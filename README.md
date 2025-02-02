# check-docker-fe-be

- [モノリシックなバージョン管理の利点](https://postd.cc/monorepo/)
- [20億行のコードを保存し、毎日4万5000回のコミットを発行しているGoogleが、単一のリポジトリで全社のソースコードを管理している理由](https://www.publickey1.jp/blog/15/2045000google.html)

```text
プロジェクトが1つでも、フロントエンドとバックエンドでリポジトリを分けず、モノリポジトリを採用する
海外の大企業でもモノレポにしているらしい
```

## 使い方

```text
このリポジトリの
- dockerディレクトリ
- Makefile
- .gitignore
を取り込みたいプロジェクトにコピー
あとはMakefileを参考にすればすぐに環境が構築できるように構成している
makeすればhelpが出力されるようになっているし、
とりあえず、まずは下記コマンドでOK
```

```sh
make setup
```

- なんかコンテナに入れないんだけど、、、

```text
make ps もしくは docker ps で、きちんとコンテナが立ち上がっているのに、
make backend もしくは dockerディレクトリに移動して docker compose exec backend bash したら、下記エラーが出る

OCI runtime exec failed: exec failed: unable to start container process: current working directory is outside of container mount namespace root -- possible container breakout detected: unknown

原因は不明だが、コンテナをリスタートするか(docker restart backend)
make down -> make up すると直る、、、謎！
```

## make コマンドと Makefile について

- make コマンド

```text
Makefile に定義されたルールに基づいて、プログラムのビルドやその他のタスクを自動化するツール
主にソフトウェアのビルドプロセスを管理するために使われるが、一般的なタスクの自動化にも利用できる
```

- Makefile

```text
プログラムのビルドや管理を自動化するためのファイル
make コマンドを使用する前提
ビルド自動化、タスクの自動化(テストの実行やファイルのコピー、クリーンアップ等)、依存関係の管理
```

## Docker Hub

- [Docker Hub 公式](https://hub.docker.com/)

```text
ポート公開、ファイル等のバインドマウント
バインドマウントでホスト側の場合
ls -al $(pwd)/../..
等で参照できているか確認すること
(-itオプションなしだとすぐに落ちるので標準入力と端末をつけておく)

コマンドからDockerfile、docker-compose.ymlを作成していく
```

## Node.js

`下記URLからバージョン確認`
- [nodesource/distributions](https://github.com/nodesource/distributions)

```sh
docker container run \
-it \
--name node \
--rm \
--detach \
--platform linux/amd64 \
--publish 3000:3000 \
--mount type=bind,src=$(pwd)/../..,dst=/home/app \
--mount type=bind,src=$(pwd)/.bashrc,dst=/root/.bashrc \
node:21.6.2
```

## PHP-FPM

- HostMachine: コンテナから設定ファイルをダウンロードするために起動
```sh
docker container run \
--name php \
--rm \
--detach \
--platform linux/amd64 \
php:8.3.9-fpm
```

- Container: 設定ファイルの配置場所を確認しておく
```sh
docker exec -it web bash
```

- HostMachine: コンテナから設定ファイルをダウンロード
```sh
docker cp php:/usr/local/etc/php/php.ini-development .
docker cp php:/usr/local/etc/php/php.ini-production .
```

- HostMachine: オリジナル用のファイル名に変更しておく
```sh
mv php.ini-development 8.3.9-php.ini-development.org
mv php.ini-production 8.3.9-php.ini-production.org
```

- HostMachine: コピー(.orgはorgディレクトリに移動させておく)
```sh
cp -api 8.3.9-php.ini-development.org 8.3.9-php.ini-development
cp -api 8.3.9-php.ini-production.org 8.3.9-php.ini-production
```


## Nginx

- HostMachine: .confを追加したら下記コマンドで反映させる
```sh
docker container restart {container_name}

Or

docker compose down
docker compose up -d --build
```

- HostMachine: コンテナから設定ファイルをダウンロードするために起動
```sh
docker container run \
--name web \
--rm \
--detach \
--platform linux/amd64 \
--publish 80:80 \
nginx:1.27.0
```

- Container: 設定ファイルの配置場所を確認しておく
```sh
docker exec -it web bash
```

- HostMachine: コンテナから設定ファイルをダウンロード
```sh
docker cp web:/etc/nginx/nginx.conf .
docker cp web:/etc/nginx/conf.d/default.conf .
```

- HostMachine: オリジナル用のファイル名に変更しておく
```sh
mv nginx.conf 1.27.0-nginx.conf.org
mv default.conf 1.27.0-default.conf.org
```

- HostMachine: コピー(.orgはorgディレクトリに移動させておく)
```sh
cp -api 1.27.0-nginx.conf.org 1.27.0-nginx.conf
cp -api 1.27.0-default.conf.org 1.27.0-default.conf
```

- HostMachine: 設定ファイルをバインドマウントする
```sh
docker container run \
--name web \
--rm \
--detach \
--platform linux/amd64 \
--publish 80:80 \
--mount type=bind,src=$(pwd)/nginx.conf,dst=/etc/nginx/nginx.conf \
--mount type=bind,src=$(pwd)/conf.d,dst=/etc/nginx/conf.d \
--mount type=bind,src=$(pwd)/../../../code,dst=/home/htdocs \
nginx:1.27.0
```

- HostMachine: ネットワークを作成する（既に作成済の場合は不要）
```sh
docker network create \
laravel-network
```

- HostMachine: ログファイルマウント動作確認
```sh
docker container run \
--name ProjectName-nginx \
--rm \
--detach \
--platform linux/amd64 \
--publish 80:80 \
--mount type=bind,src=$(pwd)/nginx.conf,dst=/etc/nginx/nginx.conf \
--mount type=bind,src=$(pwd)/conf.d,dst=/etc/nginx/conf.d \
--mount type=bind,src=$(pwd)/logs,dst=/home/log/nginx \
--mount type=bind,src=$(pwd)/../../../code,dst=/home/htdocs \
--network laravel_default \
--network-alias nginx \
nginx:1.27.0
```

## MySQL

- HostMachine: コンテナから設定ファイルをダウンロードするために起動
```sh
docker container run \
--name db \
--rm \
--detach \
--platform linux/amd64 \
--env MYSQL_ROOT_PASSWORD=root \
mysql:9.0.0
```

- HostMachine: コンテナから設定ファイルをダウンロード
```sh
docker cp db:/etc/my.cnf .
```

- HostMachine: 設定ファイルをバインドマウントする
```sh
docker container run \
--name db \
--rm \
--detach \
--platform linux/amd64 \
--env MYSQL_ROOT_PASSWORD=root \
--mount type=bind,src=$(pwd)/my.cnf,dst=/etc/my.cnf \
mysql:9.0.0
```

- Container: 文字コードがutf8mb4になっているか確認
```sh
docker exec -it web bash
mysql -u root -proot

mysql> show variables like "chara%";
+--------------------------+--------------------------------+
| Variable_name            | Value                          |
+--------------------------+--------------------------------+
| character_set_client     | utf8mb4                        |
| character_set_connection | utf8mb4                        |
| character_set_database   | utf8mb4                        |
| character_set_filesystem | binary                         |
| character_set_results    | utf8mb4                        |
| character_set_server     | utf8mb4                        |
| character_set_system     | utf8mb3                        |
| character_sets_dir       | /usr/share/mysql-9.0/charsets/ |
+--------------------------+--------------------------------+
```



- HostMachine: volumeを作成してデータの永続化
```sh
docker volume create \
--name db-store
```

- HostMachine: マウントオプションをつける
```sh
docker container run \
--name db \
--rm \
--detach \
--platform linux/amd64 \
--env MYSQL_ROOT_PASSWORD=root \
--mount type=bind,src=$(pwd)/my.cnf,dst=/etc/my.cnf \
--mount type=volume,src=db-store,dst=/var/lib/mysql \
mysql:9.0.0
```

- HostMachine: ネットワークを作成する（既に作成済の場合は不要）
```sh
docker network create \
laravel-network
```

- HostMachine: ネットワークオプションをつける
```sh
docker container run \
--name db \
--rm \
--detach \
--platform linux/amd64 \
--env MYSQL_ROOT_PASSWORD=root \
--mount type=bind,src=$(pwd)/my.cnf,dst=/etc/my.cnf \
--mount type=volume,src=db-store,dst=/var/lib/mysql \
--network laravel-network \
--network-alias db \
mysql:9.0.0
```
