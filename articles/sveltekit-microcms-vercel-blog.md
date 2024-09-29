---
title: "SvelteKitとmicroCMSで自作ブログを作ってVercelで公開してみた"
emoji: "🚀"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Svelte", "SvelteKit", "microCMS", "Vercel"]
published: false
---
## はじめに

こんにちは！
SvelteKit、microCMSを使って自作ブログを作成してみたので共有したいと思います。

### 対象読者

- 自作ブログを作成したい方
- Svelte, SvelteKit, microCMS, Vercelを使ってみたい方

### 記事を読むメリット

- Svelte, SvelteKit, microCMSを使った自作ブログの作成方法がわかります
- Vercelへのデプロイ方法がわかります

### 結論

作成したブログは[こちら](https://siiiiisar-blog.vercel.app)です！爆速で作成できます。

![](/images/sveltekit-microcms-vercel-blog/blog.gif)

https://github.com/siiiiisar/my-blog

## 使用技術

### フロントエンド：Svelte(4.2.7) + SvelteKit v2

フロントエンドにはSvelteとそのフレームワークであるSvelteKitを使用します。
Svelteはコードの記述量が少ないことや、仮想DOMを使用しないのでコンパイル後のバンドルサイズが小さくなり高速に動作するなどのメリットがあります。
https://svelte.dev/blog/virtual-dom-is-pure-overhead
SvelteKitはReactで言うところのNext.jsの立ち位置で、ルーティング等の機能を提供します。
https://kit.svelte.dev/

:::message
スタイリングにtailwindcssを使用していますが、今回の記事では範囲外とします。
:::

### ヘッドレスCMS：microCMS

ヘッドレスCMSとはコンテンツ管理の機能のみをもつ、ビューのないCMSのことです。
https://blog.microcms.io/what-is-headlesscms/
microCMSは国産のCMSなのでコンソールやドキュメントが日本語に対応していると言う良さがあります。

### ホスティングサービス：Vercel

ホスティングサービスはVercelを使用します。
SvelteKitにも対応しており、Githubと連携して簡単にアプリケーションをデプロイできます。
https://vercel.com/solutions/svelte

## プロジェクトの作成
下記のコマンドを実行してプロジェクトを作成して、開発環境を立ち上げます。

```shell
npm create svelte@latest my-app
cd my-app
npm install
npm run dev
```

:::message
ここで、ツール設定の質問が聞かれます。
最初の質問でSkelton projectを選択して最小構成で作成します。
その他は任意で選択しましょう。（この記事ではTypeScriptを選択しています。）
:::

下記の画面が立ち上がれば成功です。

![](/images/sveltekit-microcms-vercel-blog/svelte-welcome-page.png)

## microCMSのセットアップ

### サービスの作成

microCMSにサインアップしたら、任意のサービス名、サービスIDでサービスを作成します。

![](/images/sveltekit-microcms-vercel-blog/cms-service-info-form.png)

### APIの作成

次にAPIを作成します。「APIを作成」画面でブログのテンプレートを選択します。
ブログAPIが自動で作成されます。

![](/images/sveltekit-microcms-vercel-blog/cms-api-create.png)

下記のように、「コンテンツ一覧」画面にはサンプルの記事が作成されています。

![](/images/sveltekit-microcms-vercel-blog/cms-api-list.png)

画面右上のAPIプレビューを押してみましょう！
下記のように、JavaScriptと[microcms-js-sdk](https://github.com/microcmsio/microcms-js-sdk)でのAPI使用例と
APIを叩いた結果が表示されています。

![](/images/sveltekit-microcms-vercel-blog/cms-api-preview.png)

## 早速APIを叩いてみる

### APIキーの設定

microCMSでAPIを叩くにはリクエストにAPIキーを含める必要があります。
外部に漏らしてしまうのはセキュリティの観点からよくないので、環境変数として定義します。
プロジェクト直下に`.env`を作成して、下記のように記載しておきましょう。

```
MICROCMS_SERVICE_DOMAIN=microCMS管理画面URL(XXXX.microcms.io)のXXXX部分
MICROCMS_API_KEY=「権限管理」>「APIキー管理」から確認
```

設定した環境変数は`$env-static-private`というモジュールを使用して取得できます。
https://kit.svelte.dev/docs/modules#$env-static-private

### ディレクトリ構成

SvelteKitはファイルベースのルーティングを提供しています。
今回、下記のようなディレクトリ構成としています。

```
├── lib
│   └── microcms.ts
└── routes
    ├── +page.svelte // ルート
    └── blogs
        ├── +page.server.ts // ブログ一覧取得
        ├── +page.svelte // ブログ一覧
        └── [slug]
            ├── +page.server.ts // ブログ詳細取得
            └── +page.svelte // ブログ詳細
```

`src/routes`がルートとなっており、`blogs`でブログの一覧を表示します。
`slug`と言うパラメータを使用して、`/blog/hello-world`など動的にルーティングすることが可能です。
今回は`slug`に記事のidを渡して記事詳細を表示します。

また、ファイル名は`+`の接頭語で識別されて下記のような責務で分かれています。

- `+page.svelte`: 画面を定義します。

- `+page.server.ts`: サーバのみで実行される`load`関数を定義します。`+page.svelte`に`data`propsを介してデータを渡すことができます。

https://kit.svelte.dev/docs/routing

### ブログ一覧/詳細の取得

microCMSのAPIを簡単に扱うためのSDKが用意されています。
https://microcms.io/features/sdk

JavaScriptからの利用なので[microcms-js-sdk](https://github.com/microcmsio/microcms-js-sdk)をインストールします。

```shell
npm install microcms-js-sdk
```

SDKを使用してAPIを叩く`src/lib/microcms.ts`を作成していきます。

:::details microcms.ts
```ts
import { createClient, type MicroCMSImage, type MicroCMSQueries } from 'microcms-js-sdk';
import { MICROCMS_SERVICE_DOMAIN, MICROCMS_API_KEY } from '$env/static/private';

const client = createClient({
	serviceDomain: MICROCMS_SERVICE_DOMAIN,
	apiKey: MICROCMS_API_KEY
});

export type Blog = {
	id: string;
	createdAt: string;
	updatedAt: string;
	publishedAt: string;
	revisedAt: string;
	title: string;
	content: string;
	eyecatch?: MicroCMSImage;
};
export type BlogResponse = {
	totalCount: number;
	offset: number;
	limit: number;
	contents: Blog[];
};

// ブログ一覧取得
export const getList = async (queries?: MicroCMSQueries) => {
	return await client.get<BlogResponse>({
		endpoint: 'blogs',
		queries
	});
};

// ブログ詳細取得
export const getDetail = async (contentId: string, queries?: MicroCMSQueries) => {
	return await client.getListDetail<Blog>({
		endpoint: 'blogs',
		contentId: contentId,
		queries
	});
};
```
:::

サーバ側でブログ一覧と詳細を取得する処理を実装していきます。
`src/routes/blogs`と`src/routes/blogs/[slug]`にそれぞれ`+page.server.ts`を作成します。

:::details src/routes/blogs/+page.server.ts
```ts
import type { PageServerLoad } from './$types';
import { getList } from '$lib/microcms';

export const load: PageServerLoad = async () => {
	return await getList();
};

export const prerender = true;
```
:::

:::details src/routes/blogs/[slug]/+page.server.ts
```ts
import type { PageServerLoad } from './$types';
import { getDetail } from '$lib/microcms';

export const load: PageServerLoad = async ({ params }) => {
	return await getDetail(params.slug);
};

export const prerender = true;
```
:::

`export const prerender = true;`を指定することで、ビルド時に表示用にHTMLを生成する事前レンダリングを行います。

https://kit.svelte.dev/docs/page-options

SvelteKitはこのオプションにより、SSRとSSGを切り替えることができ便利ですね！

:::message
このままの設定だと、microCMSからコンテンツを更新してもビルドがされません。
以降で、microCMSのWebhook機能を使ってコンテンツを更新した際にVercelでビルドされるように設定します。
:::

ブログ一覧とブログ詳細画面を作成していきます。
`src/routes/blogs`と`src/routes/blogs/[slug]`にそれぞれ`+page.svelte`を作成します。

:::details ブログ一覧画面（src/routes/blogs/+page.svelte）
```svelte
<script lang="ts">
	import type { PageData } from './$types';
	import BlogCard from '../../components/BlogCard.svelte';
	import CommonLayout from '../../components/CommonLayout.svelte';

	export let data: PageData;
</script>

<CommonLayout title="Blogs">
	<div class="grid grid-cols-1 gap-10 sm:grid-cols-2 md:grid-cols-3">
		{#each data.contents as content}
			<BlogCard {content} />
		{/each}
	</div>
</CommonLayout>
```
:::

:::details ブログ詳細画面（src/routes/blogs/[slug]/+page.svelte）
```svelte
<script lang="ts">
	import CommonLayout from '../../../components/CommonLayout.svelte';
	import type { PageData } from './$types';

	export let data: PageData;
</script>

<CommonLayout title={data.title}>
	<article class="znc p-16 rounded-lg bg-white">
		{@html data.content}
	</article>
</CommonLayout>
```
:::

`+page.server.ts`の`load()`で取得したデータは`export let data: PageData;`で受け取ります。
ブログ詳細は、HTML形式で取得されるので`{@html}`を使用して表示しています。
https://svelte.jp/docs/special-tags#html

:::message alert
`{@html}`は記事の内容が外部のユーザーによって投稿されたり、
自由に入力されたりする場合は、XSS（クロスサイトスクリプティング）攻撃のリスクがあります。
今回は自分が管理するソースのため問題ありませんが、使用する際は注意しましょう。
:::

表示用のブログカードやレイアウト等のコンポーネントの実装は省略しますが、自由に[Githubリポジトリ](https://github.com/siiiiisar/my-blog)を参考にしてください。

画面の実装ができたら`npm run dev`で動作確認しましょう！

## Vercelへのデプロイ

VercelにサインアップしてGithubと連携しましょう。
下記のリポジトリ一覧から「Import」します。

![](/images/sveltekit-microcms-vercel-blog/vercel-repository-list.png)

「Environment Variables」から`.env`に設定した`MICROCMS_SERVICE_DOMAIN`と`MICROCMS_API_KEY`を設定します。
「Deploy」を押してデプロイ完了です。
サイトにアクセスできるか確認しましょう！

## Webhookの設定

microCMSのコンテンツ更新でVercelでビルドされるようにWebhookの設定をします。

Vercelの管理画面から「Settings」>「Git」>「Deploy Hooks」にてWebhookを作成します。
Webhook URLをコピーしておきます。

次にmicroCMSの管理画面右上の「API設定」から「Webhook」にて「追加」を選択。
サービス選択画面で「Vercel」を選択して下記画面でコピーしたWebhook URLを設定します。

![](/images/sveltekit-microcms-vercel-blog/cms-vercel-webhook.png)

通知タイミングの設定は下記で良いかなと思います。

- コンテンツの公開時・更新時
- コンテンツの公開終了時
- 公開中コンテンツの削除時

これで設定は完了です。
microCMSで記事の作成や削除をしてみて、Vercelでビルドが走るかを確認しましょう！

## まとめ

SvelteKitとmicroCMSを使って自作ブログを作成してVercelにデプロイする方法でした！
APIまで作り込むと大変ですがmictoCMSを使って楽をしつつ、フロントでエンジニアのこだわりを入れられる構成かなと感じました。
