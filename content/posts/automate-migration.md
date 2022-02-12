---
title: "Rails: デプロイ後に自動でmigrateする"
description: RailsもDockerでデプロイを行うようになってからは何かと頭を抱えることも多いです。CI経由でデプロイした後に手動で`rake db:migrate`を実行しなければならず面倒です。何かと後回しにしていましたが、その都度生産性が落ちてしまうので調べてみました。
date: 2021-01-09
draft: false
tags:
  - rails
  - docker
layout: layouts/post.njk
---

RailsもDockerでデプロイを行うようになってからは何かと頭を抱えることも多いです。CI経由でデプロイした後に手動で`rake db:migrate`を実行しなければならず面倒です。何かと後回しにしていましたが、その都度生産性が落ちてしまうので調べてみました。

<!--more-->

Railsの起動時に`rails db:migrate`を行ってから`rails server`を実行するようにしたいです。まず最初にDockerfileにはこのように記述しました:

```diff
-CMD ["rails", "s", "-b", "0.0.0.0"]
+CMD rails db:migrate && rails server -b 0.0.0.0
```

**変更前 (`["rails", "s", "-b", "0.0.0.0"]`)**:

```text
app_1  | => Booting Puma
app_1  | => Rails 6.1.0 application starting in development
app_1  | => Run `bin/rails server --help` for more startup options
app_1  | Puma starting in single mode...
app_1  | * Puma version: 5.1.1 (ruby 2.7.2-p137) ("At Your Service")
app_1  | *  Min threads: 5
app_1  | *  Max threads: 5
app_1  | *  Environment: development
app_1  | *          PID: 1
app_1  | * Listening on http://0.0.0.0:3000
app_1  | Use Ctrl-C to stop
```

**変更後 (`rails db:migrate && rails server -b 0.0.0.0`):**

```text
app_1  | => Booting Puma
app_1  | => Rails 6.1.0 application starting in development
app_1  | => Run `bin/rails server --help` for more startup options
app_1  | Puma starting in single mode...
app_1  | * Puma version: 5.1.1 (ruby 2.7.2-p137) ("At Your Service")
app_1  | *  Min threads: 5
app_1  | *  Max threads: 5
app_1  | *  Environment: development
app_1  | *          PID: 14
app_1  | * Listening on http://0.0.0.0:3000
app_1  | Use Ctrl-C to stop
```

**PID: 14**になってしまいました。動作上はあまり問題なさそうにも思えますが、Dockerのベストプラクティスとしては**PID: 1**のまま動かしたいですよね。どうしようか考えたところ[Redmine][redmine]のDockerfileを参考にすることにしました。どうやら[`ENTRYPOINT`][entrypoint]を使うのがよさそうです。

```diff
+COPY docker-entrypoint.sh /docker-entrypoint.sh
+ENTRYPOINT /docker-entrypoint.sh
+
-CMD rails db:migrate && rails server -b 0.0.0.0
+CMD ["rails", "s", "-b", "0.0.0.0"]
```

続いて`docker-entrypoint.sh`の中身です:

```bash
#!/usr/bin/env bash
set -e

bin/rails db:migrate
bin/rails assets:precompile

exec "$@"
```

こうすることで`docker exec`コマンドを実行するたびに`db:migrate`と`assets:precompile`が実行されるのでmigrateし忘れることもなくなるはずです。実はこの方法を調べる前に`assets:precompile`もコンテナのビルドに含めていました。もう少し実用性を高めるために書き換える必要はありそうですが、悩みのタネもなくなりスッキリしました。

`ENTRYPOINT`をよく調べていなかったのも原因ですが、DockerとRailsを使うときはぜひとも使っていきたいところです。

[redmine]: https://registry.hub.docker.com/_/redmine/
[entrypoint]: https://docs.docker.com/engine/reference/builder/#entrypoint
