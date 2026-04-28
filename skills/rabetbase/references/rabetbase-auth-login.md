# auth login

通过浏览器完成 OAuth 登录，并把登录态 cookie 写入本机配置目录。

## 命令

```bash
# 交互式登录：CLI 会询问是否开始登录，并自动打开浏览器
rabetbase auth login

# 非交互 Agent 登录：不打开本机浏览器，打印授权链接给用户
rabetbase auth login --yes

# 查看帮助
rabetbase auth login --help
```

## Agent 执行规则

- 在非交互环境、Coding Agent、CI 或无法打开浏览器的终端中，使用 `rabetbase auth login --yes`。
- 命令会启动本地回调服务，并打印登录 URL；Agent 应把该 URL 原样发给用户，让用户在自己的浏览器中打开并完成授权。
- 登录 URL 有效期为 **10 分钟**。过期后重新执行 `rabetbase auth login --yes` 获取新链接。
- 不要让 Agent 自己访问登录 URL，也不要要求用户提供账号密码或 cookie。
- 如果命令提示已有有效 session，则无需重复登录。

## 行为

1. 检查本机是否已有有效 session。
2. 无有效 session 时启动本地 OAuth 回调服务。
3. 交互式模式会自动打开浏览器；非交互 `--yes` 模式只打印登录 URL。
4. 用户授权成功后，CLI 保存 cookie 并输出登录成功。
5. 10 分钟内未完成授权时，登录服务退出，需要重新执行命令。

## 常见错误

```text
Browser login cannot run in non-interactive mode.
```

说明当前环境是非交互模式，但没有传 `--yes`。改用：

```bash
rabetbase auth login --yes
```

## 后续

登录完成后，通常继续执行：

```bash
rabetbase app remote --format compress
rabetbase app list --format compress
```
