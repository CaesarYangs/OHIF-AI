# OHIF-AI 项目架构详解

## 项目概述

**OHIF-AI** 是一个集成AI能力的医学影像查看与分析系统，基于OHIF (Open Health Imaging Foundation) Viewer构建。该系统将多种先进的医学影像AI模型整合到浏览器端，支持实时交互式分割和报告生成。

### 核心功能
1. **交互式AI分割** - 支持点、涂鸦、套索、边界框等多种视觉提示方式
2. **多模型支持** - nnInteractive、SAM2、MedSAM2、SAM3、VoxTell
3. **文本提示分割** - 通过自然语言描述进行分割 (VoxTell)
4. **报告生成** - 基于MedGemma 1.5的放射学风格报告生成
5. **3D传播** - 单次提示即可在整个3D体积上传播分割

---

## 系统架构

### 整体架构图

```
┌─────────────────────────────────────────────────────────────────┐
│                        前端 (Frontend)                           │
│                     OHIF Viewer (React)                          │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────────┐  │
│  │ Longitudinal│  │Segmentation │  │    Cornerstone 3D       │  │
│  │    Mode     │  │    Mode     │  │  (医学影像渲染引擎)      │  │
│  └─────────────┘  └─────────────┘  └─────────────────────────┘  │
│         │                │                    │                 │
│         └────────────────┴────────────────────┘                 │
│                          │                                      │
│                   AI工具箱交互界面                                │
│     (点/涂鸦/套索/边界框工具 + 模型选择 + 实时推理)               │
└──────────────────────────┬──────────────────────────────────────┘
                           │ HTTP/Web API
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│                      后端 (Backend)                              │
│                  MONAI Label (FastAPI)                           │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────────┐  │
│  │   推理API    │  │  数据存储    │  │      模型管理           │  │
│  │  (infer.py) │  │ (Datastore) │  │  (basic_infer.py)       │  │
│  └─────────────┘  └─────────────┘  └─────────────────────────┘  │
│                          │                                      │
│         ┌────────────────┼────────────────┐                     │
│         ▼                ▼                ▼                     │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────────┐  │
│  │  nnInteractive│  │  SAM2/3    │  │      VoxTell           │  │
│  │  (3D分割)   │  │  (视频分割) │  │   (文本提示分割)        │  │
│  └─────────────┘  └─────────────┘  └─────────────────────────┘  │
│                          │                                      │
│                   ┌──────┴──────┐                               │
│                   ▼             ▼                               │
│            ┌──────────┐   ┌──────────┐                         │
│            │ MedGemma │   │  MedSAM2 │                         │
│            │(报告生成) │   │(医学SAM2)│                         │
│            └──────────┘   └──────────┘                         │
└─────────────────────────────────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│                     数据层 (Data Layer)                          │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────────┐  │
│  │   Orthanc   │  │   DICOM     │  │   本地文件存储           │  │
│  │  (PACS服务器)│  │   Web       │  │  (NIfTI, NRRD等)        │  │
│  └─────────────┘  └─────────────┘  └─────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 前端架构 (Frontend)

### 技术栈
- **框架**: React 18 + TypeScript
- **构建工具**: RSBuild / Webpack
- **状态管理**: Zustand
- **UI组件**: Tailwind CSS + Radix UI
- **医学影像渲染**: Cornerstone3D

### 目录结构

```
Viewers/
├── extensions/                 # 扩展模块
│   ├── cornerstone/           # 核心影像渲染扩展
│   │   ├── src/
│   │   │   ├── services/      # 服务层 (ViewportService, SegmentationService等)
│   │   │   ├── components/    # UI组件
│   │   │   ├── tools/         # 自定义工具
│   │   │   └── utils/         # 工具函数
│   │   └── index.tsx          # 扩展入口
│   ├── cornerstone-dicom-seg/ # DICOM SEG支持
│   ├── cornerstone-dicom-rt/  # DICOM RT支持
│   ├── default/               # 默认扩展 (布局、工具栏等)
│   ├── measurement-tracking/  # 测量追踪
│   └── ...
├── modes/                      # 应用模式
│   ├── longitudinal/          # 纵向查看模式 (含AI工具)
│   ├── segmentation/          # 分割模式
│   └── ...
├── platform/                   # 平台核心
│   ├── core/                  # 核心服务
│   ├── ui/                    # UI组件库
│   └── app/                   # 应用配置
└── rsbuild.config.ts          # 构建配置
```

### 关键模块详解

#### 1. Cornerstone Extension (`extensions/cornerstone`)

这是前端的核心扩展，负责医学影像的渲染和交互。

**核心服务**:

| 服务名称 | 功能 |
|---------|------|
| `CornerstoneViewportService` | 视口管理，负责渲染医学影像 |
| `SegmentationService` | 分割数据管理，处理分割的加载、显示和保存 |
| `ToolGroupService` | 工具组管理，配置不同的交互工具 |
| `SegmentationService` | 分割服务，管理分割状态 |

**关键文件**:
- `src/index.tsx` - 扩展入口，注册所有服务
- `src/services/SegmentationService/SegmentationService.ts` - 分割服务实现
- `src/commandsModule.ts` - 命令模块，定义UI交互命令

#### 2. Longitudinal Mode (`modes/longitudinal`)

这是AI功能的主要入口模式，包含AI工具箱。

**关键配置**:
```typescript
// toolbarButtons.ts 中定义的AI工具
{
  'aiToolBoxSection': [
    'Probe2',           // 点提示工具
    'PlanarFreehandROI2', // 涂鸦工具
    'PlanarFreehandROI3', // 套索工具
    'RectangleROI2',    // 边界框工具
    'nninter',          // nnInteractive模型选择
  ],
  'textPromptSegmentationSection': [
    'textPromptSegmentation'  // VoxTell文本提示
  ],
  'testMedgemmaSection': [
    'testMedgemma'      // MedGemma报告生成
  ]
}
```

---

## 后端架构 (Backend)

### 技术栈
- **框架**: FastAPI (Python)
- **深度学习**: PyTorch, MONAI
- **医学影像处理**: SimpleITK, pydicom, nibabel
- **模型推理**: nnInteractive, SAM2, transformers

### 目录结构

```
monai-label/
├── monailabel/                 # 核心代码
│   ├── app.py                 # FastAPI应用入口
│   ├── main.py                # 命令行入口
│   ├── config.py              # 配置管理
│   ├── endpoints/             # API端点
│   │   ├── infer.py           # 推理API
│   │   ├── datastore.py       # 数据存储API
│   │   ├── ohif.py            # OHIF静态文件服务
│   │   └── ...
│   ├── interfaces/            # 接口定义
│   │   ├── app.py             # MONAILabelApp基类
│   │   ├── tasks/             # 任务接口
│   │   │   ├── infer.py       # 推理任务接口
│   │   │   ├── infer_v2.py    # 推理任务接口v2
│   │   │   └── ...
│   │   └── datastore.py       # 数据存储接口
│   ├── tasks/                 # 任务实现
│   │   └── infer/
│   │       └── basic_infer.py # 核心推理实现
│   ├── datastore/             # 数据存储实现
│   │   ├── local.py           # 本地存储
│   │   ├── dicom.py           # DICOM Web存储
│   │   └── ...
│   └── utils/                 # 工具函数
│       └── others/
│           ├── medgemma.py    # MedGemma工具
│           └── helper.py      # 辅助函数
├── sample-apps/               # 示例应用
│   └── radiology/             # 放射学应用
├── Dockerfile                 # 容器构建
└── requirements.txt           # Python依赖
```

### 核心模块详解

#### 1. FastAPI应用 (`app.py`)

```python
# 主要路由配置
app.include_router(info.router, prefix=settings.MONAI_LABEL_API_STR)
app.include_router(model.router, prefix=settings.MONAI_LABEL_API_STR)
app.include_router(infer.router, prefix=settings.MONAI_LABEL_API_STR)  # 推理API
app.include_router(datastore.router, prefix=settings.MONAI_LABEL_API_STR)
```

#### 2. 推理端点 (`endpoints/infer.py`)

处理来自前端的推理请求，核心流程：

```
1. 接收请求 (model, image, params)
2. 解析参数 (points, boxes, texts等)
3. 调用BasicInferTask
4. 返回分割结果 (NIfTI格式或DICOM-SEG)
```

**关键API**:
```python
@router.post("/{model}")
async def api_run_inference(
    model: str,           # 模型名称
    image: str,           # 图像ID
    params: str,          # JSON参数 (包含提示信息)
    output: ResultType,   # 输出格式
    ...
)
```

#### 3. 核心推理实现 (`tasks/infer/basic_infer.py`)

这是整个系统的核心，实现了多种AI模型的推理逻辑。

**模型初始化**:
```python
# nnInteractive
session = nnInteractiveInferenceSession(device=torch.device("cuda:0"), ...)

# SAM2
predictor_sam2 = build_sam2_video_predictor(model_cfg, sam2_checkpoint)

# MedSAM2  
predictor_med = build_sam2_video_predictor_npz(medsam2_model_cfg, medsam2_checkpoint)

# SAM3 (可选)
if os.path.exists(sam3_checkpoint):
    sam3_model = build_sam3_video_model(checkpoint_path=sam3_checkpoint)

# VoxTell
vox_predictor = VoxTellPredictor(model_dir=vox_model_path, device=torch.device("cuda:0"))

# MedGemma
gem_processor = transformers.AutoProcessor.from_pretrained(gem_model_id, ...)
gem_model = transformers.AutoModelForImageTextToText.from_pretrained(gem_model_id, ...)
```

**推理流程**:

```
┌─────────────────────────────────────────────────────────┐
│                     __call__ 方法                        │
└─────────────────────────────────────────────────────────┘
                           │
           ┌───────────────┼───────────────┐
           ▼               ▼               ▼
    ┌──────────┐    ┌──────────┐    ┌──────────┐
    │ nnInter  │    │  SAM2/3  │    │ VoxTell  │
    │  分支    │    │   分支   │    │  分支    │
    └──────────┘    └──────────┘    └──────────┘
```

**nnInteractive分支**:
- 支持点、涂鸦、套索、边界框提示
- 3D交互式分割
- 自动处理坐标变换 (处理DICOM方向)

**SAM2/3分支**:
- 基于视频跟踪的分割
- 支持点提示和边界框提示
- 3D体积传播

**VoxTell分支**:
- 文本提示分割
- 使用自然语言描述目标结构

**MedGemma分支**:
- 放射学报告生成
- 支持自定义指令和查询
- 可指定切片范围

---

## 数据流

### 分割推理数据流

```
┌─────────┐     ┌─────────────┐     ┌─────────────┐     ┌─────────┐
│  用户   │────▶│   OHIF      │────▶│  MONAI      │────▶│  AI模型 │
│ 交互    │     │   Viewer    │     │  Label      │     │  推理   │
└─────────┘     └─────────────┘     └─────────────┘     └─────────┘
     │                │                   │                  │
     │                │                   │                  │
     │  1. 绘制提示    │                   │                  │
     │  (点/框/涂鸦)   │                   │                  │
     │────────────────▶│                   │                  │
     │                │                   │                  │
     │                │  2. HTTP POST     │                  │
     │                │  /infer/{model}   │                  │
     │                │  + 提示数据        │                  │
     │                │──────────────────▶│                  │
     │                │                   │                  │
     │                │                   │  3. 调用模型      │
     │                │                   │  推理            │
     │                │                   │─────────────────▶│
     │                │                   │                  │
     │                │                   │  4. 返回         │
     │                │                   │  分割结果        │
     │                │                   │◀─────────────────│
     │                │                   │                  │
     │                │  5. 返回         │                  │
     │                │  DICOM-SEG       │                  │
     │                │◀──────────────────│                  │
     │                │                   │                  │
     │  6. 显示        │                  │                  │
     │  分割结果       │                  │                  │
     │◀───────────────│                   │                  │
```

### 请求参数格式

```json
{
  "model": "segmentation",
  "image": "/path/to/dicom_series",
  "nninter": true,
  "pos_points": [[x, y, z], ...],
  "neg_points": [[x, y, z], ...],
  "pos_boxes": [[[x1,y1,z1], [x2,y2,z2]], ...],
  "pos_scribbles": [[x, y, z], ...],
  "texts": ["liver"],
  "instruction": "You are a radiology assistant",
  "startSlice": 10,
  "endSlice": 50
}
```

---

## Docker部署架构

### 服务组成

```yaml
# docker-compose.yml
services:
  ohif_viewer:     # Nginx + OHIF前端
  orthanc:         # DICOM PACS服务器
  monai_sam2:      # MONAI Label后端 (GPU)
```

### 容器配置

**MONAI Label容器**:
- 基础镜像: `nvidia/cuda:12.1.1-devel-ubuntu22.04`
- GPU支持: NVIDIA Container Toolkit
- 共享内存: `shm_size: '10gb'`
- 端口: `8002`

---

## 模型架构

### nnInteractive
- **类型**: 3D交互式分割模型
- **输入**: 3D医学影像 + 交互提示
- **特点**: 
  - 支持点、涂鸦、套索、边界框
  - AutoZoom自动补丁
  - 3D体积分割

### SAM2 (Segment Anything Model 2)
- **类型**: 视频分割模型
- **输入**: 图像序列 + 点/框提示
- **特点**:
  - 时序一致性
  - 3D体积传播
  - 支持正/负提示

### MedSAM2
- **类型**: 医学适配版SAM2
- **训练数据**: 医学影像数据集
- **特点**: 针对医学影像优化

### SAM3 (可选)
- **类型**: 概念感知分割
- **特点**: 更强大的语义理解

### VoxTell
- **类型**: 文本提示分割
- **输入**: 3D影像 + 自然语言描述
- **输出**: 3D分割掩码

### MedGemma 1.5 4B
- **类型**: 多模态医学大语言模型
- **输入**: CT/MRI切片序列 + 指令/查询
- **输出**: 放射学风格报告
- **VRAM需求**: ~35GB

---

## 关键技术点

### 1. DICOM方向处理

系统需要处理DICOM影像的方向和坐标变换：

```python
# 检查InstanceNumber方向
if instanceNumber > instanceNumber2:
    # 需要翻转Z轴
    point[2] = img_np.shape[1] - 1 - point[2]

# 方向转换
orig_orient = sitk.DICOMOrientImageFilter_GetOrientationFromDirectionCosines(
    img.GetDirection()
)
img_ras = sitk.DICOMOrient(img, "RAS")  # 转换为RAS坐标系
```

### 2. 窗宽窗位处理

```python
# CT影像使用固定窗宽窗位
def window(ct_slice, window_center=40, window_width=400):
    min_val = window_center - window_width // 2
    max_val = window_center + window_width // 2
    windowed = np.clip(ct_slice, min_val, max_val)
    windowed = (windowed - min_val) / window_width * 255
    return windowed.astype(np.uint8)

# MRI影像使用DICOM元数据
if contrast_window and contrast_center:
    windowed_slice = window_mri(mr_slice, 
                                contrast_center - contrast_window/2, 
                                contrast_center + contrast_window/2)
```

### 3. 会话管理

```python
# nnInteractive会话状态
_session_image = {
    "seriesInstanceUID": None,
}

# 已使用的交互记录 (用于避免重复计算)
_session_used_interactions = {
    "pos_points": set(),
    "neg_points": set(),
    "pos_boxes": set(),
    ...
}
```

---

## 扩展开发指南

### 添加新的AI模型

1. **后端实现** (`basic_infer.py`):
```python
# 1. 初始化模型
my_model = load_my_model(checkpoint_path)

# 2. 在__call__方法中添加分支
if data['model_type'] == 'my_model':
    result = my_model.predict(image, prompts)
    return result, metadata
```

2. **前端配置** (`toolbarButtons.ts`):
```typescript
{
  id: 'myModel',
  label: 'My Model',
  icon: 'icon-ai-model',
  commands: [
    {
      commandName: 'setAIModel',
      commandOptions: { model: 'my_model' }
    }
  ]
}
```

---

## 性能优化

### 1. 并发处理
- 使用ThreadPoolExecutor处理多个WSI任务
- 支持多GPU推理

### 2. 缓存机制
- 图像预处理缓存
- 模型权重缓存

### 3. 内存管理
- 及时释放GPU缓存: `torch.cuda.empty_cache()`
- 使用bfloat16降低显存占用

---

## 安全与权限

- 基于RBAC的权限控制
- DICOM Web认证支持
- API Token管理

---

## 总结

OHIF-AI是一个复杂的多组件系统，整合了现代Web前端技术和深度学习后端：

1. **前端**: 基于React和Cornerstone3D的现代医学影像查看器
2. **后端**: 基于FastAPI的高性能推理服务
3. **AI模型**: 集成多种SOTA医学影像分割和报告生成模型
4. **部署**: Docker容器化，支持GPU加速

系统设计上注重模块化和可扩展性，方便集成新的AI模型和功能。
