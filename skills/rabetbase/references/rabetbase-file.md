# file upload / query-url

应用文件命令。参数与输出约定和 `lovrabet file` 一致，通过 rabetbase 当前登录态与应用权限执行。

## 上传文件

```bash
rabetbase file upload --file ./evidence.png --appcode app-demo --format compress
```

`upload` 是 write 命令，支持 `--dry-run`。本地文件必须存在、必须是普通文件且不能超过 50 MB。成功后保留返回的 `filePath`，用于业务字段或后续查询 URL。

## 查询访问 URL

```bash
# 短效预览 URL
rabetbase file query-url --filepath '20260717/hash-evidence.png' --appcode app-demo --format compress

# 短效下载 URL
rabetbase file query-url --filepath '20260717/hash-evidence.png' --download --appcode app-demo --format compress

# 3 年 URL，供 Markdown、HTML、富文本等只能保存 URL 的内容使用
rabetbase file query-url --filepath '20260717/hash-evidence.png' --long-term --appcode app-demo --format compress
```

## 选择原则

- 可保存文件引用时保存 `filePath`，访问时再获取短效 URL。
- 只有 Markdown、HTML 或富文本等 URL-only 内容才使用 `--long-term`。
- 使用 rabetbase 当前登录态上传应用文件；不要改用 `lovrabet file upload`，避免混用两套应用权限。

## 参考

- [SKILL.md](../SKILL.md)
- [rabetbase-issue-report.md](rabetbase-issue-report.md)
