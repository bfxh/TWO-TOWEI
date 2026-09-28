# 架构

## 1. 总览

| 层 | 载体 | 职责 | 运行环境 |
|---|---|---|---|
| 定义 authoring | `two-towei-core` | ShaderModule 抽象、注册表 | 纯 Rust，可单测 |
| 组合 composition | `two-towei-core` | 依赖解析、符号表、参数合并、特化 | 纯 Rust，可单测 |
| 生成 emit | `two-towei-emit-wgsl` | 最终 WGSL 文本 + 管线布局描述 | 纯 Rust，可单测 |
| 运行 runtime | `two-towei-runtime` | device / pipeline / canvas / render loop | wasm（浏览器） |
| 宿主 host | `web/` | 静态页 + 引导 JS | 浏览器 |

原则：前三层不依赖 wasm，可在纯 Rust 环境开发与测试；浏览器只出现在最后两层。渲染目标只有浏览器，不做原生路径。

## 2. 核心概念

### 2.1 ShaderModule

模块 = 可复用的着色器构造单元：

| 字段 | 说明 |
|---|---|
| `id` | 稳定标识，kebab-case（如 `sdf-shapes`） |
| `deps` | 依赖模块列表，构成 DAG，禁止环 |
| `params` | 参数声明：名字 / 类型 / 默认值（类型系统见 §5） |
| `provides` | 对外提供的 WGSL 符号（函数 / 结构体） |
| `requires` | 要求外部提供的符号，由依赖满足，缺失即报错 |
| `fragments` | stage 片段：vertex / fragment 的 WGSL 片段 + 挂载点 |

模块只声明"我提供什么、需要什么"，不感知最终组合结果。

### 2.2 Composer

输入：入口模块 + 特化参数。流程：

1. **依赖闭包** → 拓扑排序 → 环检测
2. **符号表**：`provides` / `requires` 匹配；重名冲突检测；必要时按模块 id 前缀重写引用
3. **参数合并**：全部模块的 `params` 合成单一 uniform 布局（对齐规则固定）
4. **stage 拼装**：按挂载点顺序拼接片段，注入参数绑定点
5. **产出** `ComposedShader`：
   - `wgsl: String` —— 可直接交给浏览器的最终文本
   - `layout: PipelineLayoutDesc` —— binding 布局，与 wgsl 一一对应
   - `report: CompositionReport` —— 参与了哪些模块、折叠了哪些开关

### 2.3 特化（specialization）

- 布尔开关 → 组合期条件拼接（WGSL 无宏，不做文本预处理）
- 常量 → 组合期内联折叠
- 目标：同一模块库按特化产出不同 WGSL，产物始终是完整、可直接调试的文本

## 3. crate 划分与依赖方向

```
two-towei-modules ──▶ two-towei-core ◀── two-towei-emit-wgsl
                          ▲
              two-towei-runtime（wasm-bindgen，聚合以上三层）
```

- `core`：trait、注册表、composer、校验、诊断。零 wasm 依赖。
- `modules`：标准模块库，纯声明数据，只依赖 core。
- `emit-wgsl`：`ComposedShader` 内部 IR → WGSL 文本。唯一知道 WGSL 语法的层，为后置的 `emit-glsl` 留同类接口（`Emitter` trait）。
- `runtime`：wasm-bindgen 导出 `init(canvas)` / `set_entry(...)` 等；持 device / pipeline / 缓冲；驱动 render loop；支持热替换。
- `web/`：`index.html` + 引导 JS，零框架零构建链（M1 细化工具链）。

## 4. 标准模块族

| 族 | 首批模块（规划） | 说明 |
|---|---|---|
| 2D | `gradient-linear`、`sdf-shapes`、`noise-value` | 全屏 quad 上的片段效果，组合机制的最小试炼场 |
| fx | `fx-blur`、`fx-bloom`、`fx-color-grade` | 后处理 pass，链式组合，2D / 3D 通用 |
| 3D | `transform`、`camera`、`lighting-basic`、`material-unlit` | 引入 mesh / 深度 / 相机状态，复杂度上一个台阶 |

推进顺序：2D → fx → 3D（理由见 roadmap）。

## 5. 参数系统（接口 M2 预留，实装 M5）

- 声明式：`Param::f32("radius").default(0.5).range(0.0..=1.0)`（`range` 仅元数据，不做 UI）
- 布局：Rust 侧按固定对齐规则合成 uniform buffer 布局，与 WGSL binding 一一对应
- **明确非目标：调参 UI** —— 参数可被代码修改并热生效，但不上 inspector / 滑杆面板（2026-09-28 决定缓）

## 6. 渲染后端

- 唯一首目标：**WebGPU + WGSL**。2026 年中状态：Chrome / Edge 113+、Safari 26、Firefox 141/147+ 均默认启用（W3C Candidate Recommendation）；Linux / Android 覆盖仍不完整。
- WebGL2 + GLSL ES 3.0 降级：后置决策（decisions D2），emit 层留 `Emitter` trait。
- 不做双后端同时支持的抽象层 —— 有真实降级需求再引入。

## 7. 数据流

```
模块定义(Rust) ─组合─▶ ComposedShader{wgsl,layout} ─wasm 边界─▶ 浏览器创建 pipeline
参数值(Rust,可变) ──────uniform buffer────────────────────────▶ render loop ─▶ canvas
```

## 8. 错误处理

| 阶段 | 错误 | 处理 |
|---|---|---|
| 组合期 | 依赖缺失 / 环 / 符号冲突 / 参数重名 | 结构化诊断（模块 id + 行号 + 原因），Rust 侧拦截 |
| 生成期 | 挂载点不合法 / 片段顺序矛盾 | 同上 |
| 浏览器编译期 | WGSL 编译错误 | 错误信息经日志桥回传 Rust 侧 |
| 运行期 | device 丢失 / 上下文重建 | runtime 层报告，页面显示 |

原则：能在组合期拦住的错误，绝不留给浏览器。

## 9. 热替换（为节点编辑器预留的底层能力）

runtime 支持运行时：换入口模块 / 改特化 / 改参数值 → 重新组合 → 重建 pipeline，canvas 不重载。节点编辑器（M6，缓做）将来只是这层能力之上的一个前端。
