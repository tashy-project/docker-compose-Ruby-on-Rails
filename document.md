# クイックスタート。ComposeとRails

このクイックスタートガイドでは、Docker Composeを使用してRails/PostgreSQLアプリをセットアップして実行する方法を紹介します。始める前に、Composeをインストールします。

## プロジェクトを定義します。
アプリのビルドに必要なファイルを設定することから始めます。アプリは依存関係を含むDockerコンテナ内で実行されます。依存関係の定義は、Dockerfileというファイルを使って行います。そもそもDockerfileは以下のように構成されています。

```Docker
FROM ruby:2.5
RUN apt-get update -qq && apt-get install -y nodejs postgresql-client
WORKDIR /myapp
COPY Gemfile /myapp/Gemfile
COPY Gemfile.lock /myapp/Gemfile.lock
RUN bundle install
COPY . /myapp

# Add a script to be executed every time the container starts.
COPY entrypoint.sh /usr/bin/
RUN chmod +x /usr/bin/entrypoint.sh
ENTRYPOINT ["entrypoint.sh"]
EXPOSE 3000

# Start the main process.
CMD ["rails", "server", "-b", "0.0.0.0"]
```

このファイルは、Ruby、Bundler、そしてすべての依存関係を含むコンテナを構築するイメージの中にアプリケーションのコードを入れます。Dockerfileの書き方の詳細については、DockerユーザーガイドとDockerfileリファレンスを参照してください。

次に、RailsをロードするだけのブートストラップGemfileを作成します。これはrails newで一瞬で上書きされます。

```Gemfile
source 'https://rubygems.org'
gem 'rails', '~>5'
```

空のGemfile.lockを作成して、Dockerfileを構築します。

```
touch Gemfile.lock
```

次に、特定のserver.pidファイルが事前に存在するとサーバーが再起動できなくなるというRails特有の問題を修正するためのエントリポイントスクリプトを提供します。このスクリプトはコンテナが起動するたびに実行されます。 entrypoint.shは次のように構成されています。

```
#!/bin/bash
set -e

# Remove a potentially pre-existing server.pid for Rails.
rm -f /myapp/tmp/pids/server.pid

# Then exec the container's main process (what's set as CMD in the Dockerfile).
exec "$@"
```

最後に、docker-compose.ymlでマジックが起こります。このファイルには、アプリを構成するサービス（データベースとWebアプリ）、それぞれのDockerイメージを取得する方法（データベースはあらかじめ作成されたPostgreSQLイメージ上で実行され、Webアプリはカレントディレクトリからビルドされます）、それらを一緒にリンクしてWebアプリのポートを公開するために必要な設定が記述されています。

```Docker
version: "3.9"
services:
  db:
    image: postgres
    volumes:
      - ./tmp/db:/var/lib/postgresql/data
    environment:
      POSTGRES_PASSWORD: password
  web:
    build: .
    command: bash -c "rm -f tmp/pids/server.pid && bundle exec rails s -p 3000 -b '0.0.0.0'"
    volumes:
      - .:/myapp
    ports:
      - "3000:3000"
    depends_on:
      - db
```

このファイルには.ymlまたは.yamlの拡張子を使用することができます。

## プロジェクトをビルドする

これらのファイルが用意されたので、docker-compose runを使ってRailsのスケルトンアプリを生成できるようになりました。

```
docker-compose run --no-deps web rails new . --force --database=postgresql
```

まず、ComposeはDockerfileを使ってWebサービスのイメージをビルドします。--no-depsは、リンクされたサービスを起動しないようにComposeに指示します。次に、そのイメージを使って新しいコンテナ内で rails new を実行します。これが完了すると、新しいアプリが生成されているはずです。

ファイルをリストアップします。

```
$ ls -l
total 64
-rw-r--r--   1 vmb  staff   222 Jun  7 12:05 Dockerfile
-rw-r--r--   1 vmb  staff  1738 Jun  7 12:09 Gemfile
-rw-r--r--   1 vmb  staff  4297 Jun  7 12:09 Gemfile.lock
-rw-r--r--   1 vmb  staff   374 Jun  7 12:09 README.md
-rw-r--r--   1 vmb  staff   227 Jun  7 12:09 Rakefile
drwxr-xr-x  10 vmb  staff   340 Jun  7 12:09 app
drwxr-xr-x   8 vmb  staff   272 Jun  7 12:09 bin
drwxr-xr-x  14 vmb  staff   476 Jun  7 12:09 config
-rw-r--r--   1 vmb  staff   130 Jun  7 12:09 config.ru
drwxr-xr-x   3 vmb  staff   102 Jun  7 12:09 db
-rw-r--r--   1 vmb  staff   211 Jun  7 12:06 docker-compose.yml
-rw-r--r--   1 vmb  staff   184 Jun  7 12:08 entrypoint.sh
drwxr-xr-x   4 vmb  staff   136 Jun  7 12:09 lib
drwxr-xr-x   3 vmb  staff   102 Jun  7 12:09 log
-rw-r--r--   1 vmb  staff    63 Jun  7 12:09 package.json
drwxr-xr-x   9 vmb  staff   306 Jun  7 12:09 public
drwxr-xr-x   9 vmb  staff   306 Jun  7 12:09 test
drwxr-xr-x   4 vmb  staff   136 Jun  7 12:09 tmp
drwxr-xr-x   3 vmb  staff   102 Jun  7 12:09 vendor
```

Linux上でDockerを実行している場合、rails newが作成したファイルはrootが所有しています。これはコンテナがrootユーザーとして実行されているために起こります。このような場合は、新しいファイルの所有権を変更します。

```
sudo chown -R $USER:$USER .
```

MacやWindowsでDockerを実行している場合は、rails newで生成されたファイルも含めて、すべてのファイルの所有権はすでに持っているはずです。

新しいGemfileができたので、もう一度イメージをビルドする必要があります。(再構築が必要になるのは、これとGemfileやDockerfileの変更だけです)。

```
docker-compose build
```

## データベースを接続する
アプリが起動できるようになりましたが、まだまだです。デフォルトでは、Railsはデータベースがlocalhost上で実行されることを期待しています。また、データベースとユーザー名を変更して、postgresイメージで設定されたデフォルトに合わせる必要があります。

config/database.ymlの内容を以下のように置き換えます。

```
default: &default
  adapter: postgresql
  encoding: unicode
  host: db
  username: postgres
  password: password
  pool: 5

development:
  <<: *default
  database: myapp_development


test:
  <<: *default
  database: myapp_test
```

これで、docker-compose upでアプリを起動できるようになりました。

```
docker-compose up
```

すべてが順調であれば、PostgreSQLの出力が表示されるはずです。

```
rails_db_1 is up-to-date
Creating rails_web_1 ... done
Attaching to rails_db_1, rails_web_1
db_1   | PostgreSQL init process complete; ready for start up.
db_1   |
db_1   | 2018-03-21 20:18:37.437 UTC [1] LOG:  listening on IPv4 address "0.0.0.0", port 5432
db_1   | 2018-03-21 20:18:37.437 UTC [1] LOG:  listening on IPv6 address "::", port 5432
db_1   | 2018-03-21 20:18:37.443 UTC [1] LOG:  listening on Unix socket "/var/run/postgresql/.s.PGSQL.5432"
db_1   | 2018-03-21 20:18:37.726 UTC [55] LOG:  database system was shut down at 2018-03-21 20:18:37 UTC
db_1   | 2018-03-21 20:18:37.772 UTC [1] LOG:  database system is ready to accept connections
```

最後に、データベースを作成する必要があります。別のターミナルで実行します。

```
docker-compose run web rake db:create
```

以下は、そのコマンドの出力例です。

```
vmb at snapair in ~/sandbox/rails
$ docker-compose run web rake db:create
Starting rails_db_1 ... done
Created database 'myapp_development'
Created database 'myapp_test'
```

## Railsのウェルカムページを見よう!
これで完了です。これでアプリがDockerデーモンのポート3000で動作しているはずです。

Docker Desktop for MacとDocker Desktop for Windowsでは、Webブラウザでhttp://localhost:3000 にアクセスしてRailsのウェルカムページを表示します。

![welcome](rails-welcome.png)

## アプリケーションを停止します。
アプリケーションを停止するには、プロジェクトディレクトリで docker-compose を実行してください。データベースを起動したときと同じターミナルウィンドウを使うか、コマンドプロンプトにアクセスできる別のウィンドウを使うことができます。これはアプリケーションを停止するためのクリーンな方法です。

```
vmb at snapair in ~/sandbox/rails
$ docker-compose down
Stopping rails_web_1 ... done
Stopping rails_db_1 ... done
Removing rails_web_run_1 ... done
Removing rails_web_1 ... done
Removing rails_db_1 ... done
Removing network rails_default
```

## アプリケーションを再起動する
アプリケーションを再起動するには、プロジェクトディレクトリでdocker-composeを実行してください。

## アプリケーションの再構築
異なる設定を試すために Gemfile や Compose ファイルに変更を加えた場合は、再構築が必要です。一部の変更では docker-compose up --build だけで済みますが、完全なリビルドを行うには、docker-compose run web bundle install を再実行して Gemfile.lock の変更をホストに同期させ、その後 docker-compose up --build を実行する必要があります。

ここでは、完全なリビルドが必要ない最初のケースの例を示します。ローカルホストで公開されているポートを、最初の例では 3000 から 3001 に変更したいだけだとします。コンテナ上のポート 3000 をホスト上の新しいポート 3001 を介して公開するように Compose ファイルに変更を加え、変更を保存します。

```
ports: - "3001:3000"
```

docker-compose up --buildでアプリを再構築して再起動します。

コンテナ内では、アプリは3000以前と同じポートで実行されていますが、ローカルホストの http://localhost:3001 で Rails Welcome が利用できるようになりました。

www.DeepL.com/Translator（無料版）で翻訳しました。