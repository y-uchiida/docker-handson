# commentary of training002

dockerのコンテナ間通信について、実際に設定ファイルを書いてみました。

## コンテナ間通信の必要性
dockerコンテナは、1コンテナ1プロセス(≒1サービス)が原則です。  
そのため、複数のサービス、ミドルウェアを利用してシステムを構築する場合は、コンテナ同士が通信できなければなりません。  
コンテナ間で通信をさせるには、dockerの仮想ブリッジ上でコンテナ同士を接続する必要があります。  

※「1コンテナ1プロセス」は、厳密に1プロセスを扱う訳ではないようです。  
VMや物理サーバの切り分けと同じく、Webサーバ、DBサーバ、Proxyサーバなど役割に応じて  
コンテナを分割することのほうが実際の管理上も望ましいと考えられます。

コンテナが切り分けされていれば、特定のコンテナだけクラスタリングするなどの  
「水平方向のスケーリング」がしやすくなったり、パッケージのバージョン管理がしやすくなります。

## コンテナ間通信をする2つの方法方法
1. --link オプションでコンテナをつなぐ
2. Dockerネットワークを構築して、仮想的な接続状態を作成する

## --linkオプションでのコンテナ間通信
--link オプションは古いので非推奨とのこと。2のDockerネットワークを利用するほうがよいです。
例として、2つのalpineコンテナを--link オプションでつなぐコマンド例を挙げてみます。
~~~bash
# 1つ目のコンテナ「alpine_01」を起動
$ docker run -it -d --name alpine_01 alpine
ce1d1a84aab3af9506ba1427c7c7cd4cc89787d81de2ed00833f2ebc75d959ff

# 2つ目のコンテナ「alpine_02」を起動し、--link [コンテナ名]:[コンテナ内で利用する、接続先コンテナのホスト名] で通信できるようにする
$ docker run -it -d --name alpine_02 --link alpine_01:other_container alpine
dd57496e7899c792e1d2d711d8982db45bbd7a060c113c27c9c97c9cace8412d

$ docker ps
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
dd57496e7899        alpine              "/bin/sh"           2 minutes ago       Up 2 minutes                            alpine_02
ce1d1a84aab3        alpine              "/bin/sh"           3 minutes ago       Up 3 minutes                            alpine_01

# alpine_01のIPアドレスを確認 
$ docker exec -it alpine_01 ip a s | grep inet
    inet 127.0.0.1/8 scope host lo
    inet 172.17.0.2/16 brd 172.17.255.255 scope global eth0 # 172.17.0.2が割り当てられている

# alpine_02のコンテナから、other_container にむけてpingを打ってみる
$ docker exec -it alpine_02 ping other_container
PING other_container (172.17.0.2): 56 data bytes
64 bytes from 172.17.0.2: seq=0 ttl=64 time=0.067 ms
64 bytes from 172.17.0.2: seq=1 ttl=64 time=0.060 ms
64 bytes from 172.17.0.2: seq=2 ttl=64 time=0.049 ms
64 bytes from 172.17.0.2: seq=3 ttl=64 time=0.091 ms
^C
--- other_container ping statistics ---
4 packets transmitted, 4 packets received, 0% packet loss
round-trip min/avg/max = 0.049/0.066/0.091 ms
# 通信できることがわかる
~~~

上記のように、--link オプションで指定した内容でネットワークがつなげられていることがわかります。  
alpine_02コンテナ内では、alpine_01コンテナは「other_container」というホスト名で認識されています。  
`--link [コンテナ名]:[コンテナ内で利用する接続先のコンテナ名]` という記述の意味が分かると思います。

## Dockerネットワークでのホスト間通信の設定
ここからは参考元のページの内容に戻ります。  
nginxが稼働しているコンテナを介して、別のコンテナ上で動作しているtomcatを表示する、ということをしてみます。

### Dockerfileなどの作成
まずはページの指示に従って、必要なファイルを用意します。  
なお注意点として、2020年9月16日時点では、tomcat-9.0.6の配布が停止しております。  
それに伴って、./tomcat/Dockerfileの内容そのままだとうまくいきません。  
現時点で配布されているapache-tomcat-9.0.37をwgetし、このバージョンのファイル名に変更するとよいかと思います。  
`https://archive.apache.org/dist/tomcat/tomcat-9/v9.0.37/bin/apache-tomcat-9.0.37.tar.gz`

### イメージファイルの作成とコンテナの生成
ファイルの用意ができたら、`docker build` コマンドでイメージファイルを作成します。
~~~bash
$ cd /path/to/docker-handson-202009/training002

# nginx-tomcat イメージの作成
$ docker build -t nginx-tomcat:1 ./nginx
Sending build context to Docker daemon  3.584kB
Step 1/3 : FROM nginx:latest
 ---> 7e4d58f0e5f3
Step 2/3 : RUN rm -f /etc/nginx/conf.d/default.conf
 ---> Using cache
 ---> 710deec7a71f
Step 3/3 : COPY ./files/tomcat.conf /etc/nginx/conf.d/
 ---> c043fd59bf45
Successfully built c043fd59bf45
Successfully tagged nginx-tomcat:1

# tomcat イメージの作成
$ docker build -t tomcat:1 ./tomcat
Sending build context to Docker daemon  11.21MB
Step 1/4 : FROM centos:7
 ---> 7e6257c9f8d8
Step 2/4 : RUN  yum install -y java
 ---> Using cache
 ---> e4ae88a87b48
Step 3/4 : ADD ./files/apache-tomcat-9.0.37.tar.gz /opt/
 ---> 784ebe0d2871
Step 4/4 : CMD ["/opt/apache-tomcat-9.0.37/bin/catalina.sh", "run"]
 ---> Running in dfa62ab1b643
Removing intermediate container dfa62ab1b643
 ---> 02c55524c7ed
Successfully built 02c55524c7ed
Successfully tagged tomcat:1

# 「tomcat」イメージと「nginx-tomcat」イメージができていればOK
$ docker images
REPOSITORY               TAG                 IMAGE ID            CREATED             SIZE
tomcat                   1                   02c55524c7ed        43 seconds ago      506MB
nginx-tomcat             1                   c043fd59bf45        9 minutes ago       133MB
nginx                    latest              7e4d58f0e5f3        4 days ago          133MB
centos                   7                   7e6257c9f8d8        5 weeks ago         203MB
~~~

続いて、Dockerネットワークを作成し、コンテナを起動・ネットワークに接続します。

~~~bash
$ docker network create tomcat-network
95f21956755afc8ee1bb3203a33102da2dd705a5a4f5666a679730c0a6621a9f

# 「tomca-network」が作成されたことを確認
$ docker network ls
NETWORK ID          NAME                            DRIVER              SCOPE
...
...
95f21956755a        tomcat-network                  bridge              local
...

$ docker run -it -d --name tomcat-1 --network tomcat-network tomcat:1
$ docker run -it -d --name nginx-tomcat-1 --network tomcat-network -p 10080:80 nginx-tomcat:1

$ docker ps -a
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS                   NAMES
ecc2959bc3fd        nginx-tomcat:1      "/docker-entrypoint.…"    9 seconds ago      Up 11 minutes       0.0.0.0:10080->80/tcp   nginx-tomcat-1
b3bed7a565c6        tomcat:1            "/opt/apache-tomcat-…"   26 seconds ago      Up 12 minutes                               tomcat-1
~~~

### 接続確認
2つのコンテナの起動まで確認出来たら、以下のURLでtomcatの管理画面が表示されるか確認します。  
http://127.0.0.1:10080/tomcat/

上記URLで、tomcatの画面が表示されたら完了です。

## Dockerネットワークにつなぐことでできるようになること
Dockerネットワークを指定してコンテナを作成すると、コンテナ名で通信ができるようになります。

Dockerをインストールすると、自動的にdokcer0という仮想ブリッジが作成されます。  
IPアドレスは172.17.0.1/16です。  
なお、ホストがWindowsだと、docker0は表示されないそうです。  
代わりに、自分の環境では、bridgeというネットワークがあり、これに172.17.0.1が割り当てられていました。  
一応、便宜的にデフォルトの仮想ブリッジの名前はdocker0としておきます。  

通常、Dockerコンテナは何も設定しないと、docker0に接続されます。  
特に指定せずにコンテナを作った場合、ipが172.17.0.2から連番で付けられるのは、docker0に接続されるからです。  

このとき、コンテナ同士はIPを用いて通信をすることができます。
~~~bash
$ docker run -it -d --name test_container alpine
067e6793e0c37d997f22fa0f0be2e776f30271bba83da07f55d27e7db855e961

$ docker exec -it test_container ip a s | grep inet
    inet 127.0.0.1/8 scope host lo
    inet 172.17.0.4/16 brd 172.17.255.255 scope global eth0
# 172.0.17.4 が割り当てられていることを確認

$ docker run --rm -it alpine ping -c 1 172.17.0.4
PING 172.17.0.4 (172.17.0.4): 56 data bytes
64 bytes from 172.17.0.4: seq=0 ttl=64 time=0.076 ms

--- 172.17.0.4 ping statistics ---
1 packets transmitted, 1 packets received, 0% packet loss
round-trip min/avg/max = 0.076/0.076/0.076 ms
# ipアドレスで通信できた

$ docker run --rm -it alpine ping -c 1 test_container
ping: bad address 'test_container'
# コンテナ名では宛先がわからない
~~~

コンテナ名を指定できるほうが、複数のコンテナを組み合わせるときには便利です。  
ネットワークを指定してコンテナを起動してあげると、Dockerネットワーク内のコンテナには、  
そのコンテナ名をホスト名として通信を行うことができます。  

~~~bash
# 通信確認用に、ping_testというコンテナをtomcat-networkに追加
$ docker run -it -d --name ping_test --network tomcat-network alpine
5866943b5c6c5486904279bba336deef119b735f54ba9aa731eb8c98cfef6b29

$ docker exec -it ping_test ip a s | grep inet
    inet 127.0.0.1/8 scope host lo
    inet 172.20.0.4/16 brd 172.20.255.255 scope global eth0
# デフォルトではなく、tomcat-network(172.20.0.1)配下に接続された

# ping_testコンテナから、 「nginx-tomcat-1」というホスト名（コンテナ名）でpingが通る
$ docker exec -it ping_test ping -c 1 nginx-tomcat-1
PING nginx-tomcat-1 (172.20.0.3): 56 data bytes
64 bytes from 172.20.0.3: seq=0 ttl=64 time=0.067 ms

--- nginx-tomcat-1 ping statistics ---
1 packets transmitted, 1 packets received, 0% packet loss
round-trip min/avg/max = 0.067/0.067/0.067 ms
~~~

これは、nginx-tomcat-1コンテナのconfigファイルを見てみてもわかります。
```
server {
    location /tomcat/ {
        proxy_pass    http://tomcat-1:8080/;
    }
}
```

`proxy_pass`の部分で、URLのホスト名にtomcat-1を指定しています。  
Dockerネットワークを指定してコンテナ生成を行ったことで、  
nginxのコンテナはtomcat-1というホスト名で宛先がわかるということですね。  

## おつかれさまでした！
以上で、training002 でやってみたことは終了です。  
とりあえずは、docker-composeに頼らずに基本的なコンテナ間通信ができるようになりました。  

docker-compose等を利用すると、Dockerネットワークの作成も自動で行ってくれるようなので、  
ふつうに利用している分にはあまり意識しなくてよい部分かもしれません。

ですが、トラブルシュートの際や、細かい設定を行いたい場合には必要になる知識と思います。  
読んでいただいた方のお役に立てればうれしいです。  

お付き合いいただきありがとうございました。
