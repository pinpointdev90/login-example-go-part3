Qiitaに[投稿した記事](https://qiita.com/kins/items/3695458182a6d58cc350)のソースコードです。
# 目標

- MySQLを起動する
- GoからMySQLに接続する

## MySQLを起動する

### MySQLの設定ファイルを準備します。

早速、MySQLを起動の前に、後々必要になるかもしれない設定などをしています。
そのために必要なディレクトリを作成します。

```
$ mkdir -p _tools/mysql/{init.d,conf.d}
```

今回で言うとMySQLに関するファイルを_toolsディレクトリ以下にまとめます。

_tools/mysql/conf.dにはMySQLの設定ファイルを置きます。

_tools/mysql/conf.d/my.cnf
```
[mysqld]
character-set-server=utf8mb4

[client]
default-character-set=utf8mb4
```

_tools/mysql/init.dには今回使うユーザーのテーブル作成用のファイルを作成します。

_tools/mysql/init.d/init.sql
```
CREATE TABLE `user` (
  `id` BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
  `email` VARCHAR(255) NOT NULL UNIQUE,
  `password` VARCHAR(60) NOT NULL,
  `salt` VARCHAR(30) NOT NULL,
  `state` VARCHAR(8) NOT NULL,
  `activate_token` VARCHAR(8) NOT NULL,
  `updated_at` DATETIME(6) NOT NULL,
  `created_at` DATETIME(6) NOT NULL,
  PRIMARY KEY (`id`),
  INDEX email_idx (email)
) Engine=InnoDB DEFAULT CHARSET=utf8mb4 AUTO_INCREMENT=100001;
```

これで設定ファイルの準備は完了です。

### MySQLを起動する

docker-compose.yml
```
version: "3.8"

services:
  api:
    container_name: login-example-api
    build:
      dockerfile: Dockerfile
      context: .
    volumes:
      - ".:/app"
    ports:
      - "8000:8000"
    environment:
      DB_USER: login-user
      DB_PASSWORD: login-pass
      DB_NAME: login-db
      DB_HOST: db
      DB_PORT: 3306
    depends_on:
      db:
        condition: service_healthy
  db:
    container_name: login-example-db
    image: mysql:8.0.33
    platform: linux/amd64
    environment:
      MYSQL_ROOT_PASSWORD: root
      MYSQL_DATABASE: login-db
      MYSQL_USER: login-user
      MYSQL_PASSWORD: login-pass
    ports:
      - "33306:3306"
    volumes:
      - type: volume
        source: login-example-db
        target: /var/lib/mysql
      - type: bind
        source: ./_tools/mysql/conf.d
        target: /etc/mysql/conf.d
      - type: bind
        source: ./_tools/mysql/init.d
        target: /docker-entrypoint-initdb.d
    healthcheck:
      test: ["CMD", "mysqladmin", "ping"]
      interval: 5s
      timeout: 5s
      retries: 5
  mail:
    container_name: login-example-mail
    image: mailhog/mailhog
    platform: linux/x86_64
    ports:
      - "8025:8025"
      - "1025:1025"
volumes:
  login-example-db:
```

part2で作成したdocker-compose.ymlを修正いたしました。

サービスを起動します。
```
$ docker compose up -d # -d：バックグラウンドで起動
```
これで起動ができました。
これでMySQLの準備はOKです！

## GoからMySQLに接続する

接続するために必要なパッケージをダウンロードしておきます。

```
$ go get github.com/go-sql-driver/mysql
$ go get github.com/jmoiron/sqlx
```

必要なパッケージが準備できたら接続用のコードを書いていきます。
db/db.go
```
package db

import (
	"context"
	"database/sql"
	"fmt"
	"os"
	"time"

	_ "github.com/go-sql-driver/mysql"
	"github.com/jmoiron/sqlx"
)

func NewDB() (*sqlx.DB, error) {
	// 環境変数からDBの情報を取得します
	dbUser := os.Getenv("DB_USER")         // login-user
	dbPassword := os.Getenv("DB_PASSWORD") // login-pass
	dbHost := os.Getenv("DB_HOST")         // db
	dbName := os.Getenv("DB_NAME")         // login-db
	dbPort := os.Getenv("DB_PORT")         // 3306
	src := fmt.Sprintf("%s:%s@tcp(%s:%s)/%s?parseTime=true",
		dbUser, dbPassword, dbHost, dbPort, dbName)

	db, err := sql.Open("mysql", src)
	if err != nil {
		return nil, fmt.Errorf("failed to Open DB: %w", err)
	}

	// mysqlに実際に接続できているかpingで確認します。
	ctx, cancel := context.WithTimeout(context.Background(), 2*time.Second)
	defer cancel()
	if err := db.PingContext(ctx); err != nil {
		db.Close()
		return nil, fmt.Errorf("failed to Ping DB: %w", err)
	}

	xdb := sqlx.NewDb(db, "mysql")
	return xdb, nil
}
```

MySQLに接続するためのコードが書けました。

では、本当にこのコードで接続できるか確認いたします。

### 確認

MySQLに本当に接続できるか確認するためのmain.goを修正します。

main.go
```
package main

import (
	"fmt"
	"login-example/db"
)

func main() {
	db, err := db.NewDB()
	if err != nil {
		fmt.Println(err)
		return
	}
	defer db.Close()

	fmt.Println("MySQL接続OK")
}
```

logで確認
```
$ docker compose logs api -f

# 最後に「MySQL接続OK」と出れば接続できています！
login-example-api   |   __    _   ___  
login-example-api   |  / /\  | | | |_) 
login-example-api   | /_/--\ |_| |_| \_ , built with Go 
login-example-api   | 
login-example-api   | mkdir /app/tmp
login-example-api   | watching .
login-example-api   | watching _tools
login-example-api   | watching _tools/mysql
login-example-api   | watching _tools/mysql/conf.d
login-example-api   | watching _tools/mysql/init.d
login-example-api   | watching db
login-example-api   | !exclude tmp
login-example-api   | building...
login-example-api   | go: downloading github.com/go-sql-driver/mysql v1.7.1
login-example-api   | go: downloading github.com/jmoiron/sqlx v1.3.5
login-example-api   | running...
login-example-api   | MySQL接続OK

```
