# zenn-content

[Zenn](https://zenn.dev/) の記事を管理する**公開**リポジトリ。GitHub 連携で Zenn に自動デプロイする。

> ⚠️ **公開リポジトリ**。鍵・トークン・実IP/MAC 等の秘密情報は絶対に置かない。
> 下書きの原本は非公開の作業用 vault（`~/Workspaces/agy_work/articles/`）側にあり、
> ここには公開してよい状態のものだけを移す。

## 使い方

```bash
npm install          # 初回のみ
npm run preview      # http://localhost:8000 でローカルプレビュー
npx zenn new:article # 新規記事のひな形を作る
```

- 記事: `articles/<slug>.md`（`published: false` の間は下書き。`true` で公開）
- 画像: `images/` に置き、本文からは `/images/xxx.png` で参照する

## 記事一覧

- `wyse-adguardhome-home-dns` — 余ったWyse 3040を家庭内DNS(AdGuardHome)にした話（下書き）
