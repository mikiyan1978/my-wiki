# My Wiki

GitHub Pages + GitHub API だけで動く、自分専用のミニWikiです。記事はすべて `articles/*.md` として保存され、ブラウザからそのまま閲覧・編集・追加・削除ができます。

## サイトURL

https://mikiyan1978.github.io/my-wiki/

## 編集するには

1. 上のURLを開く
2. 右上の「🔑 設定」をクリックし、GitHubのPersonal Access Token(PAT)を入力して保存
   - トークン発行: GitHub → Settings → Developer settings → Personal access tokens → Fine-grained tokens (または Tokens classic)
   - 権限は `repo`(classicの場合)、または Fine-grained で対象リポジトリに `Contents: Read and write` を付与
   - トークンはこの端末のブラウザ(localStorage)にのみ保存され、どこにも送信されません
3. 「＋ 新規記事」またはページ上の「編集」から記事を書いて保存すると、このリポジトリに直接コミットされます

## 注意

このリポジトリはPublicです。ソースコードも記事内容も誰でも閲覧できます。個人情報や公開したくない内容は書かないでください。
