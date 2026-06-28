# Prompt 配置文档

> 使用此文档向 AI 助手描述项目配置，AI ���协助您完成整个项目的部署和配置。

## 项目概述

这是一个将 kikoeru 音乐播放器与 JapaneseASMR 字幕系统集成的项目。用户通过浏览器注入脚本，在 kikoeru 播放器中显示 ASMR 音频作品列表、AI 翻译的中文标题、字幕等信息。

## 系统架构

### 数据流

```
用户输入 JapaneseASMR URL
        │
        ▼
后端 Flask API 解析页面 → 下载 HLS 音频 → 按章节分割
        │
        ▼
Whisper 语音识别生成字幕 (LRC/VTT)
        │
        ▼
字幕 + 元数据同步到云端数据库
        │
        ▼
kikoeru 前端通过注入脚本展示：
  ├─ 作品列表（含中文标题）
  ├─ 字幕注入到播放器
  ├─ 封面图片展示
  └─ 翻译/搜索增强功能
```

### 核心服务

| 域名/端口 | 服务 | 用途 |
|-----------|------|------|
| `localhost:5000` | Flask API | 后端管理接口 |
| `localhost:7890` | kikoeru Nginx | 前端播放器 |
| Cloudflare D1 | 数据库 | 字幕与元数据存储 |
| Cloudflare R2/KV | 缓存 | 封面与索引缓存 |

## 部署步骤

请 AI 根据以下步骤协助部署：

### 1. 后��� API 服务

Flask 服务提供以下 API：

- `GET /api/local-works` - 获取本地作品列表
- `GET /api/local-work/<rj>` - 获取单个作品详情
- `GET /api/local-tracks/<rj>` - 获取作品的章节/音轨列表
- `POST /api/download-full` - 下载音频（参数: url, split）
- `POST /api/translate-meta/<rj>` - AI 翻译作品标题
- `POST /api/translate-kikoeru` - 翻译歌词
- `GET /api/active-tasks` - 查看活跃任务
- `POST /api/sync-cloud` - 手动同步到云端

### 2. kikoeru 前端

kikoeru 是开源的音频播放器，支持歌词显示。需要：

1. 构建 Docker 镜像
2. 配置 Nginx 注入脚本
3. 部署到网络服务

### 3. 注入脚本

`source-switcher.js` 是核心注入脚本（约 6500 行），实现以下功能：

- 在 kikoeru 页面侧边栏添加数据源切换按钮
- 拦截 fetch/XHR ��求，重写响应数据
- 注入字幕文件到播放器
- 添加翻译、字幕管理等 UI 元素
- 增强搜索功能

### 4. 云端存储

需要 Cloudflare 账号配置：

1. 创建 D1 数据库（`subtitles` 和 `subtitle_meta` 两张表）
2. 配置 Worker 服务提供 REST API
3. 可选：R2 存储桶用于封面缓存

### 5. AI 翻译

需要配置 AI API 用于标题翻译和歌词翻译。

## 环境变量

| 变量名 | 说明 | 是否必需 |
|--------|------|----------|
| `EDGEONE_TOKEN` | EdgeOne Pages 部署 token | 否 |

## 常见问题

### 字幕不显示
- 检查云端是否已同步字幕
- 检查音频文件名与字幕文件名是否匹配
- 检查浏览器 Console 是否有错误

### 翻译失败
- AI API 可能触发内容过滤，建议逐条翻译而非批量
- 检查翻译 API 的认证信息

### 封面不显示
- 确认封面文件已生成
- 检查 CDN 配置是否正确

### 下载卡住
- N_m3u8DL-RE 可能挂起，Python httpx 回退机制已实现
- 检查网络连接和磁盘空间

## 数据库 Schema

### subtitles 表
```sql
CREATE TABLE subtitles (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  rj_number TEXT NOT NULL,
  chapter_name TEXT NOT NULL,
  content_lrc TEXT,
  content_vtt TEXT,
  language TEXT DEFAULT 'ja',
  created_at TEXT DEFAULT (datetime('now')),
  updated_at TEXT DEFAULT (datetime('now')),
  UNIQUE(rj_number, chapter_name)
);
```

### subtitle_meta 表
```sql
CREATE TABLE subtitle_meta (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  rj_number TEXT NOT NULL UNIQUE,
  title TEXT,
  title_cn TEXT,
  circle TEXT,
  cv TEXT,
  chapter_count INTEGER DEFAULT 0,
  created_at TEXT DEFAULT (datetime('now')),
  updated_at TEXT DEFAULT (datetime('now'))
);
```

## Worker API

| 方法 | 路径 | 说明 |
|------|------|------|
| GET | `/` | 视觉索引页面 |
| GET | `/api/subtitles/list` | 列出所有作品 |
| GET | `/api/subtitles/:rj` | 获取指定作品字幕 |
| POST | `/api/subtitles/:rj` | 上传字幕 |
| DELETE | `/api/subtitles/:rj` | 删除字幕 |

## 一键启动命令

```bash
# 启动后端 API
cd japanese-asmr/japanese-asmr-parser
pip install -r requirements.txt
python web/app.py &

# 构建并启动 kikoeru 前端
cd kikoeru
docker build -t kikoeru-frontend .
docker run -d --network host kikoeru-frontend
```

## 测试命令

```bash
# 测试作品列表
curl http://localhost:5000/api/local-works

# 测试作品详情
curl http://localhost:5000/api/local-work/RJ01638438

# 测试页面解析
curl -X POST http://localhost:5000/api/fetch-page \
  -H 'Content-Type: application/json' \
  -d '{"url":"https://japaneseasmr.com/147164/"}'

# 测试翻译
curl -X POST http://localhost:5000/api/translate-meta/RJ01638438 \
  -H 'Content-Type: application/json' \
  -d '{"api_type":"cnb.cool"}'

# 测试下载
curl -X POST http://localhost:5000/api/download-full \
  -H 'Content-Type: application/json' \
  -d '{"url":"https://japaneseasmr.com/147164/","split":true}'
```

## 注意事项

1. 所有封面图片通过后端代理提供，不直接暴露外部 CDN 地址
2. 字幕云端同步为可选功能，本地运行时可以跳过
3. AI 翻译 API 需要自行配置密钥
4. 发布公开版本时，注意不要暴露敏感 URL 和 R18 内容
5. 建议在独立的 git 工作区操作公开版本，避免影响私有开发环境
