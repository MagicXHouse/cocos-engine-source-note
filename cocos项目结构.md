## 一、顶层架构 — 项目根目录

```plaintext
cocos4/
├── cocos/          ← TypeScript 引擎核心（用户层 API）
├── native/         ← C++ 原生引擎层（高性能底层）
├── pal/            ← 平台抽象层 (Platform Abstraction Layer)
├── exports/        ← 公开 API 模块（用户能 import 的入口）
├── editor/         ← 编辑器相关代码
├── extensions/     ← 引擎扩展（目前仅有 ccpool）
├── platforms/      ← 平台适配相关
├── external/       ← 外部依赖（引擎自带的第三方库）
├── vendor/         ← 供应商代码
├── scripts/        ← 构建脚本
├── templates/      ← 项目模板
├── tests/          ← 测试用例
├── docs/           ← 文档
├── @types/         ← TypeScript 类型声明
├── .circleci/      ← CI 配置
├── .github/        ← GitHub 配置
├── predefine.ts    ← 入口：导出 legacyCC 全局对象
└── package.json    ← 版本 4.0.0-alpha.30
```

## 二、分层架构总图

```
┌─────────────────────────────────────────────────────┐
│                    用户层 (TypeScript API)           │
│  cocos/  — 场景、节点、组件、2D/3D、UI、动画、物理等  │
│  exports/  — 公开模块入口，用户 `import { ... } from 'cc'` │
├─────────────────────────────────────────────────────┤
│                  平台抽象层 (PAL)                     │
│  pal/  — 音频、输入、屏幕适配、系统信息、环境检测等    │
├─────────────────────────────────────────────────────┤
│                   C++ 原生引擎层                       │
│  native/cocos/  — 渲染器、场景管理、物理、绑定桥接等   │
│  native/cocos/bindings/  — JS←→C++ 绑定 (JSB)      │
├─────────────────────────────────────────────────────┤
│                  编辑器集成层                          │
│  editor/  — 编辑器面板、Inspector、国际化等            │
└─────────────────────────────────────────────────────┘
```

## 三、逐层总结

### 第 1 层：TypeScript 引擎核心（`cocos/`）

这是游戏开发者直接调用的 API 层，涵盖所有游戏开发功能模块：

| 目录                        | 用途                                                         |
| :-------------------------- | :----------------------------------------------------------- |
| **`cocos/core/`**           | **核心基础库**：全局导出(`global-exports`)、算法、曲线、数据结构、事件系统、几何工具、数学库、平台抽象、工具函数、值类型（`Vec3`/`Mat4`/`Quat` 等） |
| **`cocos/scene-graph/`**    | **场景图**：`Node`、`Scene`、`Component` 等场景管理核心，含 prefab 支持 |
| **`cocos/asset/`**          | **资源管理**：`Asset` 基类、`asset-manager` 加载/缓存/释放管线 |
| **`cocos/rendering/`**      | **渲染管线 (TS 侧)**：自定义渲染管线、forward/deferred、后处理、阴影、反射探针、legacy 兼容 |
| **`cocos/render-scene/`**   | **渲染场景**：场景内 render scene 对象（core + scene 子模块） |
| **`cocos/gfx/`**            | **图形抽象层 (TS 侧)**：`base` 定义、`webgl`/`webgl2`/`webgpu` 后端实现 |
| **`cocos/2d/`**             | **2D 渲染**：assembler、2D assets、组件、事件、framework、renderer |
| **`cocos/3d/`**             | **3D 渲染**：3D assets、framework、lights、LOD、models、reflection-probe、skeletal-animation、skinned-mesh-renderer |
| **`cocos/ui/`**             | **UI 系统**：editbox 等 UI 组件                              |
| **`cocos/animation/`**      | **动画系统**：核心动画、embedded-player、事件、exotic-animation、marionette(状态机)、tracks、value-proxy |
| **`cocos/physics/`**        | **物理引擎**：bullet/cannon/cocos/physx 后端适配 + framework + spec |
| **`cocos/physics-2d/`**     | **2D 物理**                                                  |
| **`cocos/particle/`**       | **粒子系统**：animator、emitter、models、renderer            |
| **`cocos/particle-2d/`**    | **2D 粒子**                                                  |
| **`cocos/audio/`**          | **音频系统**                                                 |
| **`cocos/input/`**          | **输入系统**：键盘、鼠标、触摸、手柄等                       |
| **`cocos/terrain/`**        | **地形系统**                                                 |
| **`cocos/tiledmap/`**       | **瓦片地图**                                                 |
| **`cocos/spine/`**          | **Spine 骨骼动画**                                           |
| **`cocos/dragon-bones/`**   | **DragonBones 骨骼动画**                                     |
| **`cocos/tween/`**          | **Tween 缓动系统**                                           |
| **`cocos/video/`**          | **视频播放**                                                 |
| **`cocos/web-view/`**       | **WebView 组件**                                             |
| **`cocos/webgpu/`**         | **WebGPU 支持**                                              |
| **`cocos/xr/`**             | **XR 扩展**                                                  |
| **`cocos/serialization/`**  | **序列化**                                                   |
| **`cocos/sorting/`**        | **排序**                                                     |
| **`cocos/primitive/`**      | **基本几何体**（立方体、球体等）                             |
| **`cocos/profiler/`**       | **性能分析器**                                               |
| **`cocos/game/`**           | **游戏循环**：主循环、Game 对象                              |
| **`cocos/gi/`**             | **全局光照**                                                 |
| **`cocos/misc/`**           | **杂项工具**                                                 |
| **`cocos/native-binding/`** | **原生绑定适配**                                             |

### 第 2 层：平台抽象层（`pal/`）

为上层提供统一平台接口，屏蔽不同平台（Web、Native、小游戏等）的差异：

| 目录                      | 用途             |
| :------------------------ | :--------------- |
| **`pal/audio/`**          | 音频抽象         |
| **`pal/input/`**          | 输入抽象         |
| **`pal/env/`**            | 环境检测         |
| **`pal/minigame/`**       | 小游戏平台适配   |
| **`pal/pacer/`**          | 帧率控制         |
| **`pal/screen-adapter/`** | 屏幕适配         |
| **`pal/system-info/`**    | 系统信息         |
| **`pal/wasm/`**           | WebAssembly 支持 |

### 第 3 层：C++ 原生引擎（`native/cocos/`）

这是引擎的高性能底层，渲染、物理、场景管理等计算密集型工作在此实现：

| 目录                               | 用途                                                         |
| :--------------------------------- | :----------------------------------------------------------- |
| **`native/cocos/renderer/`**       | **渲染系统核心**：                                           |
| └── `core/`                        | 渲染核心数据结构                                             |
| └── `pipeline/`                    | 渲染管线（forward、deferred、custom、shadow、reflection-probe、xr） |
| └── `frame-graph/`                 | 帧图（Frame Graph）资源管理                                  |
| └── `gfx-base/`                    | 图形抽象层基类                                               |
| └── `gfx-gles2/` / `gfx-gles3/`    | OpenGL ES 后端                                               |
| └── `gfx-vulkan/`                  | Vulkan 后端（Windows/Android）                               |
| └── `gfx-metal/`                   | Metal 后端（macOS/iOS）                                      |
| └── `gfx-wgpu/`                    | WebGPU 后端                                                  |
| └── `gfx-agent/`                   | GFX 代理（多线程）                                           |
| └── `gfx-validator/`               | GFX 验证层（调试用）                                         |
| └── `gfx-empty/`                   | 空后端（占位）                                               |
| **`native/cocos/scene/`**          | **C++ 场景管理**（含 raytracing）                            |
| **`native/cocos/2d/`**             | 2D 渲染（C++ 侧）                                            |
| **`native/cocos/3d/`**             | 3D 渲染（C++ 侧）：assets、models、skeletal-animation、misc  |
| **`native/cocos/physics/`**        | 物理引擎（C++ 侧）：physx、sdk、spec                         |
| **`native/cocos/animation/`**      | 动画（C++ 侧）                                               |
| **`native/cocos/assets/`**         | 资源（C++ 侧）                                               |
| **`native/cocos/core/`**           | 核心基础库（C++ 侧）：data、event、geometry、math、memop、platform、scene-graph、utils |
| **`native/cocos/base/`**           | 基础设施：job-system、memory、std（扩展）、threading         |
| **`native/cocos/math/`**           | 数学库                                                       |
| **`native/cocos/bindings/`**       | **JSB (JavaScript Bindings)**：JS←→C++ 绑定桥接              |
| └── `jswrapper/`                   | V8 / SpiderMonkey 的 JS 包装器                               |
| └── `sebind/`                      | 新一代绑定方案                                               |
| └── `dop/`                         | 数据导向优化绑定                                             |
| └── `event/`                       | 绑定事件                                                     |
| └── `manual/`                      | 手动绑定代码                                                 |
| └── `utils/`                       | 绑定工具                                                     |
| **`native/cocos/application/`**    | 应用入口                                                     |
| **`native/cocos/engine/`**         | 引擎初始化/生命周期                                          |
| **`native/cocos/main/`**           | 主函数入口                                                   |
| **`native/cocos/platform/`**       | 平台适配（Win/Mac/iOS/Android 等）                           |
| **`native/cocos/audio/`**          | 音频（C++ 侧）                                               |
| **`native/cocos/input/`**          | 输入（C++ 侧）                                               |
| **`native/cocos/network/`**        | 网络                                                         |
| **`native/cocos/storage/`**        | 本地存储                                                     |
| **`native/cocos/ui/`**             | UI（C++ 侧）                                                 |
| **`native/cocos/gi/`**             | 全局光照（C++ 侧）                                           |
| **`native/cocos/profiler/`**       | 性能分析器（C++ 侧）                                         |
| **`native/cocos/primitive/`**      | 基本几何体（C++ 侧）                                         |
| **`native/cocos/plugins/`**        | 插件系统                                                     |
| **`native/cocos/editor-support/`** | 编辑器支持（如 spine-wasm）                                  |
| **`native/cocos/xr/`**             | XR（C++ 侧）                                                 |
| **`native/cocos/...`**             | 其他                                                         |

### 第 4 层：编辑器集成（`editor/`）

| 目录                          | 用途                                       |
| :---------------------------- | :----------------------------------------- |
| **`editor/src/`**             | 编辑器核心源码（含 marionette 动画编辑器） |
| **`editor/assets/`**          | 编辑器资源                                 |
| **`editor/dashboard/`**       | 编辑器 Dashboard                           |
| **`editor/engine-features/`** | 引擎功能特性配置（`render-config.json`）   |
| **`editor/exports/`**         | 编辑器专用公开模块                         |
| **`editor/i18n/`**            | 国际化                                     |
| **`editor/inspector/`**       | Inspector 面板                             |

### 第 5 层：公开 API 模块（`exports/`）

`exports/` 目录下的文件是用户可以看到的公开 API 入口。用户的 `import { ... } from 'cc'` 最终会从这些模块重新导出。通过 `cc.config.json` 配置哪些模块被包含。