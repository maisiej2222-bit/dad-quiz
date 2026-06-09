# 《让爱开口》父亲节活动 — 完整技术架构方案

## 一、项目概述

| 项目 | 内容 |
|------|------|
| 项目名称 | 让爱开口 |
| 活动副标题 | 父亲节，把平时没说出口的话认真说出来 |
| 活动定位 | 公司提供轻量化表达工具，员工可选择以文字海报、语音明信片、消息卡或说明书海报表达 |
| 核心价值 | 公司不替员工表达，不要求员工表达，只是降低开口门槛 |
| 目标用户 | 全体员工（写给父亲 / 作为父亲写给孩子） |

---

## 二、产品结构

```
让爱开口（首页）
├── 入口一：写给父亲的话
│   ├── 文字海报（表单 → 海报预览 → 保存）
│   ├── 语音明信片（录音 → 上传 → 二维码明信片）
│   └── 未发送的消息（表单 → 聊天气泡卡预览 → 保存）
│
└── 入口二：爸爸的使用说明
    └── 说明书海报（表单 → 海报预览 → 保存）
```

---

## 三、用户主流程

```
首页 → 选择入口 → 填写内容/录制语音 → 确认授权 → 生成海报 → 保存/发送/匿名展示 → 数据沉淀
```

---

## 四、前端技术方案

### 4.1 技术选型

| 层级 | MVP | 正式版本 |
|------|-----|---------|
| 页面框架 | 原生 HTML/CSS/JS 单文件 | 同 MVP，可拆分模块 |
| 录音 | `MediaRecorder` API (webm) | `MediaRecorder` API + 后端转码 |
| 二维码 | 占位框 | `qrcode.js` 前端库 或 后端 `/api/qrcode/create` |
| 海报生成 | DOM 预览 + 占位下载 | `html2canvas` 前端截图 或 后端 Puppeteer/Sharp 渲染 |
| 语音播放 | 本地 Blob URL 试听 | 后端托管播放页 `GET /fathersday/voice/{voice_id}` |
| 数据埋点 | `console.log` 占位 | `POST /api/event/track` |
| 润色 | 前端预设文案模拟 | 接入公司大模型 API |

### 4.2 页面结构

```
page-home               — 首页（双入口）
page-entry1-choose      — 写给父亲的话 → 选择表达方式
page-form-text-poster   — 文字海报表单
page-form-voice-postcard— 语音明信片表单（含录音模块 + 隐私授权）
page-form-message-card  — 未发送消息卡表单
page-form-dad-manual    — 爸爸使用说明书表单
poster-overlay          — 海报预览弹窗（通用，动态渲染）
privacy-footer          — 全局底部隐私提示条
```

### 4.3 关键前端逻辑

**录音流程（MediaRecorder）：**
```
用户点击"开始录音"
  → navigator.mediaDevices.getUserMedia({ audio: true })
  → MediaRecorder.start()
  → 计时器每秒更新（超60秒自动停止）
  → 用户点击"停止录音"
  → MediaRecorder.onstop → 生成 Blob → 创建本地 URL
  → 显示"试听"/"重新录制"按钮
  → 用户勾选隐私授权 → "确认生成"按钮启用
  → 模拟上传 → 生成海报预览
```

**海报渲染：**
- 每种海报类型独立 `generate*()` 函数
- 动态拼接 HTML 注入 `#poster-content`
- 弹窗显示预览，支持关闭
- 正式版本：`html2canvas` 截图 → `canvas.toBlob()` → 触发下载

**润色模拟（MVP）：**
- `mockPolish(textareaId, mode)` 
- 保存原文到 `state.originalTexts`
- `gentle` 模式：添加温柔修饰词
- `casual` 模式：去除过于煽情的表达
- `original` 模式：恢复原文
- 正式版本：`POST /api/polish` 接入大模型

---

## 五、后端接口设计

### 5.1 接口列表

| 接口 | 方法 | 路径 | 说明 |
|------|------|------|------|
| 语音上传 | POST | `/api/voice/upload` | 上传音频文件，返回 voice_id + 播放链接 |
| 语音播放页 | GET | `/fathersday/voice/{voice_id}` | 语音播放 H5 页面 |
| 语音流 | GET | `/api/voice/stream/{voice_id}` | 返回音频文件流 |
| 二维码生成 | POST | `/api/qrcode/create` | 根据 URL 生成二维码图片 |
| 海报生成 | POST | `/api/poster/create` | 后端渲染海报（可选，也可前端 html2canvas） |
| 数据埋点 | POST | `/api/event/track` | 活动数据上报 |
| 活动统计 | GET | `/api/stats/summary` | 活动数据汇总（管理后台） |

### 5.2 接口详情

#### POST /api/voice/upload

**请求：** `multipart/form-data`

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| audio_file | File | 是 | 音频文件（webm/mp3/m4a，≤10MB） |
| sender_name | string | 否 | 发送者称呼 |
| receiver_name | string | 否 | 接收者称呼 |
| scene_type | string | 是 | 场景类型：`voice_postcard` |
| expire_days | int | 否 | 有效期（天），默认 30 |

**响应：**
```json
{
  "code": 0,
  "data": {
    "voice_id": "v_abc123xyz",
    "play_url": "https://company.com/fathersday/voice/v_abc123xyz",
    "qr_url": "https://company.com/api/qrcode/v_abc123xyz.png",
    "expire_at": "2026-07-08T00:00:00Z"
  }
}
```

#### GET /fathersday/voice/{voice_id}

语音播放 H5 页面（服务端渲染或静态页面 + API 获取音频信息）。

页面内容：
- 播放按钮
- 发送者称呼
- 简短标题
- 如已过期，显示"该语音明信片已过期"

#### POST /api/qrcode/create

**请求：**
```json
{
  "target_url": "https://company.com/fathersday/voice/v_abc123xyz",
  "size": 256
}
```

**响应：**
```json
{
  "code": 0,
  "data": {
    "qr_id": "qr_xyz789",
    "qr_image_url": "https://company.com/static/qr/qr_xyz789.png"
  }
}
```

#### POST /api/event/track

**请求：**
```json
{
  "event_name": "text_poster_generate",
  "entry_type": "write_to_father",
  "card_type": "text_poster",
  "page": "page-form-text-poster",
  "permission": "private",
  "timestamp": "2026-06-15T10:30:00Z",
  "user_agent": "Mozilla/5.0 ..."
}
```

#### GET /api/stats/summary

管理后台统计接口，返回聚合数据。

---

## 六、数据库设计

### 6.1 voice_cards 表

```sql
CREATE TABLE voice_cards (
    id              BIGINT AUTO_INCREMENT PRIMARY KEY,
    voice_id        VARCHAR(32)  NOT NULL UNIQUE COMMENT '语音唯一ID',
    audio_url       VARCHAR(512) NOT NULL COMMENT '音频文件存储路径',
    audio_format    VARCHAR(16)  DEFAULT 'webm' COMMENT '音频格式',
    audio_size      INT          DEFAULT 0 COMMENT '文件大小(bytes)',
    audio_duration  INT          DEFAULT 0 COMMENT '时长(秒)',
    sender_name     VARCHAR(64)  DEFAULT '' COMMENT '发送者称呼',
    title           VARCHAR(128) DEFAULT '' COMMENT '简短标题',
    scene_type      VARCHAR(32)  NOT NULL DEFAULT 'voice_postcard' COMMENT '场景类型',
    expire_at       DATETIME     NOT NULL COMMENT '过期时间（默认30天）',
    status          TINYINT      DEFAULT 1 COMMENT '1:有效 0:已过期 -1:已删除',
    play_count      INT          DEFAULT 0 COMMENT '播放次数',
    last_played_at  DATETIME     DEFAULT NULL COMMENT '最后播放时间',
    created_at      DATETIME     DEFAULT CURRENT_TIMESTAMP,
    updated_at      DATETIME     DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    INDEX idx_voice_id (voice_id),
    INDEX idx_expire_at (expire_at),
    INDEX idx_status (status)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='语音明信片';
```

### 6.2 expression_cards 表

```sql
CREATE TABLE expression_cards (
    id              BIGINT AUTO_INCREMENT PRIMARY KEY,
    card_id         VARCHAR(32)  NOT NULL UNIQUE COMMENT '卡片唯一ID',
    entry_type      VARCHAR(32)  NOT NULL COMMENT '入口: write_to_father | dad_manual',
    card_type       VARCHAR(32)  NOT NULL COMMENT '类型: text_poster | voice_postcard | message_card | dad_manual',
    content_json    JSON         NOT NULL COMMENT '用户填写内容（结构化JSON）',
    permission_type VARCHAR(32)  DEFAULT 'private' COMMENT '权限: private | family_share | anonymous_wall',
    poster_url      VARCHAR(512) DEFAULT NULL COMMENT '海报图片URL',
    voice_id        VARCHAR(32)  DEFAULT NULL COMMENT '关联语音ID（语音明信片）',
    qr_url          VARCHAR(512) DEFAULT NULL COMMENT '二维码图片URL',
    download_count  INT          DEFAULT 0 COMMENT '下载次数',
    share_count     INT          DEFAULT 0 COMMENT '分享次数',
    status          TINYINT      DEFAULT 1 COMMENT '1:有效 0:已删除',
    created_at      DATETIME     DEFAULT CURRENT_TIMESTAMP,
    updated_at      DATETIME     DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    INDEX idx_card_id (card_id),
    INDEX idx_entry_type (entry_type),
    INDEX idx_card_type (card_type),
    INDEX idx_permission (permission_type),
    INDEX idx_created_at (created_at)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='表达卡记录';
```

### 6.3 event_logs 表

```sql
CREATE TABLE event_logs (
    id              BIGINT AUTO_INCREMENT PRIMARY KEY,
    event_name      VARCHAR(64)  NOT NULL COMMENT '事件名称',
    entry_type      VARCHAR(32)  DEFAULT NULL COMMENT '入口类型',
    card_type       VARCHAR(32)  DEFAULT NULL COMMENT '卡片类型',
    page            VARCHAR(64)  DEFAULT NULL COMMENT '页面标识',
    permission      VARCHAR(32)  DEFAULT NULL COMMENT '权限选择',
    extra_data      JSON         DEFAULT NULL COMMENT '扩展数据',
    user_agent      VARCHAR(512) DEFAULT NULL COMMENT 'UA',
    ip_address      VARCHAR(64)  DEFAULT NULL COMMENT 'IP地址',
    created_at      DATETIME     DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_event_name (event_name),
    INDEX idx_created_at (created_at),
    INDEX idx_entry_type (entry_type)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='活动事件日志';
```

### 6.4 poster_records 表（可选）

```sql
CREATE TABLE poster_records (
    id              BIGINT AUTO_INCREMENT PRIMARY KEY,
    card_id         VARCHAR(32)  NOT NULL COMMENT '关联表达卡ID',
    poster_url      VARCHAR(512) NOT NULL COMMENT '海报图片URL',
    qr_url          VARCHAR(512) DEFAULT NULL COMMENT '二维码URL',
    download_count  INT          DEFAULT 0 COMMENT '下载次数',
    created_at      DATETIME     DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_card_id (card_id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='海报生成记录';
```

---

## 七、数据统计指标体系

| 序号 | 指标 | 数据来源 | 说明 |
|------|------|---------|------|
| 1 | 首页 PV/UV | event_logs (page_view, page=page-home) | 活动触达率 |
| 2 | 入口1点击量 | event_logs (entry_click, entry_type=write_to_father) | 写给父亲入口 |
| 3 | 入口2点击量 | event_logs (entry_click, entry_type=dad_manual) | 爸爸说明书入口 |
| 4 | 文字海报生成数 | event_logs (text_poster_generate) | 文字海报 |
| 5 | 语音明信片生成数 | event_logs (voice_postcard_generate) | 语音明信片 |
| 6 | 未发送消息卡生成数 | event_logs (message_card_generate) | 消息卡 |
| 7 | 爸爸说明书生成数 | event_logs (dad_manual_generate) | 说明书 |
| 8 | 录音启动次数 | event_logs (recording_start) | 录音参与 |
| 9 | 录音完成次数 | event_logs (recording_stop) | 录音完成率 |
| 10 | 语音播放页 PV | 语音播放页埋点 | 父亲扫码收听 |
| 11 | 语音实际播放次数 | voice_cards.play_count | 播放统计 |
| 12 | 海报保存点击数 | event_logs (save_poster_click) | 保存意愿 |
| 13 | 匿名展示授权数 | expression_cards (permission=anonymous_wall) | 公开展示意愿 |
| 14 | 润色功能使用次数 | event_logs (polish_click) | 辅助功能使用 |
| 15 | 总卡片生成数 | expression_cards COUNT | 活动总产出 |

---

## 八、部署架构

### 8.1 推荐部署拓扑

```
                       ┌─────────────┐
                       │   Nginx     │ (静态资源 + 反向代理)
                       └──────┬──────┘
                              │
              ┌───────────────┼───────────────┐
              │               │               │
     ┌────────▼────────┐ ┌───▼────┐  ┌───────▼──────┐
     │ 静态文件服务     │ │ API服务 │  │ 对象存储      │
     │ (H5页面)        │ │ (Python │  │ (音频/海报)   │
     │                 │ │  /Go)  │  │ MinIO/OSS     │
     └─────────────────┘ └───┬────┘  └──────────────┘
                             │
                      ┌──────▼──────┐
                      │   MySQL     │
                      │  (数据存储)  │
                      └─────────────┘
```

### 8.2 部署清单

| 组件 | 技术选型建议 | 说明 |
|------|------------|------|
| Web 服务器 | Nginx | 静态文件 + 反向代理 |
| 应用后端 | Python (Flask/FastAPI) 或 Go (Gin) | RESTful API |
| 数据库 | MySQL 8.0 | 活动数据存储 |
| 对象存储 | MinIO（自建）或公司已有文件服务器 | 音频文件 + 海报图片 |
| 二维码 | Python `qrcode` 库 或 Go `go-qrcode` | 服务端生成 |
| 海报渲染 | Puppeteer（Node.js）或 Python `imgkit` | 服务端海报生成 |
| 大模型润色 | 接入公司内部大模型 API | 可选功能 |

### 8.3 语音文件管理策略

1. **存储路径：** `{storage_root}/fathersday/voices/{yyyyMM}/{voice_id}.webm`
2. **命名规则：** 使用随机 `voice_id`（UUID 短码），不可遍历
3. **有效期：** 默认 30 天，到期后通过定时任务标记 `status=0`
4. **清理策略：** 活动结束后 7 天，统一清理过期语音文件
5. **播放链接：** `https://company.com/fathersday/voice/{voice_id}` — 服务端渲染播放页，不暴露原始文件路径

### 8.4 安全与合规要点

- 语音播放链接使用随机 ID，不可遍历
- 链接设置有效期，过期自动失效
- 音频文件不直接暴露 URL，通过服务端流式返回
- 录音前须显式勾选隐私授权
- 所有上传内容限制大小（音频 ≤10MB，文本 ≤500字）
- 内容安全审核（接入公司内容安全服务）
- 用户可随时申请删除自己的内容
- HTTPS 全链路加密
- 数据库连接使用内网地址

---

## 九、隐私与合规文案

### 9.1 录音授权文案（页面中展示）

```
你上传的语音仅用于生成本次父亲节语音明信片。
系统将生成一个专属播放链接和二维码。
语音默认保存30天，活动结束后统一清理。
请勿上传涉及隐私、敏感或不适合传播的内容。

☐ 我已知晓并同意上传语音用于生成父亲节语音明信片。
```

### 9.2 全局隐私提示（页面底部）

```
🔒 你填写的内容仅用于本次父亲节表达卡生成。如选择公开展示，将默认以匿名方式呈现。
```

### 9.3 合规原则

1. 员工自愿参与，不强制
2. 内容可仅自己保存（默认选项）
3. 不评选、不排名、不比较
4. 不教育员工如何表达
5. 不定义父亲角色
6. 语音仅用于生成父亲节语音明信片
7. 语音默认保存 30 天，活动结束后统一清理
8. 公开展示默认匿名

---

## 十、正式上线 Checklist

- 前端部署至公司 Web 服务器（Nginx 静态文件）
- 配置 HTTPS 证书
- 后端 API 服务部署（语音上传 + 播放 + 埋点 + 统计）
- 音频存储配置（MinIO / OSS / 文件服务器）
- 播放链接使用随机 voice_id（不可遍历）
- 语音播放页（服务端渲染或 API + 静态播放页）
- 链接设置 30 天有效期，定时任务标记过期
- 活动结束后定时清理语音文件
- 海报生成：接入 html2canvas（前端）或 Puppeteer（后端）
- 二维码生成：qrcode.js（前端）或后端 qrcode 库
- 数据埋点接入：`POST /api/event/track`
- 管理后台统计面板（可选）
- 内容安全审核接入
- 灰度测试 → 全员推送

---

## 十一、汇报总结（适合放进汇报材料）

> **《让爱开口》** 不是一场教育式父亲节活动，而是公司提供的轻表达场景。通过"写给父亲的话"和"爸爸的使用说明"两个入口，员工可以选择以文字海报、语音明信片、消息卡或说明书海报的方式，把平时没说出口的话表达出来。公司不介入家庭关系，也不定义爱的表达方式，只是降低开口门槛，提供一个温暖、体面、可选择的节日表达工具。语音明信片采用公司服务器部署，确保数据可控、链接可管理、活动结束后可统一清理。活动数据全部沉淀至公司数据库，支持后续复盘分析。
>
> **关键设计原则：**
> - 不教育、不评选、不强制、不定义
> - 默认仅自己保存，公开须主动授权且匿名
> - 语音 30 天有效，活动后统一清理
> - 全链路公司服务器部署，数据安全可控
>
> **预期效果：**
> - 轻量化参与路径，降低表达门槛
> - 多类型海报产出，丰富表达方式
> - 语音明信片增加互动感与仪式感
> - 数据沉淀支持活动复盘与未来优化

---

## 十二、原型文件说明

原型文件：`let-love-speak.html`

**已实现功能：**

| 功能 | 状态 |
|------|------|
| 首页双入口 | ✅ |
| 文字海报表单 + 预览 | ✅ |
| 语音明信片录音 MVP（MediaRecorder） | ✅ |
| 未发送消息卡表单 + 预览 | ✅ |
| 爸爸使用说明书表单 + 预览 | ✅ |
| 海报预览弹窗（4 种海报风格） | ✅ |
| 隐私授权勾选 | ✅ |
| 全局隐私提示条 | ✅ |
| 开口提示 + 随机换一批 | ✅ |
| 润色模拟（3 种模式） | ✅ |
| 数据埋点函数占位 | ✅ |
| 海报保存按钮占位 | ✅ |
| 二维码占位框 | ✅ |
| 移动端优先响应式 | ✅ |
| 无外部 CDN 依赖 | ✅ |

**待正式版本接入：**

| 功能 | 接入方式 |
|------|---------|
| 真实语音上传 | `POST /api/voice/upload` |
| 语音播放页 | `GET /fathersday/voice/{voice_id}` |
| 二维码生成 | `qrcode.js` 或后端接口 |
| 海报保存下载 | `html2canvas` 或后端渲染 |
| 数据埋点上报 | `POST /api/event/track` |
| AI 润色 | 接入公司大模型 API |
| 公司 Logo | 替换占位文字 |
