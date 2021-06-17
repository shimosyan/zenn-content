---
title: "開発環境の準備"
---

## ✅ 必要なサービス

本書の目的の達成のために、以下のサービス契約・アカウントの用意が必要になります。

- **Okta**
- **GitHub**
  - Private リポジトリで運用する場合は、GitHub Actions の従量課金契約が必要です。
- **Amazon Web Service**
  - S3 と IAM を使用します。

## ✅ Terraformのインストール

Terraform を既にインストール済みでしたら、こちらのセクションはスキップしてください。
（ちなみに、本書では Terraform 1.0.0 に準拠しています）

:::message
インストールには[Homebrew](https://brew.sh/index_ja)を使用します。Homebrew が導入されていない場合は各自導入してください。
:::

```shell
brew install terraform
```

## ✅ AWS CLIのインストール・設定

Terraform では、コードの内容と Okta の実データとの照らし合わせのために`terraform.tfstate`ファイルが生成されます。

通常はローカルに作成され Git 管理を行いますが、今回の目的のように GitHub Actions を使って自動反映を行うと最新の`terraform.tfstate`が Git 管理にできません。
常に最新の`terraform.tfstate`を保持するために今回は保存先に AWS S3 を使います。

そのため、AWS とやり取りするためにコマンドラインツールのインストール・設定を行います。

AWSCLI を既にインストール済みでしたら、こちらのセクションはスキップしてください。

### インストール

```shell
brew install awscli
```

### 設定

```shell
$ aws configure
AWS Access Key ID [None]: （自分の AWS アカウントのアクセスキー）
AWS Secret Access Key [None]: （自分の AWS アカウントのシークレットアクセスキー）
Default region name [None]: ap-northeast-1（東京リージョン）
Default output format [None]: json　　（空だと JSON）
```
