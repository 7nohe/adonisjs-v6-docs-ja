---
summary: AdonisJSアプリケーションでのSQLライブラリとORMの利用可能なオプション。
---

# SQLとORM

SQLデータベースは、アプリケーションのデータを永続的なストレージに保存するためによく使われます。AdonisJSアプリケーション内でSQLクエリを実行するために、任意のライブラリやORMを使用できます。

:::note
AdonisJSコアチームは[Lucid ORM](./lucid.md)を開発しましたが、強制的に使用することはありません。AdonisJSアプリケーション内で他のSQLライブラリやORMを使用することもできます。
:::

## 人気のあるオプション

以下は、AdonisJSアプリケーション内で他の人気のあるSQLライブラリやORMを使用することができるリストです（他のNode.jsアプリケーションと同様です）。

- [**Lucid**](./lucid.md)は、[Knex](https://knexjs.org)をベースにしたSQLクエリビルダーおよび**Active Record ORM**で、AdonisJSコアチームによって作成およびメンテナンスされています。
- [**Prisma**](https://prisma.io/orm)は、Node.jsエコシステムで人気のある別のORMです。大規模なコミュニティがあります。直感的なデータモデル、自動マイグレーション、型安全性、自動補完を提供します。
- [**Kysely**](https://kysely.dev/docs/getting-started)は、Node.js向けのエンドツーエンドのタイプセーフなクエリビルダーです。モデルなしでスリムなクエリビルダーが必要な場合には、Kyselyが適しています。[AdonisJSアプリケーション内でKyselyを統合する方法について説明した記事](https://adonisjs.com/blog/kysely-with-adonisjs)もあります。
- [**Drizzle ORM**](https://orm.drizzle.team/)は、多くのAdonisJS開発者によって使用されています。私たちはこのORMを使用した経験はありませんが、あなたのユースケースに適しているかどうかを確認するためにチェックしてみる価値があります。
- [**Mikro ORM**](https://mikro-orm.io/docs/guide/first-entity)は、Node.jsエコシステムで評価が低いORMです。MikroORMはLucidに比べてやや冗長ですが、積極的にメンテナンスされており、Knexをベースにしています。
- [**TypeORM**](https://typeorm.io)は、TypeScriptエコシステムで人気のあるORMです。

## 他のSQLライブラリとORMの使用

もう1つのSQLライブラリやORMを使用する場合、いくつかのパッケージの設定を手動で変更する必要があります。

### 認証

[AdonisJSの認証モジュール](../authentication/introduction.md)は、認証されたユーザーを取得するためにLucidを使用する機能を備えています。他のSQLライブラリやORMを使用する場合は、ユーザーを取得するために`SessionUserProviderContract`または`AccessTokensProviderContract`インターフェイスを実装する必要があります。

これは、`Kysely`を使用する場合に`SessionUserProviderContract`インターフェイスを実装する方法の例です。

```ts
import { symbols } from '@adonisjs/auth'
import type { SessionGuardUser, SessionUserProviderContract } from '@adonisjs/auth/types/session'
import type { Users } from '../../types/db.js' // Specific to Kysely

export class SessionKyselyUserProvider implements SessionUserProviderContract<Users> {
  /**
   * Used by the event emitter to add type information to the events emitted by the session guard.
   */   
  declare [symbols.PROVIDER_REAL_USER]: Users

  /**
   * Bridge between the session guard and your provider.
   */
  async createUserForGuard(user: Users): Promise<SessionGuardUser<Users>> {
    return {
      getId() {
        return user.id
      },
      getOriginal() {
        return user
      },
    }
  }

  /**
   * Find a user using the user id using your custom SQL library or ORM.
   */
  async findById(identifier: number): Promise<SessionGuardUser<Users> | null> {
    const user = await db
      .selectFrom('users')
      .selectAll()
      .where('id', '=', identifier)
      .executeTakeFirst()

    if (!user) {
      return null
    }

    return this.createUserForGuard(user)
  }
}
```

Once you have implemented the `UserProvider` interface, you can use it inside your configuration.

```ts
const authConfig = defineConfig({
  default: 'web',

  guards: {
    web: sessionGuard({
      useRememberMeTokens: false,
      provider: sessionUserProvider({
        model: () => import('#models/user'),
      }),
      
      provider: configProvider.create(async () => {
        const { SessionKyselyUserProvider } = await import(
          '../app/auth/session_user_provider.js' // Path to the file
        )

        return new SessionKyselyUserProvider()
      }),
    }),
  },
})
```
