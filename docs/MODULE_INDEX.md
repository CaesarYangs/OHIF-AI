# OHIF-AI 功能模块索引

本文档提供项目各功能模块的快速索引，帮助开发者快速定位代码。

---

## 📁 目录结构总览

```
OHIF-AI/
├── Viewers/                    # 前端代码 (OHIF Viewer)
│   ├── extensions/            # 扩展模块
│   ├── modes/                 # 应用模式
│   ├── platform/              # 平台核心
│   └── ...
├── monai-label/               # 后端代码 (MONAI Label)
│   ├── monailabel/           # 核心Python包
│   ├── sample-apps/          # 示例应用
│   └── ...
├── sam2/                      # SAM2模型代码
├── sam3/                      # SAM3模型代码
├── docker-compose.yml         # Docker部署配置
└── start.sh                   # 启动脚本
```

---

## 🎨 前端模块 (Viewers)

### 核心扩展

| 模块 | 路径 | 主要功能 |
|-----|------|---------|
| **Cornerstone Core** | `extensions/cornerstone/` | 医学影像渲染核心 |
| | `src/services/CornerstoneViewportService/` | 视口管理 |
| | `src/services/SegmentationService/` | 分割服务 |
| | `src/services/ToolGroupService/` | 工具组管理 |
| | `src/commandsModule.ts` | 命令定义 |
| **Default Extension** | `extensions/default/` | 默认UI组件 |
| | `src/Toolbar/` | 工具栏组件 |
| | `src/ViewerLayout/` | 布局组件 |
| | `src/customizations/` | 自定义配置 |

### 应用模式

| 模式 | 路径 | 功能描述 |
|-----|------|---------|
| **Longitudinal** | `modes/longitudinal/` | 主要AI交互模式 |
| | `src/index.ts` | 模式配置入口 |
| | `src/toolbarButtons.ts` | AI工具栏按钮定义 |
| | `src/initToolGroups.ts` | 工具组初始化 |
| **Segmentation** | `modes/segmentation/` | 纯分割模式 |
| | `src/index.tsx` | 模式配置 |

### 平台核心

| 模块 | 路径 | 功能 |
|-----|------|------|
| **Core** | `platform/core/` | 核心服务框架 |
| | `src/services/` | 服务基类 |
| | `src/types/` | TypeScript类型定义 |
| **UI** | `platform/ui/` | UI组件库 |
| | `src/components/` | React组件 |
| | `src/hooks/` | 自定义Hooks |

---

## 🔧 后端模块 (MONAI Label)

### 核心应用

| 文件 | 路径 | 功能描述 |
|-----|------|---------|
| **应用入口** | `monailabel/app.py` | FastAPI应用配置 |
| **命令行** | `monailabel/main.py` | CLI入口，服务器启动 |
| **配置** | `monailabel/config.py` | 全局配置设置 |

### API端点

| 端点 | 文件路径 | 功能 |
|-----|---------|------|
| **推理** | `endpoints/infer.py` | 模型推理API |
| | `run_inference()` | 处理推理请求 |
| | `send_response()` | 格式化响应 |
| **数据存储** | `endpoints/datastore.py` | 数据管理API |
| **信息** | `endpoints/info.py` | 应用信息API |
| **模型** | `endpoints/model.py` | 模型管理API |
| **OHIF** | `endpoints/ohif.py` | 静态文件服务 |

### 推理实现

| 文件 | 路径 | 核心功能 |
|-----|------|---------|
| **BasicInferTask** | `tasks/infer/basic_infer.py` | 核心推理类 |
| | `__call__()` | 推理流水线入口 |
| | `nnInteractive分支` | nnInteractive模型推理 |
| | `SAM2/3分支` | SAM系列模型推理 |
| | `VoxTell分支` | 文本提示分割 |
| | `MedGemma分支` | 报告生成 |
| **推理接口** | `interfaces/tasks/infer_v2.py` | 推理任务接口定义 |

### 应用接口

| 文件 | 路径 | 功能 |
|-----|------|------|
| **MONAILabelApp** | `interfaces/app.py` | 应用基类 |
| | `infer()` | 推理方法 |
| | `init_datastore()` | 初始化数据存储 |
| | `init_infers()` | 初始化推理任务 |

### 数据存储

| 文件 | 路径 | 功能 |
|-----|------|------|
| **本地存储** | `datastore/local.py` | LocalDatastore实现 |
| **DICOM Web** | `datastore/dicom.py` | DICOMWebDatastore实现 |
| **DSA** | `datastore/dsa.py` | DSADatastore实现 |
| **XNAT** | `datastore/xnat.py` | XNATDatastore实现 |
| **接口** | `interfaces/datastore.py` | 数据存储接口 |

### 工具函数

| 文件 | 路径 | 功能 |
|-----|------|------|
| **MedGemma工具** | `utils/others/medgemma.py` | 窗宽窗位处理、图像编码 |
| **辅助函数** | `utils/others/helper.py` | 涂鸦处理、套索填充、球形核 |
| **通用工具** | `utils/others/generic.py` | 设备管理、文件操作 |
| **流处理** | `utils/others/stream.py` | 多部分表单流 |

---

## 🤖 AI模型模块

### nnInteractive

| 组件 | 路径 | 说明 |
|-----|------|------|
| **包导入** | 外部包 | `nnInteractive.inference.inference_session` |
| **初始化** | `tasks/infer/basic_infer.py:107-118` | session初始化 |
| **图像设置** | `basic_infer.py:526-538` | `session.set_image()` |
| **点提示** | `basic_infer.py:747-757` | `add_point_interaction()` |
| **涂鸦提示** | `basic_infer.py:855-897` | `add_scribble_interaction()` |
| **套索提示** | `basic_infer.py:807-853` | `add_lasso_interaction()` |
| **边界框** | `basic_infer.py:771-787` | `add_bbox_interaction()` |

### SAM2 / MedSAM2

| 组件 | 路径 | 说明 |
|-----|------|------|
| **包导入** | 外部包 | `sam2.build_sam` |
| **SAM2初始化** | `basic_infer.py:128` | `build_sam2_video_predictor()` |
| **MedSAM2初始化** | `basic_infer.py:139` | `build_sam2_video_predictor_npz()` |
| **状态初始化** | `basic_infer.py:1068-1072` | `predictor.init_state()` |
| **添加提示** | `basic_infer.py:1143-1180` | `add_new_points_or_box()` |
| **视频传播** | `basic_infer.py:1187-1201` | `propagate_in_video()` |

### SAM3

| 组件 | 路径 | 说明 |
|-----|------|------|
| **包导入** | 外部包 | `sam3.model_builder` |
| **条件初始化** | `basic_infer.py:130-137` | 检查checkpoint存在性 |
| **模型构建** | `basic_infer.py:131` | `build_sam3_video_model()` |
| **推理** | `basic_infer.py:1188-1194` | SAM3分支推理 |

### VoxTell

| 组件 | 路径 | 说明 |
|-----|------|------|
| **包导入** | 外部包 | `voxtell.inference.predictor` |
| **初始化** | `basic_infer.py:103-104` | `VoxTellPredictor()` |
| **推理** | `basic_infer.py:661-693` | `predict_single_image()` |

### MedGemma

| 组件 | 路径 | 说明 |
|-----|------|------|
| **包导入** | 外部包 | `transformers` |
| **初始化** | `basic_infer.py:145-154` | `AutoProcessor`, `AutoModelForImageTextToText` |
| **推理** | `basic_infer.py:542-659` | 报告生成分支 |
| **工具函数** | `utils/others/medgemma.py` | `window()`, `window_mri()`, `_encode()` |

---

## 🐳 部署配置

### Docker

| 文件 | 路径 | 功能 |
|-----|------|------|
| **Compose** | `docker-compose.yml` | 多服务编排 |
| **MONAI Dockerfile** | `monai-label/Dockerfile` | 后端镜像构建 |
| **OHIF Dockerfile** | `Viewers/platform/app/.recipes/Nginx-Orthanc/dockerfile` | 前端镜像 |
| **启动脚本** | `start.sh` | 一键启动命令 |

### 配置项

| 配置 | 位置 | 说明 |
|-----|------|------|
| **模型权重** | `Dockerfile:44-52` | 自动下载SAM2/MedSAM2 |
| **HF Token** | `tasks/infer/basic_infer.py:143` | HuggingFace令牌 |
| **端口** | `docker-compose.yml:21-22,71` | 1026(前端), 8002(后端) |
| **GPU** | `docker-compose.yml:46-68` | NVIDIA运行时配置 |

---

## 📊 数据流关键路径

### 推理请求流

```
1. 前端收集提示
   Viewers/modes/longitudinal/src/toolbarButtons.ts
   └── 工具激活 → 提示收集

2. 发送HTTP请求
   (前端网络层)
   └── POST /monai/infer/segmentation

3. 后端接收请求
   monailabel/endpoints/infer.py
   └── run_inference()
       └── instance.infer(request)

4. 执行推理
   monailabel/tasks/infer/basic_infer.py
   └── BasicInferTask.__call__()
       ├── nnInteractive分支
       ├── SAM2/3分支
       ├── VoxTell分支
       └── MedGemma分支

5. 返回结果
   └── send_response()
       └── DICOM-SEG格式

6. 前端显示
   (前端渲染层)
   └── 加载分割到视口
```

### 模型初始化流

```
应用启动
└── monailabel/app.py: lifespan()
    └── app_instance()
        └── MONAILabelApp.__init__()
            └── init_infers()
                └── 加载 BasicInferTask
                    └── basic_infer.py 模块级初始化
                        ├── 下载nnInteractive模型
                        ├── 初始化SAM2/MedSAM2
                        ├── 初始化VoxTell
                        └── 初始化MedGemma
```

---

## 🔍 常见问题代码定位

### 1. 推理失败

| 问题 | 检查位置 |
|-----|---------|
| 模型未找到 | `interfaces/app.py:infer()` |
| 图像加载失败 | `endpoints/infer.py:run_inference()` |
| GPU内存不足 | `tasks/infer/basic_infer.py` 各模型分支 |
| 坐标错误 | `tasks/infer/basic_infer.py:460-475` (InstanceNumber检查) |

### 2. 分割显示问题

| 问题 | 检查位置 |
|-----|---------|
| 方向翻转 | `tasks/infer/basic_infer.py:683-686`, `1218-1221` |
| DICOM-SEG解析 | `endpoints/infer.py:read_seg_file()` |
| 窗宽窗位 | `utils/others/medgemma.py:window()` |

### 3. 会话问题

| 问题 | 检查位置 |
|-----|---------|
| nnInteractive重置 | `tasks/infer/basic_infer.py:449-454` |
| 会话超时 | `interfaces/app.py:cleanup_sessions()` |
| 提示重复 | `tasks/infer/basic_infer.py:394-402` |

---

## 📝 扩展开发指南

### 添加新模型

1. **后端推理** (参考 `basic_infer.py:976-1234`):
```python
# 在 __call__ 方法中添加新分支
if data['model_type'] == 'my_model':
    # 模型推理逻辑
    return result, metadata
```

2. **前端工具栏** (参考 `modes/longitudinal/src/toolbarButtons.ts`):
```typescript
{
  id: 'myModel',
  label: 'My Model',
  commands: [...]
}
```

### 添加新工具

1. **Cornerstone工具** (参考 `extensions/cornerstone/src/tools/`)
2. **工具注册** (参考 `extensions/cornerstone/src/init.tsx`)
3. **工具栏配置** (参考 `modes/longitudinal/src/toolbarButtons.ts`)

---

## 🔗 外部依赖

### Python包

| 包名 | 用途 | 配置位置 |
|-----|------|---------|
| fastapi | Web框架 | `requirements.txt` |
| torch | 深度学习 | `requirements.txt` |
| monai | 医学AI | `requirements.txt` |
| simpleitk | 医学影像处理 | `requirements.txt` |
| pydicom | DICOM处理 | `requirements.txt` |
| transformers | HuggingFace模型 | `requirements.txt` |
| nninteractive | 交互式分割 | 运行时下载 |
| sam2 | SAM2模型 | Dockerfile |
| sam3 | SAM3模型 | 手动安装 |
| voxtell | 文本分割 | 运行时下载 |

### Node.js包

| 包名 | 用途 | 配置位置 |
|-----|------|---------|
| @cornerstonejs/core | 影像渲染 | `package.json` |
| @cornerstonejs/tools | 影像工具 | `package.json` |
| react | UI框架 | `package.json` |
| zustand | 状态管理 | `package.json` |

---

## 📚 参考文档

- [ARCHITECTURE.md](./ARCHITECTURE.md) - 架构概览
- [IMPLEMENTATION_DETAILS.md](./IMPLEMENTATION_DETAILS.md) - 实现原理
- [README.md](./README.md) - 项目说明

---

*最后更新: 2026-02-18*
