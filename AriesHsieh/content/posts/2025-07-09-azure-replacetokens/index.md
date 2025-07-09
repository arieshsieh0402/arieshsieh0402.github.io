---
title: "Azure ReplaceTokens 字串轉義問題"
date: 2025-07-09T21:11:20+08:00
draft: false
categories: ["devops"]
tags: ["azure", "escape 字元", "pipeline"]
description: "記錄 ReplaceTokens task 在處理字串換行符號時的問題與解法"
showToc: true
TocOpen: false
searchHidden: false
comments: true
---

## 問題現象

在某個專案中，CI/CD 過程需要將設定以 token 替換的方式寫入 JSON 檔案，其中一個欄位包含多行文字內容。在 pipeline 中使用了 `qetza.replacetokens.replacetokens-task.replacetokens@5` 這個 Azure DevOps Task：

```yaml
- task: qetza.replacetokens.replacetokens-task.replacetokens@5
  displayName: 'Replace token in xxx-config.json'
  inputs:
    targetFiles: '$(System.ArtifactsDirectory)\\drop\\$(Build.BuildId)\\xxx-config.json'
```

完成替換並部署到測試環境後，發現無法正常啟動。檢查部署產出的 JSON 檔案時，發現原本應為多行的字串中的 `\n`，在處理後變成了 `\\n`。

這導致程式在解析該欄位時，無法還原出原本的格式，進而發生錯誤。

## 問題原因

查閱文件後發現，ReplaceTokens 預設會對變數值進行 escape 處理。這對一般情況有幫助，但對於需要保留格式的多行字串（如憑證或編碼內容）就會導致格式錯誤。

## 解法：使用 noescape() 函數停用自動 escape

ReplaceTokens 支援開啟轉換函數（Transformations）功能，可用來自訂變數處理方式。具體做法如下：

### 步驟 1：啟用 enableTransforms

在 ReplaceTokens 的 YAML 設定中加上 `enableTransforms: true`

```yaml
- task: qetza.replacetokens.replacetokens-task.replacetokens@5
  displayName: 'Replace token in xxx-config.json'
  inputs:
    targetFiles: '$(System.ArtifactsDirectory)\\drop\\$(Build.BuildId)\\xxx-config.json'
    enableTransforms: true # 要加這行
```

### 步驟 2：修改 JSON，使用 noescape()

將目標檔案中的 token 改寫為：

```json
{
  "config_content": "#{noescape(multiline_config)}#"
}
```

如此可讓 `\n` 在替換後仍保持原始格式，不會被轉成 `\\n`。

### 其他支援的轉換函數

- base64(): 將變數值編碼為 Base64
- upper(): 將變數值轉為大寫字母
- lower(): 將變數值轉為小寫字母

### Ref

- [Replace Tokens task on GitHub (qetza)](https://github.com/qetza/replacetokens)
