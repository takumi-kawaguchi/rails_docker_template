# README

## Rails/Docker/MySQLでの環境構築
1. 下記の通りにファイルを準備
- Gemfile
    ```
    source "https://rubygems.org"
    ruby "3.0.2"
    gem "rails", "~> 7.1.2"
    ```
- Gemfile.lock
    ```
    # 空ファイル
    ```
- Dockerfile
    ```
    FROM ruby:3.0.2
    RUN curl -fsSL https://deb.nodesource.com/setup_lts.x | bash - && apt-get install -y nodejs
    RUN npm install --global yarn

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
- docker-compose.yml
    ```
    version: '3.7'
    services:
    db:
        image: mysql:8.0
        platform: linux/x86_64
        command: --default-authentication-plugin=mysql_native_password
        ports:
        - "4306:3306"
        volumes:
        - db:/var/lib/mysql
        environment:
        MYSQL_ALLOW_EMPTY_PASSWORD: 'yes'
        security_opt:
        - seccomp:unconfined
    webpacker:
        build: .
        volumes:
        - .:/myapp
        - bundle:/usr/local/bundle
        command: ./bin/webpack-dev-server
        environment:
        WEBPACKER_DEV_SERVER_HOST: 0.0.0.0
        ports:
        - "3035:3035"
    web:
        build: .
        stdin_open: true
        tty: true
        volumes:
        - .:/myapp
        - bundle:/usr/local/bundle
        command: bash -c "rm -rf tmp/pids/server.pid && bundle exec rails s -p 3000 -b '0.0.0.0'"
        depends_on:
        - db
        ports:
        - "3000:3000"
        environment:
        WEBPACKER_DEV_SERVER_HOST: webpacker
    volumes:
    db:
        driver: local
    bundle:
        driver: local
    ```
- entrypoint.sh
    ```
    #!/bin/bash
    set -e

    # Remove a potentially pre-existing server.pid for Rails.
    rm -f /myapp/tmp/pids/server.pid

    # Then exec the container's main process (what's set as CMD in the Dockerfile).
    exec "$@"
    ```

2. webコンテナを指定してrails newする
```
docker-compose run web rails new . --force --no-deps --database=mysql --skip-bundle
```

3. webコンテナ内でbundle install
```
docker-compose run web bundle install
```

4. database.ymlの設定でhostをdbコンテナで指定する
```
host: db
```

5. 接続するDBを作成する
```
docker-compose run web rails db:create
```

6. コンテナを立ち上げる
```
docker compose up -d
```