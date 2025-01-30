# OpenAI 代码评审工具配置指南

本工具利用 OpenAI 的能力来自动进行代码评审，并通过微信公众号推送评审结果。

## 目录

- [前置条件](#前置条件)
- [配置步骤](#配置步骤)
  - [1. 工作流配置](#1-工作流配置)
  - [2. GitHub Secrets 配置](#2-github-secrets-配置)
  - [3. 微信公众平台配置](#3-微信公众平台配置)
- [使用说明](#使用说明)
- [常见问题](#常见问题)

## 前置条件

- GitHub 仓库的管理员权限
- 微信公众平台账号（已认证）

## 配置步骤

### 1. 工作流配置

1. 在项目根目录创建以下目录结构：

   ```
   .github/
   └── workflows/
       └── main-remote-jar.yml
   ```

2. 将提供的工作流配置复制到 `main-remote-jar.yml` 文件中。该工作流将在以下情况触发：

   - 向 master 分支推送代码时
   - 创建或更新针对 master 分支的 Pull Request 时

   ```yml
   name: OpenAI Code Review
   
   on:
     push:
       branches:
         - master
     pull_request:
       branches:
         - master
   
   jobs:
     code-review:
       runs-on: ubuntu-latest
   
       steps:
         - name: Checkout repository
           uses: actions/checkout@v3
           with:
             fetch-depth: 0
   
         - name: Set up JDK 11
           uses: actions/setup-java@v3
           with:
             distribution: 'temurin'
             java-version: '11'
   
         - name: Download SDK JAR
           run: |
             mkdir -p ./libs
             wget -O ./libs/openai-code-review-sdk-1.0.jar \
               https://github.com/Qwh100/openai-code-review-log/releases/download/v1.0/openai-code-review-sdk-1.0.jar
   
         - name: Set Push Event Variables
           if: github.event_name == 'push'
           run: |
             echo "BEFORE_COMMIT=${{ github.event.before }}" >> $GITHUB_ENV
             echo "AFTER_COMMIT=${{ github.event.after }}" >> $GITHUB_ENV
             echo "BRANCH_NAME=${GITHUB_REF#refs/heads/}" >> $GITHUB_ENV
             echo "TARGET_BRANCH=origin/${GITHUB_REF#refs/heads/}" >> $GITHUB_ENV
   
         - name: Set PR Event Variables
           if: github.event_name == 'pull_request'
           run: |
             echo "BRANCH_NAME=${{ github.head_ref }}" >> $GITHUB_ENV
             echo "TARGET_BRANCH=origin/${{ github.base_ref }}" >> $GITHUB_ENV
             echo "BEFORE_COMMIT=0000000000000000000000000000000000000000" >> $GITHUB_ENV
             echo "AFTER_COMMIT=0000000000000000000000000000000000000000" >> $GITHUB_ENV
   
         - name: Set Common Variables
           run: |
             echo "REPO_NAME=${GITHUB_REPOSITORY##*/}" >> $GITHUB_ENV
             echo "COMMIT_AUTHOR=$(git log -1 --pretty=format:'%an <%ae>')" >> $GITHUB_ENV
             echo "COMMIT_MESSAGE=$(git log -1 --pretty=format:'%s')" >> $GITHUB_ENV
   
         - name: Run Code Review
           run: java -jar ./libs/openai-code-review-sdk-1.0.jar
           env:
             GITHUB_EVENT_NAME: ${{ github.event_name }}
             BEFORE_COMMIT: ${{ env.BEFORE_COMMIT }}
             AFTER_COMMIT: ${{ env.AFTER_COMMIT }}
             TARGET_BRANCH: ${{ env.TARGET_BRANCH }}
             COMMIT_PROJECT: ${{ env.REPO_NAME }}
             COMMIT_BRANCH: ${{ env.BRANCH_NAME }}
             COMMIT_AUTHOR: ${{ env.COMMIT_AUTHOR }}
             COMMIT_MESSAGE: ${{ env.COMMIT_MESSAGE }}
             GITHUB_TOKEN: ${{ secrets.CODE_TOKEN }}
             GITHUB_REVIEW_LOG_URI: ${{ secrets.CODE_REVIEW_LOG_URI }}
             WEIXIN_APPID: ${{ secrets.WEIXIN_APPID }}
             WEIXIN_SECRET: ${{ secrets.WEIXIN_SECRET }}
             WEIXIN_TOUSER: ${{ secrets.WEIXIN_TOUSER }}
             WEIXIN_TEMPLATE_ID: ${{ secrets.WEIXIN_TEMPLATE_ID }}
             MODEL_NAME: ${{ secrets.MODEL_NAME }}
   ```

### 2. GitHub Secrets 配置

在仓库的 Settings -> Secrets and variables -> Actions 中添加以下 secrets：

| Secret 名称         | 说明                                                         |
| ------------------- | ------------------------------------------------------------ |
| CODE_TOKEN          | GitHub 个人访问令牌，用于访问仓库                            |
| CODE_REVIEW_LOG_URI | 代码评审日志服务的 URI 如(https://github.com/Qwh100/openai-code-review-log) |
| MODEL_NAME          | 使用的 OpenAI 模型名称（目前可选值"DeepSeek"或"ChatGLM"）    |
| WEIXIN_APPID        | 微信公众号 appID                                             |
| WEIXIN_SECRET       | 微信公众号 appsecret                                         |
| WEIXIN_TOUSER       | 接收消息的用户 OpenID                                        |
| WEIXIN_TEMPLATE_ID  | 消息模板 ID                                                  |

如
![image](https://github.com/user-attachments/assets/8bdd9f89-af46-4dc2-ad3f-ed41b61abcf0)


### 3. 微信公众平台配置

1. 登录[微信公众平台](https://mp.weixin.qq.com/)

2. 获取接口配置信息：

   - 获取 appID 和 appsecret（测试号信息）
   - 将这些值添加到 GitHub Secrets 中

3. 配置消息模板：

   - 进入"模板消息接口"

   - 新建测试模板，使用以下格式：

     ```
     项目：{{repo_name.DATA}}
     分支：{{branch_name.DATA}}
     作者：{{commit_author.DATA}}
     说明：{{commit_message.DATA}}
     ```

4. 获取用户 OpenID： （用户微信扫描测试公众号二维码可获取）

   - 可通过公众号的用户管理功能获取
   - 将 OpenID 添加到 GitHub Secrets 的 WEIXIN_TOUSER 中

如
![image](https://github.com/user-attachments/assets/2dcff054-182f-4b27-a4d5-37809dd4653d)

## 使用说明

配置完成后，系统将自动在以下情况触发代码评审：

1. 当代码推送到 master 分支时
2. 当创建或更新针对 master 分支的 Pull Request 时

评审结果将通过微信公众号推送给配置的接收用户。

## 常见问题

1. **评审没有触发？**
   - 检查工作流文件是否正确配置
   - 确认 GitHub Actions 是否启用
   - 查看 Actions 日志了解详细错误信息

2. **没有收到微信通知？**
   - 确认微信公众号配置是否正确
   - 验证模板消息权限是否开启
   - 检查用户 OpenID 是否正确

3. **评审结果不准确？**
   - 考虑调整使用的 OpenAI 模型
   - 检查代码提交的格式是否规范

## 注意事项

- 请确保 GitHub Token 具有足够的权限
- 定期检查 GitHub Actions 的运行状态
- 注意 OpenAI API 的使用配额
- 保护好各项密钥信息

