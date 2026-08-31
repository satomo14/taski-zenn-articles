# taski-zenn-articles

[Taski](https://taski-app.com/) の開発記録をZennに投稿するための専用リポジトリです。

`articles/` 配下のMarkdownを push すると、Zennの GitHub 連携により自動で公開されます（`published: true` の場合）。

## ローカルプレビュー

```
npm install
npm run preview
```

## 運用ルール

- 投稿済みの記事は、このリポジトリからは削除しない（Zenn側は削除するとpush連携で記事も削除されるため、`published: false` に変更する運用とする）
- 下書きは `published: false` で管理する
