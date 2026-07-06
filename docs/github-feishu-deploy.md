# GitHub Actions + 飞书部署说明

本文说明如何使用 GitHub Actions 定时运行 TrendRadar，并把筛选后的热点内容推送到飞书群机器人。

请不要把飞书 Webhook 明文写入仓库。GitHub Actions 部署时，Webhook 必须保存到 GitHub Secret，名称固定为 `FEISHU_WEBHOOK_URL`。

## 1. Fork 或使用仓库

1. 打开 TrendRadar 仓库。
2. 点击右上角 **Fork**，复制到自己的 GitHub 账号或组织。
3. 进入 Fork 后的仓库。
4. 确认仓库内存在 `.github/workflows/crawler.yml`、`config/config.yaml` 和 `pyproject.toml`。

如果你直接维护原仓库，也可以在原仓库中完成下面的 Secret 配置。

## 2. 创建飞书自定义机器人

1. 打开需要接收消息的飞书群。
2. 进入群设置，找到群机器人或群助手。
3. 添加 **自定义机器人**。
4. 设置机器人名称，例如 `TrendRadar`。
5. 按你的组织安全要求配置关键词、签名或 IP 白名单。

如果启用了关键词校验，请确保 TrendRadar 推送内容能包含对应关键词，或改用飞书允许的其他安全方式。

## 3. 复制飞书 Webhook

1. 创建机器人后，复制飞书提供的 Webhook 地址。
2. Webhook 是敏感凭证，只在 GitHub Secret 页面粘贴。
3. 不要把它写入 `config/config.yaml`、README、Issue、提交记录或截图。

## 4. 添加 GitHub Secret

1. 打开你的 GitHub 仓库。
2. 进入 **Settings**。
3. 进入 **Secrets and variables**。
4. 点击 **Actions**。
5. 点击 **New repository secret**。
6. 填写：

```text
Name: FEISHU_WEBHOOK_URL
Value: 飞书机器人 Webhook
```

7. 点击 **Add secret** 保存。

Secret 名称必须完全一致：`FEISHU_WEBHOOK_URL`。大小写或拼写不同，TrendRadar 都无法读取。

## 5. 配置文件检查

项目真实配置文件是 `config/config.yaml`。

飞书推送对应配置项是：

```yaml
notification:
  enabled: true
  channels:
    feishu:
      webhook_url: ""
```

GitHub Actions 部署时保持 `webhook_url` 为空即可。项目代码会优先读取环境变量 `FEISHU_WEBHOOK_URL`，该环境变量由 `.github/workflows/crawler.yml` 从 GitHub Secret 注入。

## 6. 手动运行 GitHub Actions

1. 打开仓库的 **Actions** 页面。
2. 选择 **TrendRadar Crawler**。
3. 点击 **Run workflow**。
4. 选择要运行的分支。
5. 再次点击 **Run workflow** 确认。

该工作流也会自动定时运行：

- 北京时间 08:00
- 北京时间 20:00

GitHub Actions 使用 UTC cron，所以对应配置是：

```yaml
- cron: "0 0 * * *"
- cron: "0 12 * * *"
```

## 7. 查看运行日志

1. 打开 **Actions**。
2. 点击最近一次 **TrendRadar Crawler** 运行记录。
3. 打开 `crawl` 任务。
4. 优先查看这些步骤：

- **Install dependencies**
- **Verify required files**
- **Validate Feishu secret**
- **Run crawler**

## 8. 如何判断部署成功

部署成功通常满足以下条件：

- **TrendRadar Crawler** 工作流运行结果是绿色对勾。
- **Validate Feishu secret** 步骤通过。
- **Run crawler** 步骤执行完成。
- 飞书群收到 TrendRadar 推送消息。

如果没有推送消息，但日志显示运行成功，也可能是本次没有匹配到关键词或没有新增热点内容。

## 9. 常见错误排查

### Secret 未配置

现象：`Validate Feishu secret` 步骤报错，提示 `FEISHU_WEBHOOK_URL is not configured`。

处理：进入 **Settings → Secrets and variables → Actions**，新增 Secret。

### Secret 名称错误

现象：你已经添加了 Secret，但工作流仍提示未配置。

处理：确认名称是 `FEISHU_WEBHOOK_URL`，不要写成 `FEISHU_WEBHOOK`、`FEISHU_URL` 或其他名字。

### 项目入口错误

本项目真实入口是 Python 包入口：

```bash
uv run python -m trendradar
```

如果日志显示找不到模块，优先检查依赖安装是否成功，以及仓库结构是否完整。

### 依赖安装失败

本项目使用 `uv` 和 `uv.lock` 管理依赖。工作流安装命令是：

```bash
uv sync --frozen --no-dev
```

如果失败，优先查看 **Install dependencies** 步骤。常见原因包括 `uv.lock` 与 `pyproject.toml` 不一致，或 Python 版本不满足项目要求。

### 飞书未收到消息

优先检查：

- **Validate Feishu secret** 是否通过。
- **Run crawler** 是否有飞书发送失败日志。
- 飞书机器人 Webhook 是否复制完整，是否已经失效或被重新生成。
- 飞书机器人安全设置是否拦截消息。
- `config/config.yaml` 中 `notification.enabled` 是否为 `true`。
- 本次是否确实匹配到了关键词或新增内容。

### 飞书机器人 Webhook 失效

如果飞书侧重新生成、删除或禁用了机器人，原来的 Webhook 会失效。

处理：在飞书重新复制最新 Webhook，并更新 GitHub Secret `FEISHU_WEBHOOK_URL`。

### 飞书机器人安全策略拦截

如果飞书机器人启用了关键词、签名或 IP 白名单，消息可能被飞书拒绝。

处理：检查飞书机器人安全设置。若使用关键词校验，请确保推送内容包含对应关键词。

### 没有匹配到关键词

TrendRadar 会按配置筛选热点内容。如果日志显示没有匹配内容，可以检查 `config/config.yaml` 中的关键词、RSS、AI 筛选和展示区域配置。

### GitHub Actions 定时任务有延迟

GitHub 的 `schedule` 不是精确到秒的任务系统，可能会延迟几分钟甚至更久。可以先用 **Run workflow** 手动测试，确认配置正确。

### Python 版本不匹配

本项目 `pyproject.toml` 要求 Python `>=3.12`。工作流使用 Python 3.12。

处理：不要改成 Python 3.11，否则可能导致依赖安装或运行失败。
