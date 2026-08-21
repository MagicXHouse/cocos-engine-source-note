## 前言

理解 RenderScene 和 Node 指针树，是掌握 Cocos 渲染架构的两个关键。本文将讲清楚二者的概念。

**注：本文基于 4.0.0 之前的版本。**

## 一、什么是 RenderScene？

场景（Scene）是游戏引擎的展示容器。在一个场景中，我们可以放置各种对象（光源、相机、角色、敌人、环境等）。

场景在编辑器中保存为 `.scene` 资源文件（序列化后的静态数据），引擎运行时通过 `RenderScene` 实例来管理渲染逻辑。

在 Cocos 引擎中，每个场景都由 `RenderScene` 对象管理：场景是资源层的外部呈现，`RenderScene` 是运行时的代码逻辑单元。

`RenderScene` 维护着多个渲染对象数组，分别管理场景中不同类型的渲染元素：

- `_models`：3D 模型（`MeshRenderer`、`SkinningModel` 等）
- `_cameras`：相机
- `_sphereLights` / `_spotLights` / `_pointLights`：各类光源
- `_batches`：2D 渲染批次（`Sprite` 等）
- `_lodGroups`：LOD 组

当往场景中添加新模型时，会先通过 TS 层的 `mesh-renderer.ts` 将模型添加到场景，再传递给原生平台，如下所示：

```
Component.onEnable()
  → _attachToScene()
    → renderScene.addModel(this._model)   // TS 层
      → RenderScene::addModel(Model*)     // C++ 层
        → _models.emplace_back(model)     // 模型指针存入 vector
        → model->attachToScene(this)      // 反向存 RenderScene* 指针
```

在游戏的每一帧，`RenderScene` 都会更新场景中所有需要更新的模型（带有 dirty 标记，例如被用户修改后下一帧必须更新）。

引擎遍历场景中的所有模型，并根据模型信息依次渲染。

## 二、什么是 Node 指针树？

Node 指针树管理了场景中各种节点（Node）的父子关系。使用 Node 指针树的好处是：对父节点的某些属性（如 `transform`、可见性）设置可以传递到子节点。

比如有一个人物模型非常复杂，我们可以用一个根节点代表这个人物，头、四肢、身躯分别是一级子节点，眼睛、头发则是头的二级子节点。假设我们希望这个人物不可见，只需要在根节点将可见性设置为 `false` 即可；假设我们需要这个人物放大两倍，只需要将根节点的缩放设为两倍，子节点会自动继承这个属性。

在游戏初始化或更新阶段，开发者需要先创建场景，然后在场景中创建 Node 及其子节点。每次添加、删除 Node，或者往 Node 中添加模型，都会更新 Node 指针树。

Node 树的更新核心是 `transform` 的层级传播。当父节点的 `transform`（位置、旋转、缩放）发生变化时，Node 指针树会自动触发级联更新：

- 父节点标记 `_changedFlags`（脏标记），表示 `transform` 已变更。
- 遍历所有子节点，递归传播脏标记。
- 每帧渲染时，`RenderScene` 遍历 Model 数组，Model 通过持有的 `_node` 指针检查脏标记，若脏则重新计算世界矩阵。

Model 持有 `_node` 指针，但不持有 Node 的所有权。Node 的生命周期由场景图管理，Model 只是引用它。Model 通过 `_node` 获取：

- 世界矩阵（`_node->getWorldMatrix()`）——用于渲染 `transform`。
- 层级信息（`_node->getLayer()`）——用于剔除。
- 脏标记（`_node->getChangedFlags()`）——判断是否需要更新。

## 三、RenderScene 和 Node 指针树如何协作

```
       用户/编辑器          Node 指针树              RenderScene              Pipeline
           │                   │                        │                        │
  ═══════════════════════════ 场景初始化阶段 ═══════════════════════════════════════
           │                   │                        │                        │
           │──创建 Scene─────→│                        │                        │
           │                   │──创建 RenderScene───→│                        │
           │                   │                        │──初始化各数组          │
           │                   │                        │  (_models、_cameras 等)│
           │                   │                        │                        │
           │──创建 Node 树───→│                        │                        │
           │  (parent/children)│                        │                        │
           │                   │                        │                        │
  ═══════════════════════════ 组件激活阶段 ═════════════════════════════════════════
           │                   │                        │                        │
           │──挂载组件────────→│                        │                        │
           │  MeshRenderer     │                        │                        │
           │  CameraComponent  │                        │                        │
           │  ...              │                        │                        │
           │                   │                        │                        │
           │           Component.onEnable()             │                        │
           │           _attachToScene()  ──────────────→│                        │
           │                   │           addModel()   │                        │
           │                   │           addCamera()  │                        │
           │                   │                        │──_models.emplace_back() │
           │                   │                        │──model->attachToScene()│
           │                   │                        │  (存 RenderScene* 引用) │
           │                   │                        │──插入 Octree（若启用） │
           │                   │                        │                        │
  ═══════════════════════════ 每帧更新阶段 ═════════════════════════════════════════
           │                   │                        │                        │
           │──用户修改────────→│                        │                        │
           │  transform/       │                        │                        │
           │  visibility       │                        │                        │
           │                   │                        │                        │
           │                   │──标记脏节点             │                        │
           │                   │  _changedFlags |= DIRTY│                        │
           │                   │──递归传播脏标记到子节点  │                        │
           │                   │                        │                        │
           │                   │          RenderScene::update(stamp)              │
           │                   │      ◄──────────────────│                        │
           │                   │                        │                        │
           │                   │                        │──更新主光源             │
           │                   │                        │  _mainLight->update()   │
           │                   │                        │                        │
           │                   │                        │──遍历 _models[]        │
           │                   │                        │  for each model:        │
           │                   │                        │                        │
           │                   │  model->updateTransform(stamp)                   │
           │                   │      ◄──────────────────│                        │
           │                   │                        │                        │
           │                   │  检查 _node->getChangedFlags()                  │
           │                   │  ├──脏：重新计算世界矩阵  │                        │
           │                   │  │  _node->updateWorldTransform()               │
           │                   │  │  更新 _worldBounds    │                        │
           │                   │  │  _localDataUpdated = true                     │
           │                   │  └──不脏：跳过            │                        │
           │                   │                        │                        │
           │                   │                        │  model->updateUBOs()    │
           │                   │                        │  上传 transform 到 GPU UBO│
           │                   │                        │                        │
           │                   │                        │──更新 LOD、Octree      │
           │                   │                        │──更新各光源             │
           │                   │                        │                        │
  ═══════════════════════════ 渲染管线阶段 ═════════════════════════════════════════
           │                   │                        │                        │
           │                   │                        │  Pipeline 获取场景数据─→│
           │                   │                        │  renderScene->getModels()│
           │                   │                        │  renderScene->getCameras()│
           │                   │                        │                        │
           │                   │                        │             Camera::update()
           │                   │      ◄─────────────────────────────────────────────│
           │                   │  检查 _node->getChangedFlags()                      │
           │                   │  若脏：重新计算 _viewMatrix、_matViewProj          │
           │                   │                        │                        │
           │                   │                        │            视锥剔除 ────→│
           │                   │                        │            遍历 models[] │
           │                   │                        │            frustumCull() │
           │                   │                        │                        │
           │                   │                        │            排序 ────────→│
           │                   │                        │            按材质 / 深度 │
           │                   │                        │                        │
           │                   │                        │            提交 Draw Call→│
           │                   │                        │            绑定 VBO / EBO│
           │                   │                        │            绑定材质      │
           │                   │                        │            drawIndexed() │
           │                   │                        │                        │
  ═══════════════════════════ 帧结束 ═══════════════════════════════════════════════

```

## 四、重要文件汇总

| 文件                                   | 含义                                                         |
| -------------------------------------- | ------------------------------------------------------------ |
| `native/cocos/core/scene-graph/Node.h` | Node 指针树，管理节点的父子关系和 `transform` 层级传播       |
| `native/cocos/scene/RenderScene.h`     | `RenderScene` 管理类，维护所有渲染对象的数组（`_models` / `_cameras` / `_lights` 等） |
| `native/cocos/scene/Model.h`           | `Model` 类，最小可渲染单元，持有 `_node` 指针和 `SubModel` 列表 |
| `cocos/3d/framework/mesh-renderer.ts`  | 通过 `_attachToScene()` → `addModel()` 将 `Model` 注册到 `RenderScene` |
| `cocos/misc/camera-component.ts`       | 通过 `_attachToScene()` → `addCamera()` 将 `Camera` 注册到 `RenderScene` |

注：C++ 侧仅提供头文件，实现类可通过头文件跳转查看。

## 五、总结

RenderScene 和 Node 指针树是 Cocos 渲染架构中两个独立但协作的管理系统，分别负责“渲染什么”和“在哪里渲染”。

**Node 指针树负责层级关系**：它管理节点的父子结构和 `transform` 的层级传播。父节点位置变化时，脏标记会递归传播到所有子节点，但 Node 树本身不触发渲染更新——它只负责“标记”。

**RenderScene 负责渲染调度**：它维护着场景中所有渲染对象（模型、相机、光源）的数组。每帧遍历这些数组，交给 Pipeline 做剔除、排序和绘制。RenderScene 不关心父子关系，只关心“有哪些东西要画”。

二者通过 Model 对象连接：Model 同时持有 `_node` 指针（引用 Node 树中的 `transform` 数据）和 `SubModel` 列表（引用网格和材质资源）。每帧 `RenderScene` 遍历 Model 时，Model 主动检查 `_node` 是否脏了，脏了才更新世界矩阵——这种“拉取”模式使得没变化的 Model 零开销，避免了全量更新的浪费。

理解这一层关系，是理解 Cocos 渲染器的基础。