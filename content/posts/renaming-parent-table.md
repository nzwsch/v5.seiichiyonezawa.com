---
title: "Rails: 既存の親テーブルをリネームするとき"
description: 2021年は新しいことに挑戦しようとまずはブログをHugoに乗り換えてみました。過去の投稿はこのドメインには持ってこずにリンクのアーカイブとして残しておく予定なので今後はここで発信していけたらなと思っています。
date: 2021-01-09
draft: false
tags:
  - rails
layout: layouts/post.njk
---

2021年は新しいことに挑戦しようとまずはブログをHugoに乗り換えてみました。過去の投稿はこのドメインには持ってこずにリンクのアーカイブとして残しておく予定なので今後はここで発信していけたらなと思っています。

<!--more-->

近況としては再びRailsでブログを作っています。今回はゲームを題材にしようと思っているのですが、そのなかに`platforms`というテーブルがあり、他のテーブルは接頭語として`game_`がついているのでこのテーブルも`game_platforms`に変更したいと思っています。純粋にテーブルを変更する方法は[Rails: How can I rename a database column in a Ruby on Rails migration?][1992019]に書かれてあるとおり`rename_column`というメソッドを使うだけです。

```ruby
class RenamePlatformsToGamePlatforms < ActiveRecord::Migration[6.1]
  def change
    rename_table :platforms, :game_platforms
  end
end
```

このまま終わるにはあまりにもいまさら過ぎるので今回はもう少し掘り下げてみましょう。それではこのモデルの定義をご覧ください:

```ruby
class Platform < ApplicationRecord
  has_many :game_titles
end
```

`game_titles`の親にあたるのですが、あくまでmigrationのファイルはSQLを実行してくれるだけです。当然この`Platform`というクラスとファイル名をリネームする作業が出てきます。といっても単純なリネームなのでそこまで難しくありません。続いて`schema.rb`を見てみましょう:

```ruby
ActiveRecord::Schema.define(version: 2021_01_09_022457) do
  create_table "game_platforms", force: :cascade do |t|
    t.string "name"
  end

  create_table "game_titles", force: :cascade do |t|
    t.string "name"
    t.bigint "platform_id"
    t.index ["platform_id"], name: "index_game_titles_on_platform_id"
  end

  add_foreign_key "game_titles", "game_platforms", column: "platform_id"
end
```

`game_titles`のカラムが`platform_id`のままで、さらに見落としがちなのは`add_foreign_key`に`column: "platform_id"`が使われているところです。私はテストにRailsのminitestと[Shoulda Matchers][shoulda-matchers]で以下のようなコードを書いていました。

```ruby
require "test_helper"

class GamePlatformTest < ActiveSupport::TestCase
  context "associations" do
    should have_many(:game_titles)
  end
end

class GameTitleTest < ActiveSupport::TestCase
  context "associations" do
    should belong_to(:game_platform).optional
  end
end
```

このテストを実行した時に出たエラーが:

```text
# Running:

F

Failure:
GamePlatformTest#test_: associations should have many game_titles.  [/usr/local/bundle/gems/shoulda-context-2.0.0/lib/shoulda/context/context.rb:62]:
Expected GamePlatform to have a has_many association called game_titles (GameTitle does not have a game_platform_id foreign key.)


rails test /usr/local/bundle/gems/shoulda-context-2.0.0/lib/shoulda/context/context.rb:62

F

Failure:
GameTitleTest#test_: associations should belong to game_platform optional: true.  [/usr/local/bundle/gems/shoulda-context-2.0.0/lib/shoulda/context/context.rb:62]:
Expected GameTitle to have a belongs_to association called game_platform (GameTitle does not have a game_platform_id foreign key.)


rails test /usr/local/bundle/gems/shoulda-context-2.0.0/lib/shoulda/context/context.rb:62
```

最初はリネーム作業だけで終わると思いきや、案外そういうわけにもいきませんでした。ただ親切にも *(GameTitle does not have a game_platform_id foreign key.)* というログが残ってくれていたおかげで一番最初のmigrationファイルを修正できました。

```diff
 class RenamePlatformsToGamePlatforms < ActiveRecord::Migration[6.1]
   def change
     rename_table :platforms, :game_platforms
+    rename_column :game_titles, :platform_id, :game_platform_id
   end
 end
```

これでテストが通るようになったのであとは残りのリネーム作業を進めていくだけです。

しばらくRailsの世界から離れていましたが、よく使うgemは今でもメンテされていますし、まだ調べれば大抵のことはすぐに検索結果に出てくるのでストレスなく開発するという点においては悪くないかなと思いました。

[1992019]: https://stackoverflow.com/q/1992019
[shoulda-matchers]: https://matchers.shoulda.io/
