# run

执行 `package.json` 中定义的脚本。

> **风险等级：write** — 任意 package script 都可能修改本地文件、调用远端服务或执行发布动作。执行前必须先审阅 `package.json` 中对应脚本；嵌入式 Rabetbase 工具暂不开放 `run`。

## 命令

```bash
# 列出所有可用脚本
rabetbase run

# 执行指定脚本
rabetbase run build
rabetbase run start
rabetbase run dev
rabetbase run preview

# preview watch 模式（并行 vite build --watch + vite preview）
rabetbase run preview --watch

# 传递额外参数
rabetbase run build -- --mode production
```

## 参数

| Flag | 类型 | 必填 | 默认 | 说明 |
|------|------|------|------|------|
| `<script>` | positional | 否 | — | package.json 中的脚本名；省略时列出所有脚本 |
| `--watch` | boolean | 否 | `false` | 仅 `preview` 脚本可用：并行执行 `vite build --watch` + `vite preview` |
| `[-- extra-args]` | positional | 否 | — | 传递给脚本的额外参数（`--` 分隔） |

## 特殊行为

- **自动检测包管理器**：按 lock 文件判断 bun / pnpm / yarn / npm
- **`start` / `dev` 脚本**：执行前自动检查 CLI 和 SDK 版本，有更新时打印升级警告
- **`preview --watch`**：并行启动 `vite build --watch` 和 `vite preview`，适合开发时预览生产构建

## 输出

- 脚本不存在时报错并列出可用脚本
- 脚本输出直接透传（`stdio: inherit`）

## 提示

- 必须在包含 `package.json` 的项目根目录执行
- 先审阅目标 script 的实际命令体和参数；不要把脚本名当成只读能力
- 所有项目脚本统一使用 `rabetbase run <script>`

## 参考

- [SKILL.md](../SKILL.md)
