---
title: Rails7+PostgreSQL+Docker
pubDate: 2025-4-28
author: nax2
tags: ['Rails,PosgreSQL,Docker']
---
# Rails7 API mode+PostgreSQL+Docker setup
## Prerequisites
- Docker インストール済み

## Versions
| Category  | Version              |
| :-------- | :------------------- |
| OS        | macOS Sequoia 15.4.1 |
| Framework | Rails 7.1.5          |
| Language  | Ruby 3.2.8           |
| Database  | PostgreSQL 14        |

## Step1 必要ファイルを作成
以下コマンドを実行しプロジェクトのルートフォルダに必要なファイルを作成する

``` bash
touch Dockerfile docker-compose.yml entrypoint.sh Gemfile Gemfile.lock
```

以下の内容でファイルの内容を更新する

### Dockerfile
```Dockerfile
# Base image
FROM ruby:3.2.8-slim AS base

# Set working directory
WORKDIR /app

# Install required packages for runtime
RUN apt-get update -qq && \
    apt-get install --no-install-recommends -y \
      curl \
      libvips \
      postgresql-client \
      libyaml-dev \
    && rm -rf /var/lib/apt/lists /var/cache/apt/archives

# Build stage
FROM base AS build

# Install required packages for build
RUN apt-get update -qq && \
    apt-get install --no-install-recommends -y \
      build-essential \
      git \
      libpq-dev \
      libvips \
      pkg-config \
      libyaml-dev

# Copy Gemfile and install gems
COPY Gemfile Gemfile.lock ./
RUN bundle install && \
    gem install rails && \
    rm -rf ~/.bundle/ "${BUNDLE_PATH}"/ruby/*/cache "${BUNDLE_PATH}"/ruby/*/bundler/gems/*/.git && \
    bundle exec bootsnap precompile --gemfile

# Copy application source code
COPY . .

# Precompile bootsnap for faster boot time
RUN bundle exec bootsnap precompile app/ lib/

# Final production image
FROM base

# Copy build artifacts
COPY --from=build /app /app

# Set entrypoint
ENTRYPOINT ["./entrypoint.sh"]

# Expose port 3000
EXPOSE 3000

# Start Rails server
CMD ["rails", "server", "-b", "0.0.0.0"]
```

### docker-compose.yml
``` yaml
services:
  db:
    image: postgres:14
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: password
    ports:
      - "5432:5432"
    volumes:
      - db-data:/var/lib/postgresql/data

  web:
    build: .
    command: bash -c "rm -f tmp/pids/server.pid && bundle exec rails s -b 0.0.0.0"
    volumes:
      - .:/app
    ports:
      - "3000:3000"
    depends_on:
      - db
    environment:
      RAILS_ENV: development  # Set Rails environment to development

volumes:
  db-data:
```

### entrypoint.sh
```bash
#!/bin/bash
set -e

# Prepare the database (create, migrate if needed)
bundle exec rails db:prepare

# Execute the given CMD (defined in Dockerfile)
exec "$@"
```
ファイル作成後、実行権限を付与する
```bash
chmod +x entrypoint.sh
```

### Gemfile
```ruby
source 'https://rubygems.org'

gem 'rails', '~> 7.1.0'
```

### Gemfile.lock（can be empty）

## Step2 Railsプロジェクトを作成
```bash
docker compose run web rails new . --api --force --database=postgresql
```

## Step3 config/database.yml を修正
```yaml
default: &default
  adapter: postgresql
  encoding: unicode
  username: postgres
  password: password
  host: db
  pool: <%= ENV.fetch("RAILS_MAX_THREADS") { 5 } %>
```
## Step4 Dockerfileをビルド
初回のみ--no-cacheをつける(キャッシュを使わずに最初からビルド実行するため)
```bash
docker compose build --no-cache
```

## Step5 Railsサーバーを起動
```bash
docker compose up
```

## Step6 起動確認
- localhost:3000にアクセスしRailsの画面が表示されること