---
title: Mac新机环境搭配
date: 2024-03-05 22:41:09
tags: [Mac]
categories: [Mac]
---

# Homebrew
一款Mac OS平台下的软件包管理工具，拥有安装、卸载、更新、查看、搜索等很多实用的功能。简单的一条指令，就可以实现包管理，而不用你关心各种依赖和文件路径的情况，十分方便快捷。
```bash
/bin/zsh -c "$(curl -fsSL https://gitee.com/cunkai/HomebrewCN/raw/master/Homebrew.sh)"

brew install google-chrome iina adrive maczip orbstack thunder android-studio visual-studio-code tencent-lemon 4k-video-downloader motrix tinymediamanager
```
# JDK

```bash
brew install openjdk@17

# 配置环境变量
echo 'export PATH=/opt/homebrew/opt/openjdk@17/bin:$PATH' >> ~/.zshrc
echo 'export ANDROID_HOME=/Users/fandun666/android-sdk' >> ~/.zshrc
echo 'export PATH=$PATH:$ANDROID_HOME/tools:$ANDROID_HOME/platform-tools' >> ~/.zshrc
source .zshrc
```

# Ohmyzsh

## 一键安装

```bash
sh -c "$(curl -fsSL https://install.ohmyz.sh)"
```
## 配置插件

```bash
# 插件目录
cd .oh-my-zsh/plugins
```
### 自带插件

- **git** git相关命令的别名和功能函数脚本
- **aliases** 管理命令别名的工具
- **tmux** tmux相关命令的别名
- **z** 目录之间的快速跳转
### 三方插件

- [**zsh-syntax-highlighting**](https://github.com/zsh-users/zsh-syntax-highlighting) 命令行语法高亮
- [**zsh-autosuggestions**](https://github.com/zsh-users/zsh-autosuggestions) 自动补全
### 主题

[官方的主题](https://github.com/ohmyzsh/ohmyzsh/wiki/Themes)目录包含了150多个易于安装的zsh主题。安装这些主题是定制你的zsh终端外观的最简单方法

为了安装一个主题，你只需要在官方目录中找到该主题的名称，并将该名称添加到 **.zshrc** 文件的主题部分。

例如，如果你想安装名为 "simple "的主题，我们需要做的就是用主题名称替换 **.zshrc** 文件中 **ZSH_THEME** 变量的当前值。
```bash
ZSH_THEME="simple"
```
# Node

```bash
brew install node@20

# 配置环境变量
echo 'export PATH=/opt/homebrew/opt/node@20/bin:$PATH' >> ~/.zshrc
source .zshrc
```

# Pake

```bash
# 安装rust环境
brew install rust

# 使用 npm 进行安装
npm install -g pake-cli

# 命令使用
pake url [OPTIONS]...

# 随便玩玩，首次由于安装环境会有些慢，后面就快了
pake https://weekly.tw93.fun --name Weekly --transparent
```

# Flutter

```bash
brew install flutter

# 配置国内镜像
echo 'export PUB_HOSTED_URL=https://pub.flutter-io.cn' >> ~/.zshrc
echo 'export FLUTTER_STORAGE_BASE_URL=https://storage.flutter-io.cn' >> ~/.zshrc
source .zshrc

```

# AList

## 阿里云盘
刷新令牌：eyJ0eXAiOiJKV1QiLCJhbGciOiJSUzI1NiJ9.eyJzdWIiOiIzYzJjODBhNjhkMDQ0MzI3YWRlNmRmNjNjOTU4OWU3OCIsImF1ZCI6Ijc2OTE3Y2NjY2Q0NDQxYzM5NDU3YTA0ZjYwODRmYjJmIiwiZXhwIjoxNzE3Njk0MTM5LCJpYXQiOjE3MDk5MTgxMzksImp0aSI6ImIwNmNjYmY1OTliNjQ3ZjViOTc2Njg5OTI3MDk0ZTZjIn0.b2sc13188Q5H3tDUP4P-Vy2aUXC44xth9dw0PufwZiuo_KkWJ8WzwrVyGXt1CkMqB3X1TT2oAwy5wjeItsXy4w
动漫世界：65abc853fd3f1000f39944d8ab0eae2f20a15756
热门电影：6586af3a72f2e2727bc345109a11de5a205d8276
热播新剧：65acec7b90edc681be294d479f316454d8cc36fb

## 百度云盘
122.95080eb6163b49f9242bedfca0e2d6c6.Y5lRwzqoY0-epibKC5J2z2RCn91ueqsolj9WGxp.oYleYw

## 迅雷云盘
根文件夹ID：VNrFL9LI6QnVzx4zhjRiukZgA1
用户名：+8617681839063
密码：Chenjie@19901221
验证码：ck0.vma4ay8xzp7iQHi_dcffzPvjSJffcjDvBnB-e3EIYOR_7S7jEMsBOhnV4uggGDwP6Z7Kz06HmTs_n8pmGARGj5riXA_2V-ZCX-0_qinYMgXVto0K_ozqdNtn3ex64w5wx6RAHR8g4lcK7K_rbAhLFCDJsr6NLtqlEXV97Dmv4xPq6e2sLVMxTlb8y3UV_hkLQtRNSTGXlI_YcaPCDXAKvHBUAYtmWU6_I3uTRK4DIiNCciTD2hnewiM0UTRZrb4Jr6U_r1YHZBQovBrqH-tq8HgVJoxiSEvetQpgFHjeUUuf7lRtEGPvxWVqKk_L6wOipk2UVWYlHl59cygwoTvkCw.CmUIs-bOkeIxEhBYcDZ2c3h6XzdJWVZ3MkJCGgs3LjUxLjAuODE5NiIbY29tLnh1bmxlaS5kb3dubG9hZHByb3ZpZGVyKiBmYWViM2I0ZDBjMGMxMGJjNWIwZGEyYmU1NTFhZjc3ZhKAAQ_X4fu9eYurVaBDYo9Ot58o5JZM6818yS51hjNpgXHT7JZphiUBm30aYeVqxZ7pg0o1-ypl3bESWdU9JZC-g8LPYA8wwtbo2Fe-PuiJbOrV0YL12eNP3HikzfHE8GkEonKoiDqfr4cPlEcA7ZOzhybyrAdug6wAiISR1JLJ6Xdq







