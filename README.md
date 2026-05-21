# dissect

一个 Claude Code 自定义命令，让你以「功能」而非「文件」的视角理解代码库。

- `/dissect` — 扫描项目，列出所有功能模块
- `/dissect <功能名>` — 追踪完整调用链，提取所有相关代码，并写入 `.dissect/<功能名>/` 文件夹，直接拷走即可在其他项目复用

---

## 效果演示

```
/dissect
```

输出项目功能图谱：

```
# 项目特性图谱

## 核心特性
- auth / login
- user management
- payment

## 支持模块
- logger
- config
- error handling

## 外部集成
- stripe / s3 / redis

---
▎ 运行 /dissect <功能名> 提取某一特性的完整代码
```

---

```
/dissect frida-injection
```

自动完成：
1. 扫描整个代码库，定位所有相关文件
2. 追踪完整调用链（CLI → 进程定位 → 脚本加载 → 注入 Hook → 版本配置）
3. 输出结构化报告，标注每个代码块的文件路径和行号
4. 发现潜在 Bug 和迁移注意事项（Gotchas）
5. 将所有代码写入 `.dissect/frida-injection/` 文件夹

输出文件结构：

```
.dissect/
└── frida-injection/
    ├── index.md                          ← 迁移指南
    ├── src_index.ts                      ← 部分提取（调用链入口）
    ├── src_cli.ts                        ← 部分提取（CLI 参数）
    ├── src_logger.ts                     ← 部分提取（日志通道）
    ├── frida_hook.js                     ← 完整文件
    └── frida_config_addresses.19823.json ← 完整文件
```

---

## 安装

**前提：** 已安装 [Claude Code](https://claude.ai/code)

```bash
# 在你的项目根目录下
mkdir -p .claude/commands
curl -o .claude/commands/dissect.md \
  https://raw.githubusercontent.com/Sharky-shark-Blue/dissect/main/dissect.md
```

或者手动下载 `dissect.md` 放到 `.claude/commands/dissect.md`。

---

## 使用方法

在项目根目录打开 Claude Code：

```bash
claude
```

然后输入：

| 命令 | 效果 |
|---|---|
| `/dissect` | 列出项目所有功能模块 |
| `/dissect login` | 提取登录相关的全部代码 |
| `/dissect payment` | 提取支付相关的全部代码 |
| `/dissect <任意功能名>` | 提取该功能的全部代码 |

提取完成后，`.dissect/<功能名>/` 文件夹即可直接拷贝到目标项目。

---

## 输出说明

### 代码文件命名规则

原始路径的分隔符替换为 `_`，保留扩展名：

```
src/services/auth.ts  →  src_services_auth.ts
frida/hook.js         →  frida_hook.js
```

### 文件头注释

完整提取：
```js
// dissect: full file — src/utils/jwt.js
```

部分提取：
```js
// dissect: extracted from src/index.ts (lines 139–246)
// ⚠ 这是部分提取，迁移时需要结合原文件上下文
```

### index.md 迁移指南

每次提取都会自动生成 `index.md`，包含：
- 文件对照表（提取文件 ↔ 原始路径）
- 依赖安装命令
- 迁移步骤
- Gotchas（潜在问题和注意事项）

---

## 适用场景

- **功能迁移** — 把老项目的某个模块搬到新项目
- **代码审查** — 快速理解一个功能涉及哪些文件
- **重构准备** — 在动刀之前搞清楚调用链
- **新人上手** — 用功能名而不是文件名来学习项目

---

## 支持的语言

JavaScript / TypeScript / Python / Go / Java / Ruby / PHP / C# 及任何可以用 `grep` 搜索的文本代码。

---

## License

MIT
