# OHIF-AI 项目文档总览

欢迎查阅OHIF-AI项目的完整技术文档。本文档索引帮助您快速找到所需信息。

---

## 📚 文档清单

### 核心文档 (必看)

| 文档 | 文件名 | 大小 | 内容概要 |
|-----|--------|------|---------|
| **项目总览** | [README.md](./README.md) | ~17KB | 项目介绍、功能说明、快速开始 |
| **架构详解** | [ARCHITECTURE.md](./ARCHITECTURE.md) | ~21KB | 系统架构、模块关系、数据流 |
| **实现原理** | [IMPLEMENTATION_DETAILS.md](./IMPLEMENTATION_DETAILS.md) | ~25KB | 核心算法、模型集成、通信协议 |
| **模块索引** | [MODULE_INDEX.md](./MODULE_INDEX.md) | ~11KB | 快速查找代码位置 |

### 详细参考文档

| 文档 | 文件名 | 大小 | 内容概要 |
|-----|--------|------|---------|
| **完整文件参考** | [COMPLETE_FILE_REFERENCE.md](./COMPLETE_FILE_REFERENCE.md) | ~20KB | 每个目录和文件的详细说明 |
| **前端文件指南** | [DETAILED_FRONTEND_GUIDE.md](./DETAILED_FRONTEND_GUIDE.md) | ~14KB | 前端各文件详解 |
| **后端文件指南** | [DETAILED_BACKEND_GUIDE.md](./DETAILED_BACKEND_GUIDE.md) | ~35KB | 后端各文件详解 |
| **配置文件指南** | [DETAILED_CONFIG_GUIDE.md](./DETAILED_CONFIG_GUIDE.md) | ~20KB | 所有配置项详解 |

### 导航文档

| 文档 | 文件名 | 内容概要 |
|-----|--------|---------|
| **文档导航** | [DOCUMENTATION.md](./DOCUMENTATION.md) | 文档入口和导航 |
| **本文档** | [PROJECT_DOCUMENTATION_INDEX.md](./PROJECT_DOCUMENTATION_INDEX.md) | 文档总览 |

---

## 🎯 根据角色选择文档

### 👨‍💻 前端开发者

**阅读顺序**:
1. [README.md](./README.md) - 了解项目整体
2. [ARCHITECTURE.md](./ARCHITECTURE.md#前端架构) - 前端架构
3. [DETAILED_FRONTEND_GUIDE.md](./DETAILED_FRONTEND_GUIDE.md) - 前端文件详解
4. [MODULE_INDEX.md](./MODULE_INDEX.md#前端模块) - 快速定位代码

**重点关注**:
- `Viewers/extensions/cornerstone/` - 影像渲染核心
- `Viewers/modes/longitudinal/` - AI功能主模式
- `Viewers/platform/core/` - 核心服务

---

### 👨‍💻 后端开发者

**阅读顺序**:
1. [README.md](./README.md) - 了解项目整体
2. [ARCHITECTURE.md](./ARCHITECTURE.md#后端架构) - 后端架构
3. [IMPLEMENTATION_DETAILS.md](./IMPLEMENTATION_DETAILS.md#后端推理实现) - 推理实现
4. [DETAILED_BACKEND_GUIDE.md](./DETAILED_BACKEND_GUIDE.md) - 后端文件详解

**重点关注**:
- `monai-label/monailabel/tasks/infer/basic_infer.py` - 核心推理实现
- `monai-label/monailabel/endpoints/infer.py` - API端点
- `monai-label/monailabel/interfaces/app.py` - 应用基类

---

### 🔬 AI/算法工程师

**阅读顺序**:
1. [ARCHITECTURE.md](./ARCHITECTURE.md#模型架构) - 模型架构
2. [IMPLEMENTATION_DETAILS.md](./IMPLEMENTATION_DETAILS.md#模型集成原理) - 模型集成
3. [DETAILED_BACKEND_GUIDE.md](./DETAILED_BACKEND_GUIDE.md#tasksinfer-推理任务目录) - 推理实现细节

**重点关注**:
- nnInteractive集成
- SAM2/MedSAM2/SAM3集成
- VoxTell文本分割
- MedGemma报告生成

---

### 🚀 运维/DevOps工程师

**阅读顺序**:
1. [README.md](./README.md#getting-started) - 部署说明
2. [ARCHITECTURE.md](./ARCHITECTURE.md#docker部署架构) - 部署架构
3. [DETAILED_CONFIG_GUIDE.md](./DETAILED_CONFIG_GUIDE.md) - 配置详解
4. [DETAILED_CONFIG_GUIDE.md](./DETAILED_CONFIG_GUIDE.md#docker部署配置) - Docker配置

**重点关注**:
- `docker-compose.yml` - 服务编排
- `monai-label/Dockerfile` - 后端镜像
- Nginx配置
- 环境变量配置

---

### 📊 技术负责人/架构师

**阅读顺序**:
1. [README.md](./README.md) - 项目概览
2. [ARCHITECTURE.md](./ARCHITECTURE.md) - 完整架构
3. [IMPLEMENTATION_DETAILS.md](./IMPLEMENTATION_DETAILS.md) - 实现细节
4. [COMPLETE_FILE_REFERENCE.md](./COMPLETE_FILE_REFERENCE.md) - 文件结构

---

## 📖 文档内容速查

### 架构相关

| 查询内容 | 参考文档 |
|---------|---------|
| 系统整体架构图 | [ARCHITECTURE.md](./ARCHITECTURE.md#整体架构图) |
| 前端架构详解 | [ARCHITECTURE.md](./ARCHITECTURE.md#前端架构) |
| 后端架构详解 | [ARCHITECTURE.md](./ARCHITECTURE.md#后端架构) |
| 数据流说明 | [ARCHITECTURE.md](./ARCHITECTURE.md#数据流) |
| Docker部署架构 | [ARCHITECTURE.md](./ARCHITECTURE.md#docker部署架构) |
| 模型架构 | [ARCHITECTURE.md](./ARCHITECTURE.md#模型架构) |

### 前端代码相关

| 查询内容 | 参考文档 |
|---------|---------|
| 前端目录结构 | [COMPLETE_FILE_REFERENCE.md](./COMPLETE_FILE_REFERENCE.md#viewers-extensions---扩展模块) |
| Cornerstone扩展详解 | [DETAILED_FRONTEND_GUIDE.md](./DETAILED_FRONTEND_GUIDE.md#extensionscornerstone-详解) |
| 纵向模式(AI功能) | [DETAILED_FRONTEND_GUIDE.md](./DETAILED_FRONTEND_GUIDE.md#modeslongitudinal-纵向模式) |
| AI工具栏配置 | [DETAILED_FRONTEND_GUIDE.md](./DETAILED_FRONTEND_GUIDE.md#toolbarbuttonsts) |
| 前端数据流 | [DETAILED_FRONTEND_GUIDE.md](./DETAILED_FRONTEND_GUIDE.md#前端数据流详解) |
| 添加新AI工具 | [DETAILED_FRONTEND_GUIDE.md](./DETAILED_FRONTEND_GUIDE.md#添加新的ai工具) |

### 后端代码相关

| 查询内容 | 参考文档 |
|---------|---------|
| 后端目录结构 | [COMPLETE_FILE_REFERENCE.md](./COMPLETE_FILE_REFERENCE.md#monai-label---后端代码目录) |
| FastAPI应用入口 | [DETAILED_BACKEND_GUIDE.md](./DETAILED_BACKEND_GUIDE.md#apppy) |
| 命令行入口 | [DETAILED_BACKEND_GUIDE.md](./DETAILED_BACKEND_GUIDE.md#mainpy) |
| 全局配置管理 | [DETAILED_BACKEND_GUIDE.md](./DETAILED_BACKEND_GUIDE.md#configpy) |
| 推理API端点 | [DETAILED_BACKEND_GUIDE.md](./DETAILED_BACKEND_GUIDE.md#inferpy) |
| MONAILabelApp基类 | [DETAILED_BACKEND_GUIDE.md](./DETAILED_BACKEND_GUIDE.md#apppy-1) |
| BasicInferTask实现 | [DETAILED_BACKEND_GUIDE.md](./DETAILED_BACKEND_GUIDE.md#basic_inferpy) |
| nnInteractive集成 | [DETAILED_BACKEND_GUIDE.md](./DETAILED_BACKEND_GUIDE.md#basic_inferpy-详解) |
| SAM2/3集成 | [DETAILED_BACKEND_GUIDE.md](./DETAILED_BACKEND_GUIDE.md#basic_inferpy-详解) |

### 模型集成相关

| 查询内容 | 参考文档 |
|---------|---------|
| nnInteractive原理 | [IMPLEMENTATION_DETAILS.md](./IMPLEMENTATION_DETAILS.md#31-nninteractive集成) |
| SAM2集成 | [IMPLEMENTATION_DETAILS.md](./IMPLEMENTATION_DETAILS.md#32-sam2medsam2sam3集成) |
| VoxTell集成 | [IMPLEMENTATION_DETAILS.md](./IMPLEMENTATION_DETAILS.md#33-voxtell集成) |
| MedGemma集成 | [IMPLEMENTATION_DETAILS.md](./IMPLEMENTATION_DETAILS.md#34-medgemma集成) |
| 坐标系统转换 | [IMPLEMENTATION_DETAILS.md](./IMPLEMENTATION_DETAILS.md#42-dicom到nifti转换) |
| 窗宽窗位处理 | [IMPLEMENTATION_DETAILS.md](./IMPLEMENTATION_DETAILS.md#窗宽窗位计算) |

### 配置相关

| 查询内容 | 参考文档 |
|---------|---------|
| Docker Compose配置 | [DETAILED_CONFIG_GUIDE.md](./DETAILED_CONFIG_GUIDE.md#docker-composeyml) |
| Dockerfile详解 | [DETAILED_CONFIG_GUIDE.md](./DETAILED_CONFIG_GUIDE.md#monai-labeldockerfile) |
| Python依赖 | [DETAILED_CONFIG_GUIDE.md](./DETAILED_CONFIG_GUIDE.md#monai-labelrequirementstxt) |
| Nginx配置 | [DETAILED_CONFIG_GUIDE.md](./DETAILED_CONFIG_GUIDE.md#nginx配置) |
| Orthanc配置 | [DETAILED_CONFIG_GUIDE.md](./DETAILED_CONFIG_GUIDE.md#orthanc配置) |
| 前端构建配置 | [DETAILED_CONFIG_GUIDE.md](./DETAILED_CONFIG_GUIDE.md#viewersrsbuildconfigts) |
| 环境变量列表 | [DETAILED_CONFIG_GUIDE.md](./DETAILED_CONFIG_GUIDE.md#环境变量配置) |

### 开发相关

| 查询内容 | 参考文档 |
|---------|---------|
| 添加新模型 | [IMPLEMENTATION_DETAILS.md](./IMPLEMENTATION_DETAILS.md#添加新的ai模型) |
| 添加新工具 | [DETAILED_FRONTEND_GUIDE.md](./DETAILED_FRONTEND_GUIDE.md#添加新的ai工具) |
| 修改视口布局 | [DETAILED_FRONTEND_GUIDE.md](./DETAILED_FRONTEND_GUIDE.md#修改视口布局) |
| 修改服务端口 | [DETAILED_CONFIG_GUIDE.md](./DETAILED_CONFIG_GUIDE.md#修改服务端口) |
| 关键文件依赖关系 | [COMPLETE_FILE_REFERENCE.md](./COMPLETE_FILE_REFERENCE.md#-关键文件依赖关系) |

---

## 🔍 常见问题速查

### 推理失败

1. 检查 [DETAILED_BACKEND_GUIDE.md#inferpy-详解](./DETAILED_BACKEND_GUIDE.md#inferpy-详解) 了解API接口
2. 检查 [DETAILED_CONFIG_GUIDE.md#环境变量配置](./DETAILED_CONFIG_GUIDE.md#环境变量配置) 确认配置
3. 查看 [IMPLEMENTATION_DETAILS.md#错误处理与容错](./IMPLEMENTATION_DETAILS.md#错误处理与容错)

### 分割显示问题

1. 检查 [IMPLEMENTATION_DETAILS.md#坐标系统转换](./IMPLEMENTATION_DETAILS.md#坐标系统转换)
2. 查看 [DETAILED_FRONTEND_GUIDE.md#segmentationservicets](./DETAILED_FRONTEND_GUIDE.md#segmentationservicets)

### 模型加载失败

1. 检查 [DETAILED_CONFIG_GUIDE.md#monai-labeldockerfile](./DETAILED_CONFIG_GUIDE.md#monai-labeldockerfile) 权重下载配置
2. 查看 [DETAILED_BACKEND_GUIDE.md#basic_inferpy-详解](./DETAILED_BACKEND_GUIDE.md#basic_inferpy-详解) 模型初始化代码

### 性能优化

1. 查看 [IMPLEMENTATION_DETAILS.md#性能优化实现](./IMPLEMENTATION_DETAILS.md#性能优化实现)
2. 检查 [DETAILED_CONFIG_GUIDE.md#环境变量配置](./DETAILED_CONFIG_GUIDE.md#环境变量配置) 并发配置

---

## 📂 项目文件结构速览

```
OHIF-AI/
├── 📄 文档文件 (8个.md文件)
│
├── 🐳 docker-compose.yml          # Docker部署配置
├── ▶️  start.sh                   # 启动脚本
│
├── 📁 Viewers/                     # 前端代码
│   ├── 📁 extensions/             # 扩展模块
│   │   ├── cornerstone/          # 核心影像渲染
│   │   ├── cornerstone-dicom-seg/ # DICOM SEG支持
│   │   ├── cornerstone-dicom-rt/  # DICOM RT支持
│   │   ├── default/              # 默认UI扩展
│   │   └── ...
│   ├── 📁 modes/                  # 应用模式
│   │   ├── longitudinal/         # 主模式(AI功能)
│   │   ├── segmentation/         # 分割模式
│   │   └── ...
│   ├── 📁 platform/               # 平台核心
│   │   ├── app/                  # 应用主体
│   │   ├── core/                 # 核心库
│   │   └── ui/                   # UI组件库
│   └── 📄 package.json            # npm配置
│
├── 📁 monai-label/                 # 后端代码
│   ├── 📁 monailabel/             # Python包
│   │   ├── 📄 app.py             # FastAPI入口
│   │   ├── 📄 main.py            # CLI入口
│   │   ├── 📄 config.py          # 全局配置
│   │   ├── 📁 endpoints/         # API端点
│   │   │   └── 📄 infer.py      # 推理API
│   │   ├── 📁 interfaces/        # 接口定义
│   │   │   └── 📄 app.py        # 应用基类
│   │   ├── 📁 tasks/infer/       # 推理任务
│   │   │   └── 📄 basic_infer.py # 核心推理
│   │   ├── 📁 datastore/         # 数据存储
│   │   └── 📁 utils/             # 工具函数
│   ├── 📁 sample-apps/           # 示例应用
│   ├── 📄 Dockerfile             # 镜像构建
│   └── 📄 requirements.txt       # Python依赖
│
├── 📁 sam2/                        # SAM2模型
├── 📁 sam3/                        # SAM3模型
└── 📁 sample-data/                 # 示例数据
```

---

## 🛠️ 开发工作流

### 1. 前端开发

```bash
# 进入前端目录
cd Viewers

# 安装依赖
npm install

# 开发模式
npm run dev

# 构建
npm run build
```

**修改文件后查看**:
- [DETAILED_FRONTEND_GUIDE.md](./DETAILED_FRONTEND_GUIDE.md) - 确认文件作用
- 使用浏览器开发者工具调试
- 查看Cornerstone3D文档

### 2. 后端开发

```bash
# 进入后端目录
cd monai-label

# 安装依赖
pip install -r requirements.txt

# 启动开发服务器
python -m monailabel.main start_server \
    --app sample-apps/radiology \
    --studies http://localhost:8042/dicom-web \
    -p 8002
```

**修改文件后查看**:
- [DETAILED_BACKEND_GUIDE.md](./DETAILED_BACKEND_GUIDE.md) - 确认文件作用
- 查看FastAPI自动生成的API文档: http://localhost:8002/docs
- 查看日志输出

### 3. 添加新AI模型

1. 后端实现: [IMPLEMENTATION_DETAILS.md#添加新的ai模型](./IMPLEMENTATION_DETAILS.md#添加新的ai模型)
2. 前端配置: [DETAILED_FRONTEND_GUIDE.md#添加新的ai工具](./DETAILED_FRONTEND_GUIDE.md#添加新的ai工具)
3. Docker配置: [DETAILED_CONFIG_GUIDE.md#添加新模型](./DETAILED_CONFIG_GUIDE.md#添加新模型)

---

## 📞 获取更多帮助

### 项目相关

- GitHub Issues: 提交Bug报告和功能请求
- GitHub Discussions: 讨论使用问题

### 外部文档

- [OHIF Documentation](https://docs.ohif.org/)
- [MONAI Label Documentation](https://docs.monai.io/projects/label/)
- [Cornerstone3D Documentation](https://www.cornerstonejs.org/)
- [FastAPI Documentation](https://fastapi.tiangolo.com/)

### 模型文档

- [nnInteractive](https://github.com/MIC-DKFZ/nnInteractive)
- [SAM2](https://github.com/facebookresearch/sam2)
- [SAM3](https://github.com/facebookresearch/sam3)
- [VoxTell](https://github.com/MIC-DKFZ/VoxTell)
- [MedGemma](https://github.com/Google-Health/medgemma)

---

## 📝 文档更新记录

| 日期 | 更新内容 |
|-----|---------|
| 2026-02-18 | 创建完整技术文档套件 |

---

*本文档索引最后更新: 2026-02-18*

*如有问题，请查阅相应详细文档或提交Issue*
