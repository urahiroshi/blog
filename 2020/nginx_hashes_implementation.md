tag: nginx

# NGINXのhashまわりの実装について

### 経緯

NGINX (厳密にはingress-nginx) の設定時に以下のようなエラーが出た。

```
nginx: [warn] could not build optimal proxy_headers_hash, you should increase either proxy_headers_hash_max_size: 512 or proxy_headers_hash_bucket_size: 64; ignoring proxy_headers_hash_bucket_size                     
```

`proxy_headers_hash_max_size` もしくは `proxy_headers_hash_bucket_size` を上げろと言っている。NGINXのドキュメントには以下のように書いてある:
http://nginx.org/en/docs/hash.html 

また、こちらのコメントもある程度理解には役立つ。
https://serverfault.com/questions/419847/nginx-setting-server-names-hash-max-size-and-server-names-hash-bucket-size/786726#786726

上の情報から、まず `max_size` を上げてみて、それが駄目なら `bucket_size` を上げるべきと言ったことがわかる。
ただ、これらの値がどのように作用するのか厳密にわからないので、どれだけ `max_size` を上げていいのか、 `bucket_size` を上げることがどういう意味を持つのかが理解できなかった。
ということでソースコードを見て確認してみたいと思い至った。

### 事前知識

hash tableの実装: https://ja.wikipedia.org/wiki/%E3%83%8F%E3%83%83%E3%82%B7%E3%83%A5%E3%83%86%E3%83%BC%E3%83%96%E3%83%AB

### ソースコードを見てわかったこと

- NGINXのhash tableはbucketの配列によって構成され、bucketはhash tableを用いるkey/valueの構造体の配列を持っている(これは通常のhash tableの実装と同じ)。
- hash tableから対象のkeyの値を検索するのは以下のような処理で行われる
  1. keyのhash値を取得
  2. hash値をhash tableの長さ(size)で割って剰余を求め、その値をindexとして対象のbucketを取り出す
  3. bucket内の要素のキー名とkey値を比較し一致すればその値を返す。一致しなければ次の要素で比較、を繰り返す
- `max_size` はhash tableが持つbucketの数の最大値を示す
- `bucket_size` はbucketに割り当てられるメモリのサイズを示す
- bucketが持つ各要素(key/valueを保持する構造体)のメモリサイズは可変であり、keyが長くなればその分大きくなる。すべての要素のメモリサイズが `bucket_size` に収まらない場合、 `max_size` もしくは `bucket_size` を増やすべきというwarningが出る (ただwarningなので、これが出た場合はhash tableのsizeは `max_size` の値となり、 `bucket_size` を超えるサイズのbucketが生成される結果になるものと思われる)

### 補足

bucketから対象のkeyを探索する処理において、bucketのサイズがCPUのL1 cache lineに収まっていれば、bucketの中身はすべてCPUのL1 cacheに収まるので処理が早くなる。そのため `bucket_size` はL1 cache lineのサイズ以下になるのが良い (参考: https://techracho.bpsinc.jp/hachi8833/2020_06_24/93115)

### 対象コード

- `ngx_hash_init` メソッド (https://github.com/nginx/nginx/blob/c3fd5f7e76343c899747ec58ae703540e7e9e69a/src/core/ngx_hash.c#L252)
  ここで各種hashのパラメータを取得して、hash mapを初期化している。上記のwarningメッセージはこの時点で出ている
- `ngx_hash_find` メソッド (https://github.com/nginx/nginx/blob/c3fd5f7e76343c899747ec58ae703540e7e9e69a/src/core/ngx_hash.c#L13)
  ここでhash mapの探索を行っている


### ngx_hash_init

#### 宣言

```c
ngx_hash_init(ngx_hash_init_t *hinit, ngx_hash_key_t *names, ngx_uint_t nelts)
```

**hinit**: 設定するmax_sizeとかbucket_sizeの値を持っている構造体

(関連する型定義)
```c
typedef struct {
    ngx_hash_t       *hash;
    ngx_hash_key_pt   key;

    ngx_uint_t        max_size;
    ngx_uint_t        bucket_size;

    char             *name;
    ngx_pool_t       *pool;
    ngx_pool_t       *temp_pool;
} ngx_hash_init_t;
```

**names**: hash tableに格納する構造体の配列(構造体(ngx_hash_key_t)にはキーの値, ハッシュ値, valueのポインタが入ってる)

(関連する型定義)
```c
typedef struct {
    ngx_str_t         key;
    ngx_uint_t        key_hash;
    void             *value;
} ngx_hash_key_t;

typedef struct {
    size_t      len;
    u_char     *data;
} ngx_str_t;
```

**nelts**: number of elements の略と思われる。namesの配列の数

(関連する型定義)
```c
typedef uintptr_t       ngx_uint_t;

typedef unsigned long           uintptr_t;
```

#### バリデーション

https://github.com/nginx/nginx/blob/c3fd5f7e76343c899747ec58ae703540e7e9e69a/src/core/ngx_hash.c#L260-L286

入力パラメータのバリデーション。このへんで失敗するとemergエラーになる。
- max_sizeが0ではないか
- bucket sizeが大きすぎないか(65536-ngx_cacheline_size以内かどうか。ngx_cacheline_sizeはCPUに応じて32|64|128のいずれかの値になる: https://github.com/nginx/nginx/blob/f8d59e33f34185c28d1d3c6625a897e214b7ca73/src/core/ngx_cpuinfo.c#L98-L128。これを超えているとCPUのL1キャッシュにbucketが収まらず、パフォーマンスに相当悪影響が出るから?)
- bucket sizeが小さすぎないか(namesの各構造体を格納できるだけの大きさがあるか)

なお、namesの各構造体のサイズ計算について、 `NGX_HASH_ELT_SIZE(&names[n])` というマクロの利用がよく出てくる。 `NGX_HASH_ELT_SIZE` は以下のようなマクロである

```c
#define NGX_HASH_ELT_SIZE(name)                                               \
    (sizeof(void *) + ngx_align((name)->key.len + 2, sizeof(void *)))
```

ngx_alignは以下となる。

```c
#define ngx_align(d, a)     (((d) + (a - 1)) & ~(a - 1))
```

ngx_alignはnameの値をメモリサイズに変換する過程で、メモリ配列に割り当てた場合に発生するパディングなどを計算しているものと思われる。詳細は https://ja.wikipedia.org/wiki/%E3%83%87%E3%83%BC%E3%82%BF%E6%A7%8B%E9%80%A0%E3%82%A2%E3%83%A9%E3%82%A4%E3%83%A1%E3%83%B3%E3%83%88#%E3%83%91%E3%83%87%E3%82%A3%E3%83%B3%E3%82%B0%E3%81%AE%E8%A8%88%E7%AE%97 参照。
つまり `ngx_align` の結果はdの値をoffsetとしてpaddingの増分を加えたもので、dの値は `(name)->key.len + 2` となるので、最終的な `NGX_HASH_ELT_SIZE` の値はキーの長さ(`(name)->key.len`)がベースとなっている。

なお、 `name` は `ngx_hash_key_t` 型で、以下のようになっている(再掲)

```c
typedef struct {
    ngx_str_t         key;
    ngx_uint_t        key_hash;
    void             *value;
} ngx_hash_key_t;

typedef struct {
    size_t      len;
    u_char     *data;
} ngx_str_t;
```

最終的にbucketの各要素には `name.value`, `name.key.data`, `name.len` に対応する値が入るので、`NGX_HASH_ELT_SIZE` の `(sizeof(void *) + ngx_align((name)->key.len + 2, sizeof(void *)))` は、 `sizeof(void *)` が `value` のメモリ, `(name)->key.len` がkey.dataのメモリ、`2` がlenのメモリ分に対応していると思われる

#### hash tableのサイズ (size, 配列の長さ) 決め

https://github.com/nginx/nginx/blob/c3fd5f7e76343c899747ec58ae703540e7e9e69a/src/core/ngx_hash.c#L287-L342

以下の処理により、sizeの値を算出している

1. sizeの値をstartの値に初期化。bucket_sizeよりも要素数が少なければ1からスタート

```
    start = nelts / (bucket_size / (2 * sizeof(void *)));
    start = start ? start : 1;
```

2. namesの各要素(`names[n]`)をbucket(`names[n].key_hash % size` により、そのbucketを取得できるhash tableのindexが求められる。つまりsizeはhash tableのindexの長さを表す)に対応させ、`names[n]`のメモリ使用量をbucketのサイズとして増分させていく。
  - bucketのサイズが設定されたbucket_size値を超えればsizeの値を増やして再度チェック
  - namesの全要素をbucketに入れてもbucket_sizeに収まれば、そのsizeの値を使う
  - sizeをmax_sizeまで増やしても何かしらbucket_sizeを超えてしまう場合、sizeの値はmax_sizeとしてwarningログを出力

```c
    for (size = start; size <= hinit->max_size; size++) {

        ngx_memzero(test, size * sizeof(u_short));

        for (n = 0; n < nelts; n++) {
            if (names[n].key.data == NULL) {
                continue;
            }

            key = names[n].key_hash % size;
            len = test[key] + NGX_HASH_ELT_SIZE(&names[n]);

#if 0
            ngx_log_error(NGX_LOG_ALERT, hinit->pool->log, 0,
                          "%ui: %ui %uz \"%V\"",
                          size, key, len, &names[n].key);
#endif

            if (len > bucket_size) {
                goto next;
            }

            test[key] = (u_short) len;
        }

        goto found;

    next:

        continue;
    }

    size = hinit->max_size;

    ngx_log_error(NGX_LOG_WARN, hinit->pool->log, 0,
                  "could not build optimal %s, you should increase "
                  "either %s_max_size: %i or %s_bucket_size: %i; "
                  "ignoring %s_bucket_size",
                  hinit->name, hinit->name, hinit->max_size,
                  hinit->name, hinit->bucket_size, hinit->name);
```

#### hash table構築

https://github.com/nginx/nginx/blob/c3fd5f7e76343c899747ec58ae703540e7e9e69a/src/core/ngx_hash.c#L344-L449

まずバリデーションとして、上で再出されたsizeの値を用いて各bucketの長さを計算し、それが `65536 - ngx_cacheline_size` を超える場合はmax_sizeを増やすようにemergエラーを出す処理がある

```c
    for (i = 0; i < size; i++) {
        test[i] = sizeof(void *);
    }

    for (n = 0; n < nelts; n++) {
        if (names[n].key.data == NULL) {
            continue;
        }

        key = names[n].key_hash % size;
        len = test[key] + NGX_HASH_ELT_SIZE(&names[n]);

        if (len > 65536 - ngx_cacheline_size) {
            ngx_log_error(NGX_LOG_EMERG, hinit->pool->log, 0,
                          "could not build %s, you should "
                          "increase %s_max_size: %i",
                          hinit->name, hinit->name, hinit->max_size);
            ngx_free(test);
            return NGX_ERROR;
        }
```

以降の処理では、各bucketのメモリ割り当てをしてからbucket内の要素の値を設定していっている。
メモリ割り当ての処理は複雑なのと、元々の問題を解消するためにあまり重要ではないので割愛。
bucketの要素に値を割り当てているところは以下の箇所になる
(`test[key]` にはbucketの先頭から数えて現在の要素の位置を示す値が入る)

```c
        key = names[n].key_hash % size;
        elt = (ngx_hash_elt_t *) ((u_char *) buckets[key] + test[key]);

        elt->value = names[n].value;
        elt->len = (u_short) names[n].key.len;

        ngx_strlow(elt->name, names[n].key.data, names[n].key.len);

        test[key] = (u_short) (test[key] + NGX_HASH_ELT_SIZE(&names[n]));
```

### ngx_hash_find

#### 宣言

```c
void *
ngx_hash_find(ngx_hash_t *hash, ngx_uint_t key, u_char *name, size_t len)
```

**hash**: ハッシュテーブル

**key**: ハッシュテーブルのキー(キー名のハッシュ値)

**name**: 検索対象のキー名

**len**: キー名の長さ

#### 処理内容

1. 対象のbucketの最初の要素を取得

https://github.com/nginx/nginx/blob/c3fd5f7e76343c899747ec58ae703540e7e9e69a/src/core/ngx_hash.c#L22
```c
    elt = hash->buckets[key % hash->size];
```

2. 要素のキー名を比較し、一致すればその値を返す。一致しなければ次の要素へ

```c
    while (elt->value) {
        if (len != (size_t) elt->len) {
            goto next;
        }

        for (i = 0; i < len; i++) {
            if (name[i] != elt->name[i]) {
                goto next;
            }
        }

        return elt->value;

    next:

        elt = (ngx_hash_elt_t *) ngx_align_ptr(&elt->name[0] + elt->len,
                                               sizeof(void *));
        continue;
    }
```
