---
summary: アプリケーションのパフォーマンス向上のためにデータをキャッシュする
---

# キャッシュ

AdonisJS Cache（`@adonisjs/cache`）は、[bentocache.dev](https://bentocache.dev) 上に構築されたシンプルで軽量なラッパーです。データをキャッシュし、アプリケーションのパフォーマンスを向上させます。Redis、DynamoDB、PostgreSQL、インメモリキャッシュなど、さまざまなキャッシュドライバーとやり取りするための統一されたAPIを提供します。

Bentocacheのドキュメントもぜひご覧ください。Bentocacheは、[マルチティア](https://bentocache.dev/docs/multi-tier)、[グレース期間](https://bentocache.dev/docs/grace-periods)、[タグ付け](https://bentocache.dev/docs/tagging)、[タイムアウト](https://bentocache.dev/docs/timeouts)、[スタンピードプロテクション](https://bentocache.dev/docs/stampede-protection) など、状況によって非常に便利な高度なオプション機能を提供しています。

## インストール

以下のコマンドで `@adonisjs/cache` パッケージをインストール・設定します。

```sh
node ace add @adonisjs/cache
```

:::disclosure{title="addコマンドで実行される手順を見る"}

1. 検出されたパッケージマネージャーを使って `@adonisjs/cache` パッケージをインストールします。
2. `adonisrc.ts` ファイルに以下のサービスプロバイダーを登録します。

  ```ts
  {
    providers: [
     // ...他のプロバイダー
     () => import('@adonisjs/cache/cache_provider'),
    ]
  }
  ```

3. `config/cache.ts` ファイルを作成します。
4. 選択したキャッシュドライバー用の環境変数を `.env` ファイルに定義します。

:::

## 設定

キャッシュパッケージの設定ファイルは `config/cache.ts` にあります。デフォルトのキャッシュドライバーや、利用可能なドライバー、その個別設定を行えます。

参考: [Config stub](https://github.com/adonisjs/cache/blob/1.x/stubs/config.stub)

```ts
import { defineConfig, store, drivers } from '@adonisjs/cache'

const cacheConfig = defineConfig({
  default: 'redis',

  stores: {
   /**
    * DynamoDB のみでキャッシュ
    */
   dynamodb: store().useL2Layer(drivers.dynamodb({})),

   /**
    * Lucidで設定したデータベースを利用
    */
   database: store().useL2Layer(drivers.database({ connectionName: 'default' })),

   /**
    * プライマリストアにインメモリ、セカンダリストアにRedisを利用
    * 複数サーバーで動作する場合、インメモリキャッシュはbusで同期が必要
    */
   redis: store()
    .useL1Layer(drivers.memory({ maxSize: '100mb' }))
    .useL2Layer(drivers.redis({ connectionName: 'main' }))
    .useBus(drivers.redisBus({ connectionName: 'main' })),
  },
})

export default cacheConfig
```

上記の例では、各キャッシュストアに複数のレイヤーを設定しています。これは[マルチティアキャッシュシステム](https://bentocache.dev/docs/multi-tier)と呼ばれ、まず高速なインメモリキャッシュ（第一層）を確認し、見つからなければ分散キャッシュ（第二層）を利用します。

### Redis

Redisをキャッシュシステムとして利用するには、`@adonisjs/redis` パッケージのインストールと設定が必要です。詳細は[Redis](../database/redis.md)を参照してください。

`config/cache.ts` では `connectionName` を指定します。この値は `config/redis.ts` の設定キーと一致させてください。

### データベース

`database` ドライバーは `@adonisjs/lucid` が必要です。利用する場合はインストールと設定を行ってください。

`config/cache.ts` では `connectionName` を指定します。この値は `config/database.ts` の設定キーと一致させてください。

### その他のドライバー

`memory`、`dynamodb`、`kysely`、`orchid` など他のドライバーも利用できます。

詳細は [Cache Drivers](https://bentocache.dev/docs/cache-drivers) を参照してください。

## 使い方

キャッシュの設定後、`cache` サービスをインポートして利用できます。以下はユーザー情報を5分間キャッシュする例です。

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

ご覧の通り、`user.toJSON()` でユーザーデータをシリアライズしています。キャッシュに保存するにはデータのシリアライズが必要です。Lucidモデルや `Date` インスタンスなどは直接キャッシュできません。

:::

`ttl` はキャッシュキーの有効期間（Time To Live）です。期限切れ後はキャッシュが無効となり、次のリクエストでfactoryメソッドから再取得されます。

### タグ付け

キャッシュエントリにタグを付与し、タグ単位で一括無効化できます。

```ts
await bento.getOrSet({
  key: 'foo',
  factory: getFromDb(),
  tags: ['tag-1', 'tag-2']
});

await bento.deleteByTag({ tags: ['tag-1'] });
```

### 名前空間

キーをグループ化するもう一つの方法が名前空間です。後からまとめて無効化できます。

```ts
const users = bento.namespace('users')

users.set({ key: '32', value: { name: 'foo' } })
users.set({ key: '33', value: { name: 'bar' } })

users.clear()
```

### グレース期間

`grace` オプションを使うと、キャッシュキーが期限切れでもグレース期間内なら古いデータを返しつつ裏で再取得できます。`SWR` や `TanStack Query` と同様の動作です。

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

上記では、1時間でデータは期限切れとなりますが、6時間のグレース期間内なら古いデータを返しつつ裏で再取得・更新します。

### タイムアウト

`timeout` オプションでfactoryメソッドの最大実行時間を設定できます。デフォルトは0ms（常に古いデータを返しつつ裏で再取得）。

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

この例では、factoryメソッドは最大200msまで実行されます。超過した場合は古いデータを返し、裏でfactoryメソッドが継続します。

`grace` を定義しない場合でも、`hardTimeout` でfactoryメソッドの実行を強制終了できます。

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

この場合、200ms経過でfactoryメソッドは停止し、エラーがスローされます。

:::note

`timeout` と `hardTimeout` は同時に指定可能です。`timeout` は古いデータを返すまでの最大時間、`hardTimeout` はfactoryメソッドの最大実行時間です。

:::

## キャッシュサービス

`@adonisjs/cache/services/main` からエクスポートされるキャッシュサービスは、`config/cache.ts` の設定を使って作成された [BentoCache](https://bentocache.dev/docs/named-caches) クラスのシングルトンインスタンスです。

アプリケーション内でインポートして利用できます。

```ts
import cache from '@adonisjs/cache/services/main'

/**
 * `use` メソッドを呼ばなければ、デフォルトストア（config/cache.tsで定義）を利用します。
 */
cache.put({ key: 'username', value: 'jul', ttl: '1h' })

/**
 * `use` メソッドで、config/cache.tsで定義した他のストアを利用できます。
 */
cache.use('dynamodb').put({ key: 'username', value: 'jul', ttl: '1h' })
```

利用可能なメソッド一覧: [BentoCache API](https://bentocache.dev/docs/methods)

```ts
await cache.namespace('users').set({ key: 'username', value: 'jul' })
await cache.namespace('users').get({ key: 'username' })

await cache.get({ key: 'username' })

await cache.set({key: 'username', value: 'jul' })
await cache.setForever({ key: 'username', value:'jul' })

await cache.getOrSet({
  key: 'username',
  factory: async () => fetchUserName(),
  ttl: '1h',
})

await cache.has({ key: 'username' })
await cache.missing({ key: 'username' })

await cache.pull({ key: 'username' })

await cache.delete({ key: 'username' })
await cache.deleteMany({ keys: ['products', 'users'] })
await cache.deleteByTag({ tags: ['products', 'users'] })

await cache.clear()
```

## Edgeヘルパー

`cache` サービスはEdgeテンプレート内でも利用できます。テンプレート内でキャッシュ値を直接取得できます。

```edge
<p>
  Hello {{ await cache.get('username') }}
</p>
```

## Aceコマンド

`@adonisjs/cache` パッケージは、ターミナルからキャッシュ操作できるAceコマンドも提供します。

### cache:clear

指定したストアのキャッシュをクリアします。未指定の場合はデフォルトストアをクリアします。

```sh
# デフォルトキャッシュストアをクリア
node ace cache:clear

# 特定のキャッシュストアをクリア
node ace cache:clear redis

# 特定の名前空間をクリア
node ace cache:clear store --namespace users

# 複数のタグをクリア
node ace cache:clear store --tags products --tags users
```

### cache:delete

指定したストアから特定のキャッシュキーを削除します。未指定の場合はデフォルトストアから削除します。

```sh
# 特定のキャッシュキーを削除
node ace cache:delete cache-key

# 特定のストアからキャッシュキーを削除
node ace cache:delete cache-key store
```
