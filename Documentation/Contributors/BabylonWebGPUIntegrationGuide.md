# CesiumJS 渲染后端设计（WebGL + WebGPU）

> 目标：让 CesiumJS 同时支持 **两种渲染后端**（`webgl` / `webgpu`），并允许用户在创建 `Viewer` 时选择后端。

本文档描述了一条可落地的引擎演进路线：保留 Cesium 现有的场景/数据管线，同时新增 WebGPU 渲染实现，并参考 Babylon.js WebGPU 架构中的成熟模式。

## 产品需求

应用启动时，用户可以显式指定渲染接口：

```js
const viewer = new Cesium.Viewer("cesiumContainer", {
  renderingBackend: "webgpu", // "webgl" | "webgpu"
});
```

当请求的后端不可用时，行为应清晰且可配置（抛错、回退、或告警后回退）。

## 高层架构

不采用双 Canvas 叠加，而是采用 **单 Cesium 场景图 + 可插拔渲染后端**。

```text
Viewer / Scene / FrameState / DrawCommand
                  |
                  v
             渲染抽象层
          /                      \
      WebGL 后端             WebGPU 后端
   （现有实现路径）     （新增实现，参考 Babylon）
```

### 核心原则

- 保持 Cesium 高层对象稳定（如 `Scene`、`Primitive`、`Model`、`DrawCommand`、`RenderState` 等概念）。
- 将 API 相关逻辑（资源创建、管线绑定、pass 编码/提交）下沉到后端接口。
- 避免在核心场景逻辑中泄露 WebGPU 专有对象。

## 公共 API 提案

为 `Viewer` 增加新选项：

- `renderingBackend?: "webgl" | "webgpu"`
- 默认值：`"webgl"`（保证兼容性）

可选的健壮性配置：

- `renderingBackendFallback?: boolean`（默认 `true`）
- `onRenderingBackendFallback?: (requested, actual, reason) => void`

行为示例：

1. 用户请求 `webgpu`。
2. 引擎检查浏览器支持并尝试创建 adapter/device。
3. 成功则启用 WebGPU 后端。
4. 失败时：
   - 若允许回退：切到 WebGL，并触发回调/告警。
   - 若禁止回退：抛出明确的初始化错误。

## 渲染抽象接口面

定义内部接口（示意）：

- `createBuffer`、`createTexture`、`createSampler`
- `createShaderModule` / 着色器转换钩子
- `createRenderPipeline` / `createComputePipeline`（如需要）
- `beginFrame`、`beginPass`、`draw`、`endPass`、`submit`
- `readPixels`、`destroy`、能力查询

Cesium 的 `Context` 可演进为 `IRenderBackend` 的薄封装。

## WebGPU 后端实现策略（参考 Babylon）

Babylon.js WebGPU 引擎中有一些已在生产中验证的模式，可按 Cesium 特性进行映射：

1. **按状态键进行管线缓存**
   - 对顶点布局 + 混合/深度/光栅状态 + 着色器变体进行哈希。
   - 尽可能复用 `GPURenderPipeline`。
2. **保持 Bind Group Layout 稳定**
   - 规范资源绑定槽位，减少 bind group 频繁重建。
3. **逐帧资源暂存策略**
   - 动态 uniform 使用 ring buffer / staging buffer。
4. **按渲染 pass 编码命令**
   - 将不透明、半透明、后处理等 pass 分开编码。
5. **能力驱动的特性开关**
   - 依据 adapter 能力开启/关闭 MSAA、纹理格式、时间戳查询等功能。

这些思路与 Cesium 现有按 pass 组织渲染的模型较为契合。

## 着色器迁移计划

Cesium 当前以 GLSL 为中心。WebGPU 后端需要可确定、可维护的着色器管线：

- 短期：GLSL -> SPIR-V -> WGSL（工具链路径）或 GLSL -> WGSL 转换。
- 中期：统一到引擎 IR / 模块化着色器生成，同时产出 WebGL 兼容 GLSL 与 WebGPU WGSL。
- 对用户保持材质/外观层 API 稳定。

## 帧生命周期对齐

Cesium 主帧循环保持唯一权威：

1. `Scene.update` 构建命令列表。
2. 后端渲染器执行命令列表。
3. 输出并呈现最终帧。

不引入第二套引擎渲染循环。

## 兼容性与分阶段落地

### 阶段 1：基础设施
- 在 `Viewer` 增加后端选择参数。
- 引入后端抽象接口。
- WebGL 后端继续作为默认实现。

### 阶段 2：最小可用 WebGPU 纵切
- 清屏、基础几何、相机移动。
- 核心状态映射（depth/blend/cull）。
- 基础纹理与 uniform 绑定。

### 阶段 3：核心能力对齐
- 地形、影像图层、3D Tiles 主路径。
- 拾取路径。
- 阴影/后处理关键路径。

### 阶段 4：优化与加固
- 管线缓存调优。
- 内存生命周期审计。
- 跨浏览器一致性与性能基准测试。

## 错误处理与诊断

- 为 `Scene/Viewer` 增加后端选择与回退原因遥测。
- WebGPU 资源/渲染 pass 增加调试标签（GPU markers）。
- 现有 Cesium Inspector 工具尽可能保持后端无关。

## 测试矩阵

最小 CI/运行时矩阵：

- 后端维度：`webgl`、`webgpu`
- 功能维度：地形、影像图层、3D Tiles、模型渲染、拾取
- 浏览器维度：Chromium 系 WebGPU 基线 + WebGL 参考浏览器

对相同相机/时间输入，按后端做截图回归对比。

## 风险与约束

- WebGPU 可用性在不同浏览器/平台上存在差异。
- 着色器转换链可能成为长期维护热点。
- 两后端“像素级完全一致”通常需要设置合理容差。

## 总结

为满足该需求，Cesium 应演进为 **单引擎、双后端渲染架构**：

- 创建 `Viewer` 时可选择 `webgl` 或 `webgpu`。
- WebGPU 后端可借鉴 Babylon 的成熟模式（管线缓存、bind group、pass 编码），但实现仍保持 Cesium 原生后端。
- 为保证兼容性，分阶段推进，初期保持 WebGL 为默认值。
