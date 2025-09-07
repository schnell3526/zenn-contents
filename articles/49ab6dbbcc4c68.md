---
title: "Ubuntu 上に Docker をインストールする方法"
emoji: "👋"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ['Docker', 'Ubuntu']
published: false
---

[公式ドキュメント](https://docs.docker.com/engine/install/debian/)を参考にインストールする際、Ubuntuのバージョンが新しい場合リポジトリに対応するバイナリが存在しないためにエラーが出ますのでその対処法を含めて備忘録として書き残します。

記事を執筆した際に利用した環境
```shell
$ cat /etc/os-release
PRETTY_NAME="Ubuntu 22.04.1 LTS"
NAME="Ubuntu"
VERSION_ID="22.04"
VERSION="22.04.1 LTS (Jammy Jellyfish)"
VERSION_CODENAME=jammy
ID=ubuntu
ID_LIKE=debian
HOME_URL="https://www.ubuntu.com/"
SUPPORT_URL="https://help.ubuntu.com/"
BUG_REPORT_URL="https://bugs.launchpad.net/ubuntu/"
PRIVACY_POLICY_URL="https://www.ubuntu.com/legal/terms-and-policies/privacy-policy"
UBUNTU_CODENAME=jammy
```

# 古いバージョンのDockerをアンインストール
```shell
sudo apt-get remove docker docker-engine docker.io containerd runc
```

# リポジトリの設定
```shell
sudo apt-get update
sudo apt-get install \
    ca-certificates \
    curl \
    gnupg \
    lsb-release

sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```
```
[Service]
Environment="www-proxy.waseda.jp:8080"
```

# Dockerをインストール
ここでパッケージリストの更新をしようとすると、エラーが出ます。
```shell
sudo apt-get update
```
```
Err:6 https://download.docker.com/linux/debian jammy Release
  404  Not Found [IP: 18.65.185.82 443]
Reading package lists... Done
E: The repository 'https://download.docker.com/linux/debian jammy Release' does not have a Release file.
N: Updating from such a repository can't be done securely, and is therefore disabled by default.
N: See apt-secure(8) manpage for repository creation and user configuration details.
```

これは [https://download.docker.com/linux/debian](https://download.docker.com/linux/debian) に jammy(Ubuntu 22.04) に対応するバイナリが存在しないため発生するエラーのようです。