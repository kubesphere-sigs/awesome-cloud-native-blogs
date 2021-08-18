# Awesome Cloud Native Blogs

## 关于本仓库

在本仓库中，您可以：

- 提交英文文章线索
- 参与翻译
- 提交个人原创文章
- 提交个人已发布文章的转载要求

文章题材包括但不限于：

- Kubernetes
- Service Mesh
- Cloud Native
- Serverless
- DevOps
- Container

您提交的文章将首发于：

- KubeSphere 社区官网：https://kubesphere.com.cn
- 「KubeSphere 云原生」微信公众号
- 其他渠道（保留作者署名或 URL 共享）

## 如何参与

参与该项目包括两种方式：

1. 通过[提交 Issue](https://github.com/kubesphere-sigs/awesome-cloud-native-blogs/issues/new?assignees=&labels=kind%2Ftranslation&template=docs.yaml)的方式提交文章线索
2. 通过 PR 参与文章翻译或提交原创文章
3. 原则上所有认领的文章要在 5 个工作日内完成翻译或创作，**可以多人认领同一篇文章协作翻译**

### 提交 Issue

[提交Issue](https://github.com/kubesphere-sigs/awesome-cloud-native-blogs/issues/new?assignees=&labels=kind%2Ftranslation&template=docs.yaml)，其中的选项只要打钩即可。

**注意**：请在 Issue 标题中增加类型前缀，如`Translation:`、`Original:`、`Reprint:`。

### 提交 PR

[提交PR](https://github.com/kubesphere-sigs/awesome-cloud-native-blogs/pulls)，可以为译文、原创或个人文章转载，请在文章头部添加元信息，格式如下：

```yaml
---
original: "原文链接或者原创作者的 GitHub 账号"
author: "作者姓名"
translator: "译者的 GitHub 账号"
original: "原文地址"
reviewer: ["审阅者 A 的 GitHub 账号","审阅者 B 的 GitHub 账号"]
title: "标题"
summary: "这里是文章摘要。"
categories: "译文、原创或转载"
tags: ["taga","tagb","tagc"]
originalPublishDate: 2021-08-18
publishDate: 2019-08-18
---
```

注：其中 `originalPublishDate` 为所翻译的文章原文的发布日期，`publishDate` 为原创文章或译文的 PR 合并日期。

## 关于 PR 的注意事项

提交 PR 请注意以下事项：

- 请在 PR 中标题中增加类型前缀，如`Translation:`、`Original:`、`Reprint:`
- 提交的文件名必须全英文、小写、单词间使用连字符

管理员在合并 PR 前请先确认 [PR 模板](<https://github.com/kubesphere-sigs/awesome-cloud-native-blogs/blob/master/PULL_REQUEST_TEMPLATE.md>)中的内容。

注：提交 PR 表明您授权您的创作被发布到 KubeSphere 社区或以保留您署名的方式转载。

## 文档要求

文档格式为 Markdown，尽量保留原文格式，可以根据中文阅读习惯适度调整，**中英文之间、中文和数字之间要加空格**。

**命名规则**

文件使用英文命名，单词之间使用 `-` 连接，所有字母均为小写，例如 `x509-certificate-exporter.md `。

**目录结构**

新的文档根据提交 PR 的时间在对应的目录中创建。如2021年8月18日提交的PR需要在 `2021/08/` 目录中创建新的文档，即所有文档是按照月来归档的。加入文章标题为 `new-post`，则创建的新文件为 `2021/08/new-post/index.md`。

**关于图片**

图片请随文章一同上传到 Github 中，与 `index.md` 文件位于同一目录。