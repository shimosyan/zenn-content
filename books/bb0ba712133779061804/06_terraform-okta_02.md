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

これで終わりです。以降は Terraform でグループ名などを書き換えると Okta 側も追従するようになります。

### Okta の各リソースの ID を取得するちょっとしたテクニック

上記ではグループの ID を取得しました。これはアクセスした URL にたまたま ID が載っていたからですが、中には URL からでは取得できないものもたくさんあります。

そんなときは Chrome 拡張機能の [rockstar](https://chrome.google.com/webstore/detail/rockstar/chjepkekmhealpjipcggnfepkkfeimbd)が非常に便利です。

Okta のいろいろな設定値を ID 付きで CSV 出力できたり、API を叩いてデータを取得できます。
Terraform コードを書くときはとてもお世話になるのでおすすめです。

## 🔨 Okta のグループとグループルールを組み合わせる

ここではグループルールの作成とグループへの割り当てを行います。

Web コンソール上でグループルールを編集する場合、割り当て先のグループは変更できなかったり、条件式を編集するときでも一度 Inactive にする必要があるなど少し使いづらい点がありますが、Terraform ではこれらをすべて解決してくれます。
（内部的には再作成や Inactive・Active を高速で処理してるだけですが）

今回はシンプルにグループとグループルールを 1 つづつ作成し、そのグループルールの割り当て先をグループに指定します。
グループルールの条件は例として「"@example.com"のメールアドレスを持つユーザー」とします。

```bash:./group.tf
resource "okta_group" "group_by_rule_1" {
  name        = "割り当て先グループ1"
  description = "これはグループルールによってメンバーが自動割当されるグループです"
}

resource "okta_group_rule" "group_rule_1" {
  name   = "グループルール"
  status = "ACTIVE"
  group_assignments = [
    okta_group.group_by_rule_1.id # group_by_rule_1 のグループ ID を参照
  ]
  expression_type  = "urn:okta:expression:1.0"
  expression_value = "String.stringContains(user.login, \"@example.com\")" # 条件式を記述します。条件式内の " はバックスラッシュでエスケープが必要です。
}
```

このようなコードになります。

プロパティ内の`group_assignments`に割り当てのグループ ID を指定します。
このグループ ID は前述のように Okta から引っ張ってくることもできますが、Terraform で管理下においているグループであれば、`okta_group.RESOURCE_ID.id`と記述することで上記であれば`group_by_rule_1`のグループ ID を参照できます。

また、グループを複数割り当てたいときは、`group_assignments`にはこのように記述します。

```bash
  group_assignments = [
    okta_group.group_by_rule_1.id,
    "XXXXXXXXXX" # Terraform 管理にせず、Okta から ID だけ持ってきた場合
  ]
```

### Okta のグループだけ Terraform 管理下ではないときのちょっとしたテクニック

グループだけ Terraform 管理にしないケースであれば上記のようにグループ ID を直接指定できます。
しかし、そのグループを作り直してしまうと ID が変わってしまいうため、Terraform のコードが正しく動かない恐れがあります。

こういう場合は、Terraform の`data`機能を活用できます。

簡単に表すと、「Terraform による管理は行わないけど、Okta にあるデータを検索して参照できるようにする」機能です。

では試しに、上記のコードを改変して以下のケースのコードを書いてみます。

- グループ 1 とグループ 2 を割り当て先に指定するグループルールを Terraform 管理にする。
- グループ 1 は Terraform 管理にする。
- グループ 2 は Terraform 管理にしない。名前は変わらないが、ID が変わる可能性がある。

```bash:./group.tf
resource "okta_group" "group_by_rule_1" {
  name        = "割り当て先グループ1"
  description = "これはグループルールによってメンバーが自動割当されるグループです"
}

data "okta_group" "group_by_rule_2" {
  name        = "割り当て先グループ2" # Okta からこの名前のグループに関する情報を取得する
}

resource "okta_group_rule" "group_rule_1" {
  name   = "グループルール"
  status = "ACTIVE"
  group_assignments = [
    okta_group.group_by_rule_1.id,
    okta_group.group_by_rule_2.id
  ]
  expression_type  = "urn:okta:expression:1.0"
  expression_value = "String.stringContains(user.login, \"@example.com\")" # 条件式を記述します。条件式内の " はバックスラッシュでエスケープが必要です。
}
```

## 📄 参考資料

- <https://registry.terraform.io/providers/okta/okta/latest/docs/resources/group>
- <https://registry.terraform.io/providers/okta/okta/latest/docs/resources/group_rule>
- <https://registry.terraform.io/providers/okta/okta/latest/docs/data-sources/group>
