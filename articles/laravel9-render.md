---
title: "Laravel9アプリをDocker+nginxでRender.comにデプロイする方法"
emoji: "🚘"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Laravel", "render", "PHP", "Docker", "nginx"]
published: true
---

# はじめに

Render.com 公式で Laravel アプリをデプロイする方法が紹介されていますが、古いバージョン(Laravel 5.8)の方法しか記載されていません。
本記事では Laravel 9.37.0 に対応した Docker+nginx を用いたデプロイ方法を紹介します。
解説したコードは[こちらのリポジトリ](https://github.com/ppputtyo/laravel-render-example)に置いています。

# Render.com とは

さまざまなアプリを GitHub 連携で簡単に運用できる PaaS で、近年は Heroku の移行先として注目されています。
また、[Render.com 公式の記述](https://render.com/render-vs-heroku-comparison)によると、実際に次世代 Heroku を目指して開発されたことがわかります。

> We’ve built Render to help developers and businesses avoid the cost and inflexibility traps of legacy Platform-as-a-Service solutions like Heroku.
> (開発者や企業が Heroku のようなレガシーな PaaS のコストや柔軟性の問題を回避できるように、私たちは Render.com を開発しました。)

Render.com ではアカウント内のすべての Web サービスの実行時間合計が 750 時間/月までなら無料で使用できます。つまり、1 つのサービスを運用する場合であれば 24 時間無料で稼働することができます。無料プランのその他の制限については以下に詳細が記載されています。

- [Free Plans | Render · Cloud Hosting for Developers](https://render.com/docs/free#free-web-services)

# 解説すること

- Render.com に Laravel アプリを Docker+nginx でデプロイする方法

# 解説しないこと

- Render.com での DB の利用方法

# ローカル実行環境

- OS: macOS Monterey 12.4
- CPU: Apple M1
- PHP 8.1.6
- Laravel 9.37.0

# 1. Laravel アプリの作成

下記コマンドで新規 Laravel アプリを作成します。

```sh
composer create-project laravel/laravel laravel-render-com
```

# 2. 認証機能の実装

以下のコマンドを実行して認証機能を実装します。

```sh
composer require laravel/ui
php artisan ui vue --auth
npm install
```

# 3. appServiceProvider を変更

app/Providers/AppServiceProvider.php を以下のように変更します。

```php: app/Providers/AppServiceProvider.php
<?php

namespace App\Providers;

use Illuminate\Support\ServiceProvider;
use Illuminate\Routing\UrlGenerator; // 追加

class AppServiceProvider extends ServiceProvider
{
    /**
     * Register any application services.
     *
     * @return void
     */
    public function register()
    {
        //
    }

    /**
     * Bootstrap any application services.
     *
     * @return void
     */
    // 変更
    public function boot(UrlGenerator $url)
    {
        if (env('APP_ENV') == 'production') {
            $url->forceScheme('https');
        }
    }
}
```

# 4. Docker の設定

Laravel プロジェクト直下に Dockerfile を作成します。

```Dockerfile: Dockerfile
# richarvey/nginx-php-fpmをベースとする
FROM richarvey/nginx-php-fpm:2.1.2

COPY . .

# Image config
ENV SKIP_COMPOSER 1
ENV WEBROOT /var/www/html/public
ENV PHP_ERRORS_STDERR 1
ENV RUN_SCRIPTS 1
ENV REAL_IP_HEADER 1

# Laravel config
ENV APP_ENV production
ENV APP_DEBUG false
ENV LOG_CHANNEL stderr

# Allow composer to run as root
ENV COMPOSER_ALLOW_SUPERUSER 1

CMD ["/start.sh"]
```

Laravel プロジェクト直下に.dockerignore ファイルを作成します。

```text:.dockerignore
/node_modules
/public/hot
/public/storage
/storage/*.key
/vendor
.env
.phpunit.result.cache
Homestead.json
Homestead.yaml
npm-debug.log
yarn-error.log
```

conf/nginx/nginx-site.conf を作成し、nginx の設定を記述します。

```nginx: conf/nginx/nginx-site.conf
server {
  # Render provisions and terminates SSL
  listen 80;

  # Make site accessible from http://localhost/
  server_name _;

  root /var/www/html/public;
  index index.html index.htm index.php;

  # Disable sendfile as per https://docs.vagrantup.com/v2/synced-folders/virtualbox.html
  sendfile off;

  # Add stdout logging
  error_log /dev/stdout info;
  access_log /dev/stdout;

  # block access to sensitive information about git
  location /.git {
    deny all;
    return 403;
  }

  add_header X-Frame-Options "SAMEORIGIN";
  add_header X-XSS-Protection "1; mode=block";
  add_header X-Content-Type-Options "nosniff";

  charset utf-8;

  location / {
      try_files $uri $uri/ /index.php?$query_string;
  }

  location = /favicon.ico { access_log off; log_not_found off; }
  location = /robots.txt  { access_log off; log_not_found off; }

  error_page 404 /index.php;

  location ~* \.(jpg|jpeg|gif|png|css|js|ico|webp|tiff|ttf|svg)$ {
    expires 5d;
  }

  location ~ \.php$ {
    fastcgi_split_path_info ^(.+\.php)(/.+)$;
    fastcgi_pass unix:/var/run/php-fpm.sock;
    fastcgi_index index.php;
    fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
    fastcgi_param SCRIPT_NAME $fastcgi_script_name;
    include fastcgi_params;
  }

  # deny access to . files
  location ~ /\. {
    log_not_found off;
    deny all;
  }

  location ~ /\.(?!well-known).* {
    deny all;
  }
}
```

# 5. デプロイスクリプトの作成

scripts/00-laravel-deploy.sh を作成します。

```bash: scripts/00-laravel-deploy.sh
#!/usr/bin/env bash
echo "Running composer"
composer global require hirak/prestissimo
composer install --no-dev --working-dir=/var/www/html

echo "Caching config..."
php artisan config:cache

echo "Caching routes..."
php artisan route:cache

echo "Running migrations..."
php artisan migrate --force
```

# 6. GitHub にリポジトリを作成

GitHub にリポジトリを作成し、これまでの内容を push します。

# 7. Render.com にデプロイ

1. [Render.com](https://dashboard.render.com/)でアカウントを作成します。
1. 管理画面から New Web Service を選択し、新規 Web サービスを作成します。
1. GitHub のアカウントを接続し、先ほど作成したリポジトリに Render.com のアプリをインストールします。
1. Render.com で先ほどのリポジトリを選択します。
1. 以下の項目を入力します。
   - Name: 任意の名前
   - Enviroment: Docker
   - Region: Singapore (Southeast Asia)
   - Branch: デプロイしたいブランチ
   - Plans: 使用したい料金プラン
1. Advanced を選択し、環境変数に APP_KEY を登録します。
   (APP_KEY とは Laravel プロジェクト直下の.env ファイルの 3 行目あたりに記載されている`base64:〇〇=`で表される文字列です。)
   ![APP_KEY](/images/laravel9-render/APP_KEY.png)
1. Create Web Service をクリックすると Web サービスが作成され、自動的にビルドが実行されます。
1. ビルド終了後、左上のサービス名の下に記載されている URL にアクセスすると、以下のページが表示されれば成功です。
   ![Laravel](/images/laravel9-render/Laravel.png)

# 参考文献

- [Deploy a PHP Web App with Laravel and Docker | Render](https://render.com/docs/deploy-php-laravel-docker)
- [Laravel アプリを PaaS の Render にデプロイする｜ Laravel ｜ PHP ｜開発ブログ｜株式会社 Nextat（ネクスタット）](https://nextat.co.jp/staff/archives/286)
- [【格安本番運用が可能に】Render.com のメリット・デメリットを Heroku と比較してみた](https://zenn.dev/mc_chinju/articles/compare_render_and_heroku)
- [次世代 Heroku と噂の Render.com で、Rails アプリをデプロイしてみる](https://zenn.dev/katsumanarisawa/articles/c9da48652f399d)
- [Laravel 6 系で make:auth を使う方法 - Qiita](https://qiita.com/rei67/items/d6d0f5f6e58edbb17c09)
