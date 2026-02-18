# OHIF-AI 完整文件参考手册

本文档提供项目中每个重要文件的详细说明。

---

## 📁 项目根目录

| 文件/目录 | 类型 | 说明 |
|----------|------|------|
| `README.md` | 文档 | 项目介绍、功能说明、使用指南 |
| `DOCUMENTATION.md` | 文档 | 文档导航入口 |
| `ARCHITECTURE.md` | 文档 | 架构详解 |
| `IMPLEMENTATION_DETAILS.md` | 文档 | 实现原理 |
| `MODULE_INDEX.md` | 文档 | 模块索引 |
| `COMPLETE_FILE_REFERENCE.md` | 文档 | 本文件，完整文件参考 |
| `docker-compose.yml` | 配置 | Docker Compose部署配置 |
| `start.sh` | 脚本 | 项目启动脚本 |
| `LICENSE` | 文档 | 项目许可证 |
| `.gitignore` | 配置 | Git忽略规则 |

---

## 📁 Viewers/ - 前端代码目录

### Viewers 根目录文件

| 文件 | 类型 | 说明 |
|------|------|------|
| `package.json` | 配置 | npm包配置，定义依赖和脚本 |
| `rsbuild.config.ts` | 配置 | RSBuild构建配置 |
| `tsconfig.json` | 配置 | TypeScript配置 |
| `babel.config.js` | 配置 | Babel转译配置 |
| `jest.config.js` | 配置 | Jest测试配置 |
| `lerna.json` | 配置 | Lerna monorepo配置 |
| `nx.json` | 配置 | Nx构建系统配置 |
| `Dockerfile` | 配置 | 前端Docker镜像构建 |
| `CHANGELOG.md` | 文档 | 变更日志 |
| `CONTRIBUTING.md` | 文档 | 贡献指南 |
| `LICENSE` | 文档 | 许可证 |

---

## 📁 Viewers/extensions/ - 扩展模块

### cornerstone - 核心影像渲染扩展

**目录**: `Viewers/extensions/cornerstone/`

| 文件 | 类型 | 说明 |
|------|------|------|
| `package.json` | 配置 | 扩展包配置 |
| `index.tsx` | 入口 | 扩展主入口，导出所有模块 |
| `id.js` | 常量 | 扩展ID定义 |
| `enums.ts` | 枚举 | 扩展枚举定义 |
| `state.ts` | 状态 | 全局状态管理 |

#### cornerstone/src/

| 文件 | 类型 | 说明 |
|------|------|------|
| `init.tsx` | 初始化 | Cornerstone初始化逻辑 |
| `commandsModule.ts` | 命令 | 命令定义模块 |
| `getCustomizationModule.tsx` | 配置 | 自定义配置模块 |
| `getHangingProtocolModule.ts` | 协议 | 挂片协议模块 |
| `getPanelModule.tsx` | 面板 | 面板模块 |
| `getToolbarModule.tsx` | 工具栏 | 工具栏模块 |
| `getSopClassHandlerModule.ts` | 处理器 | SOP类处理器 |

##### cornerstone/src/services/ - 核心服务

| 文件 | 类型 | 说明 |
|------|------|------|
| `ViewportService/CornerstoneViewportService.ts` | 服务 | 视口服务核心，管理医学影像渲染 |
| `ViewportService/IViewportService.ts` | 接口 | 视口服务接口 |
| `ViewportService/Viewport.ts` | 类 | 视口类定义 |
| `ViewportService/constants.ts` | 常量 | 视口常量 |
| `SegmentationService/SegmentationService.ts` | 服务 | 分割服务，管理分割数据 |
| `SegmentationService/index.ts` | 入口 | 分割服务入口 |
| `ToolGroupService/ToolGroupService.ts` | 服务 | 工具组服务 |
| `CornerstoneCacheService/CornerstoneCacheService.ts` | 服务 | 缓存服务 |
| `SyncGroupService/SyncGroupService.ts` | 服务 | 同步组服务 |
| `ColorbarService/ColorbarService.ts` | 服务 | 色阶服务 |
| `ViewportActionCornersService/ViewportActionCornersService.ts` | 服务 | 视口角标服务 |

##### cornerstone/src/components/ - UI组件

| 文件 | 类型 | 说明 |
|------|------|------|
| `OHIFViewportActionCorners.tsx` | 组件 | 视口角标组件 |
| `MeasurementItems.tsx` | 组件 | 测量项组件 |
| `MeasurementTableNested.tsx` | 组件 | 嵌套测量表 |
| `DicomUpload/DicomUpload.tsx` | 组件 | DICOM上传组件 |
| `DicomUpload/DicomUploadProgress.tsx` | 组件 | 上传进度组件 |
| `CinePlayer/CinePlayer.tsx` | 组件 | 播放控制器 |
| `WindowLevelActionMenu/WindowLevelActionMenu.tsx` | 组件 | 窗宽窗位菜单 |
| `ViewportSegmentationMenu/ViewportSegmentationMenu.tsx` | 组件 | 分割菜单 |

##### cornerstone/src/tools/ - 自定义工具

| 文件 | 类型 | 说明 |
|------|------|------|
| `CalibrationLineTool.ts` | 工具 | 校准线工具 |
| `ImageOverlayViewerTool.tsx` | 工具 | 图像叠加工具 |

##### cornerstone/src/panels/ - 面板组件

| 文件 | 类型 | 说明 |
|------|------|------|
| `PanelSegmentation.tsx` | 面板 | 分割面板 |
| `PanelMeasurement.tsx` | 面板 | 测量面板 |

##### cornerstone/src/utils/ - 工具函数

| 文件 | 类型 | 说明 |
|------|------|------|
| `segmentUtils.ts` | 工具 | 分割工具函数 |
| `hydrationUtils.ts` | 工具 | 水合工具函数 |
| `transitions.ts` | 工具 | 过渡动画 |
| `dicomLoaderService.ts` | 服务 | DICOM加载服务 |
| `getActiveViewportEnabledElement.ts` | 工具 | 获取活动视口 |

##### cornerstone/src/hooks/ - React Hooks

| 文件 | 类型 | 说明 |
|------|------|------|
| `useSegmentations.ts` | Hook | 分割状态Hook |
| `useMeasurements.ts` | Hook | 测量状态Hook |
| `useActiveViewportSegmentationRepresentations.ts` | Hook | 活动视口分割Hook |

---

### cornerstone-dicom-seg - DICOM SEG扩展

**目录**: `Viewers/extensions/cornerstone-dicom-seg/`

| 文件 | 类型 | 说明 |
|------|------|------|
| `index.tsx` | 入口 | 扩展入口 |
| `src/index.tsx` | 入口 | 源码入口 |
| `src/commandsModule.ts` | 命令 | SEG相关命令 |
| `src/getSopClassHandlerModule.ts` | 处理器 | SEG SOP类处理 |
| `src/getToolbarModule.ts` | 工具栏 | SEG工具栏 |
| `src/viewports/OHIFCornerstoneSEGViewport.tsx` | 视口 | SEG视口组件 |
| `src/utils/initSEGToolGroup.ts` | 工具 | SEG工具组初始化 |
| `src/utils/promptHydrateSEG.ts` | 工具 | SEG水合提示 |

---

### cornerstone-dicom-rt - DICOM RT扩展

**目录**: `Viewers/extensions/cornerstone-dicom-rt/`

| 文件 | 类型 | 说明 |
|------|------|------|
| `index.tsx` | 入口 | 扩展入口 |
| `src/getCommandsModule.ts` | 命令 | RT相关命令 |
| `src/getSopClassHandlerModule.ts` | 处理器 | RT SOP类处理 |
| `src/viewports/OHIFCornerstoneRTViewport.tsx` | 视口 | RT视口组件 |

---

### cornerstone-dicom-sr - DICOM SR扩展

**目录**: `Viewers/extensions/cornerstone-dicom-sr/`

| 文件 | 类型 | 说明 |
|------|------|------|
| `index.tsx` | 入口 | 扩展入口 |
| `src/getSopClassHandlerModule.ts` | 处理器 | SR SOP类处理 |
| `src/components/OHIFCornerstoneSRViewport.tsx` | 组件 | SR视口组件 |
| `src/tools/DICOMSRDisplayTool.ts` | 工具 | SR显示工具 |

---

### default - 默认扩展

**目录**: `Viewers/extensions/default/`

| 文件 | 类型 | 说明 |
|------|------|------|
| `index.ts` | 入口 | 扩展入口 |
| `src/index.ts` | 入口 | 源码入口 |
| `src/commandsModule.ts` | 命令 | 默认命令集 |
| `src/getPanelModule.tsx` | 面板 | 面板模块 |
| `src/getToolbarModule.tsx` | 工具栏 | 工具栏模块 |
| `src/ViewerLayout/index.tsx` | 布局 | 查看器布局 |
| `src/Toolbar/Toolbar.tsx` | 组件 | 工具栏组件 |
| `src/Panels/StudyBrowser/PanelStudyBrowser.tsx` | 面板 | 研究浏览器面板 |
| `src/DicomWebDataSource/index.ts` | 数据源 | DICOM Web数据源 |
| `src/customizations/*.ts` | 配置 | 各类自定义配置 |

---

### measurement-tracking - 测量追踪扩展

**目录**: `Viewers/extensions/measurement-tracking/`

| 文件 | 类型 | 说明 |
|------|------|------|
| `index.tsx` | 入口 | 扩展入口 |
| `src/contexts/TrackedMeasurementsContext/TrackedMeasurementsContext.tsx` | 上下文 | 测量追踪上下文 |
| `src/panels/PanelMeasurementTableTracking.tsx` | 面板 | 测量追踪面板 |

---

### monai-label - MONAI Label扩展 (AI功能)

**目录**: `Viewers/extensions/monai-label/`

| 文件 | 类型 | 说明 |
|------|------|------|
| `src/index.ts` | 入口 | MONAI Label扩展入口 |

---

## 📁 Viewers/modes/ - 应用模式

### longitudinal - 纵向查看模式 (主模式)

**目录**: `Viewers/modes/longitudinal/`

| 文件 | 类型 | 说明 |
|------|------|------|
| `package.json` | 配置 | 模式包配置 |
| `src/index.ts` | 入口 | 模式入口 |
| `src/id.ts` | 常量 | 模式ID |
| `src/toolbarButtons.ts` | 配置 | **AI工具栏按钮定义**，包含所有AI工具配置 |
| `src/initToolGroups.ts` | 初始化 | 工具组初始化 |

**toolbarButtons.ts 关键内容**:
- `aiToolBoxSection`: AI工具箱按钮组 (Probe2, PlanarFreehandROI2等)
- `aiToolBoxContainer`: AI工具箱容器
- `textPromptSegmentationSection`: 文本提示分割工具
- `testMedgemmaSection`: MedGemma报告生成工具

---

### segmentation - 分割模式

**目录**: `Viewers/modes/segmentation/`

| 文件 | 类型 | 说明 |
|------|------|------|
| `src/index.tsx` | 入口 | 分割模式入口 |
| `src/toolbarButtons.ts` | 配置 | 分割工具栏 |
| `src/initToolGroups.ts` | 初始化 | 分割工具组 |

---

### 其他模式

| 模式 | 路径 | 说明 |
|------|------|------|
| basic-dev-mode | `modes/basic-dev-mode/` | 基础开发模式 |
| basic-test-mode | `modes/basic-test-mode/` | 基础测试模式 |
| microscopy | `modes/microscopy/` | 显微镜模式 |
| preclinical-4d | `modes/preclinical-4d/` | 临床前4D模式 |
| tmtv | `modes/tmtv/` | TMTV模式 |

---

## 📁 Viewers/platform/ - 平台核心

### app - 应用主体

**目录**: `Viewers/platform/app/`

| 文件 | 类型 | 说明 |
|------|------|------|
| `src/index.tsx` | 入口 | 应用入口 |
| `src/App.tsx` | 组件 | 根组件 |
| `src/routes/` | 路由 | 路由配置 |
| `public/` | 静态 | 静态资源 |
| `.recipes/Nginx-Orthanc/` | 配置 | Nginx+Orthanc部署配置 |
| `.recipes/Nginx-Orthanc/dockerfile` | 配置 | 前端Dockerfile |
| `.recipes/Nginx-Orthanc/config/nginx.conf` | 配置 | Nginx配置 |
| `.recipes/Nginx-Orthanc/config/orthanc.json` | 配置 | Orthanc配置 |

---

### core - 核心库

**目录**: `Viewers/platform/core/`

| 文件 | 类型 | 说明 |
|------|------|------|
| `src/index.ts` | 入口 | 核心库入口 |
| `src/services/` | 服务 | 核心服务定义 |
| `src/types/` | 类型 | TypeScript类型定义 |
| `src/utils/` | 工具 | 工具函数 |

**核心服务**:
- `CustomizationService.ts` - 自定义服务
- `ExtensionManager.ts` - 扩展管理器
- `MeasurementService.ts` - 测量服务
- `ServicesManager.ts` - 服务管理器
- `CommandManager.ts` - 命令管理器

---

### ui / ui-next - UI组件库

**目录**: `Viewers/platform/ui/` 和 `platform/ui-next/`

| 文件 | 类型 | 说明 |
|------|------|------|
| `src/index.ts` | 入口 | UI库入口 |
| `src/components/` | 组件 | React组件库 |
| `src/hooks/` | Hooks | 自定义Hooks |
| `src/assets/` | 资源 | 图标、样式等 |

**关键组件**:
- `SegmentationTable/` - 分割表组件
- `Button/` - 按钮组件
- `Icon/` - 图标组件

---

## 📁 monai-label/ - 后端代码目录

### monai-label 根目录文件

| 文件 | 类型 | 说明 |
|------|------|------|
| `Dockerfile` | 配置 | **后端Docker镜像构建配置** |
| `requirements.txt` | 配置 | Python依赖列表 |
| `requirements-dev.txt` | 配置 | 开发依赖 |
| `setup.cfg` | 配置 | Python包配置 |
| `pyproject.toml` | 配置 | 现代Python项目配置 |
| `runtests.sh` | 脚本 | 测试运行脚本 |
| `CHANGELOG.md` | 文档 | 变更日志 |
| `README.md` | 文档 | MONAI Label说明 |
| `LICENSE` | 文档 | 许可证 |

---

### monailabel/ - Python包主目录

#### 根目录文件

| 文件 | 类型 | 说明 |
|------|------|------|
| `__init__.py` | 入口 | 包入口，定义版本 |
| `_version.py` | 常量 | 版本号定义 |
| `app.py` | 应用 | **FastAPI应用主入口**，定义路由和中间件 |
| `main.py` | CLI | **命令行入口**，服务器启动逻辑 |
| `config.py` | 配置 | **全局配置设置**，使用pydantic_settings |

#### app.py 详解

```python
# 主要功能:
# 1. FastAPI应用创建和配置
# 2. CORS中间件配置
# 3. 路由注册 (infer, datastore, info等)
# 4. 静态文件挂载
# 5. Swagger UI配置
```

**注册的路由**:
- `info.router` - 信息API
- `model.router` - 模型API
- `infer.router` - **推理API**
- `datastore.router` - 数据存储API
- `scoring.router` - 评分API
- `ohif.router` - OHIF静态文件
- `session.router` - 会话管理API

#### main.py 详解

```python
# 主要功能:
# 1. 命令行参数解析
# 2. 服务器启动配置
# 3. Uvicorn服务器启动
# 4. 环境变量设置
```

**子命令**:
- `start_server` - 启动服务器
- `apps` - 列出/下载示例应用
- `datasets` - 列出/下载数据集

---

### monailabel/endpoints/ - API端点

| 文件 | 类型 | 说明 |
|------|------|------|
| `__init__.py` | 入口 | 端点包入口 |
| `infer.py` | **核心** | **推理API端点**，处理分割请求 |
| `datastore.py` | 端点 | 数据存储管理API |
| `info.py` | 端点 | 应用信息API |
| `model.py` | 端点 | 模型管理API |
| `session.py` | 端点 | 会话管理API |
| `scoring.py` | 端点 | 评分API |
| `train.py` | 端点 | 训练API |
| `logs.py` | 端点 | 日志API |
| `proxy.py` | 端点 | 代理API |
| `ohif.py` | 端点 | OHIF静态文件服务 |
| `wsi_infer.py` | 端点 | WSI推理API |
| `batch_infer.py` | 端点 | 批量推理API |
| `activelearning.py` | 端点 | 主动学习API |

#### infer.py 详解

**核心函数**:
- `run_inference()` - 处理推理请求的主函数
- `send_response()` - 格式化响应
- `read_seg_file()` - 读取DICOM-SEG文件
- `save_combined_segmentation()` - 保存合并的分割

**API端点**:
```python
@router.post("/{model}")
async def api_run_inference(...)
# 处理POST /monai/infer/{model} 请求
```

---

### monailabel/interfaces/ - 接口定义

| 文件 | 类型 | 说明 |
|------|------|------|
| `__init__.py` | 入口 | 接口包入口 |
| `app.py` | **核心** | **MONAILabelApp基类**，应用框架核心 |
| `datastore.py` | 接口 | 数据存储接口定义 |
| `config.py` | 配置 | 配置接口 |
| `exception.py` | 异常 | 自定义异常类 |

#### app.py 详解 (MONAILabelApp)

**核心方法**:
- `__init__()` - 初始化应用，设置数据存储和任务
- `init_datastore()` - 初始化数据存储
- `init_infers()` - 初始化推理任务
- `infer()` - 执行推理
- `train()` - 执行训练
- `scoring()` - 执行评分
- `info()` - 获取应用信息

**属性**:
- `_datastore` - 数据存储实例
- `_infers` - 推理任务字典
- `_trainers` - 训练任务字典
- `_sessions` - 会话管理器

---

### monailabel/interfaces/tasks/ - 任务接口

| 文件 | 类型 | 说明 |
|------|------|------|
| `infer.py` | 接口 | 推理任务接口 |
| `infer_v2.py` | 接口 | 推理任务接口v2 |
| `train.py` | 接口 | 训练任务接口 |
| `scoring.py` | 接口 | 评分任务接口 |
| `strategy.py` | 接口 | 策略接口 |
| `batch_infer.py` | 接口 | 批量推理接口 |

---

### monailabel/tasks/infer/ - 推理任务实现

| 文件 | 类型 | 说明 |
|------|------|------|
| `__init__.py` | 入口 | 推理任务包入口 |
| `basic_infer.py` | **核心** | **基础推理任务实现**，包含所有AI模型集成 |
| `bundle.py` | 任务 | Bundle推理任务 |

#### basic_infer.py 详解

**这是整个项目最核心的文件之一**

**全局初始化**:
```python
# nnInteractive初始化
session = nnInteractiveInferenceSession(...)

# SAM2初始化
predictor_sam2 = build_sam2_video_predictor(...)

# MedSAM2初始化
predictor_med = build_sam2_video_predictor_npz(...)

# SAM3初始化 (条件)
predictor_sam3 = build_sam3_video_model(...)

# VoxTell初始化
vox_predictor = VoxTellPredictor(...)

# MedGemma初始化
gem_processor = AutoProcessor.from_pretrained(...)
gem_model = AutoModelForImageTextToText.from_pretrained(...)
```

**BasicInferTask类**:
- `__init__()` - 初始化推理任务
- `__call__()` - **主推理入口**，根据参数选择不同模型分支
- `pre_transforms()` - 预处理变换
- `post_transforms()` - 后处理变换
- `inferer()` - 推理器配置
- `writer()` - 结果写入

**模型分支逻辑**:
1. `nnInter == True` → nnInteractive分支 (517-974行)
2. `nnInter == False` → SAM2/3分支 (976-1234行)
3. `texts`非空 → VoxTell分支 (661-693行)
4. `nnInter == "medGemma"` → MedGemma分支 (542-659行)

---

### monailabel/datastore/ - 数据存储实现

| 文件 | 类型 | 说明 |
|------|------|------|
| `__init__.py` | 入口 | 数据存储包入口 |
| `local.py` | 存储 | LocalDatastore本地存储实现 |
| `dicom.py` | 存储 | DICOMWebDatastore DICOM Web存储 |
| `dsa.py` | 存储 | DSADatastore数字切片存储 |
| `xnat.py` | 存储 | XNATDatastore XNAT存储 |
| `cvat.py` | 存储 | CVATDatastore CVAT存储 |

#### local.py 详解

**LocalDatastore类**:
- 管理本地文件系统中的图像和标签
- 支持NIfTI、NRRD等格式
- 实现数据集的增删改查

#### dicom.py 详解

**DICOMWebDatastore类**:
- 通过DICOM Web协议访问PACS
- 支持DICOM-SEG格式分割
- 自动格式转换

---

### monailabel/utils/ - 工具函数

#### utils/others/ - 杂项工具

| 文件 | 类型 | 说明 |
|------|------|------|
| `generic.py` | 工具 | 通用工具函数 |
| `helper.py` | **核心** | **关键辅助函数**，涂鸦处理、套索填充 |
| `medgemma.py` | 工具 | **MedGemma工具**，窗宽窗位、图像编码 |
| `pathology.py` | 工具 | 病理图像工具 |
| `detection.py` | 工具 | 检测工具 |
| `label_colors.py` | 工具 | 标签颜色 |
| `class_utils.py` | 工具 | 类工具 |
| `planner.py` | 工具 | 规划器 |
| `stream.py` | 工具 | 流处理工具 |

#### helper.py 详解

**核心函数**:
- `get_scanline_filled_points_3d()` - 扫描线填充算法
- `clean_and_densify_polyline()` - 路径清理和致密化
- `spherical_kernel()` - 球形结构元素
- `calculate_dice()` - Dice系数计算
- `timeout_context()` - 超时上下文管理器

#### medgemma.py 详解

**核心函数**:
- `window()` - CT窗宽窗位处理
- `window_mri()` - MRI窗宽窗位处理
- `_encode()` - 图像Base64编码

---

## 📁 sam2/ - SAM2模型目录

这是Meta的SAM2官方代码，作为子项目存在。

| 文件 | 类型 | 说明 |
|------|------|------|
| `README.md` | 文档 | SAM2说明 |
| `setup.py` | 配置 | SAM2包安装配置 |
| `pyproject.toml` | 配置 | 现代Python项目配置 |
| `sam2/build_sam.py` | 构建 | 模型构建函数 |
| `sam2/sam2_video_predictor.py` | 模型 | 视频预测器 |

---

## 📁 sam3/ - SAM3模型目录

这是Meta的SAM3官方代码。

| 文件 | 类型 | 说明 |
|------|------|------|
| `README.md` | 文档 | SAM3说明 |
| `pyproject.toml` | 配置 | 项目配置 |
| `sam3/model_builder.py` | 构建 | 模型构建 |
| `sam3/sam/` | 目录 | SAM核心代码 |

---

## 📁 docs/ - 项目文档图片

**目录**: `docs/`

| 文件/目录 | 说明 |
|----------|------|
| `images/` | 文档图片资源 |
| `pdfs/` | PDF文档 |

---

## 📁 sample-data/ - 示例数据

**目录**: `sample-data/`

包含示例DICOM数据用于测试。

| 文件/目录 | 说明 |
|----------|------|
| `2.000000-PRE LIVER-76970/` | 肝脏CT示例数据 |
| `2.000000-PRE LIVER-76970.zip` | 压缩包 |

---

## 🔗 关键文件依赖关系

### 前端数据流

```
Viewers/
├── modes/longitudinal/src/toolbarButtons.ts  (定义AI工具)
│         ↓
├── extensions/cornerstone/src/commandsModule.ts  (命令映射)
│         ↓
├── extensions/cornerstone/src/services/SegmentationService.ts  (分割服务)
│         ↓
└── API调用 → monai-label/monailabel/endpoints/infer.py
```

### 后端数据流

```
monai-label/
├── monailabel/endpoints/infer.py  (API入口)
│         ↓
├── monailabel/interfaces/app.py  (MONAILabelApp.infer())
│         ↓
├── monailabel/tasks/infer/basic_infer.py  (BasicInferTask)
│         ├── nnInteractive Session
│         ├── SAM2 Predictor
│         ├── MedSAM2 Predictor
│         ├── SAM3 Predictor (可选)
│         ├── VoxTell Predictor
│         └── MedGemma Model
│         ↓
└── 返回 DICOM-SEG → 前端
```

---

*本手册涵盖项目的主要文件，更多细节请参考具体源码*
