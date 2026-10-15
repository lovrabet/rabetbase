# codegen sdk

根据 Dataset 操作定义生成 TypeScript SDK 调用代码。

## 命令

```bash
rabetbase codegen sdk --alias order --operation filter --format json
rabetbase codegen sdk --code 097b7361b76c42bcb12b923fa5a08861 --operation create --format json
rabetbase codegen sdk --alias productSku --operation getOne --skip-imports --format json > /tmp/getSku-result.json
node -e 'const fs=require("node:fs"); const r=JSON.parse(fs.readFileSync(process.argv[1],"utf8")); if(r.ok===false || typeof r.data?.code!=="string") throw new Error("Code generation failed"); fs.writeFileSync(process.argv[2], r.data.code+"\n");' /tmp/getSku-result.json ./src/api/getSku.ts
```

## 参数

| Flag | 类型 | 必填 | 默认 | 说明 |
|------|------|------|------|------|
| `--code <code>` | string | 与 alias 二选一 | — | Dataset code（32 位 hex UUID） |
| `--alias <name>` | string | 与 code 二选一 | — | `api.ts` 中定义的别名 |
| `--operation <name>` | string | 是 | — | 操作名（`filter` `getOne` `create` `update` `delete`） |
| `--skip-imports` | boolean | 否 | — | 不包含 import 语句 |
| `--format <fmt>` | string | 否 | `compress` | 标准信封输出格式 |

## 输出

输出保持 CLI 标准 JSON 信封，生成代码位于 `data.code`。写入 TypeScript 文件前提取该字段；直接重定向 stdout 会把整个信封写入文件。

## 提示

- 生成前先用 `dataset operations --operation <name>` 确认操作存在
- `--skip-imports` 适合追加到已有文件
- 生成代码是调用骨架，需补齐真实参数和项目 client；对象参数使用 `getOne({ id })`、`update({ id, ...fields })`、`delete({ id })`

## 参考

- [SKILL.md](../SKILL.md)
