# lamp-php83

WSL2 + Docker Compose で構築した、ローカル検証用の LAMP 環境です。
主に WordPress / PHP 8.3 / Apache / MariaDB の動作確認に使用します。

## 構成概要

```text
Windows 11
  └─ WSL2 Ubuntu
      └─ Docker Desktop / Docker Compose
          ├─ web: PHP 8.3 + Apache
          └─ db: MariaDB 10.11
```

## 用途

* PHP 8.3 の動作確認
* Apache + PHP の動作確認
* WordPress のローカル検証
* `.htaccess` / mod_rewrite の確認
* MariaDB 接続確認
* WordPress のパーマリンク確認

## ディレクトリ構成

```text
~/projects/lamp-php83/
  ├─ docker-compose.yml
  ├─ docker/
  │   └─ php83/
  │       └─ Dockerfile
  ├─ public/
  │   ├─ index.php
  │   ├─ .htaccess
  │   ├─ wp-admin/
  │   ├─ wp-content/
  │   └─ wp-includes/
  └─ README.md
```

## サービス構成

### web

PHP 8.3 + Apache のWebサーバーです。  
HTTP / HTTPS の両方に対応しています。

```yaml
web:
  build: ./docker/php83
  container_name: lamp_php83_web
  ports:
    - "8080:80"
    - "8443:443"
  volumes:
    - ./public:/var/www/html
    - ./docker/apache/ssl-vhost.conf:/etc/apache2/sites-available/000-default.conf
    - ./docker/certs:/etc/apache2/ssl
  depends_on:
    - db

#### HTTP:
```text
http://localhost:8080
```

#### HTTPS:
```text
https://localhost:8443
```

### db

MariaDB 10.11 のDBサーバーです。


```yaml
db:
  image: mariadb:10.11
  container_name: lamp_php83_db
  environment:
    MARIADB_ROOT_PASSWORD: root
    MARIADB_DATABASE: localdb
    MARIADB_USER: localuser
    MARIADB_PASSWORD: localpass
  ports:
    - "3306:3306"
  volumes:
    - db_data:/var/lib/mysql
```

## アクセスURL

WordPress / Webサイト:

```text
https://localhost:8443
```

phpinfo確認用として退避している場合:

```text
https://localhost:8443/phpinfo.php
```

## DB接続情報

Docker Compose 内部から接続する場合:

```text
Host: db
Database: localdb
User: localuser
Password: localpass
Port: 3306
```

Windows側のDBクライアントから接続する場合:

```text
Host: 127.0.0.1
Database: localdb
User: localuser
Password: localpass
Port: 3306
```

rootユーザー:

```text
User: root
Password: root
```

## WordPress のDB設定

`wp-config.php` では以下を設定します。

```php
define( 'DB_NAME', 'localdb' );
define( 'DB_USER', 'localuser' );
define( 'DB_PASSWORD', 'localpass' );
define( 'DB_HOST', 'db' );
```

## Dockerfile

`docker/php83/Dockerfile`

```dockerfile
FROM php:8.3-apache

RUN docker-php-ext-install mysqli pdo_mysql \
  && a2enmod rewrite \
  && echo "ServerName localhost" > /etc/apache2/conf-available/servername.conf \
  && a2enconf servername
```

## 有効化済みのPHP拡張・Apacheモジュール

確認済み:

```text
mysqli
pdo_mysql
rewrite_module
php_module
```

確認コマンド:

```bash
docker compose exec web bash
php -m | grep -E "mysqli|pdo_mysql"
apache2ctl -M | grep rewrite
apache2ctl -M | grep php
```

## 起動方法

```bash
cd ~/projects/lamp-php83
docker compose up -d
```

## 停止方法

```bash
cd ~/projects/lamp-php83
docker compose down
```

## 再ビルドして起動

Dockerfileを変更した場合などは以下を実行します。

```bash
cd ~/projects/lamp-php83
docker compose up -d --build
```

## 起動状態の確認

```bash
docker compose ps
```

## ログ確認

```bash
docker compose logs
```

Webコンテナのみ確認:

```bash
docker compose logs web
```

DBコンテナのみ確認:

```bash
docker compose logs db
```

## Webコンテナに入る

```bash
docker compose exec web bash
```

コンテナ内でよく使う確認コマンド:

```bash
php -v
php -m
apache2 -v
apache2ctl -M
```

## DBコンテナに入る

```bash
docker compose exec db bash
```

MariaDBへログイン:

```bash
mariadb -u localuser -p localdb
```

パスワード:

```text
localpass
```

rootでログインする場合:

```bash
mariadb -u root -p
```

パスワード:

```text
root
```

## よく使うSQL

```sql
SHOW DATABASES;
USE localdb;
SHOW TABLES;
SELECT VERSION();
```

終了:

```sql
exit;
```

## .htaccess

WordPressのパーマリンク用に、`public/.htaccess` を手動作成しています。

```apache
<IfModule mod_rewrite.c>
RewriteEngine On
RewriteRule .* - [E=HTTP_AUTHORIZATION:%{HTTP:Authorization}]
RewriteBase /
RewriteRule ^index\.php$ - [L]
RewriteCond %{REQUEST_FILENAME} !-f
RewriteCond %{REQUEST_FILENAME} !-d
RewriteRule . /index.php [L]
</IfModule>
```

## パーマリンク確認

WordPress管理画面で以下を実施済みです。

```text
設定
  → パーマリンク
  → 投稿名などに変更
  → 保存
```

投稿ページが正常に表示されることを確認済みです。

## DB接続確認用スクリプト

`public/dbtest.php` を作成している場合、以下でDB接続確認ができます。

```text
http://localhost:8080/dbtest.php
```

内容例:

```php
<?php
$host = 'db';
$db   = 'localdb';
$user = 'localuser';
$pass = 'localpass';

$mysqli = new mysqli($host, $user, $pass, $db);

if ($mysqli->connect_error) {
    die('DB接続失敗: ' . $mysqli->connect_error);
}

echo 'DB接続成功';
```

## DBデータを含めて完全削除する場合

注意: WordPressの投稿・設定・DBデータが削除されます。

```bash
docker compose down -v
```

## HTTPS対応

ローカル検証用に自己署名証明書を使用し、HTTPSアクセスに対応しています。

### 証明書ファイル

```text
docker/certs/localhost.crt
docker/certs/localhost.key
```

### Apache SSL設定

```text
docker/apache/ssl-vhost.conf
```

### WordPress URL固定

`public/wp-config.php` に以下を設定しています。

```php
define( 'WP_HOME', 'https://localhost:8443' );
define( 'WP_SITEURL', 'https://localhost:8443' );
```

### 確認コマンド

```bash
docker compose ps
docker compose exec web bash -lc "apache2ctl -M | grep ssl"
docker compose exec web bash -lc "apache2ctl -S"
docker compose exec web bash -lc "apache2ctl configtest"
docker compose exec web bash -lc "curl -k -I https://localhost"
```

### 補足
自己署名証明書のため、ブラウザでは証明書警告が表示されます。
ローカル検証用途では警告を許可して利用します。

## 現在の注意点

* Docker Hub から新規イメージを取得する際、Zscaler / 社内ネットワークの影響で失敗する場合があります。
* `phpmyadmin:latest` の取得は現時点では失敗しているため、phpMyAdmin追加は保留中です。
* Red Hat系検証用の `lamp-rocky8` / AlmaLinux / UBI 環境は、ネットワーク調整後に再開予定です。

## 今後追加したいもの

* phpMyAdmin
* WordPress用DBバックアップ / リストア手順
* php.ini のローカル検証用調整
* Red Hat系 Apache 検証環境
* 本番Apacheモジュールとの差分確認手順
