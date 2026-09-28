# TWO-TOWEI

[![naming-gate](https://github.com/bfxh/TWO-TOWEI/actions/workflows/naming-gate.yml/badge.svg)](https://github.com/bfxh/TWO-TOWEI/actions/workflows/naming-gate.yml)

模块化 Rust 着色器框架：着色器以可复用模块在 Rust 侧定义与组合，编译为 WASM，在浏览器里用 WebGPU 渲染。2D 与 3D 同一套模块体系。

> **当前状态：M0（文档阶段）** —— 本仓库目前只有文档与仓库基建（命名门 / CI），无代码。

## 定位

- **模块化**：着色器由模块拼装（依赖声明 + 参数声明 + stage 片段），不整段手写大文件。
- **Rust 负责组合**：依赖解析、符号契约、参数布局、代码生成全部在 Rust 侧完成；浏览器只接收最终产物。
- **只接浏览器**：渲染目标只有浏览器（wasm32 + WebGPU），不做原生渲染路径；不依赖 PhysArena / BSHSQ / ADV 等任何现有项目。
- **2D / 3D 同体系**：同一套模块与组合机制，标准库分 2D 族与 3D 族。

## 架构速览

```
Rust（编译为 WASM）
├─ two-towei-core      模块定义 / 注册表 / 组合器 / 校验与诊断
├─ two-towei-modules   标准模块库（2D 族 + 3D 族 + fx 族）
├─ two-towei-emit-wgsl WGSL 生成：拼装、符号重写、binding 布局
└─ two-towei-runtime   wasm-bindgen：device / pipeline / canvas / render loop
        │  WASM + 最终 WGSL + 布局
        ▼
浏览器：WebGPU（WGSL）→ canvas      WebGL2 降级：后置（见 decisions D2）
```

完整设计：[docs/architecture.md](docs/architecture.md)

## 目录规划（M1 起落地）

| 路径 | 内容 |
|---|---|
| `crates/core/` | 模块系统与组合器 |
| `crates/modules/` | 标准模块库 |
| `crates/emit-wgsl/` | WGSL 目标生成 |
| `crates/runtime/` | 浏览器运行时（wasm-bindgen） |
| `web/` | 宿主页与引导（零框架） |
| `docs/` | 文档（当前阶段主体） |
| `scripts/` | 门禁脚本（naming_gate.py，CI 强制） |

## 路线图

| 里程碑 | 内容 | 状态 |
|---|---|---|
| M0 | 文档 + 仓库基建（命名门 / CI / dependabot） | ✅ 当前 |
| M1 | Cargo workspace 骨架 + 最小渲染（全屏 pass） | ⬜ |
| M2 | 模块系统完整 + 首批 2D 族模块 | ⬜ |
| M3 | 多 pass 后处理链（fx 族） | ⬜ |
| M4 | 3D 族首批（transform / camera / 光照 / mesh） | ⬜ |
| M5 | 参数系统（uniform 自动绑定；**调参 UI 不做**） | ⬜ |
| M6 | 节点编辑器（**已明确缓做**） | ⬜ 缓 |

明细与验收标准：[docs/roadmap.md](docs/roadmap.md)

## 非目标（现阶段明确不做）

- 调参 UI / inspector（2026-09-28 决定缓）
- 节点式编辑器（2026-09-28 决定缓，依赖 M5）
- WebGL2 降级路径（后置决策，见 decisions D2）
- 与任何现有项目的集成（独立项目）

## 约定

- 命名遵循 `scripts/naming_gate.py`（CI 强制）：禁占位名 / 随意词；crate 前缀 `two-towei-`，模块 id kebab-case，WGSL 符号 snake_case。
- 一 PR 一功能；修复以真实 CI 通过为准，不以本地预校验为准。
- 文档中文为主，代码与标识符英文。
