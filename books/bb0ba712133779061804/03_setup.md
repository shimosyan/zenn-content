---
title: "プロジェクトの作成"
---

## ⚙️ 初期設定

Terraformで実装するには準備が必要なので、いくつか設定します。

### プロジェクトの初期化

```shell
git init okta-terraform
cd okta-terraform
```

### .gitignoreの作成

```gitignore:./.gitignore
.DS_Store

# terraform
*.tfstate
*.tfstate.*
*.tfvars
**/.terraform/*
```

## ⚙️ AWS S3バケットの用意

Terraformの`*.tfstate`ファイルを設置するS3バケットを作成します。

AWSのコンソールにサインインしてS3の管理画面から「バケットを作成」を選択。

![AWS S3のバケット作成ボタンを押す](https://storage.googleapis.com/zenn-user-upload/92ce5371d96634ee9579ce21.png)

バケットは以下の設定で作成します。それ以外はデフォルトのままで構いません。

- バケット名: `terraform`
- バケットのバージョニング: `有効にする`

![バケット名の設定](https://storage.googleapis.com/zenn-user-upload/fbc5edfded1e2bfa609c55ee.png)

![バージョニングの設定](https://storage.googleapis.com/zenn-user-upload/ccf4cc8fb29080d6f1677a08.png)

## ⚙️ Terraformの準備