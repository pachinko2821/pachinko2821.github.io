---
date: '{{ .Date }}'
draft: true
title: '{{ replace .File.ContentBaseName `-` ` ` | title }}'
cover:
    image: "images/cover.png" 
    alt: "some image that didn't load"
    caption: "CAPTION"
    responsiveImages: true
tags: ["tags"]
keywords: ["keywords"]
description: "DESCRIPTION"
showFullContent: true
readingTime: true
showFullContent: true
readingTime: true
showToc: true
tocOpen: true
showReadingTime: true
hideSummary: true
---