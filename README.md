# Podcast Cut · Web App

把 [podcastcut Claude Code skill](https://github.com/chenyusi/podcastcut-skills) 转成
浏览器 web app，让不会装 CLI 的播客主播也能用。用户自带 Aliyun + Anthropic API Key，
直接付外部 API 费用，不依赖 Claude 订阅。

## 跟 `podcastcut-skills` 仓库的关系

| 仓库 | 角色 | 受众 |
|---|---|---|
| [`podcastcut-skills`](https://github.com/chenyusi/podcastcut-skills) | 6 个 Claude Code skill 本体（剪播客 / 后期 / 质检 / 安装 / 音质处理 / 自进化） | 装了 Claude Code CLI 的开发者 |
| `podcastcut-web`（本仓库） | Next.js web app + 设计原型 | 普通播客主播（浏览器即用） |

**核心架构转型**：Claude 从 skill 模式下的「流程编排者」变成 web app 下的「分析引擎」。
skill 的 prompt 和方法论将被抽出来变成后端的 system prompt，Node.js 后端按预设
pipeline 跑，按需调 Anthropic API。

## 当前状态（2026-05-10）

- ✅ 设计原型：屏 1 / 2 / 3 已锁，屏 4–8 待出
- ⏳ Next.js 源码：未开始（5/16 后由响歌歌启动）
- ⏳ 后端 pipeline：未开始

## 仓库结构

```
podcastcut-web/
├── design/                          ← 全部设计资产，源头在这里
│   ├── MVP规格.md                   ← 8 屏 user flow + 数据契约（v2，2026-05-09 锁）
│   ├── prototype_主流程_v0.1.html  ← 屏 1→2→3 串联原型（含 topbar / 进度条 / vibe selector）
│   ├── prototype_屏1_节目档案.html ← 屏 1 单屏（早期版本，已并入主流程）
│   ├── prototype_屏2_上传.html     ← 屏 2 单屏（早期版本，已并入主流程）
│   ├── prototype_屏3_复盘.html     ← 屏 3 单屏（早期版本，已并入主流程）
│   ├── prototype_复盘驱动.html     ← 早期探索（屏 2 磁带动画来源）
│   ├── prototype_wow.html          ← 早期探索（视觉语言起点）
│   ├── userflow_v2_复盘驱动.md     ← 用户流第 2 版（复盘驱动模型）
│   └── userflow草稿_碰头用.md      ← 用户流早期草稿
├── README.md
└── .gitignore                       ← 已预置 Next.js / node / 音频文件忽略规则
```

未来响歌歌加入 Next.js 源码时，平级新建 `app/` `lib/` `components/` `public/` 即可。

## 视觉 DNA（已锁）

- **色板**：cream `#EFEDE6`（最近从 `#ECEEE8` 调浅，去掉绿调）+ sage `#B8C0A0` +
  sage-deep `#8B9275` + 黑 `#1A1A1A`
- **字体**：Noto Serif SC 标题 / system sans 正文 / SF Mono 数字标签
- **品牌**：左上 Podcast Cut wordmark + 5 条波形 logo（黑灰，中间柱有缺口暗示 "cut"）
- **vibe chip**：默认 demo 用「明心」mustard `#D4A95B`，运行时应从
  `localStorage.current_vibe` 读取

## 全套设计 DNA（所有屏复用）

- **顶部 sticky bar**：Podcast Cut logo + 7 步进度条 + 本期 vibe chip
- **进度条 7 步**：上传 / 复盘 / 说话人 / 处理 / 粗剪 / 精剪 / 后期
- **「覆盖档案默认」徽章**：用 vibe 色 outline，显式标注用户覆盖

## 屏间跳转链

```
屏 1 老用户主页
  └─ 点左侧文档「开始新剪辑」
       └─ 屏 1V vibe selector
            └─ 自动跳屏 2 上传
                 └─ CTA → 屏 3 复盘
                      └─ 提交 → 屏 4 说话人确认（待出）
                           └─ 屏 5 处理动画
                                └─ 屏 6 粗剪审核（核心三栏）
                                     └─ 屏 7 精剪处理
                                          └─ 屏 8 精剪完成预览
```

## MVP 8 屏

1. **节目档案**（首次 / 老用户分支）
2. **上传**
3. **录后复盘**（含隐私确认）
4. **说话人确认**
5. **处理动画**
6. **粗剪审核**（双模式 + Glossary 校正面板，**不要** chat box）
7. **后台剪辑**（智能时间预估）
8. **精剪完成预览**（placeholder）

## 技术栈

- **前端**：Next.js（App Router）
- **部署**：Railway / Fly.io（**不是** Vercel — FFmpeg / SSE / 大文件不兼容）
- **API Key**：MVP 阶段存 `localStorage`，后端不持久化
- **音频处理**：FFmpeg
- **转录**：Aliyun DashScope（FunASR）
- **AI 分析**：Anthropic Claude API

## 协作分工

- **阿司（chenyu@hikari-lab.co）**：UX / 产品 / 设计原型（在 `design/` 目录）
- **响歌歌**：后端 pipeline + Next.js 源码（在未来的 `app/` `lib/`）
- 阿司 5/16–5/31 休假期间响歌歌独立迭代后端，6/1 联调

## 给响歌歌的 onboarding

1. 先读 `design/MVP规格.md`（v2）— 数据契约 + 每屏输入输出 JSON
2. 浏览器打开 `design/prototype_主流程_v0.1.html` — 屏 1→2→3 实际能点
3. 老的单屏 prototype（屏 1/2/3 独立 HTML）作为对照参考即可，**主流程** HTML 才是当前 source of truth
4. skill 仓库 `podcastcut-skills/剪播客/SKILL.md` 是粗剪逻辑的方法论来源，要抽出来变成后端 system prompt
