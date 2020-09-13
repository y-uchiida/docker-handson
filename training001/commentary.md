# commentary of training001

実際に操作しながら、思考の整理のためにメモっていたことをまとめました。

## docker-compose.yml
docker-copose.ymlは、複数のdockerコンテナをまとめて管理するための設定ファイルです。  
buildでディレクトリを指定すれば、Dockerfileから自前のイメージを利用することもできます。
とりあえず今回は、それぞれのソフトウェアの公式イメージを利用します。

## docker-compose pull
`docker-compose pull` で、ymlファイルで指定されたイメージの取得を行います。
docker-compose.ymlがあるディレクトリ(今回の場合は、このtraining001ディレクトリ)で実行してください。

~~~bash
# ↓実行するとこんな感じになる
$ docker-compose pull
Pulling wordpress  ... downloading (75.7%)
Pulling mysql      ... downloading (51.2%)
Pulling phpmyadmin ... downloading (48.6%)
~~~

すべてのイメージのdownloadがおわり、doneになったら完了です。
`docker images`で、指定したイメージがあるかを確認します。
~~~bash
$ docker images
REPOSITORY               TAG                 IMAGE ID            CREATED             SIZE
... 
wordpress                latest              420b971d0f8b        2 minutes ago       546MB
mysql                    5.7                 ef08065b0a30        2 minutes ago       448MB
phpmyadmin/phpmyadmin    latest              dfd1f4649053        12 days ago         469MB
... 
~~~

## docker-compose up -d
`docker-compose up` で、ymlファイルに指定されているコンテナの起動を行います。
オプションに -d を追加すると、コンテナはdetachedモードで動作します。
~~~bash
$ docker-compose up -d
Starting training001_mysql_1 ... done
Starting training001_wordpress_1  ... done
Starting training001_phpmyadmin_1 ... done
~~~

処理が終わったら、`docker ps`で起動したコンテナを確認してみます。
~~~bash
$ docker ps
CONTAINER ID        IMAGE                          COMMAND                  CREATED             STATUS              PORTS                  NAMES
151e5ba4b503        wordpress:latest               "docker-entrypoint.s…"   16 seconds ago      Up 15 seconds       0.0.0.0:8080->80/tcp   training001_wordpress_1
85fe2d3598ea        phpmyadmin/phpmyadmin:latest   "/docker-entrypoint.…"   16 seconds ago      Up 15 seconds       0.0.0.0:8888->80/tcp   training001_phpmyadmin_1
25939241470f        mysql:5.7                      "docker-entrypoint.s…"   17 seconds ago      Up 15 seconds       3306/tcp, 33060/tcp    training001_mysql_1
~~~

## docker-compose ps
`docker-compose ps`で、docker-compose.ymlに記載されたサービスの状態を確認できます。
~~~bash
$ docker-compose ps
          Name                        Command               State          Ports
----------------------------------------------------------------------------------------
training001_mysql_1        docker-entrypoint.sh mysqld      Up      3306/tcp, 33060/tcp
training001_phpmyadmin_1   /docker-entrypoint.sh apac ...   Up      0.0.0.0:8888->80/tcp
training001_wordpress_1    docker-entrypoint.sh apach ...   Up      0.0.0.0:8080->80/tcp
# State が Up だとコンテナが起動している状態
~~~

## docker-compose logs
`docker-compose logs [サービス名]` で、指定のサービスのコンテナのログ(標準出力)が表示されます。
-f オプションをつけると、最新のログを出力し続けることができます。  
終了は `Ctrl + C` です。  
~~~bash
$ docker-compose logs wordpress
Attaching to docker-handson-202009_wordpress_1
...
...
...
wordpress_1   | [Sun Sep 13 16:12:42.665431 2020] [mpm_prefork:notice] [pid 1] AH00163: Apache/2.4.38 (Debian) PHP/7.4.10 configured -- resuming normal operations
wordpress_1   | [Sun Sep 13 16:12:42.665519 2020] [core:notice] [pid 1] AH00094: Command line: 'apache2 -D FOREGROUND'
# ↑最後の2行は、こういう出力になっていました
~~~

## ブラウザから接続確認
※ymlファイルで接続先ポートを変更している場合は、以下のリンクからでは表示できないのでお気を付けください
1. wordpress : http://127.0.0.1:8080/
2. phpmyadmin : http://127.0.0.1:8888/

## 表示されない？？うまくいかないときは…

### まずはymlの内容に間違いがないか確認  
docker-composeのコマンドでエラーなど出ていないのに、WordPressやphpMyAdminがうまく表示されない…というときは、  
DB周りの環境変数（environmentのところ）やポートフォワーディング（portのところ）に記述間違いがあるかも。  
自分も、最初WordPress用のmysqlユーザー名を間違えていて、`Error establishing a database connection`というエラーに襲われました。

### 要注意：サービス再起動しても表示されない場合は、同期しているファイルが原因かも  
ymlのファイルを完璧に直したはずなのにうまくいかない…というときは、コンテナと同期しているファイルに間違った内容が残っているかもしれません。  
同期されたファイルは、コンテナを再起動した際に**ホスト側のファイルからコンテナ側にコピー**されます。  
一度でもサービスを起動するとホスト側にvolumesで指定したファイルやディレクトリが作成されてしまうので、  
同期している設定ファイルなどを直すのが面倒なときは、ひとまず同期しているホスト側のディレクトリやファイルを削除してしまうのがいいと思います。

### どうしてもうまくいかない…それってキャッシュでは？？
自分が遭遇したWordPressの`Error establishing a database connection`というエラー画面は、ymlファイルを修正しても表示が変わりませんでした。  
もしやと思って別のブラウザで開いてみたら、WordPressの初期設定画面がちゃんと表示されます。  
エラー画面がブラウザのキャッシュとして残ってしまったのが原因だったので、Ctrl ⁺ F5 でスーパーリロードして解決です。

## ymlファイルの変更を反映する
設定ファイルを変更したら、一度コンテナを再起動します。  
detachedモードで起動している場合は、`docker-compose restart`でコンテナを再起動し、設定の変更を反映できます。
~~~bash
$ docker-compose restart
Restarting training001_wordpress_1  ... done
Restarting training001_phpmyadmin_1 ... done
Restarting training001_mysql_1      ... done
~~~

特定のサービス（コンテナ）だけ再起動する場合は、 `docoker-comose restart [サービス名]`
~~~bash
$ docker-compose restart wordpress
Restarting training001_wordpress_1 ... done
~~~

## 終了の手順
ブラウザ経由でWordPressとphpMyAdminにアクセスできたら、docker-composeでの環境構築の作業としてはいったん完了です。  

コンテナとネットワークを終了し、コンテナ周りを削除するには `docker-compose down`
~~~bash
$ docker-compose down
Stopping training001_wordpress_1  ... done
Stopping training001_phpmyadmin_1 ... done
Stopping training001_mysql_1      ... done
Removing training001_wordpress_1  ... done
Removing training001_phpmyadmin_1 ... done
Removing training001_mysql_1      ... done
Removing network training001_default

$ docker ps -a
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
# docker-composeで管理されていたコンテナ（サービス）はすべて削除されます

$ ls -ls
total 12
8 -rw-r--r-- 1 yoguchi yoguchi 6602 Sep 14 01:55 commentary.md
4 -rw-r--r-- 1 yoguchi yoguchi 2126 Sep 14 01:46 docker-compose.yml
0 drwxr-xr-x 1     999 root    4096 Sep 14 01:56 mysql
0 drwxr-xr-x 1 root    root    4096 Sep 14 01:35 php
0 drwxr-xr-x 1 root    root    4096 Sep 14 01:35 wordpress
# 同期ファイルはそのまま残されるので、サービスを再度 upしたときは前回の同期ファイルの内容が引き継がれます
~~~

利用したイメージの削除も行う場合は、 `docker-compose down --rmi all`
~~~bash
$ docker-compose down --rmi all
Stopping training001_wordpress_1  ... done
Stopping training001_phpmyadmin_1 ... done
Stopping training001_mysql_1      ... done
Removing training001_wordpress_1  ... done
Removing training001_phpmyadmin_1 ... done
Removing training001_mysql_1      ... done
Removing network training001_default
Removing image mysql:5.7
Removing image wordpress:latest
Removing image phpmyadmin/phpmyadmin:latest

$ docker images
REPOSITORY               TAG                 IMAGE ID            CREATED             SIZE
# イメージも削除されています
~~~

## おつかれさまでした！
以上で、training001 でやってみたことは終了です。
お付き合いいただきありがとうございました。