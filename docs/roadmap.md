# 路线图

约定：一里程碑 = 多个独立 PR（一 PR 一功能，旁支与依赖变更单开）；**完成 = 真实 CI 绿 + 验收项满足**，不以本地预校验为准。

## M0 — 文档与仓库基建（当前）

- 范围：README / architecture / decisions / roadmap；命名门 + CI + dependabot
- 验收：文档评审通过；naming-gate CI 绿
- 非范围：任何 Rust 代码

## M1 — workspace 骨架 + 最小渲染

- 范围：Cargo workspace（core / modules / emit-wgsl / runtime 四 crate 空骨架）；emit-wgsl 直通一份手写最小 WGSL；runtime 起 adapter / device + 全屏三角形；`web/` 宿主页；CI 加 fmt / clippy / test；dependabot 加 cargo 生态
- 验收：打开页面渲染出全屏渐变三角形；CI 绿
- 非范围：模块组合逻辑（M2）

## M2 — 模块系统 + 2D 族首批

- 范围：Module trait / 注册表 / 依赖闭包 + 环检测 / 符号表与重写 / 特化；参数声明结构（仅结构，不实装绑定）；首批模块：`gradient-linear`、`sdf-shapes`、`noise-value`
- 验收：≥2 个模块组合出真实效果；依赖缺失 / 环 / 符号冲突 / 参数重名有结构化报错；composer 单测覆盖
- 非范围：参数值热更新（M5）；多 pass（M3）

## M3 — 后处理链（fx 族）

- 范围：多 pass / framebuffer / pass 顺序图；`fx-blur`、`fx-bloom`、`fx-color-grade`
- 验收：≥3 pass 链式效果正确；pass 图可静态校验（环 / 读写冲突报错）

## M4 — 3D 族首批

- 范围：`transform`、`camera`、`lighting-basic`、`material-unlit`；最小 mesh 渲染（顶点缓冲 / 深度）
- 验收：一个由模块组合出的 3D 场景（相机 + 光照 + ≥1 mesh）
- 非范围：PBR / 阴影（后续按需）

## M5 — 参数系统（不含调参 UI）

- 范围：uniform 布局生成 + 自动绑定 + 默认值 + 代码侧改值热生效
- 明确非目标：inspector / 滑杆 / 面板（2026-09-28 用户明确缓）
- 验收：代码改一个参数值，画面随之变化，无整页刷新

## M6 — 节点编辑器（缓做）

- 依赖：M5 + 热替换
- 状态：2026-09-28 用户明确"那些节点先不要写" —— 不排期，等明确指令

## 顺序依据

先用 2D（M2）+ 后处理（M3）把组合机制与 pass 图磨熟 —— 这两族不引入相机 / 深度 / mesh 状态，模块组合的正确性容易验证；3D（M4）在组合机制稳定后再上，把状态复杂度与组合复杂度解耦。

## 后置决策项（不阻塞）

- WebGL2 降级路径（decisions D2）
- LICENSE 选择（BSHSQ 用 MIT OR Apache-2.0，倾向一致，M1 定）
- wasm 体积预算（首个可用版本目标值 M1 定）
