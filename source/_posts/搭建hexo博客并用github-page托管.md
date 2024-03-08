---
title: 搭建hexo博客并用github page托管
date: 2024-03-08 08:55:04
tags:
---

## 搭建环境

[hexo文档](https://hexo.io/zh-cn/docs/)

```
# 基础系统 ubuntu 20.04
# 安装hexo
sudo apt install git nodejs npm
npm config set registry https://registry.npm.taobao.org
npm install -g hexo-cli

# hexo创建项目
hexo init liduanjun_github_page
cd liduanjun_github_page
npm install

# hexo配置主题
npm install hexo-theme-next
cd liduanjun_github_page
git clone https://github.com/next-theme/hexo-theme-next themes/next
nano _config.yml
# theme: next
```

## 开始写作
```
hexo new post "博文标题"
hexo server
```

## Github Page 托管
```
mkdir -pv .github/workflows/
cat > .github/workflows/pages.yml <<EOF
name: Pages

on:
  push:
    branches:
      - master # default branch

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
        with:
          token: ${{ secrets.GITHUB_TOKEN }}
          # If your repository depends on submodule, please see: https://github.com/actions/checkout
          submodules: recursive
      - name: Use Node.js 21.x
        uses: actions/setup-node@v2
        with:
          node-version: '21'
      - name: Cache NPM dependencies
        uses: actions/cache@v2
        with:
          path: node_modules
          key: ${{ runner.OS }}-npm-cache
          restore-keys: |
            ${{ runner.OS }}-npm-cache
      - name: Install Dependencies
        run: npm install
      - name: Build
        run: npm run build
      - name: Upload Pages artifact
        uses: actions/upload-pages-artifact@v2
        with:
          path: ./public
  deploy:
    needs: build
    permissions:
      pages: write
      id-token: write
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    runs-on: ubuntu-latest
    steps:
      - name: Deploy to GitHub Pages
        id: deployment
        uses: actions/deploy-pages@v2
EOF
```
打开 liduanjun.github.io 项目的配置页面
```
https://github.com/liduanjun/liduanjun.github.io/settings/pages
```
点击 Code and automation > Pages

选择 Github Action 为 Github Action ，如下图

![Github Page 配置](images/github-page-config.png)

## 发布文章
```
git commit
git push origin master
```