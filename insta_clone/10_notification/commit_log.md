## callbackの設定

通知は、コメントされた時・いいねされた時・フォローされた時がトリガとなる

つまり、Notificationオブジェクトは、ポリモーフィック関連したオブジェクトであるCommentオブジェクト, Likeオブジェクト, Relationshipオブジェクトが生成された際に連動して生成されるようにする必要がある

各コントローラーのcreateアクション内にそれぞれ記述する方法もあるが、ファットコントローラーになりやすくまた、重複が発生しDRYでは泣くなるため、今回は各モデルのコールバックで実装することにする

今回使用するコールバックメソッドは`after_create_commit`。これは`after_commit`のエイリアスであり、レコードが保存・削除される度に、トランザクションがデータベースにcommitされた後で呼び出されるコールバックメソッドである(今回はcreateに限定)

[ちなみにこの`commit`とは、制約に引っかからずに正常にデータの作成・更新・削除が実行され、データベースの変更が反映されることをいう](https://qiita.com/codeship_tech/items/429f2ad383d74133ea95)

よって、`after_create_commit`コールバックメソッドは正常なデータへの変更が反映された後に実行されるということ

名前を`create_notifications`として、Comment, Like, Relationship modelにそれぞれ定義する

```
# models/Comment

after_create_commit :create_notifications

private

def create_notifications
 Notification.create(noticeable: self, user: post.user)
end
```

```
# models/Like

after_create_commit :create_notifications

private

def create_notifications
 Notification.create(noticeable: self, user: post.user)
end
```

```
# models/Relationship

after_create_commit :create_notifications

private

def create_notifications
 Notification.create(noticeable: self, user: followed)
end
```

復習(引数)

`noticeable`...ポリモーフィック関連名

`user`...referencesしてるから

<br>

## scopeの定義　通知を10件表示させるため

指定した件数の通知を新しいものから取得するためにscoepを定義する。models/userで定義したものとほとんど同じ

```
# models/notification

scope :recent, -> (count) { order(create_at: :desc).limit(count) }
```

<br>

## enumの定義

その通知が既読済みなのか未読のままなのか`read?`, `unread?`などのメソッドで確認できるようにするためにenumを定義する

```
# models/notification

enum read: { unread: false, read: true }
```

readカラムは`default: false`を指定しているため、値がfalseである`unread`を左に定義する

また、以上のenum定義により、`read!`メソッドも使えるようになり、これだけでreadの値をtrueに変更することができる。(.update(read: true)と同じ挙動)

<br>

## ダックタイピングを使ってパーシャルを取得するメソッドを定義

```
# models/notification

def call_appropiate_partial
 case self.noticeable_type
 when "Comment"
   "commented_to_own_post"
 when "Like"
   "liked_to_own_post"
 when "Relationship"
   "followed_me"
 end
end
```

`noticeable_type`には、関連先のクラスが格納されることを思い出そう

次に、それぞれのパーシャルを作成する

<br>

## パーシャルの作成

### shared/_header

ハート部分をクリックすることで、未読の通知一覧がドロップダウンで表示されるようにする

ハートのアイコンの右上に未読の通知数を表示させる

```
nav.navbar.navbar-expand-lg.navbar-light.bg^white
 = link_to 'InstaClone', root_path, class: 'navbar-brand'
 button.navbar-toggeler aria-controls="navbarTogglerDemo2" aria-expanded="false" aria-label=("Toggle navigation") data-target="#navbarTogglerDemo2" data-togglel="collapse" type="button"
   span.navbar-toggler-icon
 #navbarogglerDemo2.collapse.navbar-collapse
   = render 'posts/search_form', search_form: @search_form
   ul.navbar-nav.mt-2.mt-lg-0
     li.nav-item
       = link_to new_post_path, class: "nav-link" do
         = icon 'far', 'image', class: 'fa-lg'
     li.nav-item
       .dropdown
         a#dropdownMenuButton.nav-link.position-relative href="#" data-toggle="dropdown" aria-expanded="false" aria-haspopup="true"
           = icon 'far', 'heart', class: "fa-lg"
           = render 'shared/unread_badge'
         #header-notifications.dropdown-menu.dropdown-menu-right.m-0.p-0 aira-labelledby="dropdownMenuButton"
           = render 'shared/header_notifications'
```

[今回使用したBootstrapについてはこちらにまとめました]()

### shared/_unread_badge

このパーシャルは、未読の通知数をハートアイコンの右上に表示させるためのパーシャル。もし未読の通知があれば表示させるため

```
- if current_user.notifications.unread.count > 0
 span.badge.badge-warning.navbar-badge.position-absolute style="top: 0; right: 0;"
   = current_user.notifications.unread.count
```

`unread`メソッドはenumで定義したから使えることに注意

### shared/_header_notifications

このパーシャルは、ハートのアイコンをクリックした際に表示させる通知一覧画面

```
- if current_user.notifications.present?
 - current_user.notifications.recent(10).each do |notification|
   = render "shared/#{notification.call_appropiate_partial}", notification: notification
 - if current_user.notifications.count > 10
   = link_to "全て見る", mypage_notifications_path, class: "dropdown-item justify-content-center"
- else
 .dropdown-item
   | お知らせはありません
```

models/notificaitonで定義した`call_appropiate_partial`がここで活きてくる！

`#{notificaiton.call_appropiate_partial}`とすることで、「noticeable_typeの中身が`Comment`であれば、`commented_to_own_post`をrenderさせる・`Like`であれば、`liked_to_own_post`をrenderさせる・`Relationship`であれば、`followed_me`をrenderさせる」という複雑な処理をダックタイピングで単純化させることができる


<br>

### shared/_commented_to_own_post

このパーシャルは、コメントされた時に作成される通知を表示させるもの。つまり、「XXX(リンク)があなたの投稿にコメントしました」という通知を表示させるということ

```
= link_to notification_read_path(notification), class: "dropdown-item border-bottom #{'read' if notification.read?}", method: :post do
 = image_tag notification.noticeable.user.avatar.url, class: 'rounded-circle mr-1', size: "30x30"
 object
   = link_to notification.noticeable.user.name, user_path(notification.noticeable.user)
 | があなたの
 object
   = link_to "投稿", post_path(notification.noticeable.post)
 | に
 object
   = link_to "コメント", post_path(notification.noticeable.post, anchoe: "commet-#{notification.noticeable.id}")
 | しました
 .ml-auto
   = l notification.create_at, format: :short
```

この通知をクリックすると未読から既読にする処理を行いため、readというresourceのcreateアクションにアクセスするよう指定。また、既読であれば薄暗い背景色にするため、`read`クラスを付与。(ここの書き方なんか好きです。)

`object`タグ...画像や動画、音声、プラグインデータ、HTML文書などの様々な形式を持つデータを文書に埋め込むための汎用的なタグ。また、改行を含めずにインラインで表示することができるため今回の通知表現にぴったりなタグ

### lメソッド

`l(localize)`メソッド...日本人が読みやすい日時フォーマットにしてくれるメソッド。ただし、諸設定が必要

config/application.rbのdefault_localeを`:ja`にする(今回のアプリでは既に設定済み)

config/locales/ja.ymlの設定

```
# config/locales/ja.ymlの設定

ja:
 time:
   formats:
     default: "%Y/%m/%d %H:%M:%S"
     short: "%m/%d %H:%M"
```

デフォルトの書式を、2021/05/10 09:12:24 みたいな馴染みある形式に変更する

shortには、formatオプションにshortが指定されたときに表示する形式を記述する。05/10 09:24 のようになる

[伊藤先生！！](https://qiita.com/jnchito/items/831654253fb8a958ec25)

<br>

### shared/_liked_to_own_post

このパーシャルでは、いいねされた際に届く通知を表現する

```
= link_to notification_read_path(notification),  class: "dropdown-item border-bottom #{'read' if notification.read?}", method: :post do
 = image_tag notification.noticeable.user.avatar.url, class: "rounded-circle mr-1", size: '30x30'
 object
   = link_to notification.noticeable.user.name, user_path(notification.noticeable.user)
 |　があなたの
 object
   = link_to "投稿", post_path(notificaiton.noticeable.post)
 | にいいねしました
 .ml-auto
   = l notificaton.created_at, format: :short
```

<br>

### shared/_followed_me

このパーシャルでは、フォローされた際に届く通知を表現する

```
= link_to notification_read_path(notification), class: "dropdown-item border-bottom #{'read' if notification.read?}", method: :post do
 = image_tag notification.notifiable.follower.avatar.url, class: 'rounded-circle mr-1', size: '30x30'
 object
   = link_to notification.noticeable.followed.name, user_path(notification.noticeable.followed)
 | があなたをフォローしました
 .ml-auto
   = l notification.create_at, format: :short
```

`followed`メソッドが使えるのは、models/relationshipに定義したからで、フォローしてきた相手の情報を取得することができる

<br>

## reads_controller

ヘッダーに表示されているハートのアイコンを押すと、通知一覧が表示され、特定の通知を押すことで既読にし、通知内容に対応するリンクへ飛ばすパーシャルの作成が終わったため、次に既読にする(notification_read_path(notification)に対応する)処理・アクションを定義していく。

```
before_action :require_login, only: [:create]

def create
 @notification = current_user.notifications.find(params[:notification_id])
 @notification.read! if @notification.unread?
 redirect_to @notification.appropiate_path
end
```

他のユーザーからのnotificationsの更新を防ぐため`current_user.notifications`とする

`read!`メソッドはenumで定義したから使えるメソッドで、未読であれば既読にするロジック(update(read: true))

次に、`.appropiate_path`メソッドを使えるようにするために、models/notificationを編集する

## models/notification

```
include Rails.application.routes.url_helpers


def call_appropiate_partial
 case self.notifiable_type
 when "Comment"
   "commented_to_own_post"
 when "Like"
   "liked_to_own_post"
 when "Relationship"
   "followed_me"
 end
end

def appropiate_path
 case self.noticeable_type
 when "Commnet"
   post_path(self.noticeable.post, anchor: "comment-#{noticeable.id}")
 when "Like"
   post_path(self.noticeable.post)
 when "Relationship"
   user_path(self.noticeable.followed)
end
```

まずは、このクラス内のクラスメソッドでURLヘルパーパスが使えるようにするために、`Rails.application.routes.url_helper`を`include`する

コメントされた場合といいねされた場合はposts/showへ、フォローされた場合はusers/showへリダイレクトさせる必要があるため独自メソッドを実装する

<br>

## headerとmypageのSCSS

新規で`stylesheets/_header.scss`を作成し、`application.scss`にimportする

```
# stylesheets/_header

#header-notifications {
 width: 400px;
 .dropdown-item {
   max-width: initial;
   font-size: 12px;
 }

 .read {
   background: #f1f1f1;
 }
}
```

```
# stylesheets/application.scss

@import 'header'
```

```
# mypage.scss

@import 'header'

.read {
 background: #f1f1f1;
}
```

既読された通知の背景色の変更。`read`クラスは、各パーシャルにて付与するか判断している

<br>

## mypage配下の通知一覧を作成

通知一覧へのリンクとページネーションを追加する

```
# mypage/notifications_controller

def index
 @notifications = current_user.notifications.order(created_at: :desc).page(params[:page]).per(10)
end
```

```
# mypage/notifications/index

- if @notifications.present?
 - @notifications.each do |notification|
   = render "shared/#{notification.call_appropiate_partial}", notification: notification
 = paginate @notifications

- else
 .text-center
   | お知らせはありません
```

再確認となるが、`= render "shared/#{notification.call_appropiate_partial}"`とすることで、`commented_to_own_post`なのか`liked_to_own_post`なのか`followed_me`なのかに対応できる

```
# shared/_sidebar

li
 = link_to '通知一覧', mypage_notifications_path
 hr
```

<br>

## call_appropiate_partial / appropiate_pathメソッドのリファクタリング

ポリモーフィック関連を使った場合、caseでの分岐はアンチパターンとのこと。よって最近勉強したダックタイピングを使ってリファクタリングを行っていく

現在の状況を再確認

```
# models/notification

include Rails.application.routes.url_helpers

def call_appropiate_partial
 case self.notifiable_type
 when "Comment"
   "commented_to_own_post"
 when "Like"
   "liked_to_own_post"
 when "Relationship"
   "followed_me"
 end
end

def appropiate_path
 case self.notifiable_type
 when "Comment"
   post_path(self.notifiable.post, anchor: "comment-#{notifiable.id}")
 when "Like"
   post_path(self.notifiable.post)
 when "Relationship"
   user_path(self.notifiable.follower)
 end
end
```

リファクタリングの流れとしては、Comment, Like, Relationshipモデルそれぞれに新たな独自メソッドを定義し、Notificationモデルから呼び出す感じ

```
# models/notification

# URLヘルパーを使う必要がなくなったため
# include Rails.application.routes.url_helpersは不要になる

def call_appropiate_partial
 noticeable.partial_name
enb

def appropiate_path
 noticieable.resource_path
end
```

```
# models/comment

# 逆にこれら3つのモデル内でURLヘルパーを使えるようにしたいので
include Rails.application.routes.url_helpers

def partial_name
 "commented_to_own_post"
end

def resource_path
 post_path(post, anchor: "comment-#{id}") # commentクラス内であることを意識
end
```

```
# models/like

include Rails.application.routes.url_helpers

def partial_name
 "liked_to_own_post"
end

def resource_path
 post_path(post)
end
```

```
# models/relationship

include Rails.application.routes.url_helpers

def partial_name
 "followed_me"
end

def resource_path
 user_path(followed)
end
```

<br>

## 共通化できるものを`concerns`に切り分ける

Comment, Like, Relationship3つのモデル内で重複している箇所がちらほらとあるため、DRYの原則に従いモジュール化(共通化)しようと思う。

- include Rails.application.routes/url_helpres
- has_many :notifications, as: :noticeable
- after_create_commit :create_notifications

### module化する際に必要なこと

モジュール内に`ActiveSupport::Concern`をextendする

module内で定義したメソッドを使うためには、そのメソッドを利用したいクラス内でそのメソッドをincludeする必要がある。しかし、includeできるのはインスタンスメソッドのみなので`ActiveSupport::Concern`をextendする必要がある。

ActiveSupport::Concern`をextendすることによって、コールバックメソッドや一般的なクラスメソッドが使えるようになる認識

### raise NotImplementedError

実装されていないメソッドが呼び出されたときに、例外処理が発生するように設定した方が良いだろう。その際に用いるのが`NotImplementedError`。この例外は親クラス(今回で言えば、module)内に定義する。

<br>

以上のことを踏まえて、共通化するすると以下のようなファイル構成となる

```
# models/concerns/noticeable.rb

module Noticeable
 exend ActivSupport::Concern

 include Rails.application.routes.url_helpers

 include do
   has_many :notifications, as: :noticeable
   after_create_commit :create_notifications
 end

 def partial_name
   raise NotImplementedError
 end

 def resource_path
   raise NotImplementedError
 end

 def notification_user
   raise NotImplementedError
 end

 private
 def craete_notifications
   Notification.create(noticeable: self, user: notification_user)
 end
end
```

```
class Notification < ApplicationRecord
 belongs_to :notifications, polymorhic: true
 belongs_to :user

 scope :recent, -> (count) { order(created_at: :desc).limit(count) }

 enum read: { unread: false, read: true }

 def call_appropiate_partial
   noticeable.partial_name
 end

 def appropiate_path
   noticeable.resource_path
 end
end
```

```
class Comment < ApplicationRecord
 belongs_to :user
 belongs_to :post

 include Noticeable

 validates :body, presence: true, length: { maximum: 1000 }

 def partial_name
   "commented_to_own_post"
 end

 def resource_path
   post_path(post, anchor: "comment-#{id}")
 end

 def notification_user
   post.user
 end
end
```

```
class Like < ApplicationRecord
 belongs_to :user
 belongs_to :post

 include Noticeable

 validates :user_id, uniqueness: { scope: :post_id }

 def partial_name
   "liked_to_own_post"
 end

 def resource_path
   post_path(post)
 end

 def notification_user
   post.user
 end
end
```

```
class Relationship < ApplicationRecord
 belongs_to :followed, class_name: "User"
 belongs_to :follower, class_name: "User"

 include Noticeable

 validates :followed_id, presence: true
 validates :follower_id, presence: true
 validates :follower_id, uniqueness: { scope: :followed_id}

 def partial_name
   "followed_me"
 end

 def resource_path
   user_path(followed)
 end

 def notification_user
   followed
 end
end
```
