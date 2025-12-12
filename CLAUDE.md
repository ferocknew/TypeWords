# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 项目概述

TypeWords 是一个基于 Vue 3 + TypeScript 的英语学习应用，提供单词练习和文章背诵功能。项目使用 Vite 作为构建工具，数据存储在本地 IndexedDB 中。

## 常用命令

```bash
# 安装依赖
npm install

# 启动开发服务器 (http://localhost:3000)
npm run dev
# 或
npm start

# 构建生产版本
npm run build

# 构建并上传到 OSS
npm run build-oss

# 预览构建结果
npm run preview

# 类型检查
npm run build-tsc

# 提交代码 (使用 commitizen)
npm run commit

# 国际化相关
npm run i18n:write

# 部署到 OSS
npm run deploy-oss
```

## 项目架构

### 技术栈
- **前端框架**: Vue 3.5 + TypeScript 5.8
- **构建工具**: Vite 7.0
- **状态管理**: Pinia 3.0
- **路由**: Vue Router 4.5
- **UI 样式**: UnoCSS + SCSS
- **数据存储**: IndexedDB (idb-keyval)
- **虚拟列表**: Vue Virtual Scroller

### 核心目录结构

```
src/
├── components/     # 公共组件
├── pages/         # 页面组件
│   ├── words/     # 单词练习相关页面
│   ├── articles/  # 文章练习相关页面
│   ├── setting/   # 设置页面
│   └── user/      # 用户中心
├── stores/        # Pinia 状态管理
├── apis/          # API 接口
├── utils/         # 工具函数
├── hooks/         # Vue 组合式函数
├── types/         # TypeScript 类型定义
└── locales/       # 国际化文件
```

### 状态管理 (Pinia Stores)

主要 Store 模块：
- `base.ts`: 核心数据存储（词典、单词、文章）
- `practice.ts`: 练习状态和进度管理
- `user.ts`: 用户认证和信息
- `setting.ts`: 应用设置
- `runtime.ts`: 运行时状态

### 数据存储

使用 idb-keyval 管理 IndexedDB：
- 用户词典和学习进度本地存储
- 收藏、错词本、已掌握单词持久化
- 支持数据导入导出功能

### 路由结构

嵌套路由设计，主布局包含：
- `/words` - 单词练习
- `/articles` - 文章练习
- `/setting` - 设置
- `/user` - 用户中心
- `/login` - 登录

## 开发注意事项

### TypeScript 配置
- 目标版本：ES6
- 严格模式已关闭 (`strict: false`)
- 支持 JSX

### 构建配置
- 支持两种构建模式：普通构建和 CDN 构建
- CDN 构建使用 `build-oss` 命令
- 分析包大小使用 `report` 命令

### 代码规范
- 使用 Husky 管理 Git hooks
- 使用 Commitizen 规范提交信息
- 图标使用 unplugin-icons，自动按需加载

### 特殊功能
- 支持多种词库（CET-4/6、GRE、IELTS 等）
- 四种练习模式：跟打、辨认、复习、默写
- 智能记忆曲线算法
- 文章逐句练习和自动发音
- 高度可定制的设置选项

## 部署

项目支持两种部署方式：
1. 静态文件部署
2. 阿里云 OSS 部署（使用 scripts/deploy-oss.js）