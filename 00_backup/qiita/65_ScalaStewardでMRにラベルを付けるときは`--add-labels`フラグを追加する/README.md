https://qiita.com/drafts/343703799190594e27a3/edit

---

この結論に達するまで、ScalaStewardのソースコード調べたりしたので、また調べなくて済むように。

# MRにラベルを付けたい

GitLabではマージリクエスト(MR)にラベルをつけることで、MRを種類分けすることができます。ScalaStewardによって作られたMRにラベルをつけることで、依存ライブラリアップデートのMRのみを一覧化することができます。便利ですね。

# ラベルを付けるときは`--add-labels`フラグを追加する

ScalaStewardは`--add-labels`というフラグがあると、MRに対してラベルを付けることができます。

```diff_yaml:.gitlab-ci.yaml
  script:
    - mkdir --parents "$CI_PROJECT_DIR/.sbt" "$CI_PROJECT_DIR/.ivy2"
    - ln -sfT "$CI_PROJECT_DIR/.sbt"  "$HOME/.sbt"
    - ln -sfT "$CI_PROJECT_DIR/.ivy2" "$HOME/.ivy2"
    - chmod +x "$CI_PROJECT_DIR/askpass.sh"
    - >
      /opt/docker/bin/scala-steward \
        --workspace "$CI_PROJECT_DIR/workspace" \
        --process-timeout "30min" \
        --do-not-fork \
+       --add-labels \
        --repos-file "$CI_PROJECT_DIR/repos.md" \
        --repo-config "$CI_PROJECT_DIR/default.scala-steward.conf" \
        --git-author-email "${EMAIL}" \
        --forge-type "gitlab" \
        --forge-api-host "${CI_API_V4_URL}" \
        --forge-login "${LOGIN}" \
        --git-ask-pass "$CI_PROJECT_DIR/askpass.sh"
```

実際のラベルは`.scala-steward.conf`に設定します。

```javascript:.scala-steward.conf
pullRequests.customLabels=["LibraryUpdate"]
```

これでScalaStewardが作るMRに`LibraryUpdate`というラベルが付きます。やったね！

## 不要な標準ラベルを付けない

`--add-labels`を有効にすると、ScalaStewardは、`library-update`や`early-semver-major`など、ScalaSteward独自のラベルをつけようとします。このようなラベルが不要な場合は`pullRequests.includeMatchedLabels`で無視設定を追加します。

```diff_javascript:.scala-steward.conf
pullRequests.includeMatchedLabels="0^"
pullRequests.customLabels=["LibraryUpdate"]
```

`0^`というのは、どんな文字列にもマッチしない正規表現です。詳細はこちら。

https://qiita.com/HMMNRST/items/4b10dfb621a469034257
