https://qiita.com/kijuky/items/63d2c9d7524ca5d2250d

---

業務でタイトルのような環境構築を行い、色んな所に躓いたので備忘録です。

# 前提

- [sqlalchemy](https://www.sqlalchemy.org/) で Oracle database (oracle12c) に接続して、必要な情報をダンプするようなバッチプログラムの単体テストを書きたかった。
- このプロダクトコードでは SQL を直書きしていたので、単体テスト上で SQLite 環境を構築して確認しようとしたが、単体テスト環境と本番環境でDBクライアントは揃えたほうが良いという、ごもっともな意見を見かけたので、単体テスト環境でも oracle12c 環境を構築することにした。
    - SQL 直書き良くないよね、という意見もごもっともだが、そのためのリファクタリングするにはまずは単体テストが書いてないとだめだよね、という。

# ローカルで単体テスト環境構築

そもそも単体テスト環境が存在しないので、まずはローカルで単体テストの環境構築を行った。よくあるDBを使った単体テスト環境構築と同様、docker-compose で db を起動しつつ、単体テストを実行すれば良い。

oracle12c の docker イメージ作成はこちらを参考に。

```YAML:docker-compose.yml
version: '2'

services:
  oracle-database:
    image: oracle/database:12.1.0.2-ee
    container_name: oracle-database
    ports:
      - 1521:1521
    volumes:
      - ./startup:/opt/oracle/scripts/startup
    environment:
      - ORACLE_SID=SID
      - ORACLE_PWD=passw0rd
      - ORACLE_PDB=pdb
```

このDBに接続するためには、いくつか準備が必要になる。

Oracle データベースの場合、ユーザー（スキーマ）の中にデータベースを作ることになるため、まずはユーザーを作る必要がある。ユーザー作成コードを毎度テストコードに書くのはだるいので、データベースのスタートアップ時にユーザーが作られるようにする。

```sql:startup/startup.sql
ALTER SESSION SET container = pdb;
GRANT DBA TO PDBADMIN;
GRANT UNLIMITED TABLESPACE TO PDBADMIN;

CREATE USER testuser
    IDENTIFIED BY passw0rd
    DEFAULT TABLESPACE users
    TEMPORARY TABLESPACE temp;
GRANT DBA TO testuser;
GRANT UNLIMITED TABLESPACE TO testuser;
```

Oracle データベース docker イメージは startup フォルダにある sql ファイルを sysdb ユーザーで実行してしまうため、プラガブルDB(PDB)上にユーザーを追加するためにセッションを変更する。また、初期状態では PDBADMIN に admin 相当権限が存在しないため（何故？）、権限を付与している。

ここでは testuser/passw0rd としてユーザーを作成している。単体テストでしか使わないため、このユーザーにも admin 相当権限をつけている。

```python:tests/test.py
import unittest

import cx_Oracle
import sqlalchemy

class Test(unittest.TestCase):
    def setUp(self):
        self.sut = ... # テスト対象インスタンス
        dsn = cx_Oracle.makedsn("oracle-database", 1521, service_name = "pdb")
        self.testuser = sqlalchemy.create_engine(f"oracle+cx_oracle://testuser:passw0rd@{dsn}")
        self.create_testtable()

    def tearDown(self):
        self.drop_testtable()

    def test__testmethod__describe(self):
        # SetUp
        expected = ...
        # Exercise
        self.sut.testmethod(...)
        # Verify
        actual = ...
        self.assertEqual(expected, actual)

    def create_testtable(self):
        self.testuser.execute(f"""
            CREATE TABLE testtable (
                ... define columns ...
            )
        """)

    def drop_testtable(self):
        self.testuser.execute("DROP TABLE testtable")

if __name__ == "__main__":
    unittest.main()
```

テストの書き方は色々と流儀があると思うので、一例として。ここで重要なのは setUp メソッドで oracle12c に接続している部分。DSNを作成して、その情報を使って接続URLを構築して接続エンジンを作る。sqlalchemy には様々な接続方法が書いてあるが、PDBに接続するには(SIDではなく)ServiceNameで接続する必要があり、そのためにはDSNを作成する必要がある。DSNのホスト名は `docker-compose.yml` で定義したコンテナ名と一致させる。

tests に `__init__.py` を用意すれば、ルートで `pytest` を実行することで単体テストが実行される。

```sh
docker-compose up -d
pytest
```

## 問題点

とりあえず、これで最低限の環境は整う。ただし、この環境には問題がある。

やってみるとわかるが、`docker-compose up` 後、初期データベース構築にかなり時間がかかり、最終的にデータベースがオープンする（＝接続可能になる）までに5~10分程度時間がかかる。また、初期データベース構築は（Docker で構築しているのに）まれに失敗する。単体テストとしての環境と考えると、かなり取り回しが悪い。

この問題を解決するために初期データベースをバックアップしておく。ここで「バックアップ」と表現したのには訳がある。この初期データベースはDBMSから接続後、上書き保存の形でどんどん更新されていくからだ。単体テスト実行でデータベースが汚れたときもそうだが、何もしていなくてもファイルサイズが増えていく等でどんどん「汚く」なっていく。テストがそのような外部要因で失敗する可能性も否定できないので、安定したテスト環境を構築するために、`docker-compose up` の直前にバックアップしておいた初期データベースを復元しておく必要がある。

初期データベースを構築、バックアップを用意するにも多少テクニックが要る。ここでは一例を示す。

まず、初期データベースを構築するために Oracle データベースイメージを構築する。例では docker-compose を使っているが、同様の docker コマンドでも構わない。

```YAML:docker-compose-oradata.yml
version: '2'

services:
  oracle-database:
    image: oracle/database:12.1.0.2-ee
    container_name: oracle-database
    ports:
      - 1521:1521
    volumes:
      - ./oradata:/opt/oracle/oradata
    environment:
      - ORACLE_SID=SID
      - ORACLE_PWD=passw0rd
      - ORACLE_PDB=pdb
```
```sh
docker-compose -f docker-compose-oradata.yml up
```

実行後、oradata フォルダに初期データベースが構築される。この構築に5~10分程度時間がかかる。終わったかどうかのタイミングを見計らうために、上の例ではあえてデタッチモードではない。

ここからが重要で、構築が完了した直後に Ctrl+C でデータベースをシャットダウンさせる。ここをもたもたしていると、初期データベースがぶくぶく太っていってしまう（1.5 GB くらいにはなる）。それでもマウント自体はできるが、バックアップするファイルが無駄に大きくなっても意味はないので、構築直後にシャットダウンさせる。

```
Starting oracle-database ... done
Attaching to oracle-database
oracle-database    | ORACLE PASSWORD FOR SYS, SYSTEM AND PDBADMIN: passw0rd
oracle-database    | 
oracle-database    | LSNRCTL for Linux: Version 12.1.0.2.0 - Production on 15-OCT-2020 12:03:40
oracle-database    | 
oracle-database    | Copyright (c) 1991, 2014, Oracle.  All rights reserved.
oracle-database    | 
oracle-database    | Starting /opt/oracle/product/12.1.0.2/dbhome_1/bin/tnslsnr: please wait...
oracle-database    | 
oracle-database    | TNSLSNR for Linux: Version 12.1.0.2.0 - Production
oracle-database    | System parameter file is /opt/oracle/product/12.1.0.2/dbhome_1/network/admin/listener.ora
oracle-database    | Log messages written to /opt/oracle/diag/tnslsnr/bf429c874900/listener/alert/log.xml
oracle-database    | Listening on: (DESCRIPTION=(ADDRESS=(PROTOCOL=ipc)(KEY=EXTPROC1)))
oracle-database    | Listening on: (DESCRIPTION=(ADDRESS=(PROTOCOL=tcp)(HOST=0.0.0.0)(PORT=1521)))
oracle-database    | 
oracle-database    | Connecting to (DESCRIPTION=(ADDRESS=(PROTOCOL=IPC)(KEY=EXTPROC1)))
oracle-database    | STATUS of the LISTENER
oracle-database    | ------------------------
oracle-database    | Alias                     LISTENER
oracle-database    | Version                   TNSLSNR for Linux: Version 12.1.0.2.0 - Production
oracle-database    | Start Date                15-OCT-2020 12:03:40
oracle-database    | Uptime                    0 days 0 hr. 0 min. 0 sec
oracle-database    | Trace Level               off
oracle-database    | Security                  ON: Local OS Authentication
oracle-database    | SNMP                      OFF
oracle-database    | Listener Parameter File   /opt/oracle/product/12.1.0.2/dbhome_1/network/admin/listener.ora
oracle-database    | Listener Log File         /opt/oracle/diag/tnslsnr/bf429c874900/listener/alert/log.xml
oracle-database    | Listening Endpoints Summary...
oracle-database    |   (DESCRIPTION=(ADDRESS=(PROTOCOL=ipc)(KEY=EXTPROC1)))
oracle-database    |   (DESCRIPTION=(ADDRESS=(PROTOCOL=tcp)(HOST=0.0.0.0)(PORT=1521)))
oracle-database    | The listener supports no services
oracle-database    | The command completed successfully
oracle-database    | Cleaning up failed steps
oracle-database    | 4% complete
oracle-database    | Copying database files
oracle-database    | 5% complete
oracle-database    | 6% complete
oracle-database    | 30% complete
oracle-database    | Creating and starting Oracle instance
oracle-database    | 32% complete
oracle-database    | 35% complete
oracle-database    | 36% complete
oracle-database    | 37% complete
oracle-database    | 41% complete
oracle-database    | 44% complete
oracle-database    | 45% complete
oracle-database    | 48% complete
oracle-database    | Completing Database Creation
oracle-database    | 50% complete
oracle-database    | 53% complete
oracle-database    | 55% complete
oracle-database    | 63% complete
oracle-database    | 66% complete
oracle-database    | 74% complete
oracle-database    | Creating Pluggable Databases
oracle-database    | 79% complete
oracle-database    | 100% complete
oracle-database    | Look at the log file "/opt/oracle/cfgtoollogs/dbca/SID/SID0.log" for further details.
oracle-database    | 
oracle-database    | SQL*Plus: Release 12.1.0.2.0 Production on Thu Oct 15 12:11:34 2020
oracle-database    | 
oracle-database    | Copyright (c) 1982, 2014, Oracle.  All rights reserved.
oracle-database    | 
oracle-database    | 
oracle-database    | Connected to:
oracle-database    | Oracle Database 12c Enterprise Edition Release 12.1.0.2.0 - 64bit Production
oracle-database    | With the Partitioning, OLAP, Advanced Analytics and Real Application Testing options
oracle-database    | 
oracle-database    | SQL> 
oracle-database    | System altered.
oracle-database    | 
oracle-database    | SQL> 
oracle-database    | Pluggable database altered.
oracle-database    | 
oracle-database    | SQL> Disconnected from Oracle Database 12c Enterprise Edition Release 12.1.0.2.0 - 64bit Production
oracle-database    | With the Partitioning, OLAP, Advanced Analytics and Real Application Testing options
oracle-database    | The Oracle base remains unchanged with value /opt/oracle
oracle-database    | #########################
oracle-database    | DATABASE IS READY TO USE!
oracle-database    | #########################
oracle-database    | The following output is now a tail of the alert.log:
oracle-database    | Completed: alter pluggable database PDB open
oracle-database    | Thu Oct 15 12:11:33 2020
oracle-database    | CREATE SMALLFILE TABLESPACE "USERS" LOGGING  DATAFILE  '/opt/oracle/oradata/SID/PDB/PDB_users01.dbf' SIZE 5M REUSE AUTOEXTEND ON NEXT  1280K MAXSIZE UNLIMITED  EXTENT MANAGEMENT LOCAL  SEGMENT SPACE MANAGEMENT  AUTO
oracle-database    | Completed: CREATE SMALLFILE TABLESPACE "USERS" LOGGING  DATAFILE  '/opt/oracle/oradata/SID/PDB/PDB_users01.dbf' SIZE 5M REUSE AUTOEXTEND ON NEXT  1280K MAXSIZE UNLIMITED  EXTENT MANAGEMENT LOCAL  SEGMENT SPACE MANAGEMENT  AUTO
oracle-database    | ALTER DATABASE DEFAULT TABLESPACE "USERS"
oracle-database    | Completed: ALTER DATABASE DEFAULT TABLESPACE "USERS"
oracle-database    | Thu Oct 15 12:11:34 2020
oracle-database    | ALTER SYSTEM SET control_files='/opt/oracle/oradata/SID/control01.ctl' SCOPE=SPFILE;
oracle-database    |    ALTER PLUGGABLE DATABASE PDB SAVE STATE
oracle-database    | Completed:    ALTER PLUGGABLE DATABASE PDB SAVE STATE
^CERROR: Aborting.
```

oradata フォルダをどこかにバックアップしておき、テスト実行前に実行する `docker-compose up` の前に、この oradata フォルダの内容が復元されるようにしておく（実際の業務では、oradata の内容を zip圧縮し、S3 にバックアップした）。もともとの docker-compose.yml は oradata フォルダをマウントするようにしておく。

```YAML:docker-compose.yml
version: '2'

services:
  oracle-database:
    image: oracle/database:12.1.0.2-ee
    container_name: oracle-database
    ports:
      - 1521:1521
    volumes:
      - ./oradata:/opt/oracle/oradata
      - ./startup:/opt/oracle/scripts/startup
    environment:
      - ORACLE_SID=SID
      - ORACLE_PWD=passw0rd
      - ORACLE_PDB=pdb
```

こうすることで、データベースのオープンまでにかかる時間は1~2分程度にまで削減できた。

## GitLab CI で上記単体テストを実行

本題。

GitLab CI 上で単体テスト用のDBを使う場合、`services` を使うのが王道である。ただし、Oracle データベースのイメージはどうしても何らかのプライベートリポジトリで管理するしかなく、services でプライベートリポジトリを設定したことがなかった＆実務で他にも docker イメージをつくっていたため、ここでは Docker in Docker (dind) で環境構築した。

GitLab CI で dind 環境を構築する場合、１つ問題がある。それはボリュームがマウントされるディレクトリはホストのディレクトリになってしまうことだ。通常（GitLab CIに限らず）ジョブ実行はそれ自体が docker コンテナで行われるため、その中で今回のような docker-compose を実行してしまうと、oradata や startup フォルダはジョブ実行された docker コンテナではなく、その docker を実行しているホストのフォルダを指定してしまうことになる。

この解決方法は色々あるが、今回は oradata まで含めた docker イメージを作成することで回避した。そもそも oracle データベースイメージをプライベートリポジトリに置いているはずなので、そこにさらに oradata を含めたイメージを用意するだけなので、ローカルでの環境もさほど変わらない。

先程つくった oradata を含めた oracle データベースイメージを作る Dockerfile はこんな感じ。

```dockerfile:Dockerfile
FROM oracle/database:12.1.0.2-ee

ENV ORACLE_SID SID
ENV ORACLE_PWD passw0rd
ENV ORACLE_PDB pdb

COPY --chown=oracle:dba oradata/ /opt/oracle/oradata/
COPY startup/ /opt/oracle/scripts/startup/
```
```sh
docker build -t oracle/database:12.1.0.2-ee-with-oradata
```

注意点は 2 つある。

1つは oradata は環境変数 ORACLE_SID, ORACLE_PWD, ORACLE_PDB に依存するので、これらの環境変数も Dockerfile に追加しておくこと。これらが変更された場合は oradata も再作成する。

もう1つは oradata をコピーする際、オーナーを `oracle:dba` に変更すること。そのままコピーすると root ユーザーになるため、oracle ユーザーが初期データベースをマウントできなくなる。

あとはこのイメージを使って .gitlab-ci.yml で実行する。

```YAML:docker-compose.yml
version: '2'

services:
  oracle-database:
    image: oracle/database:12.1.0.2-ee-with-oradata
    container_name: oracle-database
    ports:
      - 1521:1521
```
```yml:.gitlab-ci.yml
pytest:
  stage: test
  image: docker:dind
  script:
    - apk update && apk add bash python3 python3-dev py3-pip docker-compose
    - python3 -m pip install pytest
    - docker-compose up -d
    - sleep 120s
    - pytest
```

（ちょっと pytest の実行環境構築自信ない、後で調べる）

初期データベースのオープンに時間がかかるので、ここでは例として `sleep 120s` としている。

とりあえず、こんな感じにすることで、GitLab CI の dind 上で Oracle database を使った単体テストが実行可能になった。

# まとめ

- GitLab-CI の dind 上で Oracle database を使った単体テスト環境を構築した。
    - 初期データベースの oradata を用意することで、データベースのスタートアップにかかる時間を削減できた。
- GitLab-CI でデータベースを使った単体テスト環境は services を使うのが王道なので、そのやり方で dind 脱却を目指したい。
    - そもそも dind は Docker 本体のデバッグのための環境なので、積極的に使うのは避けたい。
    - 実務ではリポジトリに AWS ECR を使っているので、そのへんの権限周りを今後調査したい。
- 他にも実務で Oracle database を使っていて、単体テストを書くためにCI環境構築している事例を知りたい。
    - 話せる範囲で「ウチはこんな感じだよ！」というのがあれば、コメントいただけると嬉しいです。
