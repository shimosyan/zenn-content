---
title: "GitHub Actions を使って Okta に自動で展開する"
---

ここでは編集されたコードを GitHub Actions を使って Okta に自動で適用する仕組みを整えていきます。

## ❓ そもそも GitHub Actions とは

<https://github.co.jp/features/actions>

主に CI/CD を行うために用意されたワークフローです。

各種コマンドやスクリプトを、ある条件をきっかけに実行し、様々な処理を行わせることができます。

## ⚙️ 自動で Okta に展開するために必要な処理

まずは、GitHub Actions に何を行わせるかをまとめます。

- いつ
  - `main`ブランチに Push されたら
  （`main`以外のブランチは開発ブランチとするため処理させない）
- 何を行うか
  - Okta の API トークンを用意する
  - AWS S3 を使える状態にする
  - Terraform のコードが正しいことを確認する
  - Terraform の内容を Okta に適用する

最低限必要なものをまとめてみました。

「成功したらチャットに通知する」や「失敗したらアラートを発火する」などがあるとより良いかもしれませんが今回は割愛します。
(チャット通知はググるといっぱい出てきます）

## ⚙️ クレデンシャルの設定

今回の GitHub Actions では Okta の API トークンと AWS のアクセスキーが必要になるため、これらのクレデンシャル情報の取得と GitHub Actions への登録が必要になります。

### Okta API トークンの登録

Okta のトークンは前ページで使用したものでもいいですし、新しく用意でも構わないので 1 つ用意します。

用意できたら、GitHub リポジトリのページを開いて Settings → Secret → New Repository Secret を選択。

![Secret登録画面を開く](https://storage.googleapis.com/zenn-user-upload/c917f0fce9c3b5ef109166f5.png)

トークンの入力画面になるので、以下のように入力して Add secret を選択します。

- Name：`OKTA_API_TOKEN`
- Value：`Okta の API トークン`

![OktaのAPIトークンを入力](https://storage.googleapis.com/zenn-user-upload/9895b171e2d1b04545ad8698.png)

### AWS クレデンシャル情報の取得・登録

GitHub Actions からも AWS S3 にアクセスさせる必要があるので、専用の AWS IAM ユーザーを作成します。

作成オプションでは、Web コンソールにアクセスする必要もないので「プログラムによるアクセス」のみの選択で構いません。

![AWS IAMユーザーのオプション設定](https://storage.googleapis.com/zenn-user-upload/c8016becee7e212856e0081c.png)

IAM ユーザーに割り当てる権限には、`terraform.tfstate`だけを読み書きできるこちらのポリシーを割り当てます。

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "s3:ListBucket",
      "Resource": "arn:aws:s3:::terraform"
    },
    {
      "Effect": "Allow",
      "Action": ["s3:GetObject", "s3:PutObject"],
      "Resource": "arn:aws:s3:::terraform/okta/*"
    }
  ]
}
```

IAM ユーザーの作成ができたら、作成時に発行されるアクセスキー ID とシークレットアクセスキーを Okta の　API トークンと同様に GitHub に登録します。

Secret 名は以下のようにしてください。

- アクセスキー ID：`AWS_ACCESS_KEY_ID`
- シークレットアクセスキー：`AWS_SECRET_ACCESS_KEY`

![GitHub Secretの保存状況](https://storage.googleapis.com/zenn-user-upload/bfe27040dd436da192222e71.png)

その他、必要あればチャットツールなどの API キーもここで登録できます。

## 🔨 自動デプロイ用 GitHub Actions Workflow の実装

以下の Workflow ファイルを作成し、GitHub に push します。

ブランチが `main` ブランチであれば、GitHub Actions が起動し、Okta へ Terraform コードの適用が行われます。

自動適用したくないときは別で作業ブランチを切り、そちらで実装を行いましょう。

作業が終わったら、 Pull Request から `main` ブランチへマージすることで Workflow が実行されます。

```yaml:./.github/workflows/deploy.yaml
name: TerraformDeploy

on:
  push:
    branches:
      - main

jobs:
  terraform:
    name: TerraformDeploy
    runs-on: ubuntu-latest
    env:
      OKTA_API_TOKEN: ${{ secrets.OKTA_API_TOKEN }}

    defaults:
      run:
        shell: bash

    steps:
    - name: Checkout
      uses: actions/checkout@v2

    - name: Configure AWS credentials
      uses: aws-actions/configure-aws-credentials@v1
      with:
        aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
        aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
        aws-region: ap-northeast-1

    - name: Setup Terraform
      uses: hashicorp/setup-terraform@v1
      with:
        terraform_version: 1.0.0

    - name: Init Terraform
      run: terraform init

    - name: Test Terraform
      run: |
        terraform fmt -recursive -check
        terraform validate
        terraform plan

    - name: Deploy Terraform
      id: terraform_deploy
      run: terraform apply -auto-approve # -auto-approve オプションの付与で、ユーザーに確認せずに適用を実施する

# 以下は Workflow の成否をチャットなどに通知を行うときの処理です。もし必要あったら使ってください。
    - name: Notify Success Message
      id: notify_success_message
      if: success() && steps.terraform_deploy.outcome == 'success'
      run: # デプロイ成功時の通知処理

    - name: Notify Failure Message
      if: failure() && steps.notify_success_message.outcome != 'failure'
      run: # デプロイ失敗時の通知処理
```

## 📄 参考資料

- <https://qiita.com/keitakn/items/db2e9c68019594885ac4>
- <https://qiita.com/t0yohei/items/9285e61f8358de7f60a7>
- <https://docs.github.com/ja/actions/reference/workflow-syntax-for-github-actions>
