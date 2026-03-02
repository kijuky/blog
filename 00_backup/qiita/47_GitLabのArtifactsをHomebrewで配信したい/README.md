https://qiita.com/kijuky/items/042c7ce4f89f8636fabf

---

# 前提

- あなたはGitLabにて、アプリのパッケージをCIのArtifactsで保存しています。
- そのアプリをbrewで配信したいとします。

そんな悩みを解決します。

# 解決

大体はこれに書いてあるんですが、いくつか注意点があります。

https://wheniwork.engineering/creating-a-private-homebrew-tap-with-gitlab-8800c453d893

新しいリポジトリを用意して、ルートに次のような Formula を用意します。

```foo.rb
class Foo < Formula
  desc "<<なんかイカす説明>>"
  homepage "<<よさげなホームページのURL>>"
  version "<<バージョン>>"
  license "<<ライセンス>>"

  url "https://<<GitLabホスト>>/api/v4/projects/<<プロジェクトID>>/jobs/artifacts/<<タグ名>>/raw/<<tar.gzまでのパス>>?job=<<ジョブ名>>&private_token=#{ENV['HOMEBREW_GITLAB_TOKEN']}"
  sha256 "<<tar.gzファイルのssh256>>"

  def install
    bin.install "<<tar.gzに含まれる実行ファイル名>>"
  end
end
```

ここで2点注意点があります。

- url に指定するのは api で artifacts を取ってくるようにします。
- 環境変数は `HOMEBREW_GITLAB_TOKEN` です。
    - 完全に愚痴ですが、元記事は `GITLAB_HOMEBREW_TOKEN` になってて、この調査に3時間溶かしました。
    - https://docs.brew.sh/Formula-Cookbook#using-environment-variables

## 使用方法

read_api にチェックを入れたパーソナルアクセストークンを作成します。

```shell
export HOMEBREW_GITLAB_TOKEN="<<作ったアクセストークン>>"
brew tap "<<ユーザー名>>/<<リポジトリ名>>" "ssh://git@<<GitLabホスト>>/<<ユーザー名>>/<<リポジトリ名>>.git"
brew install "<<アプリ名>>"
```

# まとめ

社内のリポジトリをbrew tapできると、社内ツールもbrewでインストール&バージョン管理できるようになるので、たくさん社内ツール作りたくなりますね。
