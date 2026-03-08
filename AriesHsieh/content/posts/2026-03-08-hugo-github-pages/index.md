---
title: "用 Hugo 寫文章並透過 GitHub Actions 部署到 GitHub Pages"
date: 2026-03-08T07:15:14+08:00
draft: false
categories: ["DevOps"]
tags: ["Hugo", "GitHub Actions", "GitHub Pages", "CI/CD"]
description: "記錄這個部落格的完整運作方式：如何在本地用 Hugo 新增文章，以及 GitHub Actions 如何自動把靜態網站部署到 GitHub Pages。"
showToc: true
TocOpen: false
searchHidden: false
comments: true
---

# 前言

這個部落格跑了一段時間，但我一直沒有好好整理它是怎麼運作的。趁現在記錄一下：本地怎麼新增文章、推上 GitHub 後又發生了什麼事，讓網站自動更新。

# 專案結構

整個 GitHub repository（`arieshsieh0402.github.io`）的結構如下：

```
arieshsieh0402.github.io/       ← git repo 根目錄
├── .github/
│   └── workflows/
│       └── hugo.yaml           ← GitHub Actions 部署設定
├── AriesHsieh/                 ← Hugo 專案目錄
│   ├── archetypes/
│   │   └── posts.md            ← 新文章的預設模板
│   ├── content/
│   │   └── posts/              ← 所有文章放這裡
│   ├── themes/
│   │   └── PaperMod/           ← 使用的 Hugo 主題
│   ├── public/                 ← hugo build 產生的靜態檔案（不進 git）
│   └── hugo.yaml               ← Hugo 網站設定
```

> Hugo 專案本身放在 repo 根目錄下的 `AriesHsieh/` 子目錄，而非 repo 根目錄。這個結構在 GitHub Actions 設定中需要特別注意 working directory。

# 本地新增文章

## Step 1：建立文章

在 `AriesHsieh/` 目錄下執行 `hugo new` 指令：

```bash
hugo new posts/2026-03-08-my-new-post/index.md
```

Hugo 會依據 `archetypes/posts.md` 模板，在 `content/posts/` 下建立對應目錄與 `index.md`，並自動填入 front matter：

```yaml
---
title: "2026 03 08 My New Post"
date: 2026-03-08T14:57:06+08:00
draft: true
categories: []
tags: []
description: ""
showToc: true
TocOpen: false
searchHidden: false
comments: true
---
```

## Step 2：編輯文章內容

修改 `title`、`categories`、`tags`、`description`，並在 front matter 下方撰寫 Markdown 內容。

## Step 3：本地預覽

```bash
hugo server -D
```

`-D` 參數讓 `draft: true` 的文章也會被渲染，方便預覽草稿。預覽網址預設為 `http://localhost:1313`。

## Step 4：確認完成後，將 draft 改為 false

```yaml
draft: false
```

## Step 5：推上 GitHub

```bash
git add .
git commit -m "Add post YYYYMMDD"
git push
```

推上 `main` branch 後，GitHub Actions 就會自動觸發部署。

# GitHub Actions 部署流程

## 為什麼需要 GitHub Actions

GitHub Pages 本身只做一件事：把 repo 裡的靜態檔案（HTML/CSS/JS）對外提供。如果直接放 HTML 進去，確實不需要任何 build 步驟。

但 repo 裡放的不是 HTML，而是 Markdown 原始碼、設定檔、主題模板——這些 GitHub Pages 看不懂。需要先跑一次 `hugo build`，把這些東西**編譯**成真正的 HTML，才能部署：

```
Markdown + 主題模板
        │
        ▼  hugo build
    public/（HTML/CSS/JS）
        │
        ▼
   GitHub Pages 服務
```

沒有 GitHub Actions 的話，就得在本地跑 `hugo build`、把 `public/` 也 commit 進去再 push。有了 GitHub Actions，這個 build 步驟交給 cloud 自動跑，只要 push Markdown 原始碼就好。

## 整體架構

```
push to main
    │
    ▼
[build job]
    ├── 安裝 Hugo extended
    ├── checkout repo（含 submodules）
    ├── 執行 hugo build（working-directory: AriesHsieh）
    │       └── 輸出靜態檔案到 AriesHsieh/public/
    └── upload artifact
    │
    ▼
[deploy job]
    └── 將 artifact 部署到 GitHub Pages
```

## 關鍵設定說明

**trigger：**
```yaml
on:
  push:
    branches:
      - main
```
只有 push 到 `main` 才會觸發，避免開發分支誤部署。

**Hugo 版本固定：**
```yaml
env:
  HUGO_VERSION: 0.147.9
```
固定版本確保每次 build 結果一致，不受 Hugo 版本更新影響。

**working-directory 指定子目錄：**
```yaml
- name: Build with Hugo
  run: |
    hugo --gc --minify --baseURL "${{ steps.pages.outputs.base_url }}/"
  working-directory: AriesHsieh
```
因為 Hugo 專案放在 `AriesHsieh/` 子目錄，必須明確指定 working directory，否則 Hugo 找不到 `hugo.yaml` 與 `content/`。

**Build 產物路徑：**
```yaml
- name: Upload artifact
  uses: actions/upload-pages-artifact@v3
  with:
    path: ./AriesHsieh/public
```
Hugo build 完的靜態檔案在 `AriesHsieh/public/`，上傳這個目錄作為 GitHub Pages 的部署內容。

**GITHUB_TOKEN 權限：**
```yaml
permissions:
  contents: read
  pages: write
  id-token: write
```
`pages: write` 與 `id-token: write` 是部署到 GitHub Pages 的必要權限，缺一不可。

## Build Cache

Workflow 中有 cache restore/save 的步驟，利用 `actions/cache` 快取 Hugo 的 build cache，加快後續 build 速度。

```yaml
- name: Cache Restore
  uses: actions/cache/restore@v4
  with:
    path: ${{ runner.temp }}/hugo_cache
    key: hugo-${{ github.run_id }}
    restore-keys: hugo-
```

# 小結

整理起來其實不複雜：本地用 `hugo new` 建立文章、寫完推上 `main`，GitHub Actions 自動接手 build 與部署，完全不需要手動操作 GitHub Pages。趁還沒忘掉把這個流程寫下來，下次忘了至少有地方查 XD
