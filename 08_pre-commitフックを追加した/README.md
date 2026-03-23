ブログビューワーのためにmeta.yamlが必要になった。自分でmeta.yamlを更新するのだるいなと思ったので、何かいい仕組みがないかと考えたところ、git pre-commitフックでREADME.mdの更新を監視してmeta.yamlを更新するのが良さそうに思った。

[作ったフックがこれ](../.githooks/pre-commit)だけど、なかなかいい感じに動いてくれて良き。
