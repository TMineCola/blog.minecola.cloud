# My Blog | blog.minecola.cloud

[![Build Status](https://travis-ci.org/TMineCola/tminecola.github.io.svg?branch=src)](https://travis-ci.org/TMineCola/tminecola.github.io)
[![Last Update](https://img.shields.io/github/last-commit/TMineCola/tminecola.github.io/src)](https://img.shields.io/)

## About Hugo

Hugo 是一個用 Go 語言撰寫的靜態網站生成器，可以將 Markdown 格式撰寫的內容轉換成對應主題的 HTML，自稱「全世界最快的網頁建構框架」

> Alternatives: Hexo (作者是台灣人 [Tommy Chen](https://github.com/tommy351)，以 Node.js 撰寫)

### Commands
> 基本指令 [Basic Usage](https://gohugo.io/getting-started/usage/)

```
Hugo [command]:
  check       驗證性檢查
  config      顯示網站設定檔內容 (config.toml)
  convert     轉換內容至不同格式 JSON/YAML/TOML
  env         顯示 Hugo 版本資訊
  gen         產生一些說明文件
  help        顯示可用的指令
  import      從別人的網站中匯入
  list        顯示目前所擁有的文章
  new         新增文章
  server      在本地建立一個 Web Server
  version     顯示 Hugo 版本
```

## About Theme
> Fork from: [hugo-tranquilpeak-theme](https://github.com/kakawait/hugo-tranquilpeak-theme/)

### Config.toml
> 用來設定網站屬性，詳細設定可以參考 [hugo-tranquilpeak-theme/user.md/configuration](https://github.com/kakawait/hugo-tranquilpeak-theme/blob/master/docs/user.md#tranquilpeak-configuration)

### 關於標記
[Reference](https://github.com/kakawait/hugo-tranquilpeak-theme/blob/master/docs/user.md)

#### 文章外表

- `autoThumbnailImage: true/false` 自動尋找縮圖
    > 預設會使用 Cover (封面) 或第一章圖片作為縮圖
- `thumbnailImagePosition: "top/left/right"` 縮圖放置的位置
- `thumbnailImage:` 縮圖圖片路徑

#### 文章結構
- `title:` 標題
- `date:` 日期
- `categories:` 分類
    > \<meta property="article:section" content="分類"\>
- `tags:` 標籤
    > \<meta property="article:tag" content="標籤"\>
- `keywords:` 關鍵字
    > \<meta name="keywords" content="關鍵字"\>

#### 文章內容

- `coverImage:` 封面圖片路徑
- `coverCaption:` 封面標題
- `coverMeta: in/out` 封面文字
    > in: 標題、日期及分類顯示在封面中
    > out: 顯示在封面下方
- `coverSize: partial/full` 調整封面大小
    > partial: 60% 螢幕高度
    > full: 滿版
- `metaAlignment: left/right/center` 標題對齊方式
- `summary:` 文章的摘要、總結 (與內文的 `<!-- more -->` 擇一使用)
- `showDate: true/false` 是否顯示日期
- `comments: true/false` 是否顯示回應
- `showTags: true/false` 是否顯示標籤
- `showPagination: true/false` 是否顯示分頁
- `showSocial: true/false` 是否顯示社群按鈕
- `showActions: true/false` 顯示分頁及社群按鈕等可操作元件
- `gallery:` 文末會以畫廊的方式顯示圖片
    > 使用方法與 tags 那些一樣


### 內文格式

#### 結構
- `<!--more-->` 摘要 (並且會做為 SEO 的 description)
    > \<meta name="description" content="摘要內容"\>

- `<!-- toc -->` Table of content 顯示標題連結表

#### 特殊顯示

- `{{< alert info >}} Content {{ </alert >}}` 提示框
    > info 藍、success 綠、warning 黃、 danger 紅

- `{{< blockquote "作者" "網址" "網址取代文字" >}} Content {{< /blockquote >}}` 引言
    > 資訊可遞減填寫

- `{{< codeblock "程式碼標題" "程式碼格式(副檔名 ex: cpp 或服務名稱 ex: http, nginx)" "連結網址" "網址連結名稱" >}} 程式碼 {{< /codeblock >}}`  程式碼區塊
    > 也可以使用一般 \`\`\`js \`\`\` 的方法
    > 或是 `{{< codeblock lang="程式語言" }} 內容 {{< /codeblock >}}`

- 分頁式程式區塊
```
{{< tabbed-codeblock tabbed_codeblock >}}
<!-- tab html -->
HTML 內容
<!-- endtab -->
<!-- tab js -->
JS 內容
<!-- endtab -->
{{< /tabbed-codeblock >}}
```

- `{{< gist GistID FileName >}}` Gist

- `{{< jsfiddle ccWP7 >}}` JsFiddle

- `{{< pullquote left >}} Content {{< /pullqote >}}` 內縮區塊
    > 有 left、right 兩種， left 會縮在下一段文字左方， right 會縮在前一段右方

- `{{< hl-text 顏色 >}} Content {{< /hl-text >}}` 文字背景顏色
    > 顏色也可以使用 primary、success 等

- `{{< youtube 影片ID >}}` 內嵌 Youtube 影片

- `{{< wide-image src="來源" title="圖片標題" >}}` 滿版圖片

其餘可以參考 [展示頁面](https://tranquilpeak.kakawait.com/2014/10/tags-plugins-showcase/) 搭配 [原始 mardown 格式](https://raw.githubusercontent.com/kakawait/hugo-tranquilpeak-theme/master/exampleSite/content/post/Tags-plugins-showcase.md)