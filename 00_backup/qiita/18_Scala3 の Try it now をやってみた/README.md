https://qiita.com/kijuky/items/4d0f0a297edeaf0b4baf

---

Scala3 出ましたね。[コチラのページ](https://dotty.epfl.ch/#getting-started)にある Try it now をやって、早速インストールしてみましょう。

# インストール

Mac の場合 brew でインストールできるようです

```
% brew install lampepfl/brew/dotty

...

==> Installing dotty from lampepfl/brew
==> Downloading https://github.com/lampepfl/dotty/releases/download/3.0.0/scala3
==> Downloading from https://github-releases.githubusercontent.com/7035651/8e8e1
######################################################################## 100.0%
xcrun: error: invalid active developer path (/Library/Developer/CommandLineTools), missing xcrun at: /Library/Developer/CommandLineTools/usr/bin/xcrun
Error: An exception occurred within a child process:
  CompilerSelectionError: lampepfl/brew/dotty cannot be built with any available compilers.
Install GNU's GCC:
  brew install gcc
```

おやおや、 command line tools が必要なようです。下記コマンドを実行して、表示されたポップアップで同意するとインストールされます。

```
% xcode-select --install
```

改めてインストール

```
% brew install lampepfl/brew/dotty

...

==> Installing dotty from lampepfl/brew
==> Downloading https://github.com/lampepfl/dotty/releases/download/3.0.0/scala3
Already downloaded: ~/Library/Caches/Homebrew/downloads/b52826048b9eb57afa9ef5eb4a4409c759f5a4a3a6cce36fb031487e85bf558c--scala3-3.0.0.tar.gz
Error: The `brew link` step did not complete successfully
The formula built, but is not symlinked into /usr/local
Could not symlink bin/scala
Target /usr/local/bin/scala
is a symlink belonging to scala. You can unlink it:
  brew unlink scala

To force the link and overwrite all conflicting files:
  brew link --overwrite dotty

To list all files that would be deleted:
  brew link --overwrite --dry-run dotty

Possible conflicting files are:
/usr/local/bin/scala -> /usr/local/Cellar/scala/2.13.5/bin/scala
/usr/local/bin/scalac -> /usr/local/Cellar/scala/2.13.5/bin/scalac
/usr/local/bin/scaladoc -> /usr/local/Cellar/scala/2.13.5/bin/scaladoc
==> Summary
🍺  /usr/local/Cellar/dotty/3.0.0: 50 files, 31.5MB, built in 3 seconds
```

すでに 2.13 がインストールされてるからだめって言われましたね。。。

```
% brew unlink scala
Unlinking /usr/local/Cellar/scala/2.13.5... 11 symlinks removed.
% brew install lampepfl/brew/dotty
Warning: lampepfl/brew/dotty 3.0.0 is already installed, it's just not linked.
To link this version, run:
  brew link dotty
% brew link dotty
Linking /usr/local/Cellar/dotty/3.0.0... 48 symlinks created.
% scalac --version
Scala compiler version 3.0.0 -- Copyright 2002-2021, LAMP/EPFL
```

よし、いけたっぽいですね。

# プロジェクトの作成

プロジェクトの生成は sbt でやるようです

```
% mkdir test
% cd test
% sbt new scala/scala3.g8
zsh: command not found: sbt
```

うーむ... sbt も brew で入れましょう

```
% brew install sbt
Updating Homebrew...
==> Auto-updated Homebrew!
Updated 1 tap (homebrew/cask).
==> Updated Casks
Updated 1 cask.

==> Downloading https://ghcr.io/v2/homebrew/core/sbt/manifests/1.5.2
######################################################################## 100.0%
==> Downloading https://ghcr.io/v2/homebrew/core/sbt/blobs/sha256:97d2ea0187ad1c
==> Downloading from https://pkg-containers.githubusercontent.com/ghcr1/blobs/sh
######################################################################## 100.0%
==> Pouring sbt--1.5.2.all.bottle.tar.gz
==> Caveats
You can use $SBT_OPTS to pass additional JVM options to sbt.
Project specific options should be placed in .sbtopts in the root of your project.
Global settings should be placed in /usr/local/etc/sbtopts
==> Summary
🍺  /usr/local/Cellar/sbt/1.5.2: 13 files, 50.7MB
```

改めてプロジェクトを作成します。

```
% sbt new scala/scala3.g8
copying runtime jar...
[info] [launcher] getting org.scala-sbt sbt 1.5.2  (this may take some time)...
[info] [launcher] getting Scala 2.12.13 (for sbt)...
[info] welcome to sbt 1.5.2 (N/A Java 15.0.2)
[info] set current project to new (in build file:/private/var/folders/nf/wnm__wgn16vgswm89npfm9jw0000gn/T/sbt_f3f3a50f/new/)
[info] downloading https://repo1.maven.org/maven2/org/scala-sbt/sbt-giter8-resolver/sbt-giter8-resolver_2.12/0.13.1/sbt-giter8-resolver_2.12-0.13.1.jar ...
[info] downloading https://repo1.maven.org/maven2/org/foundweekends/giter8/giter8-launcher_2.12/0.13.1/giter8-launcher_2.12-0.13.1.jar ...
[info] downloading https://repo1.maven.org/maven2/org/scala-sbt/template-resolver/0.1/template-resolver-0.1.jar ...
[info] downloading https://repo1.maven.org/maven2/org/foundweekends/giter8/giter8-cli-git_2.12/0.13.1/giter8-cli-git_2.12-0.13.1.jar ...
[info] downloading https://repo1.maven.org/maven2/io/get-coursier/coursier_2.12/2.0.0-RC6-20/coursier_2.12-2.0.0-RC6-20.jar ...
[info] downloading https://repo1.maven.org/maven2/org/scala-lang/scala-library/2.12.13/scala-library-2.12.13.jar ...
[info] 	[SUCCESSFUL ] org.scala-sbt.sbt-giter8-resolver#sbt-giter8-resolver_2.12;0.13.1!sbt-giter8-resolver_2.12.jar (475ms)
[info] downloading https://repo1.maven.org/maven2/com/github/scopt/scopt_2.12/3.7.1/scopt_2.12-3.7.1.jar ...
[info] 	[SUCCESSFUL ] org.foundweekends.giter8#giter8-launcher_2.12;0.13.1!giter8-launcher_2.12.jar (741ms)
[info] downloading https://repo1.maven.org/maven2/org/eclipse/jgit/org.eclipse.jgit.pgm/5.8.0.202006091008-r/org.eclipse.jgit.pgm-5.8.0.202006091008-r.jar ...
[info] 	[SUCCESSFUL ] org.scala-sbt#template-resolver;0.1!template-resolver.jar (771ms)
[info] downloading https://repo1.maven.org/maven2/com/jcraft/jsch.agentproxy.jsch/0.0.9/jsch.agentproxy.jsch-0.0.9.jar ...
[info] 	[SUCCESSFUL ] org.foundweekends.giter8#giter8-cli-git_2.12;0.13.1!giter8-cli-git_2.12.jar (886ms)
[info] downloading https://repo1.maven.org/maven2/com/jcraft/jsch.agentproxy.sshagent/0.0.9/jsch.agentproxy.sshagent-0.0.9.jar ...
[info] 	[SUCCESSFUL ] com.github.scopt#scopt_2.12;3.7.1!scopt_2.12.jar (491ms)
[info] downloading https://repo1.maven.org/maven2/com/jcraft/jsch.agentproxy.connector-factory/0.0.9/jsch.agentproxy.connector-factory-0.0.9.jar ...
[info] 	[SUCCESSFUL ] com.jcraft#jsch.agentproxy.jsch;0.0.9!jsch.agentproxy.jsch.jar (466ms)
[info] downloading https://repo1.maven.org/maven2/org/eclipse/jgit/org.eclipse.jgit.ssh.jsch/5.8.0.202006091008-r/org.eclipse.jgit.ssh.jsch-5.8.0.202006091008-r.jar ...
[info] 	[SUCCESSFUL ] com.jcraft#jsch.agentproxy.sshagent;0.0.9!jsch.agentproxy.sshagent.jar(bundle) (459ms)
[info] downloading https://repo1.maven.org/maven2/commons-io/commons-io/2.6/commons-io-2.6.jar ...
[info] 	[SUCCESSFUL ] org.eclipse.jgit#org.eclipse.jgit.pgm;5.8.0.202006091008-r!org.eclipse.jgit.pgm.jar (742ms)
[info] downloading https://repo1.maven.org/maven2/args4j/args4j/2.33/args4j-2.33.jar ...
[info] 	[SUCCESSFUL ] com.jcraft#jsch.agentproxy.connector-factory;0.0.9!jsch.agentproxy.connector-factory.jar(bundle) (583ms)
[info] downloading https://repo1.maven.org/maven2/org/apache/commons/commons-compress/1.19/commons-compress-1.19.jar ...
[info] 	[SUCCESSFUL ] org.scala-lang#scala-library;2.12.13!scala-library.jar (1605ms)
[info] downloading https://repo1.maven.org/maven2/org/eclipse/jgit/org.eclipse.jgit.archive/5.8.0.202006091008-r/org.eclipse.jgit.archive-5.8.0.202006091008-r.jar ...
[info] 	[SUCCESSFUL ] org.eclipse.jgit#org.eclipse.jgit.ssh.jsch;5.8.0.202006091008-r!org.eclipse.jgit.ssh.jsch.jar (512ms)
[info] downloading https://repo1.maven.org/maven2/org/eclipse/jgit/org.eclipse.jgit/5.8.0.202006091008-r/org.eclipse.jgit-5.8.0.202006091008-r.jar ...
[info] 	[SUCCESSFUL ] commons-io#commons-io;2.6!commons-io.jar (602ms)
[info] downloading https://repo1.maven.org/maven2/org/eclipse/jgit/org.eclipse.jgit.ui/5.8.0.202006091008-r/org.eclipse.jgit.ui-5.8.0.202006091008-r.jar ...
[info] 	[SUCCESSFUL ] args4j#args4j;2.33!args4j.jar(bundle) (469ms)
[info] downloading https://repo1.maven.org/maven2/org/eclipse/jgit/org.eclipse.jgit.gpg.bc/5.8.0.202006091008-r/org.eclipse.jgit.gpg.bc-5.8.0.202006091008-r.jar ...
[info] 	[SUCCESSFUL ] org.eclipse.jgit#org.eclipse.jgit.archive;5.8.0.202006091008-r!org.eclipse.jgit.archive.jar (453ms)
[info] downloading https://repo1.maven.org/maven2/org/eclipse/jgit/org.eclipse.jgit.http.apache/5.8.0.202006091008-r/org.eclipse.jgit.http.apache-5.8.0.202006091008-r.jar ...
[info] 	[SUCCESSFUL ] org.apache.commons#commons-compress;1.19!commons-compress.jar (697ms)
[info] downloading https://repo1.maven.org/maven2/org/eclipse/jgit/org.eclipse.jgit.ssh.apache/5.8.0.202006091008-r/org.eclipse.jgit.ssh.apache-5.8.0.202006091008-r.jar ...
[info] 	[SUCCESSFUL ] org.eclipse.jgit#org.eclipse.jgit.gpg.bc;5.8.0.202006091008-r!org.eclipse.jgit.gpg.bc.jar (449ms)
[info] downloading https://repo1.maven.org/maven2/org/apache/httpcomponents/httpclient/4.5.10/httpclient-4.5.10.jar ...
[info] 	[SUCCESSFUL ] org.eclipse.jgit#org.eclipse.jgit.ui;5.8.0.202006091008-r!org.eclipse.jgit.ui.jar (455ms)
[info] downloading https://repo1.maven.org/maven2/org/slf4j/slf4j-api/1.7.2/slf4j-api-1.7.2.jar ...
[info] 	[SUCCESSFUL ] org.eclipse.jgit#org.eclipse.jgit.http.apache;5.8.0.202006091008-r!org.eclipse.jgit.http.apache.jar (455ms)
[info] downloading https://repo1.maven.org/maven2/org/slf4j/slf4j-log4j12/1.7.2/slf4j-log4j12-1.7.2.jar ...
[info] 	[SUCCESSFUL ] org.eclipse.jgit#org.eclipse.jgit.ssh.apache;5.8.0.202006091008-r!org.eclipse.jgit.ssh.apache.jar (466ms)
[info] downloading https://repo1.maven.org/maven2/log4j/log4j/1.2.15/log4j-1.2.15.jar ...
[info] 	[SUCCESSFUL ] org.eclipse.jgit#org.eclipse.jgit;5.8.0.202006091008-r!org.eclipse.jgit.jar (1078ms)
[info] downloading https://repo1.maven.org/maven2/org/eclipse/jetty/jetty-servlet/9.4.28.v20200408/jetty-servlet-9.4.28.v20200408.jar ...
[info] 	[SUCCESSFUL ] org.slf4j#slf4j-api;1.7.2!slf4j-api.jar (465ms)
[info] downloading https://repo1.maven.org/maven2/org/eclipse/jgit/org.eclipse.jgit.lfs/5.8.0.202006091008-r/org.eclipse.jgit.lfs-5.8.0.202006091008-r.jar ...
[info] 	[SUCCESSFUL ] org.apache.httpcomponents#httpclient;4.5.10!httpclient.jar (484ms)
[info] downloading https://repo1.maven.org/maven2/org/eclipse/jgit/org.eclipse.jgit.lfs.server/5.8.0.202006091008-r/org.eclipse.jgit.lfs.server-5.8.0.202006091008-r.jar ...
[info] 	[SUCCESSFUL ] org.slf4j#slf4j-log4j12;1.7.2!slf4j-log4j12.jar (451ms)
[info] downloading https://repo1.maven.org/maven2/org/osgi/org.osgi.core/4.3.1/org.osgi.core-4.3.1.jar ...
[info] 	[SUCCESSFUL ] log4j#log4j;1.2.15!log4j.jar (477ms)
[info] downloading https://repo1.maven.org/maven2/com/googlecode/javaewah/JavaEWAH/1.1.7/JavaEWAH-1.1.7.jar ...
[info] 	[SUCCESSFUL ] org.eclipse.jetty#jetty-servlet;9.4.28.v20200408!jetty-servlet.jar (470ms)
[info] downloading https://repo1.maven.org/maven2/org/bouncycastle/bcpg-jdk15on/1.65/bcpg-jdk15on-1.65.jar ...
[info] 	[SUCCESSFUL ] org.eclipse.jgit#org.eclipse.jgit.lfs;5.8.0.202006091008-r!org.eclipse.jgit.lfs.jar (458ms)
[info] downloading https://repo1.maven.org/maven2/org/bouncycastle/bcprov-jdk15on/1.65.01/bcprov-jdk15on-1.65.01.jar ...
[info] 	[SUCCESSFUL ] org.eclipse.jgit#org.eclipse.jgit.lfs.server;5.8.0.202006091008-r!org.eclipse.jgit.lfs.server.jar (462ms)
[info] downloading https://repo1.maven.org/maven2/org/bouncycastle/bcpkix-jdk15on/1.65/bcpkix-jdk15on-1.65.jar ...
[info] 	[SUCCESSFUL ] org.osgi#org.osgi.core;4.3.1!org.osgi.core.jar (474ms)
[info] downloading https://repo1.maven.org/maven2/org/apache/httpcomponents/httpcore/4.4.12/httpcore-4.4.12.jar ...
[info] 	[SUCCESSFUL ] com.googlecode.javaewah#JavaEWAH;1.1.7!JavaEWAH.jar(bundle) (486ms)
[info] downloading https://repo1.maven.org/maven2/commons-logging/commons-logging/1.2/commons-logging-1.2.jar ...
[info] 	[SUCCESSFUL ] org.bouncycastle#bcpg-jdk15on;1.65!bcpg-jdk15on.jar (468ms)
[info] downloading https://repo1.maven.org/maven2/commons-codec/commons-codec/1.11/commons-codec-1.11.jar ...
[info] 	[SUCCESSFUL ] org.bouncycastle#bcpkix-jdk15on;1.65!bcpkix-jdk15on.jar (483ms)
[info] downloading https://repo1.maven.org/maven2/org/apache/sshd/sshd-osgi/2.4.0/sshd-osgi-2.4.0.jar ...
[info] 	[SUCCESSFUL ] org.apache.httpcomponents#httpcore;4.4.12!httpcore.jar (488ms)
[info] downloading https://repo1.maven.org/maven2/org/apache/sshd/sshd-sftp/2.4.0/sshd-sftp-2.4.0.jar ...
[info] 	[SUCCESSFUL ] commons-logging#commons-logging;1.2!commons-logging.jar (454ms)
[info] downloading https://repo1.maven.org/maven2/net/i2p/crypto/eddsa/0.3.0/eddsa-0.3.0.jar ...
[info] 	[SUCCESSFUL ] org.bouncycastle#bcprov-jdk15on;1.65.01!bcprov-jdk15on.jar (896ms)
[info] downloading https://repo1.maven.org/maven2/org/apache/sshd/sshd-core/2.4.0/sshd-core-2.4.0.jar ...
[info] 	[SUCCESSFUL ] commons-codec#commons-codec;1.11!commons-codec.jar (471ms)
[info] downloading https://repo1.maven.org/maven2/org/apache/sshd/sshd-common/2.4.0/sshd-common-2.4.0.jar ...
[info] 	[SUCCESSFUL ] org.apache.sshd#sshd-osgi;2.4.0!sshd-osgi.jar (508ms)
[info] downloading https://repo1.maven.org/maven2/com/jcraft/jsch/0.1.55/jsch-0.1.55.jar ...
[info] 	[SUCCESSFUL ] org.apache.sshd#sshd-sftp;2.4.0!sshd-sftp.jar (474ms)
[info] downloading https://repo1.maven.org/maven2/com/jcraft/jzlib/1.1.1/jzlib-1.1.1.jar ...
[info] 	[SUCCESSFUL ] net.i2p.crypto#eddsa;0.3.0!eddsa.jar(bundle) (463ms)
[info] downloading https://repo1.maven.org/maven2/javax/mail/mail/1.4/mail-1.4.jar ...
[info] 	[SUCCESSFUL ] org.apache.sshd#sshd-core;2.4.0!sshd-core.jar (484ms)
[info] downloading https://repo1.maven.org/maven2/javax/activation/activation/1.1/activation-1.1.jar ...
[info] 	[SUCCESSFUL ] org.apache.sshd#sshd-common;2.4.0!sshd-common.jar (485ms)
[info] downloading https://repo1.maven.org/maven2/org/eclipse/jetty/jetty-security/9.4.28.v20200408/jetty-security-9.4.28.v20200408.jar ...
[info] 	[SUCCESSFUL ] com.jcraft#jsch;0.1.55!jsch.jar (468ms)
[info] downloading https://repo1.maven.org/maven2/org/eclipse/jetty/jetty-server/9.4.28.v20200408/jetty-server-9.4.28.v20200408.jar ...
[info] 	[SUCCESSFUL ] com.jcraft#jzlib;1.1.1!jzlib.jar (448ms)
[info] downloading https://repo1.maven.org/maven2/javax/servlet/javax.servlet-api/3.1.0/javax.servlet-api-3.1.0.jar ...
[info] 	[SUCCESSFUL ] javax.activation#activation;1.1!activation.jar (451ms)
[info] downloading https://repo1.maven.org/maven2/org/eclipse/jetty/jetty-http/9.4.28.v20200408/jetty-http-9.4.28.v20200408.jar ...
[info] 	[SUCCESSFUL ] javax.mail#mail;1.4!mail.jar (566ms)
[info] downloading https://repo1.maven.org/maven2/org/eclipse/jetty/jetty-io/9.4.28.v20200408/jetty-io-9.4.28.v20200408.jar ...
[info] 	[SUCCESSFUL ] org.eclipse.jetty#jetty-security;9.4.28.v20200408!jetty-security.jar (466ms)
[info] downloading https://repo1.maven.org/maven2/org/eclipse/jetty/jetty-util/9.4.28.v20200408/jetty-util-9.4.28.v20200408.jar ...
[info] 	[SUCCESSFUL ] javax.servlet#javax.servlet-api;3.1.0!javax.servlet-api.jar (452ms)
[info] downloading https://repo1.maven.org/maven2/com/google/code/gson/gson/2.8.2/gson-2.8.2.jar ...
[info] 	[SUCCESSFUL ] org.eclipse.jetty#jetty-server;9.4.28.v20200408!jetty-server.jar (494ms)
[info] downloading https://repo1.maven.org/maven2/com/jcraft/jsch.agentproxy.core/0.0.9/jsch.agentproxy.core-0.0.9.jar ...
[info] 	[SUCCESSFUL ] org.eclipse.jetty#jetty-io;9.4.28.v20200408!jetty-io.jar (450ms)
[info] downloading https://repo1.maven.org/maven2/com/jcraft/jsch.agentproxy.usocket-jna/0.0.9/jsch.agentproxy.usocket-jna-0.0.9.jar ...
[info] 	[SUCCESSFUL ] org.eclipse.jetty#jetty-http;9.4.28.v20200408!jetty-http.jar (462ms)
[info] downloading https://repo1.maven.org/maven2/com/jcraft/jsch.agentproxy.usocket-nc/0.0.9/jsch.agentproxy.usocket-nc-0.0.9.jar ...
[info] 	[SUCCESSFUL ] org.eclipse.jetty#jetty-util;9.4.28.v20200408!jetty-util.jar (467ms)
[info] downloading https://repo1.maven.org/maven2/com/jcraft/jsch.agentproxy.pageant/0.0.9/jsch.agentproxy.pageant-0.0.9.jar ...
[info] 	[SUCCESSFUL ] com.jcraft#jsch.agentproxy.core;0.0.9!jsch.agentproxy.core.jar(bundle) (454ms)
[info] downloading https://repo1.maven.org/maven2/net/java/dev/jna/jna/4.1.0/jna-4.1.0.jar ...
[info] 	[SUCCESSFUL ] com.google.code.gson#gson;2.8.2!gson.jar (467ms)
[info] downloading https://repo1.maven.org/maven2/net/java/dev/jna/jna-platform/4.1.0/jna-platform-4.1.0.jar ...
[info] 	[SUCCESSFUL ] com.jcraft#jsch.agentproxy.usocket-jna;0.0.9!jsch.agentproxy.usocket-jna.jar(bundle) (442ms)
[info] downloading https://repo1.maven.org/maven2/io/get-coursier/coursier-core_2.12/2.0.0-RC6-20/coursier-core_2.12-2.0.0-RC6-20.jar ...
[info] 	[SUCCESSFUL ] com.jcraft#jsch.agentproxy.usocket-nc;0.0.9!jsch.agentproxy.usocket-nc.jar(bundle) (451ms)
[info] downloading https://repo1.maven.org/maven2/io/get-coursier/coursier-cache_2.12/2.0.0-RC6-20/coursier-cache_2.12-2.0.0-RC6-20.jar ...
[info] 	[SUCCESSFUL ] com.jcraft#jsch.agentproxy.pageant;0.0.9!jsch.agentproxy.pageant.jar(bundle) (456ms)
[info] downloading https://repo1.maven.org/maven2/com/github/alexarchambault/argonaut-shapeless_6.2_2.12/1.2.0-M12/argonaut-shapeless_6.2_2.12-1.2.0-M12.jar ...
[info] 	[SUCCESSFUL ] net.java.dev.jna#jna-platform;4.1.0!jna-platform.jar (505ms)
[info] downloading https://repo1.maven.org/maven2/io/get-coursier/coursier-util_2.12/2.0.0-RC6-20/coursier-util_2.12-2.0.0-RC6-20.jar ...
[info] 	[SUCCESSFUL ] net.java.dev.jna#jna;4.1.0!jna.jar (575ms)
[info] downloading https://repo1.maven.org/maven2/org/scala-lang/modules/scala-xml_2.12/1.3.0/scala-xml_2.12-1.3.0.jar ...
[info] 	[SUCCESSFUL ] io.get-coursier#coursier-core_2.12;2.0.0-RC6-20!coursier-core_2.12.jar (483ms)
[info] downloading https://repo1.maven.org/maven2/io/github/alexarchambault/windows-ansi/windows-ansi/0.0.3/windows-ansi-0.0.3.jar ...
[info] 	[SUCCESSFUL ] io.get-coursier#coursier-cache_2.12;2.0.0-RC6-20!coursier-cache_2.12.jar (474ms)
[info] downloading https://repo1.maven.org/maven2/org/fusesource/jansi/jansi/1.18/jansi-1.18.jar ...
[info] 	[SUCCESSFUL ] com.github.alexarchambault#argonaut-shapeless_6.2_2.12;1.2.0-M12!argonaut-shapeless_6.2_2.12.jar (460ms)
[info] downloading https://repo1.maven.org/maven2/io/argonaut/argonaut_2.12/6.2.4/argonaut_2.12-6.2.4.jar ...
[info] 	[SUCCESSFUL ] io.get-coursier#coursier-util_2.12;2.0.0-RC6-20!coursier-util_2.12.jar (474ms)
[info] downloading https://repo1.maven.org/maven2/com/chuusai/shapeless_2.12/2.3.3/shapeless_2.12-2.3.3.jar ...
[info] 	[SUCCESSFUL ] org.scala-lang.modules#scala-xml_2.12;1.3.0!scala-xml_2.12.jar(bundle) (481ms)
[info] downloading https://repo1.maven.org/maven2/org/scala-lang/scala-reflect/2.12.13/scala-reflect-2.12.13.jar ...
[info] 	[SUCCESSFUL ] io.github.alexarchambault.windows-ansi#windows-ansi;0.0.3!windows-ansi.jar (454ms)
[info] downloading https://repo1.maven.org/maven2/org/typelevel/macro-compat_2.12/1.1.1/macro-compat_2.12-1.1.1.jar ...
[info] 	[SUCCESSFUL ] org.fusesource.jansi#jansi;1.18!jansi.jar (483ms)
[info] 	[SUCCESSFUL ] io.argonaut#argonaut_2.12;6.2.4!argonaut_2.12.jar (479ms)
[info] 	[SUCCESSFUL ] com.chuusai#shapeless_2.12;2.3.3!shapeless_2.12.jar(bundle) (621ms)
[info] 	[SUCCESSFUL ] org.scala-lang#scala-reflect;2.12.13!scala-reflect.jar (628ms)
[info] 	[SUCCESSFUL ] org.typelevel#macro-compat_2.12;1.1.1!macro-compat_2.12.jar (453ms)
[info] 	[SUCCESSFUL ] io.get-coursier#coursier_2.12;2.0.0-RC6-20!coursier_2.12.jar (60373ms)
[info] resolving Giter8 0.13.1...

A template to demonstrate a minimal Scala 3 application 

name [Scala 3 Project Template]: test

Template applied in ~/test/./test
```

なるほど、別でフォルダを作る必要はなかったようですね。

フォルダを見てみましょう

```
% cd test
% ls
README.md	build.sbt	project		src
```

ふむふむ。

```
% cat README.md 
## sbt project compiled with Scala 3

### Usage

This is a normal sbt project. You can compile code with `sbt compile`, run it with `sbt run`, and `sbt console` will start a Scala 3 REPL.

For more information on the sbt-dotty plugin, see the
[scala3-example-project](https://github.com/scala/scala3-example-project/blob/main/README.md).
```

## 実行

`sbt run` で実行できるようです。やってみましょう。

```
% sbt run
[info] welcome to sbt 1.5.2 (N/A Java 15.0.2)
[info] loading project definition from /Users/kizuki.yasue/aaa/test/project
[info] Updating 
https://repo1.maven.org/maven2/jline/jline/2.14.6/jline-2.14.6.pom
  100.0% [##########] 19.4 KiB (30.8 KiB / s)
https://repo1.maven.org/maven2/org/fusesource/jansi/jansi/1.12/jansi-1.12.pom
  100.0% [##########] 3.6 KiB (5.7 KiB / s)
https://repo1.maven.org/maven2/org/fusesource/jansi/jansi-project/1.12/jansi-pr…
  100.0% [##########] 11.5 KiB (83.4 KiB / s)
https://repo1.maven.org/maven2/org/fusesource/fusesource-pom/1.11/fusesource-po…
  100.0% [##########] 14.2 KiB (103.9 KiB / s)
[info] Resolved  dependencies
[info] Fetching artifacts of 
https://repo1.maven.org/maven2/org/fusesource/jansi/jansi/1.12/jansi-1.12.jar
  100.0% [##########] 148.2 KiB (409.4 KiB / s)
https://repo1.maven.org/maven2/jline/jline/2.14.6/jline-2.14.6.jar
  100.0% [##########] 262.5 KiB (365.1 KiB / s)
[info] Fetched artifacts of 
[info] loading settings for project root from build.sbt ...
[info] set current project to scala3-simple (in build file:/Users/kizuki.yasue/aaa/test/)
[info] Updating 
[info] Resolved  dependencies
[info] Updating 
https://repo1.maven.org/maven2/org/scala-lang/scala3-library_3.0.0-RC3/3.0.0-RC…
  100.0% [##########] 3.6 KiB (25.5 KiB / s)
[info] Resolved  dependencies
[info] Updating 
https://repo1.maven.org/maven2/com/novocode/junit-interface/0.11/junit-interfac…
  100.0% [##########] 1.7 KiB (12.3 KiB / s)
https://repo1.maven.org/maven2/junit/junit/4.11/junit-4.11.pom
  100.0% [##########] 2.3 KiB (18.2 KiB / s)
https://repo1.maven.org/maven2/org/hamcrest/hamcrest-core/1.3/hamcrest-core-1.3…
  100.0% [##########] 766B (6.2 KiB / s)
[info] Resolved  dependencies
[info] Updating 
https://repo1.maven.org/maven2/org/scala-lang/scala3-compiler_3.0.0-RC3/3.0.0-R…
  100.0% [##########] 4.8 KiB (34.7 KiB / s)
https://repo1.maven.org/maven2/org/scala-lang/scala3-interfaces/3.0.0-RC3/scala…
  100.0% [##########] 3.4 KiB (26.5 KiB / s)
https://repo1.maven.org/maven2/org/scala-lang/tasty-core_3.0.0-RC3/3.0.0-RC3/ta…
  100.0% [##########] 3.6 KiB (28.1 KiB / s)
https://repo1.maven.org/maven2/org/scala-sbt/compiler-interface/1.3.5/compiler-…
  100.0% [##########] 2.8 KiB (7.6 KiB / s)
https://repo1.maven.org/maven2/org/scala-lang/modules/scala-asm/9.1.0-scala-1/s…
  100.0% [##########] 1.5 KiB (3.7 KiB / s)
https://repo1.maven.org/maven2/net/java/dev/jna/jna/5.3.1/jna-5.3.1.pom
  100.0% [##########] 1.5 KiB (12.4 KiB / s)
https://repo1.maven.org/maven2/com/google/protobuf/protobuf-java/3.7.0/protobuf…
  100.0% [##########] 5.2 KiB (35.5 KiB / s)
https://repo1.maven.org/maven2/org/scala-sbt/util-interface/1.3.0/util-interfac…
  100.0% [##########] 2.7 KiB (18.4 KiB / s)
https://repo1.maven.org/maven2/com/google/protobuf/protobuf-parent/3.7.0/protob…
  100.0% [##########] 7.1 KiB (52.0 KiB / s)
[info] Resolved  dependencies
[info] Updating 
https://repo1.maven.org/maven2/org/scala-lang/scaladoc_3.0.0-RC3/3.0.0-RC3/scal…
  100.0% [##########] 6.1 KiB (44.5 KiB / s)
https://repo1.maven.org/maven2/com/vladsch/flexmark/flexmark-ext-gfm-strikethro…
  100.0% [##########] 1.3 KiB (10.4 KiB / s)
https://repo1.maven.org/maven2/com/vladsch/flexmark/flexmark-ext-gfm-tables/0.4…
  100.0% [##########] 1.3 KiB (10.3 KiB / s)
https://repo1.maven.org/maven2/com/vladsch/flexmark/flexmark-ext-yaml-front-mat…
  100.0% [##########] 1.3 KiB (10.1 KiB / s)
https://repo1.maven.org/maven2/org/jsoup/jsoup/1.13.1/jsoup-1.13.1.pom
  100.0% [##########] 8.9 KiB (70.4 KiB / s)
https://repo1.maven.org/maven2/com/vladsch/flexmark/flexmark-ext-emoji/0.42.12/…
  100.0% [##########] 1.5 KiB (12.1 KiB / s)
https://repo1.maven.org/maven2/com/vladsch/flexmark/flexmark-ext-gfm-tasklist/0…
  100.0% [##########] 1.3 KiB (10.8 KiB / s)
https://repo1.maven.org/maven2/com/vladsch/flexmark/flexmark-html-parser/0.42.1…
  100.0% [##########] 1.4 KiB (11.4 KiB / s)
https://repo1.maven.org/maven2/nl/big-o/liqp/0.6.7/liqp-0.6.7.pom
  100.0% [##########] 5.0 KiB (34.8 KiB / s)
https://repo1.maven.org/maven2/com/fasterxml/jackson/dataformat/jackson-datafor…
  100.0% [##########] 2.1 KiB (5.7 KiB / s)
https://repo1.maven.org/maven2/com/vladsch/flexmark/flexmark-ext-autolink/0.42.…
  100.0% [##########] 1.8 KiB (4.9 KiB / s)
https://repo1.maven.org/maven2/com/vladsch/flexmark/flexmark-ext-anchorlink/0.4…
  100.0% [##########] 1.6 KiB (11.0 KiB / s)
https://repo1.maven.org/maven2/com/vladsch/flexmark/flexmark-ext-wikilink/0.42.…
  100.0% [##########] 1.5 KiB (10.1 KiB / s)
https://repo1.maven.org/maven2/com/vladsch/flexmark/flexmark/0.42.12/flexmark-0…
  100.0% [##########] 2.5 KiB (17.7 KiB / s)
https://repo1.maven.org/maven2/org/scala-lang/scala3-tasty-inspector_3.0.0-RC3/…
  100.0% [##########] 3.6 KiB (24.9 KiB / s)
https://repo1.maven.org/maven2/com/fasterxml/jackson/dataformat/jackson-datafor…
  100.0% [##########] 2.9 KiB (22.8 KiB / s)
https://repo1.maven.org/maven2/com/vladsch/flexmark/flexmark-java/0.42.12/flexm…
  100.0% [##########] 19.5 KiB (133.6 KiB / s)
https://repo1.maven.org/maven2/com/fasterxml/jackson/jackson-base/2.9.8/jackson…
  100.0% [##########] 5.4 KiB (38.9 KiB / s)
https://repo1.maven.org/maven2/com/fasterxml/jackson/jackson-bom/2.9.8/jackson-…
  100.0% [##########] 12.1 KiB (91.7 KiB / s)
https://repo1.maven.org/maven2/com/fasterxml/jackson/jackson-parent/2.9.1.2/jac…
  100.0% [##########] 7.7 KiB (55.5 KiB / s)
https://repo1.maven.org/maven2/com/fasterxml/oss-parent/34/oss-parent-34.pom
  100.0% [##########] 22.2 KiB (149.8 KiB / s)
https://repo1.maven.org/maven2/com/fasterxml/jackson/core/jackson-annotations/2…
  100.0% [##########] 1.2 KiB (9.8 KiB / s)
https://repo1.maven.org/maven2/com/fasterxml/jackson/core/jackson-databind/2.2.…
  100.0% [##########] 5.3 KiB (42.0 KiB / s)
https://repo1.maven.org/maven2/com/vladsch/flexmark/flexmark-jira-converter/0.4…
  100.0% [##########] 2.1 KiB (16.5 KiB / s)
https://repo1.maven.org/maven2/com/vladsch/flexmark/flexmark-util/0.42.12/flexm…
  100.0% [##########] 792B (6.2 KiB / s)
https://repo1.maven.org/maven2/org/yaml/snakeyaml/1.23/snakeyaml-1.23.pom
  100.0% [##########] 36.9 KiB (247.8 KiB / s)
https://repo1.maven.org/maven2/com/vladsch/flexmark/flexmark-formatter/0.42.12/…
  100.0% [##########] 1.1 KiB (9.0 KiB / s)
https://repo1.maven.org/maven2/org/antlr/antlr/3.5.1/antlr-3.5.1.pom
  100.0% [##########] 2.7 KiB (21.2 KiB / s)
https://repo1.maven.org/maven2/com/fasterxml/jackson/core/jackson-core/2.9.8/ja…
  100.0% [##########] 4.0 KiB (26.3 KiB / s)
https://repo1.maven.org/maven2/org/nibor/autolink/autolink/0.6.0/autolink-0.6.0…
  100.0% [##########] 9.0 KiB (24.5 KiB / s)
https://repo1.maven.org/maven2/org/antlr/antlr-master/3.5.1/antlr-master-3.5.1.…
  100.0% [##########] 12.0 KiB (90.9 KiB / s)
https://repo1.maven.org/maven2/com/fasterxml/oss-parent/11/oss-parent-11.pom
  100.0% [##########] 22.6 KiB (146.1 KiB / s)
https://repo1.maven.org/maven2/com/fasterxml/oss-parent/10/oss-parent-10.pom
  100.0% [##########] 22.7 KiB (143.0 KiB / s)
https://repo1.maven.org/maven2/com/vladsch/flexmark/flexmark-ext-ins/0.42.12/fl…
  100.0% [##########] 1.2 KiB (9.7 KiB / s)
https://repo1.maven.org/maven2/com/vladsch/flexmark/flexmark-ext-superscript/0.…
  100.0% [##########] 1.3 KiB (9.9 KiB / s)
https://repo1.maven.org/maven2/com/vladsch/flexmark/flexmark-ext-tables/0.42.12…
  100.0% [##########] 1.5 KiB (11.7 KiB / s)
https://repo1.maven.org/maven2/org/antlr/ST4/4.0.7/ST4-4.0.7.pom
  100.0% [##########] 12.2 KiB (95.3 KiB / s)
https://repo1.maven.org/maven2/org/antlr/antlr-runtime/3.5.1/antlr-runtime-3.5.…
  100.0% [##########] 2.2 KiB (17.1 KiB / s)
[info] Resolved  dependencies
[info] Fetching artifacts of 
https://repo1.maven.org/maven2/org/scala-lang/scala3-sbt-bridge/3.0.0-RC3/scala…
  100.0% [##########] 21.3 KiB (150.8 KiB / s)
[info] Fetched artifacts of 
[info] Fetching artifacts of 
https://repo1.maven.org/maven2/org/scala-lang/scala3-interfaces/3.0.0-RC3/scala…
  100.0% [##########] 3.4 KiB (25.2 KiB / s)
https://repo1.maven.org/maven2/com/vladsch/flexmark/flexmark-ext-emoji/0.42.12/…
  100.0% [##########] 71.7 KiB (453.5 KiB / s)
https://repo1.maven.org/maven2/com/vladsch/flexmark/flexmark-html-parser/0.42.1…
  100.0% [##########] 43.4 KiB (241.0 KiB / s)
https://repo1.maven.org/maven2/com/vladsch/flexmark/flexmark-ext-ins/0.42.12/fl…
  100.0% [##########] 12.7 KiB (86.3 KiB / s)
https://repo1.maven.org/maven2/com/vladsch/flexmark/flexmark-ext-superscript/0.…
  100.0% [##########] 13.0 KiB (47.0 KiB / s)
https://repo1.maven.org/maven2/org/jsoup/jsoup/1.13.1/jsoup-1.13.1.jar
  100.0% [##########] 384.6 KiB (775.4 KiB / s)
https://repo1.maven.org/maven2/com/vladsch/flexmark/flexmark-ext-gfm-tables/0.4…
  100.0% [##########] 32.4 KiB (250.9 KiB / s)
https://repo1.maven.org/maven2/org/nibor/autolink/autolink/0.6.0/autolink-0.6.0…
  100.0% [##########] 15.7 KiB (117.3 KiB / s)
https://repo1.maven.org/maven2/com/vladsch/flexmark/flexmark-util/0.42.12/flexm…
  100.0% [##########] 376.0 KiB (942.5 KiB / s)
https://repo1.maven.org/maven2/com/fasterxml/jackson/core/jackson-core/2.9.8/ja…
  100.0% [##########] 318.0 KiB (442.9 KiB / s)
https://repo1.maven.org/maven2/net/java/dev/jna/jna/5.3.1/jna-5.3.1.jar
  100.0% [##########] 1.4 MiB (2.0 MiB / s)
https://repo1.maven.org/maven2/org/antlr/antlr/3.5.1/antlr-3.5.1.jar
  100.0% [##########] 1.1 MiB (1.7 MiB / s)
https://repo1.maven.org/maven2/com/vladsch/flexmark/flexmark/0.42.12/flexmark-0…
  100.0% [##########] 379.3 KiB (1.4 MiB / s)
https://repo1.maven.org/maven2/com/vladsch/flexmark/flexmark-ext-tables/0.42.12…
  100.0% [##########] 74.5 KiB (543.8 KiB / s)
https://repo1.maven.org/maven2/com/fasterxml/jackson/core/jackson-databind/2.2.…
  100.0% [##########] 845.5 KiB (5.2 MiB / s)
https://repo1.maven.org/maven2/org/scala-lang/tasty-core_3.0.0-RC3/3.0.0-RC3/ta…
  100.0% [##########] 71.8 KiB (484.9 KiB / s)
https://repo1.maven.org/maven2/com/vladsch/flexmark/flexmark-ext-anchorlink/0.4…
  100.0% [##########] 16.3 KiB (117.4 KiB / s)
https://repo1.maven.org/maven2/org/scala-lang/scala3-tasty-inspector_3.0.0-RC3/…
  100.0% [##########] 16.6 KiB (130.6 KiB / s)
https://repo1.maven.org/maven2/com/novocode/junit-interface/0.11/junit-interfac…
  100.0% [##########] 29.6 KiB (233.3 KiB / s)
https://repo1.maven.org/maven2/com/vladsch/flexmark/flexmark-formatter/0.42.12/…
  100.0% [##########] 95.2 KiB (704.8 KiB / s)
https://repo1.maven.org/maven2/org/scala-lang/scaladoc_3.0.0-RC3/3.0.0-RC3/scal…
  100.0% [##########] 1.5 MiB (4.0 MiB / s)
https://repo1.maven.org/maven2/com/fasterxml/jackson/core/jackson-annotations/2…
  100.0% [##########] 32.7 KiB (235.2 KiB / s)
https://repo1.maven.org/maven2/com/vladsch/flexmark/flexmark-ext-yaml-front-mat…
  100.0% [##########] 17.9 KiB (135.3 KiB / s)
https://repo1.maven.org/maven2/org/scala-lang/scala3-library_3.0.0-RC3/3.0.0-RC…
  100.0% [##########] 1.1 MiB (3.9 MiB / s)
https://repo1.maven.org/maven2/com/fasterxml/jackson/dataformat/jackson-datafor…
  100.0% [##########] 40.4 KiB (310.9 KiB / s)
https://repo1.maven.org/maven2/com/vladsch/flexmark/flexmark-ext-autolink/0.42.…
  100.0% [##########] 9.1 KiB (71.9 KiB / s)
https://repo1.maven.org/maven2/org/hamcrest/hamcrest-core/1.3/hamcrest-core-1.3…
  100.0% [##########] 44.0 KiB (346.2 KiB / s)
https://repo1.maven.org/maven2/org/antlr/antlr-runtime/3.5.1/antlr-runtime-3.5.…
  100.0% [##########] 163.8 KiB (1.2 MiB / s)
https://repo1.maven.org/maven2/com/vladsch/flexmark/flexmark-ext-gfm-strikethro…
  100.0% [##########] 27.9 KiB (188.3 KiB / s)
https://repo1.maven.org/maven2/org/yaml/snakeyaml/1.23/snakeyaml-1.23.jar
  100.0% [##########] 294.2 KiB (1.9 MiB / s)
https://repo1.maven.org/maven2/nl/big-o/liqp/0.6.7/liqp-0.6.7.jar
  100.0% [##########] 196.6 KiB (1.5 MiB / s)
https://repo1.maven.org/maven2/org/scala-sbt/compiler-interface/1.3.5/compiler-…
  100.0% [##########] 90.3 KiB (684.1 KiB / s)
https://repo1.maven.org/maven2/org/antlr/ST4/4.0.7/ST4-4.0.7.jar
  100.0% [##########] 230.6 KiB (1.5 MiB / s)
https://repo1.maven.org/maven2/org/scala-lang/modules/scala-asm/9.1.0-scala-1/s…
  100.0% [##########] 304.8 KiB (2.0 MiB / s)
https://repo1.maven.org/maven2/com/vladsch/flexmark/flexmark-jira-converter/0.4…
  100.0% [##########] 38.9 KiB (285.8 KiB / s)
https://repo1.maven.org/maven2/junit/junit/4.11/junit-4.11.jar
  100.0% [##########] 239.3 KiB (1.7 MiB / s)
https://repo1.maven.org/maven2/org/scala-sbt/util-interface/1.3.0/util-interfac…
  100.0% [##########] 2.5 KiB (19.6 KiB / s)
https://repo1.maven.org/maven2/com/vladsch/flexmark/flexmark-ext-gfm-tasklist/0…
  100.0% [##########] 27.2 KiB (201.8 KiB / s)
https://repo1.maven.org/maven2/com/vladsch/flexmark/flexmark-ext-wikilink/0.42.…
  100.0% [##########] 24.9 KiB (184.8 KiB / s)
https://repo1.maven.org/maven2/com/google/protobuf/protobuf-java/3.7.0/protobuf…
  100.0% [##########] 1.4 MiB (3.8 MiB / s)
https://repo1.maven.org/maven2/org/scala-lang/scala-library/2.13.5/scala-librar…
  100.0% [##########] 5.6 MiB (14.7 MiB / s)
https://repo1.maven.org/maven2/org/scala-lang/scala3-compiler_3.0.0-RC3/3.0.0-R…
  100.0% [##########] 14.7 MiB (17.2 MiB / s)
[info] Fetched artifacts of 
[info] compiling 1 Scala source to ~/test/test/target/scala-3.0.0-RC3/classes ...
[info] running hello 
Hello world!
I was compiled by Scala 3. :)
[success] Total time: 8 s, completed 2021/05/16 17:22:47
```

出力が多いですね... `--warn` でログレベルを warn 以上にできるようなので、それで再度実行してみましょう。

```
% sbt --warn run
Hello world!
I was compiled by Scala 3. :)
```

なるほど。いわゆる `Hello world` がわかりやすくなりました。

ソースコードも見てみましょう。

```
% cat src/main/scala/Main.scala 
@main def hello: Unit = 
  println("Hello world!")
  println(msg)

def msg = "I was compiled by Scala 3. :)"
```

おお、噂には聞いていましたがインデント構文ですね。あと `@main` という見慣れないアノテーションもありますね。後で調べてみますか。

## テスト

テストもあるようです。

```
% cat src/test/scala/Test1.scala 
import org.junit.Test
import org.junit.Assert.*

class Test1:
  @Test def t1(): Unit = 
    assertEquals("I was compiled by Scala 3. :)", msg)
```

junit を使っているようですね。scalatest じゃないんだ、というのが最初の印象。

```
% sbt test       
[info] welcome to sbt 1.5.2 (N/A Java 15.0.2)
[info] loading project definition from /Users/kizuki.yasue/aaa/test/project
[info] loading settings for project root from build.sbt ...
[info] set current project to scala3-simple (in build file:/Users/kizuki.yasue/aaa/test/)
[info] Passed: Total 1, Failed 0, Errors 0, Passed 1
[success] Total time: 1 s, completed 2021/05/16 17:31:09
```

なるほど。`@Test` の使い方を見ると、やはり `@main` はアノテーションのようですね。

# まとめ

一通り Scala3 の導入ページにある機能を触ってみました。

- brew にてインストールができること
    - command line tools をインストールする必要がある
    - 以前に scala 2系を入れている場合は unlink する必要がある
- sbt にて新規プロジェクトを作成できる
    - sbt は brew でインストールできる
    - サンプルソースはインデント構文ベース
    - サンプルテストはjunitベース

以上です。
