---
title: "Terraform に Okta のコードを記述する - その1"
---

## ⚙️ Terraform のディレクトリ構成

Terraform は基本的に同じディレクトリに属する`*.tf`ファイルは自動で認識します。

同じディレクトリであればファイルに定義したリソースを別のファイルで参照することもできます。

異なるディレクトリに書いた`*.tf`ファイルは module と呼ばれる仕組みを使うことでリソースの参照などができるようになりますが、この要素は事前の設計が大事なのでここではディレクトリは分けずすべて同じディレクトリ（リポジトリのルート階層）に`*.tf`ファイルを置くこととします。

## ⚙️ Terraform の準備

Terraform の基本設定を書きます。

ここでは`terraform.tfstate`の保存先と Okta の Terraform Provider の指定、接続先の Okta 環境を指定します。

```bash:./main.tf
terraform {
  required_version = ">= 1.0.0"

  # terraform.tfstate の保存先に、前のページで作成した AWS S3 バケットを指定します。
  backend "s3" {
    bucket  = "terraform"
    region  = "ap-northeast-1"
    key     = "okta/terraform.tfstate"
    encrypt = true
  }

  required_providers {
    okta = {
      source  = "okta/okta"
      version = "~> 3.11.1"
    }
  }
}

# Configure the Okta Provider
provider "okta" {
  org_name = "xxxxxxxx" # Okta のテナント名(サブドメイン)を入力します。 ex) xxxxxxxx.okta.com
  base_url = "okta.com"
  # api_token = "xxxx"  # ドキュメントでは API トークンの指定が書いてありますが、このファイルは Git 管理されるためトークンは記述しません。
}
```

## ⚙️ Okta の API トークンを取得

Okta に Terraform から設定を反映させるには Okta の API トークンが必要です。

Okta の管理画面に入って Security → API → Create Token を選択。

![API画面のCreate Tokenボタン](https://storage.googleapis.com/zenn-user-upload/8d4ed799a7cfaff9dda2f638.png)

トークン名に`Terraform`と入力して Create Token をクリックします。

![トークンを作成](https://storage.googleapis.com/zenn-user-upload/4b587b135b5504449e571a73.png)

トークンが発行されるので、コピーボタンを押してどこかに控えます。Git管理下のファイルに記述しないようにだけ注意してください。

![トークン発行](https://storage.googleapis.com/zenn-user-upload/77127c11475ba66425664d3d.png)

### Terraform プロジェクトに Okta のトークンを登録

ターミナルの環境変数に Okta のトークンを保存します。変数名を`OKTA_API_TOKEN`にすると、Terraform が自動で読み取ってくれます。

```bash
export OKTA_API_TOKEN="XXXXXXXX"
```

## 🔨 Okta にグループを作成

Terraform で Okta のグループを2つ作成してみます。

```bash:./group.tf
resource "okta_group" "test_group_1" {
  name        = "テストグループ1"
  description = "これはテスト用のグループです"
}

resource "okta_group" "test_group_2" {
  name        = "テストグループ2"
  description = "これはテスト用のグループです"
}
```

これを Okta に適用します。以下のコマンドを叩いて適用させます。

```bash
terraform plan
terraform apply
```

Okta のグループに追加されていることを確認できます。

![グループが追加された](https://storage.googleapis.com/zenn-user-upload/937ecdd838c45e9a69565a40.png)

既に Okta にグループがあり、それを Terraform の設定に紐付けたい場合は「Terraform に Okta のコードを記述する - その2（記述中）」をご覧ください。

## 参考資料

- <https://qiita.com/kohey18/items/38400d8c498baa0a0ed8>
- <https://registry.terraform.io/providers/okta/okta/latest/docs>
- <https://registry.terraform.io/providers/okta/okta/latest/docs/resources/group>
