---
title: "開発環境の準備"
---

## ✅ 必要なサービス

本書の目的の達成のために、以下のサービス契約・アカウントの用意が必要になります。

- **Okta**
- **GitHub**
  - Privateリポジトリで運用する場合は、GitHub Actionsの従量課金契約が必要です。
- **Amazon Web Service**
  - S3とIAMを使用します。

## ✅ Terraformのインストール

Terraformを既にインストール済みでしたら、こちらのセクションはスキップしてください。
（ちなみに、本書ではTerraform 1.0.0に準拠しています。）

:::message
インストールには[Homebrew](https://brew.sh/index_ja)を使用します。Homebrewが導入されていない場合は各自導入してください。
:::

```shell
brew install terraform
```

## ✅ AWS CLIのインストール・設定

Terraformでは、コードの内容とOktaの実データとの照らし合わせのために`*.tfstate`ファイルが生成されます。

通常はローカルに作成されGit管理を行いますが、今回はGitHub Actionsで自動反映を行うのでGit管理せずに常に最新を維持するためにAWS S3を使います。

そのため、AWSとやり取りするためにコマンドラインツールのインストール・設定を行います。

AWSCLIを既にインストール済みでしたら、こちらのセクションはスキップしてください。

### インストール

```shell
brew install awscli
```

### 設定

```shell
$ aws configure
AWS Access Key ID [None]: （自分のAWSアカウントのアクセスキー）
AWS Secret Access Key [None]: （自分のAWSアカウントのシークレットアクセスキー）
Default region name [None]: ap-northeast-1（東京リージョン）
Default output format [None]: json　　（空だとJSON）
```
