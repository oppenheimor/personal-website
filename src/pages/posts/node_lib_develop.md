---
title: Node 库开发
date: 2024-11-27T00:00:00.000+00:00
lang: zh
author: 沈佳棋
---

## 标准的库产物：

### package.json
<!-- package.json -->
```json
{
  "types": "./dist/index.d.ts",
  "main": "./dist/index.cjs",
  "module": "./dist/index.mjs",
  "exports": {
    ".": {
      "types": "./dist/index.d.ts",
      "import": "./dist/index.mjs",
      "require": "./dist/index.cjs"
    }
  },
  "files": [
    "dist"
  ],
}
```

### dist 内容

```
dist
├─ index.cjs             # 用于 commonjs 规范
├─ index.mjs             # 用于 esm 规范
└─ index.d.ts            # 类型声明
```

## 工具
可以使用字节的 `rslib` 工具用于构建标准 node 库：https://lib.rsbuild.dev/