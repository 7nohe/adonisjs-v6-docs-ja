---
summary: アプリケーションのパフォーマンスを向上させるためにデータをキャッシュする
---

# キャッシュ

AdonisJS Cache (`@adonisjs/cache`) は、データをキャッシュしてアプリケーションのパフォーマンスを向上させるために [bentocache.dev](https://bentocache.dev) の上に構築されたシンプルで軽量なラッパーです。Redis、DynamoDB、PostgreSQL、インメモリキャッシュなど、さまざまなキャッシュドライバと対話するための簡単で統一されたAPIを提供します。

Bentocacheのドキュメントを読むことを強くオススメします。Bentocacheは、[マルチティア](https://bentocache.dev/docs/multi-tier)、[グレース期間](https://bentocache.dev/docs/grace-periods)、[タイムアウト](https://bentocache.dev/docs/timeouts)、[スタンピードプロテクション](https://bentocache.dev/docs/stampede-protection) など、特定の状況で非常に役立つ高度なオプションの概念を提供します。

## インストール

以下のコマンドを実行して `@adonisjs/cache` パッケージをインストールおよび設定します：

```sh
node ace add @adonisjs/cache
```

:::disclosure{title="addコマンドによって実行されるステップ"}

1. 検出されたパッケージマネージャーを使用して `@adonisjs/cache` パッケージをインストールします。
2. `adonisrc.ts` ファイルに次のサービスプロバイダーを登録します：

  ```ts
  {
    providers: [
     // ...他のプロバイダー
     () => import('@adonisjs/cache/cache_provider'),
    ]
  }
  ```

3. `config/cache.ts` ファイルを作成します。
4. `.env` ファイル内に選択されたキャッシュドライバの環境変数を定義します。

:::

## 設定

キャッシュパッケージの設定ファイルは `config/cache.ts` にあります。デフォルトのキャッシュドライバ、ドライバのリスト、およびそれらの特定の設定を構成できます。

参照： [Config stub](https://github.com/adonisjs/cache/blob/1.x/stubs/config.stub)

```ts
import { defineConfig, store, drivers } from '@adonisjs/cache'

const cacheConfig = defineConfig({
  default: 'redis',
  
  stores: {
   /**
    * データをDynamoDBにのみキャッシュ
    */
   dynamodb: store().useL2Layer(drivers.dynamodb({})),

   /**
    * Lucidで設定されたデータベースを使用してデータをキャッシュ
    */
   database: store().useL2Layer(drivers.database({ connectionName: 'default' })),

   /**
    * データをプライマリストアとしてインメモリに、セカンダリストアとしてRedisにキャッシュ。
    * アプリケーションが複数のサーバーで実行されている場合、インメモリキャッシュはバスを使用して同期する必要があります。
    */
   redis: store()
    .useL1Layer(drivers.memory({ maxSize: '100mb' }))
    .useL2Layer(drivers.redis({ connectionName: 'main' }))
    .useBus(drivers.redisBus({ connectionName: 'main' })),
  },
})

export default cacheConfig
```

上記のコード例では、各キャッシュストアに複数のレイヤーを設定しています。これは [マルチティアキャッシングシステム](https://bentocache.dev/docs/multi-tier) と呼ばれます。これにより、最初に高速なインメモリキャッシュ（最初のレイヤー）をチェックし、データが見つからない場合は分散キャッシュ（2番目のレイヤー）を使用します。

### Redis

Redisをキャッシュシステムとして使用するには、`@adonisjs/redis` パッケージをインストールして設定する必要があります。ドキュメントはこちらを参照してください： [Redis](../database/redis.md)。

`config/cache.ts` では、`connectionName` を指定する必要があります。このプロパティは `config/redis.ts` ファイル内のRedis設定キーと一致する必要があります。

### データベース

`database` ドライバには `@adonisjs/lucid` のピア依存関係があります。したがって、このパッケージをインストールして設定する必要があります。

`config/cache.ts` では、`connectionName` を指定する必要があります。このプロパティは `config/database.ts` ファイル内のデータベース設定キーに対応する必要があります。

### その他のドライバ

`memory`、`dynamodb`、`kysely`、`orchid` などの他のドライバを使用できます。

詳細は [Cache Drivers](https://bentocache.dev/docs/cache-drivers) を参照してください。

## 使用方法

キャッシュが設定されたら、`cache` サービスをインポートして操作できます。次の例では、ユーザーの詳細を5分間キャッシュします：

```ts
import cache from '@adonisjs/cache/services/main'
import router from '@adonisjs/core/services/router'

router.get('/user/:id', async ({ params }) => {
  return cache.getOrSet({
   key: `user:${params.id}`,
   factory: async () => {
    const user = await User.find(params.id)
    return user.toJSON()
   },
   ttl: '5m',
  })
})
```

:::warning

ご覧のとおり、`user.toJSON()` を使用してユーザーデータをシリアル化しています。これは、データをキャッシュに保存するためにシリアル化する必要があるためです。Lucidモデルや `Date` のインスタンスなどのクラスは、Redisやデータベースのようなキャッシュに直接保存することはできません。

:::

`ttl` はキャッシュキーの有効期限を定義します。TTLが切れると、キャッシュキーは古いと見なされ、次のリクエストはファクトリメソッドからデータを再取得します。

### グレース期間

キャッシュキーが期限切れでもグレース期間内であれば古いデータを返すようにするには、`grace` オプションを使用します。この変更により、Bentocacheは `SWR` や `TanStack Query` と同じように動作します。

```ts
import cache from '@adonisjs/cache/services/main'

cache.getOrSet({
  key: 'slow-api',
  factory: async () => {
   await sleep(5000)
   return 'slow-api-response'
  },
  ttl: '1h',
  grace: '6h',
})
```

上記の例では、データは1時間後に古いと見なされます。ただし、6時間のグレース期間内の次のリクエストは古いデータを返しながら、ファクトリメソッドからデータを再取得してキャッシュを更新します。

### タイムアウト

ファクトリメソッドが実行される時間を `timeout` オプションで設定できます。デフォルトでは、Bentocacheは0msのソフトタイムアウトを設定しており、常に古いデータを返しながらバックグラウンドでデータを再取得します。

```ts
import cache from '@adonisjs/cache/services/main'

cache.getOrSet({
  key: 'slow-api',
  factory: async () => {
   await sleep(5000)
   return 'slow-api-response'
  },
  ttl: '1h',
  grace: '6h',
  timeout: '200ms',
})
```

上記の例では、ファクトリメソッドは最大200ms実行されます。ファクトリメソッドが200msを超えると、古いデータがユーザーに返されますが、ファクトリメソッドはバックグラウンドで実行され続けます。

`grace` 期間を定義していない場合でも、ハードタイムアウトを使用して一定時間後にファクトリメソッドの実行を停止できます。

```ts
import cache from '@adonisjs/cache/services/main'

cache.getOrSet({
  key: 'slow-api',
  factory: async () => {
   await sleep(5000)
   return 'slow-api-response'
  },
  ttl: '1h',
  hardTimeout: '200ms',
})
```

この例では、ファクトリメソッドは200ms後に停止され、エラーがスローされます。

:::note

`timeout` と `hardTimeout` を一緒に定義できます。`timeout` はファクトリメソッドが古いデータを返す前に実行される最大時間であり、`hardTimeout` はファクトリメソッドが停止されるまでの最大時間です。

:::

## キャッシュサービス

`@adonisjs/cache/services/main` からエクスポートされるキャッシュサービスは、`config/cache.ts` で定義された設定を使用して作成された [BentoCache](https://bentocache.dev/docs/named-caches) クラスのシングルトンインスタンスです。

キャッシュサービスをアプリケーションにインポートして、キャッシュと対話するために使用できます：

```ts
import cache from '@adonisjs/cache/services/main'

/**
 * `use` メソッドを呼び出さない場合、キャッシュサービスで呼び出すメソッドは
 * `config/cache.ts` で定義されたデフォルトストアを使用します。
 */
cache.put({ key: 'username', value: 'jul', ttl: '1h' })

/**
 * `use` メソッドを使用して、`config/cache.ts` で定義された別のストアに切り替えることができます。
 */
cache.use('dynamodb').put({ key: 'username', value: 'jul', ttl: '1h' })
```

利用可能なすべてのメソッドはこちらで確認できます： [BentoCache API](https://bentocache.dev/docs/methods)。

```ts
await cache.namespace('users').set({ key: 'username', value: 'jul' })
await cache.namespace('users').get('username')

await cache.get('username')

await cache.set('username', 'jul')
await cache.setForever('username', 'jul')

await cache.getOrSet({
  key: 'username',
  factory: async () => fetchUserName(),
  ttl: '1h',
})

await cache.has('username')
await cache.missing('username')

await cache.pull('username')

await cache.delete('username')
await cache.deleteMany(['products', 'users'])

await cache.clear()
```

## Edgeヘルパー

`cache` サービスはビュー内でエッジヘルパーとして利用できます。テンプレート内でキャッシュされた値を直接取得するために使用できます。

```edge
<p>
  こんにちは {{ await cache.get('username') }}
</p>
```

## Aceコマンド

`@adonisjs/cache` パッケージは、ターミナルからキャッシュと対話するための一連のAceコマンドも提供します。

### cache:clear

指定されたストアのキャッシュをクリアします。指定されていない場合は、デフォルトのストアをクリアします。

```sh
node ace cache:clear
node ace cache:clear redis
node ace cache:clear dynamodb
```
