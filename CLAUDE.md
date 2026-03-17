# CLAUDE.md

## Project Overview

OpenUsage - macOS 菜单栏应用，一站式追踪 AI 编程订阅用量（Claude、Cursor、Copilot、Gemini 等 15+ 平台）。

**当前版本**: 0.6.10
**协议**: MIT

## Tech Stack

| 层 | 技术 |
|---|---|
| 前端 | React 19 + TypeScript 5.9 + Vite 8 + Tailwind CSS 4 + Zustand 5 |
| 桌面框架 | Tauri 2 (Rust) |
| 插件引擎 | QuickJS (rquickjs, 在 Rust 中嵌入 JS 运行时) |
| 包管理 | Bun |
| 测试 | Vitest + @testing-library/react (覆盖率 ≥ 90%) |

## Architecture

```
┌─────────────────────────────────────────┐
│  macOS Menu Bar (NSPanel/Tray)          │
├─────────────────────────────────────────┤
│  React Frontend (src/)                  │
│  ├── Zustand Stores (3个: UI/插件/偏好)  │
│  ├── Hooks (编排层: probe/tray/settings) │
│  └── Components (shadcn 风格)            │
├─────────────────────────────────────────┤
│  Tauri IPC Bridge                       │
├─────────────────────────────────────────┤
│  Rust Backend (src-tauri/)              │
│  ├── Plugin Engine (QuickJS 运行时)      │
│  ├── Panel/Tray 管理                     │
│  └── 系统集成 (Keychain, 快捷键等)       │
├─────────────────────────────────────────┤
│  Plugins (plugins/)                     │
│  └── 每个插件: plugin.json + plugin.js   │
└─────────────────────────────────────────┘
```

**核心流程**: 前端调用 `start_probe_batch` → Rust 并行加载插件 → QuickJS 执行 `probe(ctx)` → 事件返回结果 → 前端渲染。

## Directory Structure

```
src/                          # React/TypeScript 前端
├── components/
│   ├── ui/                  # shadcn 风格 UI 组件
│   └── app/                 # AppShell, AppContent
├── hooks/app/               # 核心 hooks (probe, tray, settings, plugins, bootstrap)
├── stores/                  # Zustand stores (app-plugin, app-preferences, app-ui)
├── pages/                   # 页面组件
├── lib/                     # 工具函数 (analytics, settings, formatters)
├── App.tsx                  # 主编排入口
└── main.tsx                 # React 入口

src-tauri/                    # Tauri Rust 后端
├── src/
│   ├── plugin_engine/       # JS 插件运行时
│   │   ├── mod.rs          # 插件发现与初始化
│   │   ├── manifest.rs     # 插件清单解析
│   │   ├── runtime.rs      # JS probe 执行
│   │   └── host_api.rs     # 暴露给插件的 Host API
│   ├── lib.rs              # Tauri app 初始化 & 命令定义
│   ├── tray.rs             # 托盘菜单逻辑
│   ├── panel.rs            # NSPanel 窗口管理
│   └── main.rs             # 入口
├── Cargo.toml              # Rust 依赖
└── tauri.conf.json         # Tauri 配置 (打包/签名/更新)

plugins/                      # 插件实现 (18 个 provider)
├── claude/                  # plugin.json + plugin.js + icon.svg + plugin.test.js
├── cursor/
├── copilot/
├── gemini/
├── windsurf/
├── codex/
└── ...

.github/workflows/
├── ci.yml                   # PR 检查 (lint + build + test)
└── publish.yml              # 发布自动化 (macOS 签名 + 公证)

scripts/
├── build-release.sh         # 发布构建脚本
└── bump-ccusage-version.mjs # ccusage 版本管理
```

## Build & Dev Commands

```bash
bun install                    # 安装依赖
bun run dev                    # Vite 开发服务器 (localhost:1420)
bun tauri dev                  # Tauri 开发模式 (热重载)
bun run build                  # TypeScript + Vite 构建 → /dist
bun run bundle:plugins         # 复制插件到 src-tauri/resources/bundled_plugins/
bun tauri build                # 完整原生构建 → DMG
bun run test                   # 运行测试
bun run test:coverage          # 覆盖率报告 (≥90% 阈值)
```

## Packaging

1. `bun run build` → Vite 打包前端到 `/dist`
2. `bun run bundle:plugins` → 复制 `plugins/` 到 `src-tauri/resources/bundled_plugins/`
3. `bun tauri build` → 编译 Rust + 嵌入前端 + 输出 DMG
4. 打包配置在 `src-tauri/tauri.conf.json` (Bundle ID、图标、签名、自动更新)

## Release Process

1. 同步三处版本号: `package.json` / `src-tauri/Cargo.toml` / `src-tauri/tauri.conf.json`
2. 更新 `CHANGELOG.md`
3. 提交并推送
4. 打 Git Tag: `git tag v0.x.x && git push --tags`
5. GitHub Actions 自动触发 `.github/workflows/publish.yml`:
   - 双架构矩阵: `aarch64-apple-darwin` + `x86_64-apple-darwin`
   - Apple 证书签名 + 公证
   - Tauri 更新签名 (应用内自动更新)
   - 发布到 GitHub Releases

**环境变量** (发布所需，见 `.env.example`):
- `TAURI_SIGNING_PRIVATE_KEY` / `TAURI_SIGNING_PRIVATE_KEY_PASSWORD`
- `APPLE_SIGNING_IDENTITY` / `APPLE_ID` / `APPLE_PASSWORD` / `APPLE_TEAM_ID`

## Plugin System

每个插件目录包含:
- `plugin.json`: 清单 (id, name, brandColor, 输出 schema)
- `plugin.js`: 入口，导出 `probe(ctx)` 异步函数
- `icon.svg`: 品牌图标 (必须使用 `currentColor`)
- `plugin.test.js`: 测试

**Plugin probe 接收的 context**:
```javascript
{
  nowIso: string,              // UTC 时间
  app: { version, platform, appDataDir, pluginDataDir },
  host: {                      // Host API
    fs: { readTextFile, writeTextFile, listDir, ... },
    http: { request },
    env: { get },
    keychain: { findGenericPassword },
    sqlite: { query, exec },
    log: { info, warn, error },
    ccusage: { query }
  }
}
```

## Key Modification Points

| 目标 | 改哪里 |
|---|---|
| 添加新 AI 平台 | `plugins/` 下新建目录 |
| 改 UI/样式 | `src/components/` + Tailwind |
| 改菜单栏行为 | `src-tauri/src/tray.rs` + `panel.rs` |
| 扩展插件能力 | `src-tauri/src/plugin_engine/host_api.rs` |
| 改打包配置 | `src-tauri/tauri.conf.json` |
| 改发布流程 | `.github/workflows/publish.yml` |
| 跨平台支持 | 替换 NSPanel/Keychain 等 macOS 专属代码 |

## Important Notes

- Tauri IPC: JS 端用 camelCase，Tauri 自动转 Rust snake_case，不要从 JS 发 snake_case
- 插件数据目录: `{appDataDir}/plugins_data/{pluginId}/` 按需自动创建
- ccusage 版本锁定: Claude 18.0.10, Codex 18.0.10
- 文件保持 <400 LOC，超过则拆分重构
- 测试覆盖率必须 ≥ 90%
