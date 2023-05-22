---
title: "GitHub Actions を使って Okta と Terraform に差分が起きてないか監視する"
---

ここでは編集されたコードを GitHub Actions を使って Okta の設定が Terraform（`main`ブランチ）と差異がないか監視する仕組みを整えていきます。

## ❓ なぜ監視が必要か

Terraform は Okta の設定をコード管理ができ、GitHub を組み合わせることで Okta の変更管理が実現します。

しかし、Okta 側で誰かが設定を変更してしまうと、この変更管理を活かすことができません。

Okta の設定変更を Terraform 一本に絞るためにも、Okta と `main` ブランチの Terraform コードに差が出ていないことを担保する必要があります。

そのために、Okta の設定変更がされていないかを監視する仕組みを整えます。

### もっとメタな話

実は、Okta の API トークンは有効期限が最後に使用してから一ヶ月しかありません。

Terraform コードのメンテナンスの期間が空いてしまうと、トークンの再設定が必要になるため、ここの監視の仕組みで Okta の API トークンを定期的に使用するという狙いがあります。

## 🔨 監視用 GitHub Actions Workflow の実装

以下の Workflow ファイルを作成し、GitHub に push します。

毎日正午に処理が実行されます。
実行するタイミング、頻度を変更したい場合は、`on.schedule.cron` 内の Cron 文を変更してください。

```yaml:./.github/workflows/schedule_check.yaml
# Okta で手動で変更されていないか監視するための Action
# 毎日12時に確認する
# また、Okta の API トークンが1ヶ月未使用で失効することを防ぐ目的もある
name: OktaScheduleCheck
on:
  schedule:
    # 毎日12時(12:00-JST = 03:00-UTC)に実行
    - cron:  '0 3 * * *'
jobs:
  terraform:
    name: ScheduleCheck
    runs-on: ubuntu-latest
    env:
      OKTA_API_TOKEN: ${{ secrets.OKTA_API_TOKEN }}

    defaults:
      run:
        shell: bash
    steps:
    - name: Checkout # GitHub のリポジトリを WorkingDirectory に Pull
      uses: actions/checkout@v2

    - name: Configure AWS credentials # AWSCLI をインストール及び Credential の設定
      uses: aws-actions/configure-aws-credentials@v1
      with:
        aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
        aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
        aws-region: ap-northeast-1

    - name: Setup Terraform # Terraform をインストール
      uses: hashicorp/setup-terraform@v2
      with:
        terraform_version: 1.0.0

    - name: Init Terraform
      run: terraform init

    - name: Test Terraform
      run: terraform validate

    - name: Difference Check # 差分が見つかればエラーを返す
      id: terraform_diff_check
      run: |
        RESULT=$(terraform plan -parallelism=1 -no-color -input=false)
        echo "$RESULT"
        if [ ! "`echo $RESULT | grep 'No changes.'`" ]; then
          exit 1
        fi
      continue-on-error: true

# 以下は Workflow の成否をチャットなどに通知を行うときの処理です。もし必要あったら使ってください。
    - name: Notify Difference Message
      if: steps.terraform_diff_check.outcome == 'failure'
      run: # 差分が見つかったときに通知処理
```

こちらの Workflow も同様に、Okta の差分検知時に通知できるステップを実装しています。

もし、通知を使わないときは`Notify Difference Message`ブロックを削除してください。

## 📄 参考資料

- <https://qiita.com/keitakn/items/db2e9c68019594885ac4>
- <https://qiita.com/t0yohei/items/9285e61f8358de7f60a7>
- <https://docs.github.com/ja/actions/reference/workflow-syntax-for-github-actions>
