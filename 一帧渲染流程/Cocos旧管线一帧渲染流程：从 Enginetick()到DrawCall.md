## 前言

本文的目的是讲清楚旧管线（ForwardPipeline、DefferedPipeline）一帧之内的完整渲染流程——从引擎的帧循环入口开始，经过 Director 调度、Root 管理、场景剔除、管线组织，最终在 RenderQueue 中提交 DrawCall。

## 一帧渲染的完整链路

```
原生平台 (C++)                          Web 平台 (TS)
══════════════════════                  ══════════════════════
Engine::tick()                          Game._updateCallback()
│                                       │
├── _gfxDevice->frameSync()             │
├── sleep_for (帧率限制)                │
├── _scheduler->update(dt)              │
├── mainLoopUpdate()                    │
│       │                               │
│       ▼                               ▼
│   Director.tick(dt) ←───────────────┘
│       │
│       ├── 更新阶段
│       │   ├── 组件 start/update/lateUpdate
│       │   ├── 系统 update/postUpdate
│       │   └── 延迟销毁对象
│       │
│       ├── 渲染阶段
│       │   ├── 更新脏 UI 渲染器
│       │   └── Root.frameMove(dt)
│       │       │
│       │       ├── _frameMoveBegin
│       │       │   ├── 清空 scenes 的 Batch
│       │       │   └── 清空 cameraList
│       │       │
│       │       ├── _frameMoveProcess
│       │       │   ├── 从窗口提取相机列表
│       │       │   ├── 2D 合批 (Batcher2D)
│       │       │   └── scenes[i].update(stamp)
│       │       │
│       │       └── _frameMoveEnd
│       │           ├── 按 priority 排序相机
│       │           └── pipeline.render(cameraList)
│       │               │
│       │               └── 对每个相机：
│       │                   ├── validPunctualLightsCulling
│       │                   ├── sceneCulling
│       │                   ├── updateGlobalUBO / updateCameraUBO
│       │                   └── 遍历 flows[j].render(camera)
│       │                       │
│       │                       └── 遍历 stages[k].render(camera)
│       │                           │
│       │                           ├── 清空 renderQueues
│       │                           ├── 收集 SubModel/Pass
│       │                           ├── renderQueues.sort()
│       │                           └── renderQueues.recordCommandBuffer()
│       │                               └── GPU 执行 DrawCall
│       │
│       └── 清理 Node 标记、更新总帧数
│
├── DeferredReleasePool::clear()
└── dt 平滑计算
```

## 第一步：帧入口

### 1、原生平台 (C++)

`native/cocos/engine/Engine.cpp` 中的 `Engine::tick()` 是原生平台的帧循环入口，由操作系统定时器驱动，每帧执行。源码验证如下：

```cpp
void Engine::tick() {
    // 1、帧起始与 GPU 同步
    _gfxDevice->frameSync();

    // 2、帧率限制（Android/Windows/OpenHarmony/macOS）
    if (dtNS < _preferredNanosecondsPerFrame) {
        std::this_thread::sleep_for(...);
    }

    // 3、核心逻辑更新
    _scheduler->update(dt);
    events::Tick::broadcast(dt);

    // 4、渲染主循环（C++ → TS 的桥梁）
    se::ScriptEngine::getInstance()->mainLoopUpdate();

    // 5、资源回收与异步回调
    DeferredReleasePool::clear();
    _scheduler->runFunctionsToBePerformedInCocosThread();

    // 6、DeltaTime 平滑
    dtNS = dtNS * 0.1 + 0.9 * ...;
}


```

六个阶段的作用如下：

| 阶段         | 关键代码                        | 核心任务                                        |
| :----------- | :------------------------------ | :---------------------------------------------- |
| ① 帧起始     | `_gfxDevice->frameSync()`       | 等待上一帧 GPU 命令执行完毕，回收 CommandBuffer |
| ② 帧率限制   | `sleep_for(...)`                | 省电控温，防止设备过热降频                      |
| ③ 逻辑更新   | `_scheduler->update(dt)`        | 驱动定时器、Action、用户脚本 update             |
| ④ 渲染主循环 | `mainLoopUpdate()`              | 进入 TS 层，触发整个渲染管线                    |
| ⑤ 资源回收   | `DeferredReleasePool::clear()`  | 释放标记销毁的对象，执行异步回调                |
| ⑥ dt 平滑    | `dtNS = dtNS * 0.1 + 0.9 * ...` | 指数平滑法，避免帧率波动导致逻辑跳跃            |

### 2、Web 平台 (TS)

`cocos/game/game.ts` 中的 `Game._updateCallback()` 是 Web 平台的入口，由浏览器 `requestAnimationFrame` 驱动：

```Typescript
private _updateCallback (): void {
    if (!this._inited) return;
    if (/* 闪屏阶段 */) {
        SplashScreen.instance.update(...);
    } else if (this._shouldLoadLaunchScene) {
        // 加载启动场景
        director.loadScene(launchScene, ...);
    } else {
        director.tick(this._calculateDT(false));  // 正常帧循环
    }
}
```

两条路径最终都汇聚到 `Director.tick(dt)`。

## 第二步：Director 调度

`cocos/game/director.ts` 是主循环的核心实现。它将一帧拆成两个阶段：

```typescript
public tick (dt: number): void {
    if (!this._invalid) {
        this.emit(DirectorEvent.BEGIN_FRAME);
        // 【更新阶段】
        if (!this._paused) {
            this.emit(DirectorEvent.BEFORE_UPDATE);
            this._compScheduler.startPhase();       // 组件 start
            this._compScheduler.updatePhase(dt);    // 组件 update
            this._systems[i].update(dt);            // 系统 update
            this._compScheduler.lateUpdatePhase(dt);// 组件 lateUpdate
            this.emit(DirectorEvent.AFTER_UPDATE);
            CCObject._deferredDestroy();            // 延迟销毁
            this._systems[i].postUpdate(dt);        // 系统 postUpdate
        }
        // 【渲染阶段】
        this.emit(DirectorEvent.BEFORE_DRAW);
        uiRendererManager.updateAllDirtyRenderers(); // 更新脏 UI 渲染器
        this._root!.frameMove(dt);                   // 执行实际渲染
        this.emit(DirectorEvent.AFTER_DRAW);
        // 收尾
        Node.resetHasChangedFlags();
        Node.clearNodeArray();
        this.emit(DirectorEvent.END_FRAME);
        this._totalFrames++;
    }
}
```

## 第三步：Root 总控

`cocos/root.ts` 中的 `Root` 是渲染系统的顶层管理器。`Director` 只管调度，真正的渲染工作交给 `Root`：

```typescript
public frameMove (deltaTime: number): void {
    this._frameMoveBegin();
    this._frameMoveProcess();
    this._frameMoveEnd();
}
```

三个阶段的具体任务：

**`_frameMoveBegin`**：清空上帧的 2D Batch 和相机列表。

```typescript
private _frameMoveBegin (): void {
    for (let i = 0; i < this._scenes.length; ++i) {
        this._scenes[i].removeBatches();
    }
    this._cameraList.length = 0;
}
```

**`_frameMoveProcess`**：提取相机、更新场景、2D 合批。

```typescript
private _frameMoveProcess (): void {
    // 1. 从所有窗口提取相机
    for (let i = 0; i < windows.length; i++) {
        windows[i].extractRenderCameras(cameraList);
    }
    // 2. 2D 合批
    if (this._batcher) {
        this._batcher.update();
        this._batcher.uploadBuffers();
    }
    // 3. 更新所有 RenderScene
    for (let i = 0; i < scenes.length; i++) {
        scenes[i].update(stamp);
    }
}
```

**`_frameMoveEnd`**：排序相机、调用管线渲染、提交到屏幕。

```typescript
private _frameMoveEnd (): void {
    cameraList.sort((a, b) => a.priority - b.priority);  // 按 priority 排序
    for (let i = 0; i < cameraList.length; ++i) {
        cameraList[i].geometryRenderer?.update();
    }
    this._pipeline.render(cameraList);  // 进入管线
    this._device.present();             // 提交到屏幕
}
```

Root 持有以下核心资源：

| 成员          | 含义                                   |
| :------------ | :------------------------------------- |
| `_device`     | GFX 设备，所有 GPU 操作的入口          |
| `_windows`    | 渲染窗口数组                           |
| `_scenes`     | RenderScene 数组，每个逻辑场景对应一个 |
| `_pipeline`   | 渲染管线（前向/延迟/自定义）           |
| `_batcher`    | 2D 合批器                              |
| `_cameraList` | 当前帧的活跃相机列表                   |

## 第四步：灯光剔除与场景主剔除

进入管线后，`cocos/rendering/render-pipeline.ts` 的 `RenderPipeline.render()` 对每个相机执行两种剔除：

```typescript
public render (cameras: Camera[]): void {
    this._commandBuffers[0].begin();
    for (let i = 0; i < cameras.length; i++) {
        const camera = cameras[i];
        if (camera.scene) {
            validPunctualLightsCulling(this.pipelineSceneData, camera);  // ①
            sceneCulling(this.pipelineSceneData, this.pipelineUBO, camera);  // ②
            this._pipelineUBO.updateGlobalUBO(camera.window);
            this._pipelineUBO.updateCameraUBO(camera);
            for (let j = 0; j < this._flows.length; j++) {
                this._flows[j].render(camera);  // ③
            }
        }
    }
    this._commandBuffers[0].end();
    this._device.queue.submit(this._commandBuffers);
}
```

### 1、灯光剔除 `validPunctualLightsCulling`

从场景的所有点光源、聚光灯、球光源、范围方向光中，筛选出对当前相机可见的光源，存入 `sceneData.validPunctualLights`，供后续光照计算使用。

### 2、场景主剔除 `sceneCulling`

源码验证（`cocos/rendering/scene-culling.ts`）：

```typescript
export function sceneCulling (sceneData, pipelineUBO, camera): void {
    const models = scene.models;
    const visibility = camera.visibility;

    function enqueueRenderObject (model: Model): void {
        if (model.enabled) {
            // LOD 检查
            if (scene.isCulledByLod(camera, model)) return;
            // 收集阴影投射对象
            if (model.castShadow) {
                castShadowObjects.push(...);
                csmLayerObjects.push(...);
            }
            // 图层匹配检查
            if ((model.node && ((visibility & model.node.layer) === model.node.layer))
                || (visibility & model.visFlags)) {
                // 视锥体剔除
                if (model.worldBounds && !geometry.intersect.aabbFrustum(model.worldBounds, camera.frustum)) {
                    return;
                }
                renderObjects.push(getRenderObject(model, camera));
            }
        }
    }

    for (let i = 0; i < models.length; i++) {
        enqueueRenderObject(models[i]);
    }
}

```

剔除判断条件依次为：**LOD 级别 → 图层匹配 → 视锥体相交**。通过全部检查的 Model 被放入 `renderObjects` 数组，进入后续的渲染队列。

## 第五步：管线组织 — Pipeline → Flow → Stage → Queue

旧管线采用分层架构：

```typescript
RenderPipeline
  └── RenderFlow[]        ← 按优先级排列的渲染流程
       └── RenderStage[]  ← 每个流程包含多个渲染阶段
            └── RenderQueue[] ← 每个阶段包含不透明/透明队列
```

### 1、前向管线 (ForwardPipeline)

`cocos/rendering/forward/forward-pipeline.ts` 初始化时注册三个 Flow：

```typescript
public initialize (info: IRenderPipelineInfo): boolean {
    super.initialize(info);
    if (this._flows.length === 0) {
        // 1. 阴影流程（优先级最高）
        const shadowFlow = new ShadowFlow();
        shadowFlow.initialize(ShadowFlow.initInfo);
        this._flows.push(shadowFlow);
        
        // 2. 反射探针流程
        const reflectionFlow = new ReflectionProbeFlow();
        reflectionFlow.initialize(ReflectionProbeFlow.initInfo);
        this._flows.push(reflectionFlow);

        // 3. 前向渲染主流程
        const forwardFlow = new ForwardFlow();
        forwardFlow.initialize(ForwardFlow.initInfo);
        this._flows.push(forwardFlow);
    }
    return true;
}
```

### 2、延迟管线 (DeferredPipeline)

`cocos/rendering/deferred/deferred-pipeline.ts` 注册两个 Flow：

```typescript
if (this._flows.length === 0) {
    const shadowFlow = new ShadowFlow();
    this._flows.push(shadowFlow);
    const mainFlow = new MainFlow();  // 包含 GBuffer 阶段 + 光照阶段
    this._flows.push(mainFlow);
}
```

### 3、RenderFlow 的 render 方法

`cocos/rendering/render-flow.ts` 中，`RenderFlow.render()` 遍历所有 Stage：

```typescript
public render (camera: Camera): void {
    for (let i = 0, len = this._stages.length; i < len; i++) {
        if (this._stages[i].enabled) this._stages[i].render(camera);
    }
}
```

### 4、ForwardFlow 的 Stage 组成

以 `ForwardFlow` 为例，它只包含一个 `ForwardStage`：

```typescript
public initialize (info: IRenderFlowInfo): boolean {
    super.initialize(info);
    if (this._stages.length === 0) {
        const forwardStage = new ForwardStage();
        forwardStage.initialize(ForwardStage.initInfo);
        this._stages.push(forwardStage);
    }
    return true;
}
```

### 5、ShadowFlow 的 Stage 组成

`cocos/rendering/shadow/shadow-flow.ts` 同样只包含一个 `ShadowStage`：

```typescript
public initialize (info: IRenderFlowInfo): boolean {
    super.initialize(info);
    if (this._stages.length === 0) {
        const shadowMapStage = new ShadowStage();
        shadowMapStage.initialize(ShadowStage.initInfo);
        this._stages.push(shadowMapStage);
    }
    return true;
}
```

## 第六步：RenderStage 执行渲染

`cocos/rendering/forward/forward-stage.ts` 的 `ForwardStage.render()` 是旧管线最核心的执行逻辑：

```typescript
public render (camera: Camera): void {
    // 1. 清空所有 renderQueue
    this._renderQueues.forEach(renderQueueClearFunc);

    const renderObjects = pipeline.pipelineSceneData.renderObjects;

    // 2. 遍历 renderObjects，收集 SubModel/Pass
    for (let i = 0; i < renderObjects.length; ++i) {
        const ro = renderObjects[i];
        const subModels = ro.model.subModels;
        for (m = 0; m < subModels.length; m++) {
            const passes = subModels[m].passes;
            for (p = 0; p < passes.length; p++) {
                // 将 Pass 插入对应的 RenderQueue
                for (k = 0; k < this._renderQueues.length; k++) {
                    this._renderQueues[k].insertRenderPass(ro, m, p);
                }
            }
        }
    }

    // 3. 排序
    this._renderQueues.forEach(renderQueueSortFunc);

    // 4. 录制命令缓冲
    const cmdBuff = pipeline.commandBuffers[0];
    // 不透明队列先渲染
    this._renderQueues[0].recordCommandBuffer(device, renderPass, cmdBuff);
    // 实例化队列
    this._instancedQueue.recordCommandBuffer(device, renderPass, cmdBuff);
    // 附加光源队列
    this._additiveLightQueue.recordCommandBuffer(device, renderPass, cmdBuff);
    // 透明队列后渲染
    this._renderQueues[1].recordCommandBuffer(device, renderPass, cmdBuff);
    // 平面阴影队列
    this._planarQueue.recordCommandBuffer(device, renderPass, cmdBuff);
    // UI 渲染
    this._uiPhase.render(camera, renderPass);
    // Profiler 渲染
    renderProfiler(device, renderPass, cmdBuff, pipeline.profiler, camera);
    cmdBuff.endRenderPass();
}

```

ForwardStage 初始化时定义了两个 RenderQueue：

```typescript
public static initInfo: IRenderStageInfo = {
    name: 'ForwardStage',
    priority: ForwardStagePriority.FORWARD,
    renderQueues: [
        {
            isTransparent: false,
            sortMode: RenderQueueSortMode.FRONT_TO_BACK,   // 不透明：从前往后
            stages: ['default'],
        },
        {
            isTransparent: true,
            sortMode: RenderQueueSortMode.BACK_TO_FRONT,   // 透明：从后往前
            stages: ['default', 'planarShadow'],
        },
    ],
};
```

## 第七步：RenderQueue 排序与录制

`cocos/rendering/render-queue.ts` 的 `RenderQueue` 负责 Pass 的排序和命令录制：

### 1、插入 Pass

```typescript
public insertRenderPass (renderObj, subModelIdx, passIdx): boolean {
    const subModel = renderObj.model.subModels[subModelIdx];
    const pass = subModel.passes[passIdx];
    const isTransparent = pass.blendState.targets[0].blend;
    // 过滤：透明/不透明类型必须匹配，且 Pass 的 phase 必须在队列允许的范围内
    if (isTransparent !== this._passDesc.isTransparent || !(pass.phase & this._passDesc.phases)) {
        return false;
    }
    // 生成 hash：pass.priority + subModel.priority + passIdx
    const hash = (0 << 30) | (pass.priority << 16) | (subModel.priority << 8) | passIdx;
    // 插入队列
    this.queue.push({ subModel, passIdx, hash, depth });
    return true;
}
```

### 2、排序

```typescript
export function renderQueueSortFunc (rq: RenderQueue): void {
    rq.sort();  // 按 hash 排序，同状态尽量合并
}
```

### 3、录制命令缓冲

```typescript
public recordCommandBuffer (device, renderPass, cmdBuff): void {
    for (let i = 0; i < this.queue.length; ++i) {
        const { subModel, passIdx } = this.queue.array[i];
        const { inputAssembler } = subModel;
        const pass = subModel.passes[passIdx];
        const shader = subModel.shaders[passIdx];
        // 创建或获取 PSO
        const pso = PipelineStateManager.getOrCreatePipelineState(device, pass, shader, renderPass, inputAssembler);
        // 绑定流水线状态
        cmdBuff.bindPipelineState(pso);
        // 绑定材质描述符集
        cmdBuff.bindDescriptorSet(SetIndex.MATERIAL, pass.descriptorSet);
        // 绑定本地描述符集
        cmdBuff.bindDescriptorSet(SetIndex.LOCAL, subModel.descriptorSet);
        // 绑定输入汇集器
        cmdBuff.bindInputAssembler(inputAssembler);
        // 执行绘制
        cmdBuff.draw(inputAssembler);
    }
}
```

## 重要文件汇总

| 文件                                            | 含义                                           |
| :---------------------------------------------- | :--------------------------------------------- |
| `native/cocos/engine/Engine.cpp`                | 原生平台帧循环入口，`Engine::tick()`           |
| `cocos/game/game.ts`                            | Web 平台帧循环入口，`Game._updateCallback()`   |
| `cocos/game/director.ts`                        | 主循环调度器，拆分更新阶段和渲染阶段           |
| `cocos/root.ts`                                 | 渲染系统顶层管理器，持有设备、窗口、管线、场景 |
| `cocos/rendering/render-pipeline.ts`            | 旧管线基类，定义 Pipeline → Flow → Stage 框架  |
| `cocos/rendering/render-flow.ts`                | RenderFlow 基类，遍历 Stage 执行渲染           |
| `cocos/rendering/render-stage.ts`               | RenderStage 抽象基类，定义 render 接口         |
| `cocos/rendering/scene-culling.ts`              | 场景剔除：灯光、阴影、主场景可见性筛选         |
| `cocos/rendering/render-queue.ts`               | RenderQueue：Pass 插入、排序、命令缓冲录制     |
| `cocos/rendering/forward/forward-pipeline.ts`   | 前向管线，Flow = [Shadow, Reflection, Forward] |
| `cocos/rendering/forward/forward-flow.ts`       | 前向渲染流程                                   |
| `cocos/rendering/forward/forward-stage.ts`      | 前向渲染阶段，核心渲染逻辑                     |
| `cocos/rendering/deferred/deferred-pipeline.ts` | 延迟管线，Flow = [Shadow, Main]                |
| `cocos/rendering/shadow/shadow-flow.ts`         | 阴影渲染流程                                   |

## 总结

Cocos 旧管线的一帧渲染，从 `Engine::tick()` 或 `Game._updateCallback()` 开始，经过七层调用最终到达 GPU：

```typescript
Engine::tick() / Game._updateCallback()
  → Director.tick()            ← 拆分更新与渲染
    → Root.frameMove()          ← 管理场景、相机、管线
      → Pipeline.render()       ← 对每个相机做剔除 + 遍历 Flow
        → RenderFlow.render()   ← 遍历 Stage
          → RenderStage.render()← 收集 SubModel/Pass → 排序 → 录制
            → RenderQueue.recordCommandBuffer()
              → GPU DrawCall
```

前向管线（ForwardPipeline）和延迟管线（DeferredPipeline）的区别在于 Flow 的组合不同，但底层都遵循相同的 Pipeline → Flow → Stage → Queue 分层架构。
