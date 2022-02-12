---
title: "React: Keycloakの認証を組み込む"
description: 最近はRailsでブログの管理をできるアプリケーションを作っていたのですが、またReactとExpressの構成に戻りました。そこで毎回の課題が認証です。普段はPassportを利用するのですが、今回はKeycloakという認証サーバーを試してみることにしました。
date: 2021-02-25T05:50:07+09:00
draft: true
tags:
  - react
layout: layouts/post.njk
---

最近はRailsでブログの管理をできるアプリケーションを作っていたのですが、またReactとExpressの構成に戻りました。そこで毎回の課題が認証です。普段はPassportを利用するのですが、今回は[Keycloak][keycloak]という認証サーバーを試してみることにしました。

<!--more-->

## インストール

まずは簡単にDockerを使って起動できるか試してみます。v12.0.3の[docker-compose][docker-compose]ファイルをそのまま引用します。

```yaml
version: "3"

volumes:
  postgres_data:
    driver: local

services:
  postgres:
    image: postgres
    volumes:
      - postgres_data:/var/lib/postgresql/data
    environment:
      POSTGRES_DB: keycloak
      POSTGRES_USER: keycloak
      POSTGRES_PASSWORD: password
  keycloak:
    image: quay.io/keycloak/keycloak:latest
    environment:
      DB_VENDOR: POSTGRES
      DB_ADDR: postgres
      DB_DATABASE: keycloak
      DB_USER: keycloak
      DB_SCHEMA: public
      DB_PASSWORD: password
      KEYCLOAK_USER: admin
      KEYCLOAK_PASSWORD: Pa55w0rd
    ports:
      - "8080:8080"
    depends_on:
      - postgres
```

固有の設定は`KEYCLOAK_USER`と`KEYCLOAK_PASSWORD`ですね。サーバーを起動して[http://localhost:8080](http://localhost:8080)にアクセスしてログインできれば成功です。

## Keycloakの設定

無事に`admin`でログインできたら次に左上のほうに*Add realm*というボタンがあるのでクリックしてNameを入力すると固有の*領域*が作成できます。[Realms and users][realms-and-users]に記載されていますが、それぞれ**Master realm**と**Other realms**が存在していて、それぞれの領域ごとにアプリケーションやユーザーを分けて管理できるようです。そのため、新しくアプリケーションを作るごとに領域を作成する必要がありそうです。ここでは例として`anchovy-demo`で登録しました。

続いて左側にあるメニューから*Manage*の*Users*をクリックして*Add user*をクリックします。ここでは実際にログインに使用するユーザーを作成できました。デフォルトではパスワードを指定されないため*Credentials*でパスワードを指定しました。いろいろ設定項目がありますが必要最低限はこれで問題なさそうです。

## Reactとの連携

続いてReact側との連携です。[Secure React App with Keycloak][medium]という投稿を参考に作成します。

Keycloakのメニューにある*Configure*の*Clients*で*Create*をクリックして新規にクライアントを登録します。

それぞれ以下の設定をしました:

| 項目 | 値 |
|--|--|
| Client ID | `react-test` |
| Root URL | `http://localhost:3000` |
| Valid Redirect URIs | `http://localhost:3000/*` |
| Admin URL | `http://localhost:3000` |
| Web Origins | `http://localhost:3000` |

次にCRAでアプリケーションを作成してKeycloakのクライアントをインストールします。

    npm i keycloak-js

次に`src/index.js`を以下のように書き換えました:

```js
import React from "react";
import ReactDOM from "react-dom";
import "./index.css";
import App from "./App";
import * as Keycloak from "keycloak-js";

//keycloak init options
let initOptions = {
  url: "http://localhost:8080/auth",
  realm: "anchovy-demo",
  clientId: "react-test",
  onLoad: "login-required",
};

let keycloak = Keycloak(initOptions);

keycloak
  .init({ onLoad: initOptions.onLoad })
  .then((auth) => {
    if (!auth) {
      window.location.reload();
    } else {
      console.info("Authenticated");
    }

    //React Render
    ReactDOM.render(
      <React.StrictMode>
        <App />
      </React.StrictMode>,
      document.getElementById("root")
    );

    localStorage.setItem("react-token", keycloak.token);
    localStorage.setItem("react-refresh-token", keycloak.refreshToken);

    setTimeout(() => {
      keycloak
        .updateToken(70)
        .then((refreshed) => {
          if (refreshed) {
            console.debug("Token refreshed" + refreshed);
          } else {
            console.warn(
              "Token not refreshed, valid for " +
                Math.round(
                  keycloak.tokenParsed.exp +
                    keycloak.timeSkew -
                    new Date().getTime() / 1000
                ) +
                " seconds"
            );
          }
        })
        .catch(() => {
          console.error("Failed to refresh token");
        });
    }, 60000);
  })
  .catch(() => {
    console.error("Authenticated Failed");
  });
```

コードそのものの改善は必要ですが、すべてうまく行けばコンソール上に`Authenticated`の表示が出るはずです。

## まとめ

SPAなどではAuth0のような外部サービスが人気ですが、私はKeycloakのようなセルフホスティングの認証サーバーを使ってみようと思います。設定項目は多くて戸惑うものの、最小限の操作でもとりあえずは使えそうです。このまましばらく使ってみて問題なさそうであればRailsの認証もKeycloakに任せられたらいいなあと思いました。

[keycloak]: https://www.keycloak.org
[docker-compose]: https://github.com/keycloak/keycloak-containers/tree/12.0.3/docker-compose-examples
[realms-and-users]: https://www.keycloak.org/docs/latest/getting_started/index.html#realms-and-users
[medium]: https://medium.com/keycloak/secure-react-app-with-keycloak-4a65614f7be2
