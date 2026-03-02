https://qiita.com/kijuky/items/00a9cea6c9dc9d7444ae

---

[こちら](https://qiita.com/__kijuky__/items/8182d5c5ec43c3ed87e8) のつづき。

https://qiita.com/__kijuky__/items/8182d5c5ec43c3ed87e8

生成された swagger.json は、GitLab にレンダリングさせるためにコミットに含めます。ここで問題なのは routes ファイルと swagger.json ファイルの差異をどうするか... 皆さんはこのような自動生成ファイルをどのように管理していますか？

routes ファイルの変更があれば気づくかもしれませんが、自動生成されたモデルの変更があった場合などはどうでしょうか。swagger.json ファイルの生成は忘れそうですね。そんな時は [Danger](https://danger.systems/ruby/) さんにお任せしましょう。

# Danger による解決方法

GitLab CI で sbt swagger を実行し、 git diff を使って差分を検出します。

```.gitlab-ci.yml
swagger:
  only:
    - merge_requests
  stage: build
  script:
    - mkdir logs
    - sbt swagger
    - git diff -U0 > logs/swagger-git-diff-U0.log
  artifacts:
    paths:
      - logs/
    when: always

danger:
  only:
    - merge_requests
  stage: test
  image: ruby
  script:
    - bundle install --jobs=4 --path vendor/bundle
    - bundle exec danger --fail-on-errors=true
```

```ruby:Dangerfile
base_path = "#{Dir.pwd}/"

# swagger
lines = []
File.foreach("logs/swagger-git-diff-U0.log"){ |line| lines << line.chomp }
for index in 0..(lines.count - 1) do
  line = lines[index]
  result = /\+\+\+ b\/(.*)$/.match(line)
  if result then
    filename = result[1].sub(/^#{base_path}/, "")
  end
  if filename.include? "swagger.json" then
      message = "[swagger] routes changed. Run `sbt swagger` and commit swagger.json"
      failure(message)
  end
end
```

# 余談

個人的にはこの git diff を使った差分検出、最初見た時は感動したんですけど、まぁ当たり前体操だったりするんですかね？

こちらのインスパイア

https://github.com/johnno1962/GitDiff/blob/master/FormatImpl/clang_format.sh
