---
summary: AdonisJSにおけるTypeScriptのビルドプロセスについて学びましょう
---

# TypeScriptのビルドプロセス

TypeScriptで書かれたアプリケーションは、本番環境で実行する前にJavaScriptにコンパイルする必要があります。

TypeScriptのソースファイルのコンパイルは、さまざまなビルドツールを使用して行うことができます。しかし、AdonisJSでは、最もシンプルなアプローチを採用し、以下の信頼性のあるツールを使用しています。


:::note

以下に挙げるすべてのツールは、公式のスターターキットとして開発依存関係として事前にインストールされています。

:::


- **[TSC](https://www.typescriptlang.org/docs/handbook/compiler-options.html)** は、TypeScriptの公式コンパイラです。TSCを使用して型チェックを行い、本番ビルドを作成します。

- **[TS Node Maintained](https://github.com/thetutlage/ts-node-maintained)** は、TypeScriptのJust-in-Timeコンパイラです。これにより、TypeScriptファイルをコンパイルせずに実行することができ、開発には非常に便利なツールです。

- **[SWC](https://swc.rs/)** は、Rustで書かれたTypeScriptコンパイラです。開発時にTS Nodeと組み合わせて使用し、JITプロセスを非常に高速化します。

| ツール      | 用途                  | 型チェック |
|-----------|---------------------------|---------------|
| `TSC`     | 本番ビルドの作成 | はい           |
| `TS Node` | 開発               | いいえ            |
| `SWC`     | 開発               | いいえ            |

## コンパイルせずにTypeScriptファイルを実行する

`ts-node-maintained/register/esm`フックを使用して、TypeScriptファイルをコンパイルせずに実行できます。たとえば、次のコマンドを実行してHTTPサーバーを起動できます。

```sh
node --import=ts-node-maintained/register/esm bin/server.js
```

- `--import`: インポートフラグを使用すると、モジュールの解決とロードのためのカスタマイズフックをエクスポートするモジュールを指定できます。詳細については、[Node.jsのカスタマイズフックのドキュメント](https://nodejs.org/docs/latest-v22.x/api/module.html#customization-hooks)を参照してください。

- `ts-node-maintained/register/esm`：TypeScriptソースをJavaScriptにコンパイルするためのJust-in-Timeコンパイルを実行するライフサイクルフックを登録する`ts-node-maintained/register/esm`スクリプトへのパスです。

- `bin/server.js`：AdonisJSのHTTPサーバーエントリーポイントファイルへのパスです。**詳細はこちらを参照してください：[ファイル拡張子に関する注意事項](#ファイル拡張子に関する注意事項)**

同様に、他のTypeScriptファイルに対してもこのプロセスを繰り返すことができます。たとえば：

```sh
// title: テストを実行する
node --import=ts-node-maintained/register/esm bin/test.ts
```


```sh
// title: Aceコマンドを実行する
node --import=ts-node-maintained/register/esm bin/console.ts
```

```sh
// title: 他のTypeScriptファイルを実行する
node --import=ts-node-maintained/register/esm path/to/file.ts
```

### ファイル拡張子に関する注意事項

ディスク上のファイルは`.ts`の拡張子で保存されているにもかかわらず、私たちはいたるところで`.js`の拡張子を使用していることに気付いたかもしれません。

これは、ESモジュールでは、インポートやスクリプトの実行時に`.js`拡張子を使用するようにTypeScriptが強制されるためです。この選択の背後にある理論については、[TypeScriptのドキュメント](https://www.typescriptlang.org/docs/handbook/modules/theory.html#typescript-imitates-the-hosts-module-resolution-but-with-types)で学ぶことができます。

:::tip

もしTypeScript 5.7以降を使用している場合、`.ts`拡張子を使用してTypeScriptファイルをインポートすることができます。これは、[相対パスのためのパス書き換え](https://devblogs.microsoft.com/typescript/announcing-typescript-5-7-beta/#path-rewriting-for-relative-paths)機能によって可能になります。

一部のランタイムでは、TypeScriptコードを「その場で」実行し、`.ts`拡張子を要求することができるため、将来の互換性のためにすでに`.ts`拡張子を使用することを好むかもしれません。

:::

## 開発サーバーの実行
`bin/server.js`ファイルを直接実行する代わりに、以下の理由から`serve`コマンドの使用をオススメします。

- コマンドにはファイルウォッチャーが含まれており、ファイルの変更時に開発サーバーを再起動します。
- `serve`コマンドは、アプリケーションで使用しているフロントエンドアセットバンドラーを検出し、その開発サーバーを起動します。たとえば、プロジェクトのルートに`vite.config.js`ファイルがある場合、`serve`コマンドは`vite`の開発サーバーを起動します。

```sh
node ace serve --watch
```

`--assets-args`コマンドラインフラグを使用して、Viteの開発サーバーコマンドに引数を渡すことができます。

```sh
node ace serve --watch --assets-args="--debug --base=/public"
```

Viteの開発サーバーを無効にするには、`--no-assets`フラグを使用します。

```sh
node ace serve --watch --no-assets
```

### Node.jsコマンドラインにオプションを渡す

`serve`コマンドは、開発サーバー（`bin/server.ts`ファイル）を子プロセスとして起動します。子プロセスに[node引数](https://nodejs.org/api/cli.html#options)を渡す場合は、コマンド名の前にそれらを定義できます。

```sh
node ace --no-warnings --inspect serve --watch
```

## 本番ビルドの作成

AdonisJSアプリケーションの本番ビルドは、`node ace build`コマンドを使用して作成されます。`build`コマンドは、以下の操作を実行して、`./build`ディレクトリ内に[**スタンドアロンのJavaScriptアプリケーション**](#スタンドアロンビルドとは何ですか)を作成します。

- 既存の`./build`フォルダを削除します（存在する場合）。
- `ace.js`ファイルを**ゼロから**書き直して、`ts-node/esm`ローダーを削除します。
- Viteを使用してフロントエンドアセットをコンパイルします（設定されている場合）。
- [`tsc`](https://www.typescriptlang.org/docs/handbook/compiler-options.html)を使用してTypeScriptのソースコードをJavaScriptにコンパイルします。
- [`metaFiles`](../concepts/adonisrc_file.md#metafiles)配列に登録されている非TypeScriptファイルを`./build`フォルダにコピーします。
- `package.json`および`package-lock.json/yarn.lock`ファイルを`./build`フォルダにコピーします。

:::warning
`ace.js`ファイルへの変更はビルドプロセス中に失われます。ビルドの前にAceが開始される前に実行する追加のコードがある場合は、代わりに`bin/console.ts`ファイル内で行うべきです。
:::

以上です！

```sh
node ace build
```

ビルドが作成されたら、`build`フォルダに移動し、本番依存関係をインストールしてアプリケーションを実行できます。

```sh
cd build

# 本番依存関係のインストール
npm ci --omit=dev

# サーバーの実行
node bin/server.js
```

`--assets-args`コマンドラインフラグを使用して、Viteビルドコマンドに引数を渡すことができます。

```sh
node ace build --assets-args="--debug --base=/public"
```

フロントエンドアセットのコンパイルを回避するには、`--no-assets`フラグを使用します。

```sh
node ace build --no-assets
```

### スタンドアロンビルドとは何ですか？

スタンドアロンビルドとは、元のTypeScriptソースなしで実行できるアプリケーションのJavaScript出力を指します。

スタンドアロンビルドを作成することで、デプロイするコードのサイズを削減できます。ソースファイルとJavaScript出力の両方をコピーする必要がないためです。

本番ビルドを作成した後、`./build`を本番サーバーにコピーし、依存関係をインストールし、環境変数を定義してアプリケーションを実行できます。
