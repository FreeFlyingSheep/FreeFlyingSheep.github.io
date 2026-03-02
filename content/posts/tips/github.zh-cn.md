---
title: "Github 模板"
date: 2021-05-12
description: ""
menu:
  sidebar:
    name: "Github 模板"
    identifier: tips-github
    parent: tips
    weight: 100
tags: ["Github"]
categories: ["Tips"]
---

根据 [Github Issue & PR templates](https://docs.github.com/en/communities/using-templates-to-encourage-useful-issues-and-pull-requests) 相关文档整理。

<!--more-->

## Github 模板

仓库的维护者可以添加模板，来帮助贡献者创建高质量的 Issue 和 Pull request。

### Issue 模板

#### Issue 模板介绍

Issue 模板放置于 `.github/ISSUE_TEMPLATE` 文件夹，里面可以创建 `config.yml` 来配置模板选择器。

`config.yml` 的官方示例如下：

```yaml
blank_issues_enabled: false
contact_links:
  - name: GitHub Community Support
    url: https://github.community/
    about: Please ask and answer questions here.
  - name: GitHub Security Bug Bounty
    url: https://bounty.github.com/
    about: Please report security vulnerabilities here.
```

`blank_issues_enabled` 选项用于配置是否允许发起空 Issue。

`contact_links` 用于指定一些额外的链接，比如有些问题不属于该项目，而是相关工具的问题，可以让用户去那些工具的官网提 Issue。

#### Issue 模板实例

[Pro Git, Second Edition](https://github.com/progit/progit2) 项目的 `.github/ISSUE_TEMPLATE/config.yml` 文件如下：

```yaml
contact_links:
  - name: Translation bug
    url: https://github.com/progit/progit2/blob/master/TRANSLATING.md
    about: Refer to this table to find out where to report translation bugs.

  - name: Report bugs for git-scm.com site
    url: https://github.com/git/git-scm.com/issues/
    about: Please report problems with the git-scm.com site here.

  - name: Bug is about Git program itself
    url: https://git-scm.com/community
    about: Please report problems with the Git program here.

  - name: Bug is about Git for Windows
    url: https://github.com/git-for-windows/git/issues
    about: Please report problems with Git for Windows here.
```

该项目新建 Issue 的页面如下：

![ProGit2 新建 Issue](/images/tips/progit2-issue.png)

其中，只有第一个 `Bug report` 是项目里添加的模板，位于 `.github/ISSUE_TEMPLATE/bug_report.md`，内容如下：

```markdown
---
name: Bug report
about: Create a report to help us improve
title: ''
labels: ''
assignees: ''

---

<!-- Before filing a bug please check the following: -->
<!-- * There's no existing/similar bug report. -->
<!-- * This bug report is about a single actionable bug. -->
<!-- * This bug is about the Pro Git book, version 2, English language. -->
<!-- * This bug is about the book as found on the [website](https://www.git-scm.com/book/en/v2) or the pdf. -->
<!-- * If you found an issue in the pdf/epub/mobi files, you've checked if the problem is also present in the Pro Git book on the [website](https://www.git-scm.com/book/en/v2). -->

**Which version of the book is affected?**
<!-- It's important for us to know if the problem is in the source or in the tooling for the pdf/epub/mobi files. -->
<!-- Therefore, please write whether the problem is with the files, the online book, or both. -->

**Describe the bug:**
<!-- A clear and concise description of what the bug is. -->

**Steps to reproduce:**
<!-- Please write the steps needed to reproduce the bug here. -->
<!-- 1. Go to '...' -->
<!-- 2. Click on '....' -->
<!-- 3. Scroll down to '....' -->
<!-- 4. See error -->

**Expected behavior:**
<!-- A clear and concise description of what you expected to happen. -->

**Screenshots:**
<!-- If applicable, add screenshots to help explain your problem. -->

**Additional context:**
<!-- Add any other context about the problem here. -->
<!-- You can also put references to similar bugs here. -->

**Desktop:**
<!-- If you've used a desktop/laptop to access the content, please fill in this form. -->
<!-- Example: Windows 10 Home Edition, Firefox, version 66.0.2 -->
- Operating system:
- Browser/application:
- Browser/application version:

**Smartphone:**
<!-- If you've used a smartphone to access the content, please fill in this form. -->
<!-- Example: iPhone 6, iOS 12.2, Safari, version 22 -->
- Device:
- OS:
- Browser/application:
- Browser/application version:

**E-book reader:**
<!-- If you've used an e-book reader to access the content, please fill in this form. -->
<!-- Example: Amazon Kindle Paperwhite 10th generation, software update 5.11.1 -->
- Device:
- Software Update:
```

### Pull request 模板

#### Pull request 模板介绍

Pull request 模板可以放置于 `docs` 目录或者 `.github` 目录。

如果只需要单一的模板，创建 `docs/pull_request_template.md` 或者 `.github/pull_request_template.md` 文件即可。

如果需要多个模板，将模板集中放置于 `docs/PULL_REQUEST_TEMPLATE` 或者 `.github/PULL_REQUEST_TEMPLATE` 文件夹。

#### Pull request 模板实例

[Pro Git, Second Edition](https://github.com/progit/progit2) 项目的 Pull request 模板位于 `.github/pull_request_template.md`，内容如下：

```markdown
<!-- Thanks for contributing! -->
<!-- Before you start on a large rewrite or other major change: open a new issue first, to discuss the proposed changes. -->
<!-- Should your changes appear in a printed edition, you'll be included in the contributors list. -->

<!-- Mark the checkbox [X] or [x] if you agree with the item. -->
- [ ] I provide my work under the [project license](https://github.com/progit/progit2/blob/master/LICENSE.asc).
- [ ] I grant such license of my work as is required for the purposes of future print editions to [Ben Straub](https://github.com/ben) and [Scott Chacon](https://github.com/schacon).

## Changes

- 

## Context
<!--
List related issues.
Provide the necessary context to understand the changes you made.
Are you fixing an issue with this pull-request?
Use the "Fixes" keyword, to close the issue automatically after your work is merged.
Fixes #123
Fixes #456
-->
```
