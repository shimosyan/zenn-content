---
title: "Terraform に Okta のコードを記述する - その 1"
---

## ⚙️ Terraform のディレクトリ構成

Terraform は基本的に同じディレクトリに属する`*.tf`ファイルは自動で認識します。

同じディレクトリであればファイルに定義したリソースを別のファイルで参照することもできます。

異なるディレクトリに書いた`*.tf`ファイルは module と呼ばれる仕組みを使うことでリソースの参照などができるようになりますが、この要素は事前の設計が大事なのでここではディレクトリは分けずすべて同じディレクトリ（リポジトリのルート階層）に`*.tf`ファイルを置くこととします。

## ⚙️ Terraform の準備

Terraform の基本設定を書くために`./main.tf`ファイルを作ります。

ここでは`terraform.tfstate`の保存先と Okta の Terraform Provider の指定、接続先の Okta 環境を指定します。

以下のように書いてみましょう。

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

トークンが発行されるので、コピーボタンを押してどこかに控えます。Git 管理下のファイルに記述しないようにだけ注意してください。

![トークン発行](https://storage.googleapis.com/zenn-user-upload/77127c11475ba66425664d3d.png)

### Terraform プロジェクトに Okta のトークンを登録

ターミナルの環境変数に Okta のトークンを保存します。変数名を`OKTA_API_TOKEN`にすると、Terraform が自動で読み取ってくれます。

```bash
export OKTA_API_TOKEN="XXXXXXXX"
```

## 🔨 Okta にグループを作成

Terraform を使って、空の Okta グループを 2 つ作成してみます。

`./group.tf`ファイルを新たに作り以下のように記述してみましょう。公式のリファレンスは[こちら](https://registry.terraform.io/providers/okta/okta/latest/docs/resources/group)です。

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

### Terraform の記法について

Terraform では Okta などのクラウドサービスにはリソース（`resource`）単位で設定していきます。

今回は Okta のグループが 2 つのため、リソースの定義が 2 つとなります。

`resource`が書かれている行には、今回の例ですと`okta_group`や`test_group_1`が書かれています。
リソースの構成は以下のようになります。

```bash
resource "設定の種類" "ID" {
  # 各プロパティ
}
```

「設定の種類」はクラウドサービスのどの設定かを指定します。`okta_group`は Okta のグループを定義であることを示します。例えば`okta_user`にすると Okta のユーザーを定義できます。

「ID」は Terraform 内で扱われるユニークな ID です。プロジェクト内で一意に指定する必要があります。

この`resource`ブロックの中のプロパティには、そのリソースの内容を記述します。種類ごとに記述すべき内容は異なるので、[公式リファレンス](https://registry.terraform.io/providers/okta/okta/latest/docs)や[公式サンプル](https://github.com/okta/terraform-provider-okta/tree/master/examples)を見ながら書きましょう。

今回のは`okta_group`空のグループなので、`name`と`description`を書きましたが、`user`を使用することで Okta ユーザーを指定することもできます。

### Okta に適用する

これを Okta に適用します。以下のコマンドを叩いて適用させます。

```bash
terraform plan
terraform apply
```

Okta のグループに追加されていることを確認できます。

![グループが追加された](https://storage.googleapis.com/zenn-user-upload/937ecdd838c45e9a69565a40.png)

既に Okta にグループがあり、それを Terraform の設定に紐付けたい場合は「Terraform に Okta のコードを記述する - その 2（執筆中）」をご覧ください。

## 参考資料

- <https://qiita.com/kohey18/items/38400d8c498baa0a0ed8>
- <https://registry.terraform.io/providers/okta/okta/latest/docs>
- <https://registry.terraform.io/providers/okta/okta/latest/docs/resources/group>
