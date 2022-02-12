---
title: "Rails: Draperの導入"
description: 今年の1月からRailsの仕事を始めました。Railsの保守開発は数年ぶりですが、それ以上にテストをしっかり書く文化の会社の業務に携わる機会は初めてなので興奮しています。興味本位や全く聞いたことのないgemも多いのですが、今回はDraperというgemを紹介します。
date: 2021-02-04T20:34:27+09:00
draft: true
tags:
  - rails
  - draper
layout: layouts/post.njk
---

今年の1月からRailsの仕事を始めました。Railsの保守開発は数年ぶりですが、それ以上にテストをしっかり書く文化の会社の業務に携わる機会は初めてなので興奮しています。興味本位や全く聞いたことのないgemも多いのですが、今回は[Draper](https://github.com/drapergem/draper)というgemを紹介します。

<!--more-->

Draperそのものは昔Railscastsなどで紹介されていたので存在そのものは知っていました。しかし具体的にどうやって導入するかのイメージが全くつかめませんでした。デザインパターンとかリファクタリングはなんにでも適用すれば良いわけではなく、モデルが肥大化してきてコードの見通しが悪くなってから初めて導入するものです。そもそものコードベースが少ないうちはあまり意味がないどころか、かえって冗長化を招きかねません。

以下のようなモデルがあるとします。

イベント用のモデル:

```ruby
# == Schema Information
#
# Table name: events
#
#  id           :integer          not null, primary key
#  name         :string
#  published_at :datetime
#
class Event < ApplicationRecord
  has_many :campaigns
end
```

キャンペーン用のモデル:

```ruby
# == Schema Information
#
# Table name: campaigns
#
#  id           :integer          not null, primary key
#  name         :string
#  published_at :datetime
#  event_id     :integer          not null
#
class Campaign < ApplicationRecord
  belongs_to :event
end
```

seedファイル:

```ruby
event1 = { name: 'テストイベント 第1弾',
           published_at: 2.weeks.ago }

event = Event.create(event1)

campaign = { event: event,
             name: 'テストキャンペーン 1',
             published_at: 10.days.ago }

event.campaigns.create(campagin)

event2 = { name: 'テストイベント 第2弾',
           published_at: 1.day.ago }

event = Event.create(event2)
```

コントローラー:

```ruby
class EventsController < ApplicationController
  def index
    @events = Event.all
  end
end
```

テンプレート:

```pug
h1 キャンペーン情報

table
  thead
    tr
      th ID
      th キャンペーン情報
      th キャンペーン日時

  tbody
    - @events.each do |event|
      tr
        td = event.id
        td = event.campaigns.first.try(:name) || \
             event.name
        td = (event.campaigns.first.try(:published_at) || \
              event.published_at).to_date.strftime('%Y.%m.%d')
```

あるイベントにキャンペーンがいくつか紐付いていて、キャンペーンが登録されている場合はそのキャンペーンの情報を表示します。もし表示できるキャンペーンがない場合は代わりに親のイベントの情報を表示するといったものです。これまでMVCをうたうアプリケーションはいくつか見てきましたが、Viewのロジックは極力少なくするのが理想です。最初このようなファイルを見てすぐに中身を理解できるとは限りません。実際こういったファイルを見てもとの挙動を書き換えてしまう内容のPRを出してしまいました。

かといってモデルにインスタンスメソッドを追加していけば追加していくほど今度はファットモデルという別の問題を誘発してしまいます。一体どうすれば良いのかというと、ここでDecoraterとかPresenterというパターンが出てきます。呼び方は些細な違いですが、RailsではDraperを使うとよいでしょう。

Draperをインストールすると、`EventDecorator`と`CampaignDecorator`が自動で追加されますがここでは`EventDecorator`を使います。

```ruby
class EventDecorator < Draper::Decorator
  DATE_FORMAT = '%Y.%m.%d'.freeze

  delegate :id, :campaigns

  def name
    campaign.name
  end

  def published_on
    campaign.published_at.to_date.strftime(DATE_FORMAT)
  end

  private

  def campaign
    campaigns.first.presence || object # event
  end
end
```

ここで`delegate`という単語が出てきて、Javaとかでちょろっとやったのであまり馴染みのない単語ですが、簡単にいうと`Event`のメソッドをそのまま`EventDecorator`でも使えるようにしますよというものです。

デフォルトで`delegate_all`なんて記述があるので何もしなければすべてのメソッドを自動的に委譲する設定なのですが、無限ループが発生しがちなので私は必要なものだけ`delegate`して、モデルそのものを参照するときは`object`を使う方が好ましいと思います。さすがにもう大変なので割愛しますが、これで何が嬉しいのかというとView側のテストを書くよりDraperを使ったほうが楽にテストが書けるということです。

```ruby
class EventsController < ApplicationController
  def index
    @events = Event.all.decorate
  end
end
```

使い方は簡単で`decorate`というメソッドを実行すると`Event`のインスタンスが自動で`EventDecorator`に置き換わってくれるのであとはテンプレート側を修正します。

```pug
h1 キャンペーン情報

table
  thead
    tr
      th ID
      th キャンペーン情報
      th キャンペーン日時

  tbody
    - @events.each do |event|
      tr
        td = event.id
        td = event.name
        td = event.published_on
```

テンプレートはこれ以上ないくらいに美しく書き換えることができました。

もしIDの値を変えたいと思ったらテンプレートは変更せずにデコレーター側を修正すればよいです。実際はもう少し複雑なファイルを修正しましたが、Viewのファイルはすぐに腐敗しがちなのでこういったパターンに沿っていけば複雑で一見誰も寄せ付けないようなRailsアプリケーションを楽にメンテナンスしていくことができるようになるかもしれません。

実際にまだ私のRailsアプリケーションでは導入に至っていないのですが、いずれテンプレートが肥大化してきたらその時は導入したいと思っています。
