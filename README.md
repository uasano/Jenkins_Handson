Jenkins_Handson
===============

# 第5回ゆるぎー Jenkinsハンズオン 資料

Jenkinsハンズオンの手引です。

やる事は以下のような内容を想定しています。

 1. Grails Webアプリのビルドと成果物としてwarファイルを保存する
 1. Webアプリのビルドが成功した時は、HerokuへデプロイするJobを実行する
 1. ビルドジョブとデプロイジョブのジョブ実行の流れをビルドパイプラインで見える化する
 1. デプロイジョブの前にアプリのテストを実行して、テストに失敗した時はデプロイジョブの実行しないようにする

# テンプレートJobの情報

「ここの部分が気になるので先にそこから試したい」という場合は、それぞれの個別のジョブを作っているので以下の手順でコピーしてください。

 1. `新規ジョブ作成`をクリック
 1. `ジョブ名`に、自分が作りたいジョブの名前を入力
 1. `既存のジョブをコピー`にチェック
 1. コピー元に下記の表を参照して、自分が複製したいジョブの名前を入力する。

ジョブ名 | やっていること
---------|---------------
template-build | Grailsアプリのビルドと成果物としてwarファイルを保存する
template-test-failed | Grailsアプリのテスト(テスト実行すると失敗)
template-test-success | Grailsアプリのテスト(テスト実行すると成功)
template-deploy | GrailsアプリのHerokuへのデプロイ。アプリ名はビルドパラメータで設定しています

# 演習用githubのリポジトリの状態

リポジトリURL:https://github.com/uasano/Jenkins_Handson

ブランチ名 | 状態
-----------|----------
phase1 | grailsアプリのひな形
phase2 | BookのCRUD機能が追加。テストを実行すると失敗する
phase3 | テストが成功する状態
master | phase3と同じ。README.mdが更新されている

# 各ジョブを作るヒント

## 共通

### ソースコードをGithubから取得する

Gitリポジトリからソースコードを取得するには、Jenkinsジョブで下記の設定を行います。

 1. `ソースコード管理`の`Git`にチェック
 1. `Repositories`の`Repository URL`に`git@github.com:uasano/Jenkins_Handson.git`を入力
 1. `Branches to build`でビルド対象としたいブランチ名を指定。
  * phase1の場合 `*/phase1`

### Jenkinsのジョブからシェルを実行する

Jenkinsのビルドでシェルを実行するには、Jenkinsジョブで下記の設定を行います。

 1. `ビルド`の`ビルド手順の追加`から`シェルの実行`を選択。(Windows上のサーバで動いている場合は`Windowsバッチコマンドの実行`)
 1. `シェルスクリプト`のテキストボックスに実行したいコマンドを入力

## ビルドジョブ

Grailsアプリケーションのwarファイルを生成するには以下のシェルコマンドを実行します。

```
grails war
```

このコマンドを実行すると、 `target/<app.name>-<app.version>.war` が生成されます。
`app.name`と`app.version`は`application.properties`ファイルで定義してあります。

ビルドで生成されたファイルを成果物として保存する時
 1. ジョブの設定で`ビルド後の処理`の`ビルド後の処理の追加`
 1. `成果物を保存`を選択
 1. 保存したいファイルの場所を指定

## テストジョブ

Grailsアプリケーションのtestファイルを生成するには以下のシェルコマンドを実行します。

```
grails test-app
```

このコマンドを実行すると、 `target/test-reports` ディレクトリの下にテスト結果が出力されます。

場所 | 出力されている形式
-----|------
test-repots直下 | JUnit xml形式
test-reports/html | HTML
test-reports/plain | テキストファイル

 * テストの実行結果の成功・失敗の推移を記録するには
  2. `ビルド後の処理`の`ビルド後の処理の追加`から`JUnitテスト結果の集計`を選択
  2. テスト結果XMLが出力されている場所を`テスト結果XML`に指定。
 * テストの実行結果レポートのHTMLをJenkinsのジョブから参照するには
  2. `ビルド後の処理`の`ビルド後の処理の追加`から`Publish HTML reports`を選択
  2. `追加`をクリック。
  2. `HTML directory to archive`にindex.htmlが出力されているフォルダを指定。

## デプロイジョブ

herokuにアプリをデプロイする時は、gitのチェックアウト設定に下記の設定を追加してください。

 * `Additional Behaviours`の`追加`をクリックし、`Check out to specific local branch`を選択
 * `Branch name`に`master`を入力

GrailsアプリケーションをHerokuにアップロードするには、以下のシェルコマンドを実行します。

```
if ! git ls-remote heroku; then
  heroku create [APP_NAME]
fi
git push heroku master
```

APP_NAMEを省略すると、herokuが自動的に他と重複しない値を設定します。
明示的に指定すると、重複していなければその名前で公開されます。
APP_NAMEは実行するシェルスクリプト上に直接記述しても良いですし、ビルドパラメータ等で渡しても良いと思います。

if で条件分岐しているのは、一度だけ実行すれば良いからです。

# ジョブとジョブを関連付ける

## 下流ジョブの設定

あるジョブが成功した時は、その後に別のジョブを実行させる設定。

 1. `ビルド後の処理`の`ビルド後の処理の追加`から`他のプロジェクトのビルド`を選択
 1. `ビルドするプロジェクト`に実行したいジョブの名前を入力
 1. `Trigger only if build is stable` にチェック。

## 上流ジョブの設定

あるジョブを実行する前に、別のビルドを実行させる設定。

 1. `ビルド・トリガ`の`他プロジェクトのビルド後にビルド`にチェック
 1. `プロジェクト名`に先行して実行させたいジョブの名前を設定

# ビルドの上流〜下流の流れを見える化する

 1. Jenkinsのホーム画面のジョブ一覧のタブの`+`をクリック
 1. `ビュー名`に任意の名前を設定
 1. `Build Pipeline View`を選択
 1. `OK`をクリック
 1. `Layout`の`Select Initial Job`に一連のビルドの起点になるジョブを選択
 1. `No Of Displayed Builds`は`5`を選択
 1. `保存`をクリック
