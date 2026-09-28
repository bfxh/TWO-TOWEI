# 设计决策

| # | 决策 | 日期 | 状态 |
|---|---|---|---|
| D1 | 编写方式：Rust 声明 + WGSL 片段，不做全 Rust DSL 转译 | 2026-09-28 | 已定 |
| D2 | 唯一首目标 WebGPU / WGSL；WebGL2 后置 | 2026-09-28 | 已定 |
| D3 | 宿主页零框架零构建链 | 2026-09-28 | 已定 |
| D4 | 命名门 + CI 从 M0 起生效 | 2026-09-28 | 已定 |
| D5 | 调参 UI 与节点编辑器缓做 | 2026-09-28 | 已定（用户指令） |

## D1 着色器编写方式

"着色器是 Rust 的"有两条实现路径：

- **A. 全 Rust DSL → 转译 WGSL**：类型安全到表达式级，但要长期维护一个转译器，工程量与风险不可控。
- **B. Rust 声明（依赖 / 参数 / 符号契约）+ stage 主体 WGSL 片段**：组合期完成全部校验，无转译器；产物是可直接在浏览器调试的 WGSL。

选 **B**。理由：交付确定性（产物即浏览器原生调试语言）、避免转译器长期维护成本。A 在 B 稳定且有真实诉求时再评估。

## D2 渲染后端

WebGPU 2026 年中已在 Chrome / Edge / Safari 26 / Firefox 默认启用（W3C Candidate Recommendation）；双后端抽象是当前阶段的过度设计。emit 层保留 `Emitter` trait，为后置 `emit-glsl` 留口；有真实降级需求再引入。

## D3 宿主

不引 React / Vite 等构建链；`web/` 只有静态文件 + 极少量原生 JS 引导。wasm 工具链（wasm-pack 或 wasm-bindgen 直出）在 M1 定。

## D4 仓库基建

`scripts/naming_gate.py` 自包含拷贝（与 BSHSQ / ADV 同一门脚本），CI 强制。crate 前缀 `two-towei-`；模块 id kebab-case；WGSL 符号 snake_case。本仓无退役旧名，`--forbid` 暂空。

## D5 缓做项

调参 UI（M5 注记）与节点编辑器（M6 注记）为用户 2026-09-28 明确缓做项，任何里程碑不得夹带。
