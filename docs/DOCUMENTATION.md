# OHIF-AI 项目文档

本目录包含OHIF-AI项目的详细技术文档，帮助开发者理解、使用和扩展该项目。

---

## 📖 文档列表

### 1. [ARCHITECTURE.md](./ARCHITECTURE.md)
**项目架构详解**

- 系统整体架构图
- 前端架构 (OHIF Viewer + Cornerstone3D)
- 后端架构 (MONAI Label + FastAPI)
- AI模型集成架构
- 数据流详细说明
- Docker部署架构
- 关键技术点分析

**适合读者**: 想要了解系统整体设计的架构师和开发者

---

### 2. [IMPLEMENTATION_DETAILS.md](./IMPLEMENTATION_DETAILS.md)
**代码实现原理详解**

- 前端交互实现机制
- 后端推理流水线
- 各AI模型集成细节 (nnInteractive, SAM2/3, VoxTell, MedGemma)
- DICOM数据处理流程
- 通信协议规范
- 关键算法实现 (涂鸦处理、扫描线填充、球形核)
- 性能优化策略
- 错误处理与容错机制

**适合读者**: 需要深入理解代码实现细节的开发者

---

### 3. [MODULE_INDEX.md](./MODULE_INDEX.md)
**功能模块索引**

- 完整的目录结构
- 前端模块快速定位
- 后端API端点索引
- AI模型模块索引
- 数据流关键路径
- 常见问题代码定位
- 扩展开发指南
- 外部依赖清单

**适合读者**: 需要快速查找特定代码的开发者

---

## 🚀 快速开始

### 新用户
1. 先阅读 [README.md](./README.md) 了解项目基本功能
2. 查看 [ARCHITECTURE.md](./ARCHITECTURE.md) 理解系统架构

### 前端开发者
1. 查看 [MODULE_INDEX.md](./MODULE_INDEX.md#前端模块) 定位前端代码
2. 参考 [IMPLEMENTATION_DETAILS.md](./IMPLEMENTATION_DETAILS.md#1-前端交互实现) 理解交互机制

### 后端开发者
1. 查看 [MODULE_INDEX.md](./MODULE_INDEX.md#后端模块) 定位后端代码
2. 参考 [IMPLEMENTATION_DETAILS.md](./IMPLEMENTATION_DETAILS.md#2-后端推理实现) 理解推理流程

### 算法/模型开发者
1. 查看 [ARCHITECTURE.md](./ARCHITECTURE.md#模型架构) 了解模型架构
2. 参考 [IMPLEMENTATION_DETAILS.md](./IMPLEMENTATION_DETAILS.md#3-模型集成原理) 理解模型集成
3. 查看 [MODULE_INDEX.md](./MODULE_INDEX.md#ai模型模块) 定位模型代码

---

## 📂 项目结构

```
OHIF-AI/
├── README.md                  # 项目介绍和使用指南
├── DOCUMENTATION.md           # 本文档
├── ARCHITECTURE.md            # 架构详解
├── IMPLEMENTATION_DETAILS.md  # 实现原理
├── MODULE_INDEX.md            # 模块索引
│
├── Viewers/                   # 前端代码
│   ├── extensions/           # 扩展模块
│   ├── modes/                # 应用模式
│   └── platform/             # 平台核心
│
├── monai-label/              # 后端代码
│   ├── monailabel/          # 核心Python包
│   └── sample-apps/         # 示例应用
│
├── sam2/                     # SAM2模型
├── sam3/                     # SAM3模型
│
└── docker-compose.yml        # Docker部署配置
```

---

## 🔧 技术栈

### 前端
- **框架**: React 18 + TypeScript
- **影像引擎**: Cornerstone3D
- **状态管理**: Zustand
- **UI库**: Tailwind CSS + Radix UI
- **构建工具**: RSBuild

### 后端
- **Web框架**: FastAPI (Python)
- **深度学习**: PyTorch + MONAI
- **影像处理**: SimpleITK + pydicom
- **模型推理**: transformers, nnInteractive, SAM2

### 部署
- **容器化**: Docker + Docker Compose
- **GPU支持**: NVIDIA Container Toolkit
- **PACS服务器**: Orthanc

---

## 🤝 贡献指南

如果您想要改进这些文档：

1. 确保代码修改后同步更新相关文档
2. 添加新功能时更新 [MODULE_INDEX.md](./MODULE_INDEX.md)
3. 架构变更时更新 [ARCHITECTURE.md](./ARCHITECTURE.md)
4. 算法修改时更新 [IMPLEMENTATION_DETAILS.md](./IMPLEMENTATION_DETAILS.md)

---

## 📞 问题反馈

如有问题，请参考：

1. [README.md#FAQ](./README.md#faq) - 常见问题
2. [MODULE_INDEX.md#常见问题代码定位](./MODULE_INDEX.md#常见问题代码定位) - 代码级问题定位
3. 项目GitHub Issues

---

## 📚 外部参考

- [OHIF Documentation](https://docs.ohif.org/)
- [MONAI Label Documentation](https://docs.monai.io/projects/label/en/latest/)
- [Cornerstone3D Documentation](https://www.cornerstonejs.org/)
- [nnInteractive GitHub](https://github.com/MIC-DKFZ/nnInteractive)
- [SAM2 GitHub](https://github.com/facebookresearch/sam2)

---

*文档版本: 1.0*  
*最后更新: 2026-02-18*
