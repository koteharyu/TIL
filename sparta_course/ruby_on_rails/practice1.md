# リクエストからレスポンスまでの流れについて(GET編)

<br>

routes.rbにて`resources :users`が定義してあると仮定する。HTTPメソッドのGETメソッドにて/usersへリクエストが送られた場合、まず、コントローラー層にあるrouted.rbにて、usersコントローラーのindexアクションで処理するように指定する。

次に、indexアクションで使用する全ユーザーデータをデータベース内にある該当モデルのテーブルから取得するためのSQLを発行し、インスタンス変数`@users`へ格納する

```
# users_controller.rb

def index
  @users = User.all
end
```



indexアクションに対応するビューテンプレートを使ってHTMLなどを生成する

```
- @users.each do |user|
  = user.name
```
eachメソッドが呼ばれて初めて、データの検索が行われる

最後にコントローラーがレスポンスを作成し、Webサーバーを通じてクライアントへレスポンスを返す。クライアントであるブラウザがコンテンツを解釈し、ユーザーの一覧ページを表示させる

[![Image from Gyazo](https://i.gyazo.com/d877a6b5293cc854104380323007b89d.png)](https://gyazo.com/d877a6b5293cc854104380323007b89d)

# リクエストからレスポンスまでの流れについて(POST編)

<br>
## 新規作成ページを表示する

 /tasks/new というURLで新規作成ページへとリクエストを送り、サーバがそのリクエストを受け取る

サーバー側は、リクエストに対応する処理をルーティングから、tasksコントローラーのnewアクションだと指定し、実行する。

newアクションで、空のTaskオブジェクトを生成し、インスタンス変数`@task`へ代入する

```
# tasks_controller.rb
def new
  @task = Task.new
end
```

その処理を元に、tasks/new.html.slimがレンダリングされる

<br>

## タスク名(name)を入力する

<br>

form_withメソッドを使って、フォームをtasks/new.html.slim内に作成。またbootstrapを適用している

```
# tasks/new.html.slim

= form_with model: @task, local: true do |f|
  .form-group
    = f.label :name
    = f.text_field :name, class: "form-control"
  = f.submit nil, class: "btn btn-primary"
```

以下のような作成されるHTMLが生成される

``` 
<form action="/tasks" accept-charset="UTF-8" method="post">
  <input name="utf8" type="hidden" value="✔︎">
  <input type="hidden" name="authenticity_token" value="sajfosjf...">
  <div class="form-group">
    <label for="task_name">名称</label>
    <input class="form-control" type="text" name="task[name]">
  </div>
  <input type="submit" name="commit" value="登録する" class="btn btn-primary" data-disable-with="登録する">
</form>
```

form_withメソッドで生成されたHTML内の`action="/tasks"`は、送信先のURL、`method="post"`はHTTPメソッドのPOSTメソッドを使うことを指定している。

<br>

## 作成ボタンを押す

<br>

作成ボタンが押されると、`/tasks`URLへPOSTメソッドが送信され、routes.rb内の記述を元に、tasksコントローラーのcreateアクションが実行される

```
# tasks_controller.rb

def create
  @task = Task.new(task_params)
  if @task.save
    redirect_to @task
  else
    render :new
end

private
  def task_params
    params.require(:task).permit(:name)
  end
```

`task_params`というプライベートメソッドを作成し、対象とする属性を制限。

inputタグ内の`name="task[name]"`が重要で、`task`が`require(:task)`、`[name]`が`permit(:name)`に対応している

[![Image from Gyazo](https://i.gyazo.com/172edb92e8d223e5482058eb213a3832.png)](https://gyazo.com/172edb92e8d223e5482058eb213a3832)

`task_params`を引数にとったTaskオブジェクトを生成し、インスタンス変数`@task`に格納

<br>

## タスクの詳細画面に遷移する

<br>

@taskの保存に成功したら、そのタスクの詳細ページへリダイレクトを行う。他にも`redirect_to task_path(@task)`と書ける。つまり、生成されるURLは`/tasks/:id`トいうこと

[![Image from Gyazo](https://i.gyazo.com/854f1d8e36783deb8bdef0416d3511d2.png)](https://gyazo.com/854f1d8e36783deb8bdef0416d3511d2)


<br>

よろしくお願いします

