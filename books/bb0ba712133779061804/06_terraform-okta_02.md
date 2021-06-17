---
title: "Terraform に Okta のコードを記述する - 応用編"
---

その 1 では Okta に新規で設定を追加するという基本的な操作を行いました。

こちらでは、もう少し踏み込んだ実装方法を紹介していきます。

## 🔨 Okta に既にあるグループを Terraform 管理下に置く

Web コンソール上で Okta のグループを先に作ってあったグループを Terraform で管理する方法です。

既に Okta のグループを作り込んでいる場合、一から組み直したりすることが難しいことがあると思います。

まず、新規と同じように Terraform に Okta グループの定義を書きます。

```bash:./group.tf
resource "okta_group" "exist_group_1" {
  name        = "グループ1"
  description = "Okta に既にあるグループ"
}
```

続いて、Okta の Web コンソールを開いて対象のグループのページを開きます。

URL の末尾にある文字列をコピーします。これが Okta のグループ ID です。

![OktaのグループID](https://storage.googleapis.com/zenn-user-upload/392906cb30835fb3cc9fbe2e.png)

ターミナルに以下のコマンドを入力します。

```bash
terraform import okta_group.exist_group_1 <OktaグループID>
```

これで終わりです。以降は Terraform でグループ名などを書き換えるとOkta側も追従するようになります。

## 🔨 Okta のグループとグループルールを組み合わせる

### Okta のグループだけ Terraform 管理下ではないときのちょっとしたテクニック

## 参考資料

- <https://registry.terraform.io/providers/okta/okta/latest/docs/resources/group>
- <https://registry.terraform.io/providers/okta/okta/latest/docs/resources/group_rule>
