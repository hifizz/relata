<div align="center">
  <img src="public/icon/128.png" alt="见微 Readsight 图标" width="80" />
  <h1>见微 Readsight</h1>
  <p><strong>双语阅读、语境查词与 AI 讲解，不离开原文。</strong></p>
  <p><a href="README.md">English</a> · <strong>简体中文</strong></p>
</div>

见微 Readsight 是一个 Chrome MV3 扩展 + 自托管 Next.js 后端，用来深度阅读英文文章。它面向"能看懂大部分、但会被个别词 / 句 / 语法卡住"的中文读者 —— 当前主要工作流是**英文 → 简体中文**。

后端自己部署、模型 API key 自备；模型请求经由后端发出，provider 密钥只留在服务端。

## 功能特性

- **页内划词查词** —— 选中单词、句子或段落，点击选区旁的小圆点 / "Look up" 指示打开可拖动浮窗：单词给出音标、释义和形态变化，长选区给出内联中文翻译；接入本地 ECDICT SQLite（可选、不随仓库分发）后还能显示 Collins 星级、牛津 3000、词频排名。
- **语境阅读动作** —— 在同一浮窗里继续追问：词义与用法、句子翻译、语法分析、改写、难点讲解，也可以转到侧边栏深入讨论。
- **语境 AI 对话** —— Chrome 侧边栏里的流式对话（DeepSeek 驱动），始终知道你当前选中了什么。
- **双语全文翻译** —— 在英文段落下方注入中文。article 模式会过滤导航等杂讯；page 模式覆盖更宽泛的页面文本。懒加载 / SPA 内容由重扫机制续翻，段落级缓存，失败块单独重试。
- **阅读布局** —— 全屏沉浸式阅读器，支持单列和双语对照双列两种排版。
- **学习记录** —— 后端保存你的划选原文、出处与上下文，记录查词与再遇见，可随时跳回原始页面。
- **安静可控 & 朗读** —— 默认手动触发，可按语言或按域名选择性开启自动翻译。浮窗、句子与短语朗读使用浏览器 `speechSynthesis`（`en-US`）；单词卡可优先播放词典音频、回退浏览器语音合成。可用音色因操作系统和浏览器而异。

## 使用方式

1. 在 Chrome 中打开一篇英文文章。
2. 划选单词或一段文字，点击选区旁出现的小圆点 / "Look up" 指示。
3. 在浮窗里查看词典卡或中文翻译，使用语法分析、改写、难点讲解等动作 —— 想深入讨论时打开侧边栏。
4. 用悬浮按钮、右键菜单或 `Alt+T`（macOS 为 `Option+T`）切换双语翻译。
5. 打开沉浸阅读器，切换专注的单列 / 双列排版。
6. 在侧边栏回顾学习记录。

## 快速开始

### 环境要求

- Node.js **>= 22.19.0**
- pnpm **10.32.1**
- 支持侧边栏（side panel）的 Chrome
- 一个专用的本地 Postgres 数据库（例如名为 `readsight`）—— 请先建好再跑迁移
- DeepSeek API key

### 安装

扩展和服务端是两个独立 package（不是 workspace），都要装依赖：

```bash
git clone https://github.com/hifizz/relata.git
cd relata
pnpm install
pnpm --dir server install
```

### 配置环境变量

在仓库根目录创建 `.env.local` —— 只放扩展要请求的公开后端地址（不要把任何 provider 密钥写进扩展）：

```ini
WXT_BACKEND_URL=http://localhost:4010
```

创建 `server/.env.local` —— 请把所有占位值替换为真实值：

```ini
DATABASE_URL=postgresql://readsight:change-me@localhost:5432/readsight
DEEPSEEK_API_KEY=your-deepseek-api-key
```

可选的兜底 provider 和其他变量见 [配置](#配置)。

### 数据库迁移

`server/scripts/migrate.mjs` 直接读 `process.env`，**不会**自动加载 `.env.local`，所以要在仓库根目录显式指定 env 文件：

```bash
pnpm --dir server exec node --env-file=.env.local scripts/migrate.mjs
```

该命令把已提交的 Drizzle 迁移应用到 `readsight` schema，无需手写 SQL。

### 运行

终端 1 —— 后端跑在 `localhost:4010`：

```bash
pnpm --dir server dev
```

终端 2 —— 扩展。构建指向本地后端的扩展生产包：

```bash
WXT_BACKEND_URL=http://localhost:4010 pnpm build
```

然后打开 `chrome://extensions`，开启**开发者模式**，点**加载已解压的扩展程序**，选择 `.output/chrome-mv3`。

开发时可用 `pnpm dev` 跑 WXT dev mode，产物在 `.output/chrome-mv3-dev`。使用生产 unpacked 产物时，改动源码后要重新构建并刷新扩展。

## 架构

严格的上下文隔离：只有 content script 接触页面 DOM；所有跨上下文消息都经 background service worker 中转；侧边栏从不直接读页面。

| 路径 | 职责 |
| --- | --- |
| `entrypoints/content/` | Content script：划词浮窗、双语注入、沉浸阅读器 |
| `entrypoints/background.ts` | 各上下文之间的消息中转与网络出口 |
| `entrypoints/sidepanel/` | 侧边栏 UI：对话、设置、学习记录 |
| `components/`、`lib/` | 共享 UI 卡片、设置、消息助手 |
| `server/src/app/api/` | Next.js API 路由：翻译、查词、对话、记录 |
| `server/src/db/`、`server/drizzle/` | Drizzle schema 与已提交的 Postgres 迁移 |

**技术栈：** 扩展端 WXT 0.20 + React 19 + TypeScript + Tailwind CSS 4 + assistant-ui；服务端 Next.js 16 + Drizzle + Postgres。

**模型：** 语境对话用 DeepSeek `deepseek-v4-pro`；翻译与模型辅助查词用 `deepseek-v4-flash`（均关闭 thinking）。翻译兜底链：DeepSeek → 智谱 GLM-4-Flash → OpenRouter。

## 常用命令

| 命令 | 说明 |
| --- | --- |
| `pnpm dev` | WXT dev mode → `.output/chrome-mv3-dev` |
| `pnpm build` | 扩展生产构建 → `.output/chrome-mv3` |
| `pnpm zip` | 打包已构建的扩展 |
| `pnpm compile` | 扩展端类型检查（`tsc --noEmit`） |
| `pnpm --dir server dev` | 后端开发服务器（4010 端口） |
| `pnpm --dir server build` / `pnpm --dir server start` | Next.js 生产构建 / 启动 |
| `pnpm --dir server lint` | ESLint |
| `pnpm --dir server exec tsc --noEmit` | 后端类型检查 |
| `pnpm --dir server db:generate` | 生成新的 Drizzle 迁移（仅改 schema 时用） |
| `pnpm --dir server db:migrate` | 应用迁移 —— 需要环境里有 `DATABASE_URL`；用法见[快速开始](#快速开始)中的 `--env-file` 命令 |

## 配置

**扩展（构建期、公开）：**

| 变量 | 说明 |
| --- | --- |
| `WXT_BACKEND_URL` | 打进扩展包的后端地址。默认 `http://localhost:4010`。 |

**服务端（`server/.env.local`）：**

| 变量 | 必填 | 说明 |
| --- | --- | --- |
| `DATABASE_URL` | 是 | 专用 Postgres 数据库的连接串。 |
| `DEEPSEEK_API_KEY` | 是 | 对话、模型辅助查词与主翻译。 |
| `ZHIPU_API_KEY` | 否 | 第一级翻译兜底（GLM-4-Flash）。 |
| `OPENROUTER_API_KEY` | 否 | 第二级翻译兜底。 |
| `OPENROUTER_MODEL` | 否 | OpenRouter 模型覆盖（默认 `meta-llama/llama-3.3-70b-instruct:free`）。 |
| `TRANSLATE_PRIMARY` | 否 | `deepseek`（默认）/ `zhipu` / `openrouter`，调整翻译链顺序。 |
| `ECDICT_PATH` | 否 | 兼容 ECDICT SQLite 文件的绝对路径。 |
| `CORS_ALLOW_ORIGIN` | 否 | API 路由的 CORS 来源（默认 `*`）。它不是鉴权手段。 |

**可选本地词典（ECDICT）：** 不随仓库分发。需要本地查词时，将含 `dict` 表、源自 ECDICT 数据的 SQLite 数据库放到 `server/data/ecdict.sqlite`，或用 `ECDICT_PATH` 指向它 —— 适配层格式见 [`server/src/lib/ecdict.ts`](server/src/lib/ecdict.ts)。该文件在首次查词时按需打开；没有它时查词退回在线词典 + 模型。词典数据版权：skywind3000/ECDICT，CC BY 4.0。

## 状态与注意事项

- **自托管、单用户开发基线。** 目前 API 路由没有鉴权和限流 —— 暴露到公网前请自行加鉴权、收紧 `CORS_ALLOW_ORIGIN` 并做好安全防护。provider 密钥绝不能进扩展。
- 划选文本 / 上下文和待翻译段落可能经你的后端转发给模型或词典服务（命中缓存时不一定调用 provider）；单词发音音频可能直接从外部音频地址获取。学习记录与翻译缓存存在 Postgres；Chrome sync 只用于扩展偏好设置。
- 仅支持 Chrome（依赖 side panel API）。动态页面或受限页面可能存在局限。
- 朗读使用浏览器语音合成，音色可用性与质量取决于操作系统 / 浏览器所装的语音。

## 文档

- [PRD.md](PRD.md) —— 产品需求文档
- [CLAUDE.md](CLAUDE.md) —— 开发与架构说明
- [docs/PROJECT_STATUS.md](docs/PROJECT_STATUS.md) —— 历史里程碑快照（顶部补充说明为现行模型路由）
- [docs/notes-translate-injection.md](docs/notes-translate-injection.md) —— 翻译注入内部实现
