# GEMINI.md

本文档为 Gemini 等大语言模型提供 **Prompt Optimizer（提示词优化器）** 项目的完整开发上下文。每次会话开始时请优先阅读此文档。

---

## 1. 项目概览

- **名称**: Prompt Optimizer（提示词优化器）
- **版本**: 2.5.4（AGPL-3.0-only 许可）
- **定位**: AI 提示词优化 + 图像生成 + 多模态工具，支持 Web / Desktop / Chrome Extension / Docker / MCP Server 五种部署形态
- **架构**: pnpm Monorepo，严格单向依赖
- **仓库**: `linshenkx/prompt-optimizer`

---

## 2. 核心技术栈

| 类别 | 技术 |
|---|---|
| 包管理 | **pnpm@10+**（严禁 npm/yarn，`package.json` 中已强制限制） |
| 前端框架 | Vue 3 + TypeScript + Composition API |
| UI 组件库 | **Naive UI（优先使用）** + TailwindCSS |
| 构建工具 | Vite（应用层），tsup（core 包） |
| 桌面端 | Electron + electron-builder + 自动更新 |
| 测试 | Vitest（单元/集成），Playwright（E2E） |
| 状态管理 | Vue 响应式 Composables + Pinia（stores 层），不使用 Vuex |
| 国际化 | Vue-i18n |
| 代码编辑器 | CodeMirror 6（用于模板/提示词编辑） |
| Markdown 渲染 | markdown-it + highlight.js + DOMPurify |
| 数据验证 | Zod |
| 运行时 | Node.js >= 18（支持 18/20/22） |

---

## 3. Monorepo 架构

### 3.1 依赖层级（严格单向，禁止反向依赖）

```
packages/core (基础层：平台无关的业务逻辑)
    ↓
packages/ui (UI 层：Vue 组件 + Composables + 路由 + 主题)
    ↓
packages/web | extension | desktop | mcp-server (应用层：各平台入口)
```

### 3.2 构建顺序

```
1. core (tsup → CJS + ESM + DTS)
2. ui   (Vite library mode → JS + CSS + DTS)
3. web / extension / desktop / mcp-server (可并行)
```

### 3.3 目录结构总览

```
prompt-optimizer/
├── packages/
│   ├── core/           # @prompt-optimizer/core — 核心业务逻辑
│   │   └── src/
│   │       ├── services/       # 18+ 服务模块（见下文详解）
│   │       ├── constants/      # 存储键、错误代码
│   │       ├── types/          # 高级类型定义
│   │       └── utils/          # 环境检测、IPC 序列化、patch 工具
│   ├── ui/             # @prompt-optimizer/ui — Vue 3 组件库
│   │   └── src/
│   │       ├── components/     # 60+ Vue SFC 组件
│   │       ├── composables/    # 14 个功能域的响应式逻辑
│   │       ├── stores/         # Pinia stores（session/settings 等）
│   │       ├── router/         # 路由配置（Hash 模式，兼容 Electron）
│   │       ├── i18n/           # 国际化资源
│   │       ├── styles/         # 全局样式
│   │       ├── integrations/   # 第三方集成
│   │       └── services/       # UI 层服务
│   ├── web/            # @prompt-optimizer/web — Vite Web 应用入口
│   ├── desktop/        # @prompt-optimizer/desktop — Electron 桌面应用
│   ├── extension/      # @prompt-optimizer/extension — Chrome 浏览器插件
│   └── mcp-server/     # @prompt-optimizer/mcp-server — MCP 协议服务器
├── api/                # Vercel Serverless Functions（auth.js 等）
├── docker/             # Docker 配置（nginx.conf, 启动脚本, supervisor）
├── docs/               # 项目文档（architecture/developer/testing/archives 等）
├── tests/              # E2E 测试（Playwright）
├── scripts/            # 工具脚本（版本同步、智能 E2E、进程清理）
├── .github/workflows/  # CI/CD（test.yml, release.yml, docker.yml）
└── 配置文件             # package.json, pnpm-workspace.yaml, vercel.json, Dockerfile 等
```

---

## 4. Core 服务模块详解 (`packages/core/src/services/`)

每个服务模块遵循统一模式：`types.ts` + `errors.ts` + `service/manager.ts` + `electron-proxy.ts`

| 服务模块 | 职责 | 关键导出 |
|---|---|---|
| `llm/` | LLM API 通信（OpenAI/Gemini/DeepSeek/Anthropic/智谱/SiliconFlow/ModelScope 等） | `LLMService`, `TextAdapterRegistry`, `ElectronLLMProxy` |
| `model/` | 文本模型配置管理、参数模式、高级参数定义 | `ModelManager`, `ElectronModelManagerProxy` |
| `image/` | 图像生成服务（文生图/图生图）、图像存储 | `ImageService`, `ImageStorageService` |
| `image-model/` | 图像模型配置管理 | `ImageModelManager` |
| `prompt/` | 提示词优化核心逻辑、自定义会话测试 | `PromptService`, `ElectronPromptServiceProxy` |
| `template/` | 提示词模板管理、CSP 安全处理器、多语言支持 | `TemplateManager`, `TemplateProcessor`, `StaticLoader` |
| `history/` | 优化历史记录追踪 | `HistoryManager`, `ElectronHistoryManagerProxy` |
| `storage/` | 多适配器存储（localStorage/IndexedDB/FileSystem/Memory） | `StorageFactory`, `DexieStorageProvider` 等 |
| `preference/` | 用户偏好设置（跨平台同步） | `PreferenceService`, `ElectronPreferenceServiceProxy` |
| `compare/` | 提示词版本对比 | `CompareService` |
| `context/` | 上下文仓库管理 | `createContextRepo`, `ElectronContextRepoProxy` |
| `favorite/` | 收藏管理 | `FavoriteManager`, `TagTypeConverter` |
| `evaluation/` | 提示词评估服务 | `EvaluationService` |
| `data/` | 数据导入导出管理 | `DataManager`, `ElectronDataManagerProxy` |
| `variable-extraction/` | 从提示词中智能提取变量 | `VariableExtractionService` |
| `variable-value-generation/` | 变量值自动生成 | `VariableValueGenerationService` |
| `adapters/` | 通用适配器基础 | — |
| `shared/` | 共享工具 | — |

---

## 5. UI 架构 (`packages/ui/`)

### 5.1 页面路由体系（Hash 模式）

应用采用三大功能模式，每种模式下有独立的工作区路由：

| 路由路径 | 名称 | 组件 | 说明 |
|---|---|---|---|
| `/basic/system` | basic-system | `BasicSystemWorkspace.vue` | 基础模式 - 系统提示词优化 |
| `/basic/user` | basic-user | `BasicUserWorkspace.vue` | 基础模式 - 用户提示词优化 |
| `/pro/multi` | pro-multi | `ContextSystemWorkspace.vue` | 专业模式 - 多消息上下文 |
| `/pro/variable` | pro-variable | `ContextUserWorkspace.vue` | 专业模式 - 变量模式 |
| `/image/text2image` | image-text2image | `ImageText2ImageWorkspace.vue` | 图像模式 - 文生图 |
| `/image/image2image` | image-image2image | `ImageImage2ImageWorkspace.vue` | 图像模式 - 图生图 |

### 5.2 Composables 功能分层

composables 按功能域组织，封装与 core 服务的响应式交互：

| 功能域 | 职责 |
|---|---|
| `prompt/` | 提示词操作、优化流程控制 |
| `model/` | 模型选择、参数管理 |
| `ui/` | 通用 UI 状态（toast、dialog、主题等） |
| `storage/` | 存储操作封装 |
| `variable/` | 变量管理 |
| `mode/` | 功能模式切换 |
| `context/` | 上下文管理 |
| `session/` | 会话管理 |
| `image/` | 图像相关 |
| `system/` | 系统级功能 |
| `performance/` | 性能优化 |
| `accessibility/` | 无障碍功能 |
| `app/` | 应用全局状态 |
| `workspaces/` | 工作区管理 |

### 5.3 核心组件

- **`PromptPanel.vue`** (28KB)：提示词编辑主面板
- **`TemplateManager.vue`** (51KB)：模板管理器（最复杂的组件）
- **`FavoriteManager.vue`** (30KB)：收藏管理
- **`OutputDisplayCore.vue`** (19KB)：优化结果展示
- **`DataManager.vue`** (21KB)：数据导入导出
- **`ModelParameterEditor.vue`** (19KB)：模型参数编辑器
- **`ImageModelEditModal.vue`** (20KB)：图像模型编辑弹窗

---

## 6. Electron 桌面端架构 (核心开发关注点)

针对 Windows 桌面端开发，**必须深刻理解并严格遵循 IPC 代理模式**。在 Electron 中，主进程是负责核心逻辑和网络通信（Node.js 环境）的“大脑”，渲染进程是纯负责界面展示（浏览器环境）的“四肢”。

### 6.1 服务调用数据流

渲染进程（UI 层）**严禁**直接导入核心服务实例（否则会导致内存隔离的数据孤岛和 Node 环境限制问题）。所有跨进程调用必须遵循以下流程：

1. **UI 触发**: 用户交互触发 Vue 组件逻辑。
2. **代理拦截 (Proxy)**: 调用被转交至 `core` 包下的代理类（如 `ElectronLLMProxy`）。
3. **安全桥梁 (Preload)**: Proxy 调用预加载脚本暴露的安全接口（如 `window.electronAPI.llm.testConnection`）。
4. **IPC 通信**: `preload.js` 使用 `ipcRenderer.invoke` 将指令发送给主进程。
5. **主进程执行 (Main)**: `main.js` 中的 `ipcMain.handle` 捕获指令，调用主进程内持有的真正 `core` 服务实例执行逻辑。
6. **响应返回**: 结果以纯 JSON 格式原路返回给渲染进程。

### 6.2 接口修改及更新规范（⚠️ 极易出错的点）

如果在 `core` 包中**新增或修改了一个服务接口**（例如为 `LLMService` 添加新方法），**必须同时在以下 4 个地方同步修改**，缺一不可：

1. **Core 接口与实现**: 修改 `packages/core/src/services/*/types.ts` 及对应的实现逻辑文件。
2. **Core 代理层**: 在 `packages/core/src/services/*/electron-proxy.ts` 中实现代理和转发逻辑。
3. **主进程监听**: 在 `packages/desktop/main.js` 中增加或修改 `ipcMain.handle` 捕获处理函数。
4. **预加载层暴露**: 在 `packages/desktop/preload.js` 的 `window.electronAPI` 对应域中按规范暴露此方法。

### 6.3 桌面端调试与排错指南

- **主进程报错/网络请求异常**: 核心 API 调用（如大模型通信的 `node-fetch`）发生在主进程，报错信息**不会显示在浏览器的 Console 中**，而会打印在**启动应用所在环境的 PowerShell 终端（Terminal）**里。这是一个核心排错点。
- **渲染进程异常 (UI)**: 可以使用 `Ctrl+Shift+I` 打开开发者工具查看界面控制台。
- **热更新与重置**: 进行主进程 (`main.js`, `preload.js`) 修改后，如遇热更新异常，建议直接重启终端中的 `pnpm dev:desktop`；遇到复杂环境缓存问题，直接通过 `pnpm dev:desktop:fresh` 彻底清理重装。

---

## 7. 支持的 LLM 提供商

项目通过 Adapter 注册表模式支持多家 LLM 提供商：

| 提供商 | 环境变量前缀 |
|---|---|
| OpenAI | `VITE_OPENAI_API_KEY` |
| Google Gemini | `VITE_GEMINI_API_KEY` |
| Anthropic Claude | `VITE_ANTHROPIC_API_KEY` |
| DeepSeek | `VITE_DEEPSEEK_API_KEY` |
| 智谱 AI | `VITE_ZHIPU_API_KEY` |
| SiliconFlow | `VITE_SILICONFLOW_API_KEY` |
| ModelScope（魔搭） | `VITE_MODELSCOPE_API_KEY` |
| 自定义（如 Ollama） | `VITE_CUSTOM_API_KEY` / `VITE_CUSTOM_API_BASE_URL` / `VITE_CUSTOM_API_MODEL` |

### 多自定义模型配置

使用后缀格式 `VITE_CUSTOM_API_*_<suffix>` 可配置无限数量的自定义模型：

```env
VITE_CUSTOM_API_KEY_qwen3=your-key
VITE_CUSTOM_API_BASE_URL_qwen3=http://localhost:11434/v1
VITE_CUSTOM_API_MODEL_qwen3=qwen3:8b
```

后缀规则：只允许字母、数字、下划线、连字符（`[a-zA-Z0-9_-]`）。

---

## 8. 开发指令

### 日常开发

```bash
pnpm dev:fresh             # 推荐：清理缓存 → 重装依赖 → 构建 core/ui → 启动 Web 开发服务
pnpm dev                   # 构建 core/ui → 启动 Web 开发服务
pnpm dev:desktop           # 构建 core/ui → 同时启动 Web + Desktop
pnpm dev:desktop:fresh     # 彻底重置并启动 Desktop 开发
pnpm dev:ext               # 启动 Chrome 插件开发
```

### 构建

```bash
pnpm build                 # 按序构建 core → ui → web/ext（并行）
pnpm build:desktop         # 完整构建 core → ui → web → Desktop 打包
pnpm mcp:build             # 构建 MCP 服务器
```

### 测试

```bash
pnpm test                  # 运行所有测试（单元 + E2E）
pnpm test:unit             # 仅运行单元测试（所有包）
pnpm test:e2e              # 运行 Playwright E2E 测试
pnpm test:e2e:ui           # Playwright 交互式 UI 模式
pnpm test:gate             # 门禁测试（core + ui 关键测试）
pnpm test:gate:full        # 门禁测试 + E2E 回归

# 单包测试
pnpm -F @prompt-optimizer/core test
pnpm -F @prompt-optimizer/ui test
pnpm mcp:test
```

### 代码质量

```bash
pnpm lint                  # ESLint 检查（UI 包）
pnpm lint:fix              # ESLint 自动修复
pnpm clean                 # 清理所有 dist 和 vite 缓存
```

### 版本发布

```bash
pnpm version:prepare minor  # 更新版本号（不打 Tag）
pnpm run version:tag        # 创建 Git Tag（触发 GitHub Action 构建 Desktop）
pnpm run version:publish    # 推送 Tag 到远程
pnpm version:sync           # 同步所有子包版本号
```

---

## 9. 部署架构

### Vercel 部署（Web）

- `main` 分支推送触发生产部署
- `develop` 分支不触发部署
- 输出 `packages/web/dist`
- 包含 `api/` 目录的 Serverless Functions
- 注入 `VITE_VERCEL_DEPLOYMENT=true` 环境变量

### Docker 部署（Web + MCP Server）

Docker 镜像同时包含 Web 应用和 MCP 服务器：

```
Docker Container
├── Nginx (端口 80)
│   ├── Web 应用 (/usr/share/nginx/html)
│   └── 反向代理 /mcp → MCP 服务器
├── MCP Server (Node.js 进程)
└── Supervisord (进程管理)
```

支持访问控制：`ACCESS_USERNAME` / `ACCESS_PASSWORD` 环境变量。

### GitHub Actions CI/CD

| Workflow | 触发条件 | 作用 |
|---|---|---|
| `test.yml` | PR / Push | 运行测试 |
| `release.yml` | Tag 推送 (`v*`) | 构建 Desktop 应用（Win/Mac/Linux）并发布 GitHub Release |
| `docker.yml` | Tag 推送 | 构建并推送 Docker 镜像 |

---

## 10. 测试策略

| 层级 | 工具 | 位置 | 说明 |
|---|---|---|---|
| 单元测试 | Vitest | `packages/*/tests/unit/` | 服务方法、工具函数 |
| 集成测试 | Vitest | `packages/*/tests/integration/` | 跨服务工作流 |
| E2E 测试 | Playwright | `tests/e2e/` | 完整用户流程、回归测试 |
| 门禁测试 | Vitest/Playwright | — | `pnpm test:gate` 关键路径快速验证 |

测试文件命名：`*.test.ts` 或 `*.spec.ts`

---

## 11. 大模型开发行为约束

### 11.1 环境与语言

- 操作系统：**Windows 10**，Shell：**PowerShell 5.1**
- 优先使用 PowerShell 原生命令，可用 Git for Windows 的 GNU 工具集
- 所有对话、思考和文档必须使用**中文**

### 11.2 核心开发规范

1. **严格使用 pnpm**：所有的依赖安装、脚本运行只能使用 `pnpm`
2. **测试驱动**：代码修改后必须运行对应模块测试或 `pnpm test` 确保通过
3. **隔离解耦与依赖限制**：
   - 业务逻辑和 API 通信放在 `core` 包。
   - 展示组件放在 `ui` 包（**优先使用 Naive UI**）。
   - Desktop 开发**必须**使用 **IPC 代理模式**：在渲染流程中严禁直接导入或实例化 `core` 服务（会破坏单源数据结构并导致代码运行错误）。
4. **Desktop 接口严格同步机制**：
   - 只要修改或新增 core 服务层的任一接口方法，**必须闭环修改以下 4 处文件**：
     1. `core` 里的 `types.ts` 和具体逻辑实现文件
     2. `core` 里的 `electron-proxy.ts`（用于渲染进程的代理中继）
     3. `desktop/main.js`（增加主进程里的 `ipcMain.handle` 注册）
     4. `desktop/preload.js`（增加全局 `electronAPI` 的方法暴露）
   - 最后在 UI 层修改对应的 composable 进行对接。
5. **模块规范**：在除主进程主入口以外的大多数代码中，尽量使用 ESM (`import/export`) 并使用具名导入。
6. **TypeScript 严格**：核心接口必须明确定义类型，异步调用需 `try-catch` + 友好错误提示

### 11.3 代码风格

- Vue SFC 使用 PascalCase 命名（如 `PromptPanel.vue`）
- 目录使用 kebab-case（如 `variable-extraction/`）
- 遵循 Composition API，避免 Options API
- 使用 composables 封装业务逻辑，组件仅负责展示

### 11.4 分支管理

| 分支 | 用途 | 部署 |
|---|---|---|
| `main` | 生产分支 | 触发 Vercel 部署 |
| `develop` | 日常开发 | 不触发部署 |
| `feature/*` | 功能分支 | 从 develop 创建 |

版本发布流程：`pnpm version:prepare` → 提交 → `pnpm run version:tag` → `pnpm run version:publish`

### 11.5 辅助文档

- **`scratchpad.md`**：当前任务的规划、拆解与问题追踪。复杂任务前必须先建立或更新
- **`experience.md`**：可复用组件、踩坑教训、版本兼容问题及最佳实践。Bug 修复后应更新

### 11.6 操作前须知

> 由于项目代码量庞大（core 329 个文件、ui 297 个文件）、文件分布复杂，在进行具体功能修改前，**必须**先借助工具查询相关目录和文件路径，熟悉具体实现后再提修改方案。**切勿臆想接口或文件结构。**

---

## 12. 待办功能：系统提示词优化增加测试上下文

根据开发计划，为 "优化系统提示词" 的 Prompt 中引入原始提示词、测试内容及测试结果上下文。目前尚未进行开发，仅记录所需修改的文件和思路。

### 12.1 功能需求
1. 在“优化系统提示词”时，将**用户的原始提示词**、**测试内容**和**当前测试结果**一并发送给大模型。
2. 在此基础上，将**上一版本的系统提示词的测试结果**也一并发送给大模型。

### 12.2 涉及的代码文件与修改思路

#### 1. UI 层状态读取与请求构建
- **文件**: `packages/ui/src/composables/workspaces/useBasicWorkspaceLogic.ts`
- **思路**:
  - 在 `handleOptimize` 或相关提交逻辑中，获取当前的 `testContent.value`（测试内容）和 `testResults.value`（包含原始/优化后的测试结果的简易状态）。
  - 若需要严谨的多版本（V0/Vn）测试结果，可结合 `session.testVariants`（见 `packages/ui/src/stores/session/useBasicSystemSession.ts` 及 `BasicSystemWorkspace.vue` 中的逻辑）进行提取。
  - 将这些变量通过 `request.advancedContext.variables` 传入 `promptService.optimizePromptStream` 方法。

#### 2. Core 层模板处理器机制
- **文件**: `packages/core/src/services/template/processor.ts`
- **思路**:
  - 当前 `TemplateProcessor.processTemplate` 使用 Mustache 进行变量渲染。但需要注意的是，当前逻辑中只有当模板格式为 `Message[]`（数组高级模板）时，才会执行 `Mustache.render`，直接传字符串的简单模板跳过了渲染。
  - **关键点**: 若要在提示词模板中支持 `{{testContent}}`、`{{originalTestResult}}` 等变量替换，相关模板必须以 `Message[]`（数组）格式定义。

#### 3. Core 层模板文件更新
- **文件**: `packages/core/src/services/template/default-templates/optimize/*.ts` (如 `general-optimize.ts`, `analytical-optimize.ts` 等)
- **思路**:
  - 需要将现有的纯字符串 `content` 修改为 `[ { role: 'system', content: '...' }, { role: 'user', content: '...' } ]` 的格式，以便支持 Mustache 变量。
  - 在模板的提示词中增加动态条件渲染块，例如：
    ```hbs
    {{#testContent}}
    ### 用户的测试内容
    {{testContent}}
    {{/testContent}}

    {{#originalTestResult}}
    ### 原始提示词的测试结果（上一版本）
    {{originalTestResult}}
    {{/originalTestResult}}
    
    {{#optimizedTestResult}}
    ### 当前提示词的测试结果
    {{optimizedTestResult}}
    {{/optimizedTestResult}}
    ```
  - 通过这种方式，当传递了对应变量时，大模型便会收到完整的上下文以提供更具针对性的优化建议。
