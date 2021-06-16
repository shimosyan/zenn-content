---
title: "プロジェクトの作成"
---

## ⚙️ 初期設定

Terraform で実装するには準備が必要のためいくつか設定します。

### リポジトリの初期化

```shell
git init okta-terraform
cd okta-terraform
```

### .gitignore の作成

```gitignore:./.gitignore
.DS_Store

# terraform
*.tfstate
*.tfstate.*
*.tfvars
**/.terraform/*
```

## ⚙️ AWS S3 バケットの用意

Terraform の`terraform.tfstate`ファイルを設置する S3 バケットを作成します。

AWS のコンソールにサインインして S3 の管理画面から「バケットを作成」を選択。

![AWS S3のバケット作成ボタンを押す](https://storage.googleapis.com/zenn-user-upload/92ce5371d96634ee9579ce21.png)

バケットは以下の設定で作成します。それ以外はデフォルトのままで構いません。

- バケット名: `terraform`
- バケットのバージョニング: `有効にする`

![バケット名の設定](https://storage.googleapis.com/zenn-user-upload/fbc5edfded1e2bfa609c55ee.png)

![バージョニングの設定](https://storage.googleapis.com/zenn-user-upload/ccf4cc8fb29080d6f1677a08.png)
