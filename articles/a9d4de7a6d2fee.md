---
title: "標準ユーザーに対して Microsoft Intune と Winget を組み合わせアプリケーション配布を実現させたい"
emoji: "📝"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Intune", "Windows", "Winget"]
published: true
---

## はじめに注意点

:::message
Intune と Winget を組み合わせてアプリケーションを配布する方法は Winget プロジェクトでも議論の最中にあります。そのため今後仕様が変わる可能性があります。

また独自調査のため誤った説明が入っている可能性があります。
:::

:::message
この Winget を使った方法は AutoPilot の登録状態ページ（EnrollmentStatusPage、ESP）で利用することができません。

登録状態ページでアプリケーションをインストールしたい場合は MSI を使ったインストールを検討ください。
@[card](https://zenn.dev/shimosyan/articles/2f34f66d1d5b9a)
:::

## これは何？

Intune で Windows 端末に対してアプリケーションを配布する作業、とてもめんどくさいです。

https://zenn.dev/shimosyan/articles/2f34f66d1d5b9a

こちらの記事でも書いたとおり、ある程度共通化・簡略化をしてアプリケーションを配布を実現したいです。
その方法の一つとして Microsoft 公式のパッケージマネージャーである Winget と組み合わせる方法があります。

## 今回の課題

Intune + Winget を使う方法を調べると「配布設定にて指定する`インストールの処理`では**システム**ではなく**ユーザー**を指定すること」と記載があります。

これは「Winget がユーザーコンテキストにしか対応していないため」ということが主な理由として説明されています。

Intune では「**システム**を指定すると Windows の System アカウントの資格情報（システムコンテキスト）」として、そして「**ユーザー**を指定するとログインしているユーザーの資格情報（ユーザーコンテキスト）」として処理が動きます。

ログインユーザーの資格情報で処理されるということは、つまりユーザーに管理者特権が付与されている必要があります。逆に特権を持たない標準ユーザーでは Intune で処理を実行しても UAC に阻まれてしまうわけです。

## Winget をシステムコンテキストで実行させるには

標準ユーザーでログインしている端末ではユーザーコンテキストでは管理者特権を行使することができません。必然的にシステムコンテキストで実行させるしかなくなります。

`winget install --id **.**` のような一般的なインストールコマンドのまま Intune の配布設定をユーザーからシステムに作り直しましたが結果が変わりません。

しかし、いろいろ文献を調べてみると Winget の実行バイナリは2種類あることがわかってきました。

<!-- cSpell:disable -->一つ目は `winget` コマンドの PATH 先となっている下記のパスです。

```sh
%LOCALAPPDATA%\Microsoft\WindowsApps\winget.exe
```

もう一つは下記のパスです。(`**` には Winget のバージョンが入ります)

```sh
C:\Program Files\WindowsApps\Microsoft.DesktopAppInstaller_**_x64__8wekyb3d8bbwe\winget.exe
```

<!-- cSpell:enable -->さらに調べてみると前者はユーザーコンテキストでの動作ですが、後者はシステムコンテキストで動作できさらに Intune で処理したときに UAC を回避できることがわかってきました。

つまり、今回のケースでは後者の Winget を使うことで解決できるということです。

## スクリプトの作成

下記のスクリプトを作成して `.intunewin` 形式にパッケージングしてください。ファイル名、ファイル形式は以下で作成してください。

- ファイル名：`winget.ps1`
- 文字コード：`UTF-8`
- 改行コード：`CRLF`

```powershell
Param (
  [switch]$install,
  [switch]$uninstall,
  [switch]$user,
  [switch]$system,
  [parameter(mandatory=$true)][string]$applicationId
)

if ($install -eq $uninstall){
  Write-Error "Either '-install' or '-uninstall' must be entered."
}

if ($user -eq $system){
  Write-Error "Either '-user' or '-system' must be entered."
}

if ($system -eq $TRUE) {
  # Check Administrator
  $isAdmin = [System.Security.Principal.WindowsPrincipal]::new([System.Security.Principal.WindowsIdentity]::GetCurrent()).IsInRole([System.Security.Principal.WindowsBuiltInRole]::Administrator)

  if ($isAdmin -eq $FALSE) {
    Write-Error "Administrator priviledges are required when using '-system'."
  }

  $wingetExe = Resolve-Path "C:\Program Files\WindowsApps\Microsoft.DesktopAppInstaller_*_*__8wekyb3d8bbwe\winget.exe"

  if ($wingetExe.count -gt 1){
    $wingetExe = $wingetExe[-1].Path
  }
  if (!$wingetExe){
    Write-Error "Winget not installed"
  }
}

if ($user -eq $TRUE) {
  $wingetExe = Resolve-Path "$env:LOCALAPPDATA\Microsoft\WindowsApps\winget.exe"
}

Write-Host "Winget Path: $wingetExe"

if ($install -eq $TRUE) {
  Start-Process -NoNewWindow -FilePath $wingetExe -ArgumentList "install -e --silent --accept-source-agreements --accept-package-agreements --id $applicationId" -Wait
}

if ($uninstall -eq $TRUE) {
  Start-Process -NoNewWindow -FilePath $wingetExe -ArgumentList "uninstall -e --silent --id $applicationId" -Wait
}

```

## Intune の配布設定

上記スクリプトを格納した `.intunewin` ファイルを Intune にアップロードします。

配布設定は以下のように設定します。

### インストールコマンドおよびアンインストールコマンド

下記文字列を入力します。一部書き換えないといけない箇所があるので下表から置き換えてください。

```sh
powershell -ExecutionPolicy Bypass ".\winget.ps1 $mode $context 'id'"
```

|コマンド内の文字列|置き換える値|
|---|---|
|`$mode`|`-install` または `-uninstall` どちらかを入力します。入力した値によってインストールまたはアンインストールのモードが変わります。|
|`$context`|`-user` または `-system` どちらかを入力します。入力した値によってユーザーコンテキストまたはシステムコンテキストのモードが変わります。|
|`id`|Winget で処理したいアプリケーションを指定する ID|

以下はサンプルです。

|やりたいこと|コマンド例|
|---|---|
|Google Chrome をシステムコンテキストでインストール|`powershell -ExecutionPolicy Bypass ".\winget.ps1 -install -system 'Google.Chrome'"`|
|Google Chrome をシステムコンテキストでアンインストール|`powershell -ExecutionPolicy Bypass ".\winget.ps1 -uninstall -system 'Google.Chrome'"`|

:::message

`C:¥Program Files` 以下にインストールされるアプリケーションはシステムコンテキスト、`C:¥Users¥○○¥AppData` 以下にインストールされるアプリケーションはユーザーコンテキストで動作させるのがオススメです。

:::

## 検出規則

Intune の検出ルールでスクリプトを書いて Winget コマンドでインストールの有無を検出することは技術的に可能ですが、いろいろデバッグしにくいところなので素直にファイルの有無で検出した方が良いかも。

## 割り当て

デバイスコンテキストで動かすなら「すべてのデバイス」or「EntraID のデバイスグループ」で、
ユーザーコンテキストで動かすなら「すべてのユーザー」or「EntraID のユーザーグループ」がオススメです。

## 最後に

というわけで、Intune と Winget を組み合わせて標準ユーザーの Windows 端末に対してアプリケーションの配布を実現するための解決策を紹介しました。

記載した内容について何かありましたら記事コメントか X/Twitter までご連絡ください。
