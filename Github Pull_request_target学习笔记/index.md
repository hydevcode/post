+++
date = '2025-03-15T19:20:00+08:00'
draft = false
title = 'Github Pull_request_target学习笔记'
+++

<aside> 😀 由于需要在PR上用bot发送评论，所以可能需要用到pull_request_target,但是官方文档提示该触发条件有安全风险,于是研究下并做个笔记

</aside>

[触发工作流的事件 - GitHub 文档](https://docs.github.com/zh/actions/writing-workflows/choosing-when-your-workflow-runs/events-that-trigger-workflows#pull_request_target)

在 GitHub Actions 中，`pull_request` 和 `pull_request_target` 是两种处理 Pull Request (PR) 的触发器,以下是我的理解介绍以及对比：

# 📝 什么是pull_request?

- 当 **PR 被创建**、**更新**（新提交）或 **重新打开** 时触发工作流
- 专为 **处理 PR 贡献者的代码** 设计

|特性|说明|
|---|---|
|**代码环境**|使用 **PR 分支的代码**（即贡献者的代码）|
|**权限**|`GITHUB_TOKEN` 默认只有 **读权限**|
|**Secrets**|**无法访问仓库的加密 secrets**|
|**目标分支**|无法直接访问目标分支（如 `main`）的最新代码|

使用示例：

```bash
on:
	#触发条件设置↓
  pull_request:
    branches: [ main ]

permissions:
  contents: read  # 显式声明最低权限

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4  # 自动签出 PR 代码
```

# 📝 什么是pull_request_target?

- 专门用于 **维护者审核 PR 时** 需要 **高权限操作** 的场景

|特性|说明|
|---|---|
|**代码环境**|默认使用 **目标分支的代码**（如 `main` 分支），而非 PR 代码|
|**权限**|`GITHUB_TOKEN` 默认拥有 **写权限**（可修改仓库）|
|**Secrets**|**可以访问仓库的加密 secrets**|
|**目标分支**|直接基于目标分支的最新代码运行|

## 可能的安全风险

![](Github%20Pull_request_target学习笔记/images/Pasted%20image%2020250315192253.png)

- 来自官方文档https://docs.github.com/zh/actions/writing-workflows/choosing-when-your-workflow-runs/events-that-trigger-workflows#pull_request_target
    
    对于由 `pull_request_target` 事件触发的工作流，将授予 `GITHUB_TOKEN` 读/写存储库权限，除非指定了 `permissions`密钥并且工作流可以访问机密，即使从分支触发也是如此。 虽然工作流程在拉取请求的基础上下文中运行，但你应该确保不在此事件中检出、生成或运行来自拉取请求的不受信任代码。 此外，任何缓存都与基本分支共享相同的作用域。 为帮助防止缓存中毒，如果缓存内容可能已更改，则不应保存缓存。 有关详细信息，请参阅 GitHub Security Lab 网站上的“[保护 GitHub Actions 和工作流安全：阻止 pwn 请求](https://securitylab.github.com/research/github-actions-preventing-pwn-requests)”。
    

### 比如：

由于 `pull_request_target` 默认使用的是 `main` 分支的代码，因此无法测试 PR 中修改的内容。为了解决这个问题，可以指定使用 PR 分支的代码，但这又会引发新的问题。

```
steps:
  - uses: actions/checkout@v4
    with:
      ref: ${{ github.event.pull_request.head.sha }}  # 显式签出 PR 代码
```

当前情况： • 工作流运行在高权限环境中（可访问敏感信息） • 但执行的是来自不可信 PR 的代码 • 触发的工作流仅在提交者的分叉仓库中运行 • 攻击者可能在 PR 代码中插入恶意 CI 脚本

安全风险主要源自于 `pull_request_target` 下的工作流，因为该工作流具有读取 secrets 和写入 GITHUB_TOKEN 的权限。因此，在编写 YAML 文件时需要特别注意。

在网上搜索到的一些解决办法可以有效提高 GitHub Actions 的安全性，具体措施如下：

1. **禁用凭证持久化 (persist-credentials: false)**：当使用 `actions/checkout` 动作时，默认情况下，GitHub 会将 GITHUB_TOKEN 写入本地的 Git 配置文件（存储在 `.git/config` 中）。这意味着后续的所有 Git 操作（如 `push` 或 `pull`）都会自动使用这个令牌进行身份验证。然而，为了增强安全性并避免潜在的风险，可以通过设置 `persist-credentials: false` 来主动删除这些凭证信息。这一步骤确保了即使代码库中存在敏感操作，也不会因为误用或泄露令牌而引发安全问题。
2. **最小化权限配置**：在 GitHub Actions 的工作流文件中，可以使用 `permissions` 块来显式声明每个步骤所需的权限。通过这种方式，可以确保每个作业只拥有完成任务所必需的最低权限。例如，如果某个作业只需要读取仓库内容，则不应授予其写入权限。这种做法不仅减少了不必要的权限暴露，还降低了因权限滥用而导致的安全风险。
3. **隔离敏感操作**：为了进一步提升安全性，建议将签出代码的操作与任何涉及写入的操作分离到不同的作业（job）中。这样做可以有效防止在同一环境中同时执行读取和写入操作，从而减少意外修改或恶意攻击的可能性。通过将不同类型的任务分隔开，可以更好地控制和监控每个作业的行为，确保整个工作流的安全性和稳定性。

通过以上这些方法，不仅可以提高 GitHub Actions 的安全性，还能帮助开发者更清晰地管理和控制各个作业之间的权限和操作，从而构建更加健壮和可靠的自动化流程。

# WorkFlow_Run

在官方的一篇文章中，官方提供了一种解决办法,而且给出的示例也刚好符合实现在PR发送评论的需求

- [https://securitylab.github.com/resources/github-actions-preventing-pwn-requests/](https://securitylab.github.com/resources/github-actions-preventing-pwn-requests/)
    
    与`pull_request_target`一起，引入了一个新的触发器`workflow_run`，以支持需要构建不受信任的代码并且还需要写入权限来更新 PR（例如代码覆盖率结果或其他测试结果）的方案。要以安全的方式执行此作，必须通过 `pull_request` 触发器处理不受信任的代码，以便将其隔离在非特权环境中。然后，处理 PR 的工作流应将任何结果（如代码覆盖率或失败/通过的测试）存储在构件中并退出。然后，以下工作流程从 `workflow_run` 开始，在该位置被授予对目标存储库的写入权限和对存储库密钥的访问权限，以便它可以下载项目并对存储库进行任何必要的修改，或与需要存储库密钥（例如 API 令牌）的第三方服务交互。
    

这里用的是官方的一个示例，其中分别有两个工作流，关键注意这两个工作流的触发事件一个是`pull_request`一个是`workflow_run`

当贡献者提交PR时，会触发相应的CI流程。由于`Receive PR`工作流使用的是`pull_request`事件，默认情况下无法访问机密信息且不具备写入权限。该工作流运行完毕后，会将处理结果及PR编号存储在构件中。随后，`workflow_run`检测到`Receive PR`工作流已结束，于是触发该工作流并开始执行其定义的任务。

```bash
name: Receive PR

# read-only repo token
# no access to secrets
on:
  pull_request:

jobs:
  build:
    runs-on: ubuntu-latest

    steps:        
      - uses: actions/checkout@v2

      # imitation of a build process
      - name: Build
        run: /bin/bash ./build.sh

      - name: Save PR number
        run: |
          mkdir -p ./pr
          echo ${{ github.event.number }} > ./pr/NR
      - uses: actions/upload-artifact@v2
        with:
          name: pr
          path: pr/

```

`Comment on the pull request`工作流将`Receive PR`的构件下载回来，然后根据PR号对相应的PR发送评论

```bash
name: Comment on the pull request

# read-write repo token
# access to secrets
on:
  workflow_run:
    workflows: ["Receive PR"]
    types:
      - completed

jobs:
  upload:
    runs-on: ubuntu-latest
    if: >
      github.event.workflow_run.event == 'pull_request' &&
      github.event.workflow_run.conclusion == 'success'
    steps:
      - name: 'Download artifact'
        uses: actions/github-script@v3.1.0
        with:
          script: |
            var artifacts = await github.actions.listWorkflowRunArtifacts({
               owner: context.repo.owner,
               repo: context.repo.repo,
               run_id: ${{github.event.workflow_run.id }},
            });
            var matchArtifact = artifacts.data.artifacts.filter((artifact) => {
              return artifact.name == "pr"
            })[0];
            var download = await github.actions.downloadArtifact({
               owner: context.repo.owner,
               repo: context.repo.repo,
               artifact_id: matchArtifact.id,
               archive_format: 'zip',
            });
            var fs = require('fs');
            fs.writeFileSync('${{github.workspace}}/pr.zip', Buffer.from(download.data));
      - run: unzip pr.zip

      - name: 'Comment on PR'
        uses: actions/github-script@v3
        with:
          github-token: ${{ secrets.GITHUB_TOKEN }}
          script: |
            var fs = require('fs');
            var issue_number = Number(fs.readFileSync('./NR'));
            await github.issues.createComment({
              owner: context.repo.owner,
              repo: context.repo.repo,
              issue_number: issue_number,
              body: 'Everything is OK. Thank you for the PR!'
            });

```

<aside> 💡

有关题目或文章的问题，欢迎您在底部评论区留言，一起交流~

</aside>