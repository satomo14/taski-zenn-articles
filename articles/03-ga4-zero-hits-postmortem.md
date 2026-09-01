---
title: "GA4を自前で入れたら送信0件だった。犯人は dataLayer.push(arguments)"
emoji: "📉"
type: "tech"
topics: ["googleanalytics", "ga4", "typescript", "react", "個人開発"]
published: true
---

個人開発の SPA に GA4 を入れました。測定IDを設定して、デプロイして、リアルタイムレポートを開いて──**0件**。

エラーは出ていません。`gtag/js` は 200 で読み込まれています。`dataLayer` は伸びています。それなのに `google-analytics.com` へのリクエストが1本も出ていませんでした。

原因にたどり着くまでと、その後に続けて踏んだ3つの罠を、順番に書きます。Vite + React の SPA での話ですが、1つ目と3つ目はフレームワークに関係なく効きます。

## 罠1: シムをアロー関数で書くと、何も送信されない

公式のスニペットをそのまま `<head>` に貼らず、TypeScript のモジュールから初期化する形にしていました。そのとき `gtag` のシムをこう書きました。

```ts
// ✗ これだと1件も送信されない
window.dataLayer = window.dataLayer || [];
window.gtag = (...args: unknown[]) => window.dataLayer!.push(args);

window.gtag('js', new Date());
window.gtag('config', MEASUREMENT_ID, { send_page_view: false });
```

一見なんの問題もありません。`dataLayer` に `['js', Date]` と `['config', 'G-XXXXXXXXXX', {...}]` が積まれます。デバッガで見ると、ちゃんと中身も入っています。

**しかし gtag.js は、`dataLayer` に積まれた `Arguments` オブジェクトだけをコマンドとして解釈します。素の配列は黙って無視されます。**

無視されるのが `event` だけならまだ気づけたのですが、**`config` も無視されます**。つまり測定IDが一度も設定されないまま、イベントらしきものだけが配列に溜まっていく。送信先が決まっていないので、当然リクエストは発生しません。エラーも警告も出ません。

正しくはこうです。

```ts
// ✓ Arguments オブジェクトをそのまま push する
window.dataLayer = window.dataLayer || [];
window.gtag = function gtag() {
  // eslint-disable-next-line prefer-rest-params
  window.dataLayer!.push(arguments);
};
```

Google の公式スニペットが、いまどき珍しい `function` 宣言で

```js
function gtag(){dataLayer.push(arguments)}
```

と書かれているのは、まさにこのためでした。**アロー関数には `arguments` が無いので、この形はアロー関数では書けません。** モダンに書き直そうとした瞬間に壊れる、という構造になっています。

`prefer-rest-params` の ESLint エラーを消そうとしてレストパラメータに直すのも、同じ理由で NG です。ここだけは無効化コメントを付けて残すのが正解でした。

### 切り分けに使った手順

同じ症状（読み込まれているのに送信されない）が出たとき用に、手順を残しておきます。ページのコンソールで実行します。

```js
// 正しいシムを手で作って送ってみる
function g(){ window.dataLayer.push(arguments); }
g('js', new Date());
g('config', 'G-XXXXXXXXXX');
g('event', 'debug_test');

// 実際に送信が発生したか確認する
performance.getEntriesByType('resource').filter(e => /google-analytics/.test(e.name));
```

これで送信が発生するなら、犯人はシムの書き方です。

そしてもう一点。**ブラウザ拡張のネットワーク監視では GA4 の送信を捕捉できないことがあります。** GA4 は `fetch` / `sendBeacon` で送るためです。「拡張機能に何も出ないから送っていない」と判断すると、逆方向に時間を溶かします。`performance.getEntriesByType('resource')` で見るのが確実でした。

## 罠2: プリレンダリングが、実行時に挿した script タグを焼き付ける

SPA なので、SEO 対策に Puppeteer でプリレンダリングしています。Chrome でアプリを描画して `page.content()` で DOM を保存し、静的な `index.html` として吐く、というよくあるやつです。

GA4 を有効化した最初のデプロイで、**`analytics.ts` が実行時に `document.head` へ挿入した `<script src="...gtag/js?id=G-XXXXXXXXXX">` が、本番の静的 `index.html` に焼き付きました。**

当たり前といえば当たり前で、`page.content()` は「実行後の DOM」をそのまま文字列化するので、実行時に差し込んだものは全部入ります。

対処は、ヘッドレスブラウザなら初期化しないことです。

```ts
function isHeadless(): boolean {
  if (typeof navigator === 'undefined') return true;
  if (navigator.webdriver) return true;
  return /HeadlessChrome|Puppeteer|jsdom/i.test(navigator.userAgent);
}

export function initAnalytics(): void {
  if (initialized || !MEASUREMENT_ID) return;
  if (isHeadless()) return;
  // ...
}
```

副次的な効果として、クローラや自動テストの流入も数えなくなります。これは実は結構大きくて、別プロジェクトでは**1機種のボットが Android の `first_open` の57%を占めて**、分析が丸ごと歪んでいたことがありました。最初から入れておいて損はないと思います。

## 罠3: `page_location` からクエリを落とすと、UTM が全部消える

SPA なので `send_page_view: false` にして、ルーティングに合わせて自前で `page_view` を送っています。最初はこう書いていました。

```ts
// ✗ UTM が全部消える
window.gtag('event', 'page_view', {
  page_path: path,
  page_location: window.location.origin + path,
  page_title: document.title,
});
```

**GA4 は `page_location` の中身から `utm_source` / `utm_medium` / `utm_campaign` を読んで流入元を判定します。** `origin + path` で組み立てると、クエリ文字列が消えるので UTM も一緒に消えます。結果、記事から来た人も広告から来た人も、全部「Direct」に丸められます。

しかもこれは**後から復旧できません**。送ってしまったデータに流入元は書き足せないので、気づいた日より前の「どこから来たか」は永久に分かりません。

直し方は、現在の URL をベースにしてパスだけ差し替えることです。

```ts
// ✓ 検索クエリとハッシュをそのまま残す
const url = new URL(window.location.href);
url.pathname = path;

window.gtag('event', 'page_view', {
  page_path: path,
  page_location: url.toString(),
  page_title: document.title,
});
```

自前で `page_view` を送る構成にした人は、ここを一度確認しておくといいと思います。私は罠1を直した直後、リアルタイムに数字が出るようになって満足してしまい、しばらく気づきませんでした。

## 罠4: 自分のアクセスを止めるには、`config` ごと止めないと足りない

開発者本人のアクセスを計測から外したい、という話です。最初は「送信関数の入口で弾けばいい」と考えて、こうしていました。

```ts
function canSend(): boolean {
  return initialized && !isInternal && !!window.gtag;
}

export function trackEvent(name: string, params?: Record<string, unknown>): void {
  if (!canSend()) return;
  window.gtag!('event', name, params);
}
```

これでイベントは止まります。**でもセッションは止まりません。**

`gtag('config', ...)` は、`send_page_view: false` を付けていても **GA4 へ1リクエスト飛びます。それだけでセッションが1件立ちます。** 実機で確認しました。

外部の利用者がまだ0人という段階だと、これは致命的です。「アクティブユーザー1人」が本物のユーザーなのか自分なのか、判別できなくなります。

なので、認証状態が確定するまで `initAnalytics()` を呼ばず、社内アカウントと分かったら**初期化そのものをしない**形に変えました。

```tsx
// App.tsx
const [analyticsReady, setAnalyticsReady] = useState(false);

useEffect(() => {
  if (loading || analyticsReady) return;              // 認証状態が確定するまで待つ
  if ((currentUser?.plan ?? null) === 'internal') return;  // 社内なら初期化しない
  initAnalytics();
  setAnalyticsReady(true);
}, [loading, currentUser?.plan, analyticsReady]);
```

ページビューも `analyticsReady` を見てから送ります。`canSend()` に任せると、初期化前に発生した初回のページビューが捨てられて取りこぼすためです。

```tsx
useEffect(() => {
  if (!analyticsReady) return;
  trackPageView(`/${activeTab}`);
}, [activeTab, analyticsReady]);
```

**GA4 は送ってしまったデータを後から遡って消せません。** 除外は「送る前に止める」しかない、というのがこの一件の教訓でした。

## おまけ: パラメータは、カスタム定義に登録するまで分解できない

これは今回の SPA ではなく、別のアプリで3回踏んだ話です。

```ts
trackEvent('section_reached', { section: 'vote_panel' });
```

こう送っておけば、あとで `section` 別に見られる──と思いますよね。**見られません。** GA4 管理画面の「カスタム定義」にカスタムディメンションとして登録するまで、Data API からも探索からも分解できません（`customEvent:section` は 400 が返ります）。

そして**登録前に届いたデータは、登録しても遡って分解できません**。2ヶ月ぶんの計測が「イベント総数は分かるが内訳は永久に不明」になりました。

対処は2つです。

- 最初からカスタム定義に登録しておく（以後のデータだけ分解できる）
- **急ぐなら、パラメータではなくイベント名を分ける**（`section_reached_vote_panel`）。標準の `eventName` なので即日見えます

「あとで分解すればいい」が効かない、というのが GA4 の一番ハマりやすい仕様だと思っています。

## まとめ

| # | 症状 | 原因 |
|---|---|---|
| 1 | 送信が1件も発生しない | シムがアロー関数で、`Arguments` ではなく配列を push していた |
| 2 | 静的HTMLに gtag の script タグが焼き付く | プリレンダリングが実行時に挿した DOM ごと保存していた |
| 3 | 流入元が全部 Direct になる | `page_location` を組み立て直してクエリを落としていた |
| 4 | 自分のアクセスでセッションが立つ | `config` は `send_page_view:false` でも1リクエスト飛ぶ |

1と4はどちらも「エラーが出ないまま、数字だけが静かに間違う」タイプです。GA4 を入れたら、まず `performance.getEntriesByType('resource')` で**本当に送信が発生しているか**を自分の目で確認するところから始めるのがおすすめです。

---

この記事のコードは、Redmine からの移行を入口にしたタスク管理ツール [Taski](https://taski-app.com/?utm_source=zenn&utm_medium=article&utm_campaign=ga4_zero_hits) を作りながら踏んだものです。同じところで止まっている方の役に立てばうれしいです。
