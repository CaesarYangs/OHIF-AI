# OHIF-AI 前端文件详细指南

本文档详细解释Viewers目录下每个重要文件的具体作用和实现细节。

---

## 目录结构

```
Viewers/
├── 配置文件
├── extensions/          # 扩展模块
│   ├── cornerstone/    # 核心影像渲染
│   ├── cornerstone-dicom-seg/  # DICOM SEG支持
│   ├── cornerstone-dicom-rt/   # DICOM RT支持
│   ├── cornerstone-dicom-sr/   # DICOM SR支持
│   ├── default/        # 默认UI扩展
│   ├── measurement-tracking/   # 测量追踪
│   └── ...
├── modes/              # 应用模式
│   ├── longitudinal/   # 主模式(AI功能)
│   ├── segmentation/   # 分割模式
│   └── ...
├── platform/           # 平台核心
│   ├── app/           # 应用主体
│   ├── core/          # 核心库
│   ├── ui/            # UI组件库
│   └── ...
└── tests/             # 测试
```

---

## 根目录配置文件

### package.json
**路径**: `Viewers/package.json`

项目主配置文件，定义：
- `workspaces`: 工作区配置，包含extensions、modes、platform
- `scripts`: 构建脚本
  - `build`: 生产构建
  - `dev`: 开发服务器
  - `test`: 运行测试
- `dependencies`: 项目依赖
  - `@cornerstonejs/core`: 影像渲染核心
  - `@cornerstonejs/tools`: 影像工具
  - `react`: React框架
  - `zustand`: 状态管理

### rsbuild.config.ts
**路径**: `Viewers/rsbuild.config.ts`

RSBuild构建配置：
```typescript
// 主要配置项:
- source.entry: 入口文件
- server.port: 开发服务器端口
- output.distPath: 输出目录
- tools.swc: SWC编译器配置
- moduleFederation: 模块联邦配置(扩展机制核心)
```

### tsconfig.json
**路径**: `Viewers/tsconfig.json`

TypeScript编译配置：
- `compilerOptions.strict`: 严格类型检查
- `paths`: 路径别名配置
- `include`: 包含的文件范围

---

## extensions/cornerstone/ 详解

### package.json
```json
{
  "name": "@ohif/extension-cornerstone",
  "version": "3.0.0",
  "description": "OHIF Cornerstone Extension",
  "entry": "dist/index.tsx"
}
```

### index.tsx
**路径**: `extensions/cornerstone/src/index.tsx`

**作用**: 扩展主入口，导出所有模块给OHIF核心使用

**导出内容**:
```typescript
// 主要导出:
- id: 扩展唯一标识
- preRegistration: 预注册函数，初始化Cornerstone服务
- onModeEnter: 模式进入回调
- onModeExit: 模式退出回调
- getPanelModule: 面板模块
- getToolbarModule: 工具栏模块
- getViewportModule: 视口模块
- getCommandsModule: 命令模块
- getCustomizationModule: 自定义配置模块
- getHangingProtocolModule: 挂片协议模块
```

**关键代码**:
```typescript
const cornerstoneExtension: Types.Extensions.Extension = {
  id,
  
  // 模式进入时注册工具栏事件
  onModeEnter: ({ servicesManager }) => {
    const { toolbarService } = servicesManager.services;
    toolbarService.registerEventForToolbarUpdate(...);
  },
  
  // 预注册服务
  preRegistration: async function(props) {
    const { servicesManager } = props;
    // 注册核心服务
    servicesManager.registerService(CornerstoneViewportService.REGISTRATION);
    servicesManager.registerService(SegmentationService.REGISTRATION);
    // ... 更多服务
  }
};
```

### init.tsx
**路径**: `extensions/cornerstone/src/init.tsx`

**作用**: Cornerstone3D初始化

**初始化流程**:
1. 初始化Cornerstone3D核心
2. 初始化Cornerstone3D工具
3. 配置WebWorker
4. 设置图像加载器
5. 初始化工具组

### commandsModule.ts
**路径**: `extensions/cornerstone/src/commandsModule.ts`

**作用**: 定义Cornerstone相关的命令

**命令列表**:
```typescript
{
  // 视口操作
  'setViewportActive': 设置活动视口
  'showDownloadViewportModal': 显示下载模态框
  'clearMeasurements': 清除测量
  
  // 分割操作
  'createSegmentationForDisplaySet': 为显示集创建分割
  'downloadSegmentation': 下载分割
  
  // 工具操作
  'setToolActive': 设置工具激活
  'toggleTool': 切换工具
}
```

---

## services/ 服务目录

### CornerstoneViewportService.ts
**路径**: `extensions/cornerstone/src/services/ViewportService/CornerstoneViewportService.ts`

**作用**: 视口服务核心，管理所有医学影像的渲染

**主要方法**:
```typescript
class CornerstoneViewportService {
  // 启用视口
  enableViewport(viewportId, element): void
  
  // 设置视口数据
  setViewportData(viewportId, displaySet, viewportOptions): Promise<void>
  
  // 获取视口
  getCornerstoneViewport(viewportId): Viewport
  
  // 更新视口
  updateViewport(viewportId, options): void
  
  // 销毁视口
  destroyViewport(viewportId): void
}
```

**事件**:
- `VIEWPORT_DATA_CHANGED`: 视口数据变更
- `VIEWPORT_STACK_CHANGED`: 堆栈变更

### SegmentationService.ts
**路径**: `extensions/cornerstone/src/services/SegmentationService/SegmentationService.ts`

**作用**: 分割服务，管理分割的加载、显示和保存

**主要方法**:
```typescript
class SegmentationService {
  // 添加分割
  addSegmentation(segmentation): void
  
  // 添加分割表示
  addSegmentationRepresentation(viewportId, segmentationId): Promise<void>
  
  // 获取分割
  getSegmentation(segmentationId): Segmentation
  
  // 更新分割
  updateSegmentation(segmentationId, payload): void
  
  // 移除分割
  removeSegmentation(segmentationId): void
  
  // 为视口获取分割表示
  getSegmentationRepresentationsForViewport(viewportId): Representation[]
}
```

**事件**:
- `SEGMENTATION_ADDED`: 分割添加
- `SEGMENTATION_REMOVED`: 分割移除
- `SEGMENTATION_MODIFIED`: 分割修改

### ToolGroupService.ts
**路径**: `extensions/cornerstone/src/services/ToolGroupService/ToolGroupService.ts`

**作用**: 工具组服务，管理Cornerstone工具的激活和配置

---

## components/ 组件目录

### OHIFCornerstoneViewport.tsx
**路径**: `extensions/cornerstone/src/Viewport/OHIFCornerstoneViewport.tsx`

**作用**: 主要的影像视口组件

**Props**:
```typescript
interface Props {
  viewportId: string;           // 视口ID
  displaySet: DisplaySet;       // 显示集
  viewportOptions: Options;     // 视口选项
  servicesManager: ServicesManager;
  commandsManager: CommandsManager;
}
```

**功能**:
- 渲染医学影像
- 处理用户交互
- 管理视口状态
- 响应分割变化

### PanelSegmentation.tsx
**路径**: `extensions/cornerstone/src/panels/PanelSegmentation.tsx`

**作用**: 右侧面板中的分割面板

**功能**:
- 显示分割列表
- 管理分割可见性
- 调整分割配置
- 触发AI推理

### CinePlayer.tsx
**路径**: `extensions/cornerstone/src/components/CinePlayer/CinePlayer.tsx`

**作用**: 影像播放控制器

**功能**:
- 播放/暂停
- 帧率控制
- 进度显示

---

## utils/ 工具函数

### segmentUtils.ts
**路径**: `extensions/cornerstone/src/utils/segmentUtils.ts`

**作用**: 分割相关工具函数

**函数列表**:
```typescript
// 创建分割显示集
function createSegmentationDisplaySet(...)

// 获取分割的样式
function getSegmentationStyle(...)

// 检查分割是否可见
function isSegmentVisible(...)
```

### dicomLoaderService.ts
**路径**: `extensions/cornerstone/src/utils/dicomLoaderService.ts`

**作用**: DICOM图像加载服务

**功能**:
- 配置图像加载器
- 处理WADO-RS请求
- 管理加载队列

---

## hooks/ React Hooks

### useSegmentations.ts
**路径**: `extensions/cornerstone/src/hooks/useSegmentations.ts`

**作用**: 分割状态管理Hook

**用法**:
```typescript
const { segmentations, activeSegmentation } = useSegmentations();
```

### useMeasurements.ts
**路径**: `extensions/cornerstone/src/hooks/useMeasurements.ts`

**作用**: 测量状态管理Hook

---

## extensions/default/ 默认扩展

### commandsModule.ts
**路径**: `extensions/default/src/commandsModule.ts`

**作用**: 默认命令集

**命令**:
```typescript
{
  // 导航命令
  'navigateToStudy': 导航到研究
  'navigateToViewer': 导航到查看器
  
  // UI命令
  'toggleSidePanel': 切换侧面板
  'showContextMenu': 显示上下文菜单
}
```

### ViewerLayout/index.tsx
**路径**: `extensions/default/src/ViewerLayout/index.tsx`

**作用**: 主查看器布局组件

**布局结构**:
```
┌─────────────────────────────────────┐
│  Header (ViewerHeader)              │
├──────────────────┬──────────────────┤
│  Left Panel      │  Viewport Grid    │
│  (StudyBrowser)  │  (影像显示区域)    │
│                  │                  │
│                  ├──────────────────┤
│                  │  Right Panel      │
│                  │  (Segmentation)   │
└──────────────────┴──────────────────┘
```

### Panels/StudyBrowser/PanelStudyBrowser.tsx
**路径**: `extensions/default/src/Panels/StudyBrowser/PanelStudyBrowser.tsx`

**作用**: 左侧研究浏览器面板

**功能**:
- 显示研究列表
- 显示序列缩略图
- 拖放加载影像

---

## modes/longitudinal/ 纵向模式

### index.ts
**路径**: `modes/longitudinal/src/index.ts`

**作用**: 纵向模式入口

**模式配置**:
```typescript
function modeFactory({ modeConfiguration }) {
  return {
    id: 'longitudinal',
    routeName: 'viewer',
    displayName: i18n.t('Modes:Basic Viewer'),
    
    // 模式进入时初始化
    onModeEnter: ({ servicesManager }) => {
      // 初始化工具组
      initToolGroups(...);
      
      // 添加工具栏按钮
      toolbarService.addButtons(toolbarButtons);
      
      // 创建工具栏区域
      toolbarService.createButtonSection('primary', [...]);
      toolbarService.createButtonSection('aiToolBoxSection', [...]);  // AI工具
    }
  };
}
```

### toolbarButtons.ts
**路径**: `modes/longitudinal/src/toolbarButtons.ts`

**作用**: **AI功能的核心配置文件**

**AI工具箱配置**:
```typescript
{
  id: 'aiToolBoxContainer',
  label: 'AI Toolbox',
  icon: 'icon-ai-brain',
  buttons: [
    {
      id: 'Probe2',
      label: 'Point Prompt',
      icon: 'icon-tool-point',
      commands: [{
        commandName: 'setToolActive',
        commandOptions: { toolName: 'Probe' }
      }]
    },
    {
      id: 'PlanarFreehandROI2',
      label: 'Scribble',
      icon: 'icon-tool-scribble',
      commands: [...]
    },
    {
      id: 'PlanarFreehandROI3',
      label: 'Lasso',
      icon: 'icon-tool-lasso',
      commands: [...]
    },
    {
      id: 'RectangleROI2',
      label: 'Bounding Box',
      icon: 'icon-tool-bbox',
      commands: [...]
    },
    {
      id: 'nninter',
      label: 'nnInteractive',
      commands: [{
        commandName: 'setAIModel',
        commandOptions: { model: 'nninter' }
      }]
    }
  ]
}
```

**文本提示分割配置**:
```typescript
{
  id: 'textPromptSegmentation',
  label: 'Text Prompt',
  component: TextPromptComponent,
  props: {
    placeholder: 'Enter anatomical structure...'
  }
}
```

**MedGemma配置**:
```typescript
{
  id: 'testMedgemma',
  label: 'Generate Report',
  component: MedGemmaPanel,
  props: {
    instruction: 'You are a radiology assistant',
    query: 'Generate a report for this CT scan'
  }
}
```

### initToolGroups.ts
**路径**: `modes/longitudinal/src/initToolGroups.ts`

**作用**: 初始化工具组

**工具组列表**:
```typescript
- 'default': 默认工具组
  - WindowLevelTool: 窗宽窗位
  - PanTool: 平移
  - ZoomTool: 缩放
  - StackScrollTool: 堆栈滚动
  
- 'segmentation': 分割工具组
  - BrushTool: 画笔
  - EraserTool: 橡皮擦
  
- 'ai': AI工具组
  - ProbeTool: 点提示
  - PlanarFreehandROITool: 涂鸦/套索
  - RectangleROITool: 边界框
```

---

## platform/core/ 平台核心

### services/ 服务目录

#### ServicesManager.ts
**路径**: `platform/core/src/services/ServicesManager.ts`

**作用**: 服务管理器，管理所有服务的生命周期

**方法**:
```typescript
class ServicesManager {
  registerService(registration): void
  getService(name): Service
  destroy(): void
}
```

#### ExtensionManager.ts
**路径**: `platform/core/src/services/ExtensionManager.ts`

**作用**: 扩展管理器，加载和管理扩展

**功能**:
- 动态加载扩展
- 管理扩展依赖
- 扩展生命周期管理

---

## 前端数据流详解

### 1. 工具激活流程

```
用户点击工具按钮
    ↓
toolbarButtons.ts 中的 command 定义
    ↓
commandsModule.ts 中的命令处理
    ↓
CornerstoneTools.setToolActive()
    ↓
工具激活，等待用户交互
```

### 2. AI推理流程

```
用户绘制提示 (点/涂鸦/框)
    ↓
Cornerstone Tools 捕获交互
    ↓
收集提示坐标 → 转换到图像坐标
    ↓
调用 commandsManager.runCommand('runAIInference', {...})
    ↓
发送 HTTP POST 到 /monai/infer/segmentation
    ↓
后端处理，返回 DICOM-SEG
    ↓
SegmentationService.addSegmentation()
    ↓
视口更新，显示分割结果
```

### 3. 分割显示流程

```
收到分割数据 (DICOM-SEG)
    ↓
SegmentationService.addSegmentation()
    ↓
创建 Segmentation DisplaySet
    ↓
Cornerstone Cache 加载分割体积
    ↓
SegmentationService.addSegmentationRepresentation()
    ↓
Cornerstone Viewport 渲染分割
    ↓
用户看到分割叠加在影像上
```

---

## 关键文件修改指南

### 添加新的AI工具

**修改文件**: `modes/longitudinal/src/toolbarButtons.ts`

```typescript
{
  id: 'myNewTool',
  label: 'My New Tool',
  icon: 'icon-custom',
  commands: [{
    commandName: 'runCustomAI',
    commandOptions: { type: 'custom' }
  }]
}
```

### 修改视口布局

**修改文件**: `modes/longitudinal/src/index.ts`

```typescript
layoutTemplate: () => ({
  id: ohif.layout,
  props: {
    leftPanels: [...],
    rightPanels: [...],  // 修改这里
    viewports: [...]
  }
})
```

### 添加新的命令

**修改文件**: `extensions/cornerstone/src/commandsModule.ts`

```typescript
const commandsModule = ({
  commands: {
    myNewCommand: ({ param }) => {
      // 命令逻辑
    }
  }
});
```

---

*本指南详细说明了前端各文件的作用和相互关系，更多细节请参考具体源码*
