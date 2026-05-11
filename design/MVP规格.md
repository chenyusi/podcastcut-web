# Podcastcut Web App — MVP 规格

**版本：** v2 · 2026-05-09
**协作：** 阿司（产品 / UX / UI）· 响歌歌（后端 / 系统集成）
**用途：** 这份文档是两人共同维护的 source of truth。所有屏的视觉、文案、数据契约改动都在这份 doc 里同步。

---

## 0. 总览

把现在跑在 Claude Code 里的 podcastcut skills，做成一个 web app：用户在浏览器里走完「上传 → 录后复盘 → 说话人确认 → 粗剪审核 → 精剪 → 完成」全流程，用自己的 API Key 直接调外部服务。

**MVP 阶段假设：**
- 用户一口气从录音剪到精剪完成，不分阶段保存
- API Key 用 placeholder（前端 mock 数据驱动），后端逻辑由响歌歌后续接入
- 单用户场景，没有账号系统，节目档案存 localStorage

---

## 1. 设计原则

视觉、UX、文案三方面共同遵守的调性。后续所有屏都引用这一节。

### 1.1 视觉

- **色调：** Figmint 暖白调（cream `#F3F0EA` / sage `#C5CBAD` / 黑 `#1A1A1A`），延续 `prototype_复盘驱动.html` 和 `review_roughcut.html` 的色板
- **字体：** Noto Serif SC 做标题（编辑质感）+ -apple-system / PingFang SC 做正文 + SF Mono 做小标签和数字
- **留白：** 充足且有目的，预留位置给后续插入动画或插画（不是空着，是"等位置"）
- **重点突出：** 每屏一个最大的视觉锚点（数字 / 标题 / 卡片），其他元素都退后

### 1.2 UX

- **像剪辑师，不像编辑器：** 我们不给用户无限自由编辑，每一步都是经过专业判断后的引导。少而精的决策点 > 多而杂的控件
- **进度可见：** 用户进了上传屏开始，就有持续的 progress indicator，让他知道粗剪剪什么、精剪剪什么、现在到哪了
- **专业引导：** 在阶段切换处给出专业建议（如粗剪→精剪之间提醒"先跟团队确认内容再继续"）
- **第一印象 = 整个产品的赌注：** 屏 1 必须有"惊艳贴心"的感觉，定整个产品的视觉 DNA

### 1.3 文案

- 中文为主，技术词汇英文（如 API Key / glossary）
- 第二人称 "你"，平实但有温度
- 不卖弄技术，专注用户得到什么

---

## 2. 产品架构

### 2.1 从 skill 模式到 web 模式

| | Skill 模式（现状） | Web 模式（目标） |
|---|---|---|
| 谁编排流程 | Claude（边聊边决定） | Node.js 后端按 pipeline 跑 |
| 用户入口 | Claude Code 终端 | 浏览器 |
| AI 调用方式 | Claude 调自己的工具链 | 后端通过 Anthropic API 调 Claude |
| Skill 文件的角色 | Claude 读完决定怎么做 | 抽出 prompt + 方法论变成后端 system prompt |
| 计费 | Claude 订阅 | 用户自己的 API Key 直接被后端调用 |

### 2.2 6 个 skill 在 web app 里的位置

| Skill | Web 后端模块 | 触发屏 |
|---|---|---|
| `剪播客` | `lib/transcribe.ts` + `lib/analyze.ts` + `lib/cut.ts` | 屏 5 |
| `质检` | `lib/qualityCheck.ts` | 屏 7 |
| `音质处理` | `lib/audioFix.ts` | 屏 7 |
| `后期` | `lib/postProcess.ts` | 屏 7 |
| `自进化` | `lib/preferences.ts`（MVP 简化版只更 localStorage） | 屏 1（保存档案）+ 屏 6（用户决策回流） |
| `安装` | （不需要，web app 没有安装步骤） | - |

---

## 3. MVP 范围

### 3.1 必做（本次交付）

- [x] 屏 1 节目档案（新用户建档 + 老用户摘要+编辑入口 + 主菜单 + glossary 输入）
- [x] 屏 2 上传 + 初始设置
- [x] 屏 3 录后复盘
- [x] 屏 4 说话人确认
- [x] 屏 5 处理动画
- [x] 屏 6 粗剪审核（编辑/预览双模式 + glossary 校正面板 + 重新 cut 入口 + 进入精剪 CTA）
- [x] 屏 7 后台精剪 + 倒计时（带预计时间）
- [x] 屏 8 精剪完成预览（placeholder 页面，不实装功能）

### 3.2 不做（后续版本）

- 用户账号系统（先 localStorage）
- 屏 6 的 chat box 自然语言指令（移到精剪阶段或之后）
- 多档播客管理
- 高光片段生成
- 团队协作 / 多主播复盘合并
- 移动端适配
- 屏 8 内的实际精剪编辑功能

---

## 4. 8 屏 User Flow + 数据契约

每屏的 ① 用户体验要点 ② 前端发什么 ③ 后端做什么 ④ 返回什么。

### 屏 1 — 节目档案

**职责：** 产品入口。判断新/老用户，做相应的建档或承接。

#### 1.A 入口判断逻辑
```
读 localStorage.taste_profile
  ├─ 不存在 → 走 "新用户建档" 路径
  └─ 存在   → 走 "老用户主菜单" 路径
```

#### 1.B 新用户建档（菜单式精致建档）

视觉关键词：像翻开一本编辑笔记。

字段：
| 字段 | 类型 | 说明 |
|---|---|---|
| 节目名称 | text | 必填 |
| 节目风格描述 | textarea | 2-3 句话，会作为 AI 的剪辑风格基线 |
| 主播 | dynamic list | name + role（主持/嘉宾） |
| 默认目标时长 | slider 20-90 min | |
| 默认删减力度 | pill (保守/适中/激进) | |
| **Glossary 关键词** | tag input | 节目反复出现的专有名词、人名、品牌名（如「辟客」「珊瑚词典」），转录时用来纠正 |

存储：localStorage `taste_profile` (JSON)

#### 1.C 老用户主菜单

视觉：节目档案摘要卡 + 编辑按钮 + 菜单选项。

```
┌─────────────────────────────────────┐
│  欢迎回来                            │
│                                     │
│  ▌辟客                               │
│  三人闲聊 + 嘉宾访谈，氛围轻松但有深度    │
│  阿司 · 雨林 · 主持                   │
│  默认 50 min · 适中 · 8 个 glossary 词  │
│                          [编辑档案]   │
└─────────────────────────────────────┘

主菜单：
  [+ 开始新的剪辑]  ← MVP 唯一可用项
  [打开上次剪辑]    ← V2，灰显
```

**离开方式：** 点「开始新的剪辑」→ 屏 2

---

### 屏 2 — 上传 + 初始设置

**职责：** 接收音频 + 收集本期专属信息。

字段：
| 字段 | 类型 | 说明 |
|---|---|---|
| 音频文件 | file upload | 拖拽区，支持 MP3/WAV，100-500MB |
| 本期标题 | text | 例 "EP19 潘潘期" |
| 说话人数量 | number | 用于后续说话人识别 |
| 本期专属 glossary | tag input | 嘉宾名、本期特殊词，**会和档案里的 glossary 合并** |

**前端 → 后端：** `POST /api/upload`
```
multipart/form-data { audio: File, episode_meta: JSON }
header: x-aliyun-key, x-anthropic-key
响应: { episodeId, duration, sizeBytes }
```

**离开方式：** 上传完成 → 屏 3

---

### 屏 3 — 录后复盘

**职责：** 录完音趁热让用户告诉 AI 该保留/删除/避免什么。

字段（继承 `prototype_复盘驱动.html` 屏 1 的设计）：
| 字段 | 类型 | 说明 |
|---|---|---|
| 整体录制感觉 | pill (顺畅/还行/磕磕绊绊) | 影响口癖清理力度 |
| 录制中的意外 | multi-checkbox | 录前准备 / 制作讨论 / 设备问题 / 有人离开 |
| **隐私/合规话题** | textarea | 本期是否提到不便公开的内容（个人信息、敏感话题、未公开项目） |
| 印象最深的内容 | textarea | → 保护段落 |
| 可以删的内容 | textarea | → 优先删除 |
| 目标时长 | slider | 覆盖档案默认值 |
| 删减力度 | pill | 覆盖档案默认值 |

**前端 → 后端：** `POST /api/episode/:id/debrief`
```json
{
  "smoothness": "normal",
  "incidents": ["pre_recording_chat", "production_discussion"],
  "privacy_concerns": "...",
  "must_keep": "...",
  "can_remove": "...",
  "target_duration_min": 50,
  "aggressiveness": "moderate"
}
```

**离开方式：** 提交 → 屏 4 + 后台启动 transcribe 任务

---

### 屏 4 — 说话人确认

**职责：** 转录跑完后，让用户确认每个说话人的标签。第一个 human-in-the-loop 检查点。

UI：每个说话人一张卡 — 自动识别标签 + 10-15s 试听片段 + 改名输入框。

**前端 ← 后端（SSE）：**
```
event: transcribe_done
data: { speakers: [{id, auto_label, sample_url, duration}, ...] }
```

**前端 → 后端：** `POST /api/episode/:id/speakers`
```json
{ "spk_1": "阿司", "spk_2": "雨林", "spk_3": "潘潘" }
```

**离开方式：** 确认 → 屏 5

---

### 屏 5 — 处理动画

**职责：** 用户在等粗剪结果，给愉悦的等待体验。

UI：磁带转动 + 6 步进度（上传 / 转录 / 说话人 / 内容分析 / 应用复盘指令 / 生成报告）。继承 `prototype_复盘驱动.html` 屏 2。

**前端 ← 后端（SSE）：** `progress` / `analysis_done` 事件。

**离开方式：** 后端推送完成 → 自动跳屏 6

---

### 屏 6 — 粗剪审核（核心屏）

**职责：** 让用户看 AI 粗剪结果，做必要的核心决策（不是逐句精雕）。

#### 6.A 三栏布局

**左栏 — 概要 + 章节导航：**
- 时长（原 → 粗剪）
- 删减概要文字
- 说话人占比
- 章节列表（AI 总结的章节标题，不是截断的第一句）
- 复用 `review_roughcut.html` 已有结构

**中栏 — 双模式逐字稿：**
- Tab：[ 编辑模式 (default) | 粗剪预览 ]
- 编辑模式：完整逐字稿 + 删除段灰色折叠 + 单句 toggle 删除/恢复
- 粗剪预览：只显示保留的内容 + 紧凑播放器
- 复用 `review_roughcut.html` 的双模式实现

**右栏 — Glossary 校正面板（NEW，MVP 必做）：**

为什么必做：「辟客」被识别成「屁客」会瞬间毁掉用户对工具的信任。这是个低成本高信任的功能。

UI：
```
┌─────────────────────────────┐
│  Glossary（关键词校正）       │
│                             │
│  从档案 + 本期设置同步：       │
│  · 辟客   · 阿司   · 雨林    │
│  · 潘潘   · 珊瑚词典          │
│                             │
│  AI 检测到可能的错误：         │
│  ⚠ "屁客" → 应为 "辟客"      │
│    出现 3 次  [全部修正]      │
│  ⚠ "盘盘" → 应为 "潘潘"      │
│    出现 7 次  [全部修正]      │
│                             │
│  [+ 添加新关键词]              │
└─────────────────────────────┘
```

修正点击后即时更新中栏逐字稿 + 加入档案 glossary（下次自动生效）。

#### 6.B 工具栏

- **重新 cut 一遍** 入口：弹出小窗 "想换个力度试试？"，让用户选 保守/适中/激进 后重跑分析
- **导出粗剪 MP3**（次要，非主要 CTA）

#### 6.C 主 CTA — 进入精剪

```
┌──────────────────────────────────────┐
│  粗剪完成 · 47:12                     │
│                                      │
│  接下来：精剪                          │
│  我们会自动处理 口癖词 / 重复 /         │
│  停顿 / 杂音 / 音量统一                │
│                                      │
│  💡 建议：先把粗剪版发给团队/嘉宾确认   │
│     内容删减没问题，再继续精剪          │
│                                      │
│  精剪风格：                            │
│  ○ 自然 — 保留部分口癖，更真实          │
│  ● 专业 — 精简口癖，更干练              │
│                                      │
│  [ 进入精剪 →]                        │
└──────────────────────────────────────┘
```

#### 6.D 数据契约

`GET /api/episode/:id/cut`
```json
{
  "duration_original": 5720,
  "duration_current": 2892,
  "chapters": [
    { "id": "ch_1", "title": "01 开场介绍", "start": 12.0, "end": 145.0 }
  ],
  "sentences": [
    { "id": "s_1", "speaker": "阿司", "text": "...", "start": 12.0, "end": 17.0, "decision": "keep" }
  ],
  "glossary_corrections": [
    { "wrong": "屁客", "correct": "辟客", "occurrences": 3, "sentence_ids": ["s_5", "s_42", ...] }
  ]
}
```

句子 toggle：`PATCH /api/episode/:id/sentences/:sid` `{ "decision": "remove" }`

Glossary 修正：`POST /api/episode/:id/glossary/apply` `{ "wrong": "屁客", "correct": "辟客", "scope": "all" }`

重新 cut：`POST /api/episode/:id/recut` `{ "aggressiveness": "aggressive" }`

进入精剪（确认粗剪）：`POST /api/episode/:id/finalize-rough-cut` `{ "fine_cut_style": "professional" }`

---

### 屏 7 — 后台精剪 + 倒计时

**职责：** 用户已经确认粗剪，后端开始跑精剪 pipeline（剪播客拼接 → 音质处理 → 后期），前端给智能等待体验。

UI：
- 大磁带动画或类似的「在工作」动画
- 当前阶段（精修句子 / 处理口癖 / 降噪 / 音量统一 / 拼接成品）
- **预计剩余时间** — 基于音频长度和阶段做粗略估计（如 `1.5h 音频，精剪预计 4 分钟`）
- 实时进度条

**前端 ← 后端（SSE）：** `fine_cut_progress` 事件，每个阶段推送 `step` + `eta_seconds`。

---

### 屏 8 — 精剪完成预览（MVP placeholder）

**职责：** MVP 阶段只做静态成品页，让用户感觉"东西做完了"。

UI：
- "🎉 EP19 精剪完成"
- 时长信息（1:35:20 → 47:12）
- 章节 + 时间戳
- [ 下载精剪 MP3 ] 主 CTA
- [ 下载完整逐字稿 ]
- [ 下载章节时间戳 ]
- 后续提示：精剪页面（编辑句子级 / 个性化处理）将在 V2 开放

---

## 5. 后端 API 清单

按调用顺序：

| Endpoint | 方法 | 触发屏 | 复用现有 skill |
|---|---|---|---|
| `/api/upload` | POST | 屏 2 | （新建，用 FFmpeg 转码） |
| `/api/episode/:id/debrief` | POST | 屏 3 | （新建） |
| `/api/episode/:id/transcribe` | POST | 屏 3→4 | 剪播客 `aliyun_funasr_transcribe` |
| `/api/episode/:id/transcribe/stream` | GET (SSE) | 屏 4-5 | （新建） |
| `/api/episode/:id/speakers` | GET / POST | 屏 4 | 剪播客 `identify_speakers` |
| `/api/episode/:id/analyze` | POST | 屏 5 | 剪播客 内容分析 prompt |
| `/api/episode/:id/cut` | GET | 屏 6 | 剪播客 `generate_review` |
| `/api/episode/:id/sentences/:sid` | PATCH | 屏 6 | （新建） |
| `/api/episode/:id/glossary/apply` | POST | 屏 6 | （新建，逐字稿替换 + 写回 taste_profile） |
| `/api/episode/:id/recut` | POST | 屏 6 | 剪播客 重跑分析（用不同力度） |
| `/api/episode/:id/finalize-rough-cut` | POST | 屏 6→7 | （触发精剪 pipeline） |
| `/api/episode/:id/fine-cut/stream` | GET (SSE) | 屏 7 | 剪播客 + 音质处理 + 后期 串行 |
| `/api/episode/:id/download/:type` | GET | 屏 8 | （静态文件） |

**所有请求 header：** `x-aliyun-key`, `x-anthropic-key`（MVP 阶段可以是 placeholder）

---

## 6. API Key 处理方案

### MVP 方案

- 用户在 settings 页输入两个 key → 存浏览器 localStorage
- 每次请求前端在 header 附带
- 后端拿到只在内存里用，不持久化、不写日志
- localStorage 不加密（接受这个 trade-off）
- UI 警告文案："你的 API Key 只存在你的浏览器本地，我们的服务器不保存。建议为这个工具单独创建一个限额的 API Key。"

### 本次交付

API Key 输入框做 placeholder，前端走 mock 数据，无需实际后端。后续响歌歌接入时再开通。

---

## 7. 技术栈

- **前端 + 后端：** Next.js 14（App Router）+ TypeScript
- **样式：** Tailwind CSS + 自定义 Figmint 色板
- **后端运行时：** Node.js
- **部署目标：** Railway 或 Fly.io（不要 Vercel — FFmpeg + 长连接 + 大文件不兼容）
- **文件存储：** MVP 服务器本地 `/tmp/<episodeId>/`，处理完 24 小时清理。V2 改阿里云 OSS

---

## 8. GitHub 仓库结构

新仓库：`podcastcut-web`（独立于 `podcastcut-skills`）

```
podcastcut-web/
├── README.md
├── package.json
├── next.config.js
├── tailwind.config.ts
│
├── app/                            ← 前端页面（阿司主导）
│   ├── layout.tsx
│   ├── page.tsx                    ← 入口判断
│   ├── settings/page.tsx           ← API Key 输入
│   ├── new/page.tsx                ← 屏 1B 新用户建档
│   ├── menu/page.tsx               ← 屏 1C 老用户主菜单
│   ├── upload/page.tsx             ← 屏 2
│   ├── debrief/[id]/page.tsx       ← 屏 3
│   ├── speakers/[id]/page.tsx      ← 屏 4
│   ├── processing/[id]/page.tsx    ← 屏 5
│   ├── review/[id]/page.tsx        ← 屏 6
│   ├── fine-cut/[id]/page.tsx      ← 屏 7
│   └── done/[id]/page.tsx          ← 屏 8
│
├── app/api/                        ← 后端（响歌歌主导）
│   └── …（按第 5 节 endpoint 一一对应）
│
├── components/
│   ├── ProgressIndicator.tsx       ← 全局进度条（屏 2-8 复用）
│   ├── FileCard.tsx
│   ├── ProgressCassette.tsx
│   ├── TranscriptViewer.tsx
│   ├── ChapterNav.tsx
│   ├── GlossaryPanel.tsx
│   └── DesignSystem/...
│
├── lib/                            ← 后端核心逻辑（响歌歌）
│   ├── transcribe.ts
│   ├── analyze.ts
│   ├── cut.ts
│   ├── audioFix.ts
│   ├── postProcess.ts
│   ├── preferences.ts
│   └── store.ts
│
├── prompts/                        ← 从 podcastcut-skills 抽出来的 system prompt
│   ├── analyze_system.md
│   ├── chapter_naming.md
│   ├── glossary_detection.md
│   └── style_baseline.md
│
└── mock/                           ← 前端 prototype 用的假数据
    └── ep19_mock.json
```

---

## 9. 协作分工

| 阶段 | 阿司 | 响歌歌 |
|---|---|---|
| 出 MVP 原型 | 8 屏前端，使用 mock 数据，先跑通流程和视觉 | review spec，确认 API 契约可行 |
| 推 GitHub | 把 prototype 推到 `podcastcut-web` 仓库 | 收到代码后开始搭 lib/ 和 api/ 后端 |
| 联调 | 把 mock 替换成真 API 调用 | 实现各 endpoint，让前后端跑通 |
| 后续迭代 | UX 调优 + 文案 | bug fix + 性能优化 |

### 文件分工约定

- **阿司动：** `app/(屏对应文件夹)/page.tsx` + `components/` + `app/globals.css`
- **响歌歌动：** `app/api/**` + `lib/**` + `prompts/**`
- **共同维护：** 这份 spec + `mock/ep19_mock.json`（数据契约）

---

## 10. 待决问题

跟响歌歌过的时候逐条确认：

- [ ] Q1：转录服务确定用阿里云 FunASR（vs Whisper）— 中文场景下保留
- [ ] Q2：处理 1.5h 音频大概消耗多少 token，给用户做成本预期
- [ ] Q3：屏 7 渲染失败的降级策略（重试 / 给原始音频让用户自己剪）
- [ ] Q4：用户能不能编辑 AI 生成的章节标题（MVP 只读，V2 加？）
- [ ] Q5：API Key 输入界面 — 弹窗 / 独立 settings 页 / 首次强制输入
- [ ] Q6：Glossary 作用范围 — 仅修正显示文案，还是同时改 SRT/TXT 导出
- [ ] Q7：屏 6 「重新 cut」是后端整个 analyze 重跑，还是只调整决策置信度阈值

---

## 附录 A — Glossary 数据结构

存档案的 glossary（影响所有期）：
```json
{
  "taste_profile": {
    "glossary": [
      { "term": "辟客", "common_misrec": ["屁客", "皮克"] },
      { "term": "潘潘", "common_misrec": ["盘盘"] }
    ]
  }
}
```

本期专属 glossary（仅本期生效，转录时合并到主词表）：
```json
{
  "episode_glossary": [
    { "term": "珊瑚词典", "context": "雨林的艺术项目" }
  ]
}
```

转录后 AI 检测到的疑似错误（屏 6 右栏展示）：
```json
{
  "glossary_corrections": [
    { "wrong": "屁客", "correct": "辟客", "occurrences": 3, "confidence": 0.95 }
  ]
}
```

---

## 附录 B — 进度指示器

从屏 2 开始，全屏顶部一致的进度条：

```
[●━━━━━━━━━━○○○○○○○○]
 上传  复盘  说话人  AI处理  粗剪  精剪  完成
   2     3     4     5     6     7    8
```

每屏高亮当前位置 + 已完成步骤，hover 任何节点显示该阶段做什么。

---

## 附录 C — V2 待加功能

- 屏 6 chat box（自然语言指令）— 移到精剪阶段后，避免在粗剪阶段被滥用从而稀释精剪价值
- 屏 8 句子级精剪编辑功能
- 高光片段自动推荐
- 多档播客管理
- 团队协作 / 多主播复盘合并
- 移动端

---

## Changelog

- **v2 · 2026-05-09** — 加 Glossary 功能（屏 1 + 屏 6）；屏 6 移除 chat box；屏 1 加老用户主菜单；屏 6 加重新 cut 入口；屏 6 完成 CTA 加精剪风格选择；屏 7 改为精剪 processing；屏 8 改为精剪完成预览（placeholder）；新增设计原则节；移除时间表 + 非技术指引；改成阿司+响歌歌共同 reference 风格
- **v1 · 2026-05-09** — 初版
