# OHIF-AI 配置文件详细指南

本文档详细解释项目中的所有配置文件及其配置项。

---

## 1. Docker部署配置

### docker-compose.yml (项目根目录)

**完整配置解析**:

```yaml
version: '3.8'

services:
  # ========== OHIF前端服务 ==========
  ohif_viewer:
    build:
      context: ./Viewers/           # 构建上下文
      dockerfile: ./platform/app/.recipes/Nginx-Orthanc/dockerfile
    image: webapp:latest            # 镜像名称
    container_name: ohif_orthanc    # 容器名称
    volumes:
      # Nginx配置挂载
      - ./Viewers/platform/app/.recipes/Nginx-Orthanc/config/nginx.conf:/etc/nginx/nginx.conf
      # 密码文件
      - ./Viewers/platform/app/.recipes/Nginx-Orthanc/config/.htpasswd:/etc/nginx/.htpasswd
      # 日志目录
      - ./Viewers/platform/app/.recipes/Nginx-Orthanc/logs/nginx:/var/logs/nginx
    ports:
      - '1025:1025'  # HTTPS端口
      - '1026:1026'  # HTTP端口 (主访问端口)
    depends_on:
      - orthanc       # 依赖Orthanc服务
    restart: on-failure

  # ========== Orthanc PACS服务 ==========
  orthanc:
    image: jodogne/orthanc-plugins   # 官方镜像
    hostname: orthanc
    container_name: orthancPACS
    volumes:
      # Orthanc配置
      - ./Viewers/platform/app/.recipes/Nginx-Orthanc/config/orthanc.json:/etc/orthanc/orthanc.json:ro
      # 数据持久化
      - ./Viewers/platform/app/.recipes/Nginx-Orthanc/volumes/orthanc-db/:/var/lib/orthanc/db/
    restart: unless-stopped
    ports:
      - '4242:4242'  # DICOM端口
      - '8042:8042'  # HTTP端口 (DICOM Web)

  # ========== MONAI Label AI服务 ==========
  monai_sam2:
    build:
      context: ./                    # 构建上下文为项目根目录
      dockerfile: ./monai-label/Dockerfile
    image: monai
    runtime: nvidia                  # 使用NVIDIA运行时
    container_name: monai_sam2
    volumes:
      # 预测结果持久化
      - ./monai-label/predictions:/code/predictions
    environment:
      - CUDA_VISIBLE_DEVICES=0       # 指定GPU设备
      - NVIDIA_VISIBLE_DEVICES=all   # 暴露所有GPU
      - NVIDIA_DRIVER_CAPABILITIES=all
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              capabilities: [gpu]   # 请求GPU资源
    shm_size: '10gb'                # 共享内存大小
    ports:
      - '8002:8002'                 # API端口
    depends_on:
      - ohif_viewer
      - orthanc
    restart: on-failure
```

**关键配置项说明**:

| 配置项 | 值 | 说明 |
|-------|-----|------|
| `runtime: nvidia` | - | 启用NVIDIA容器运行时 |
| `NVIDIA_VISIBLE_DEVICES` | all | 容器可见的GPU |
| `shm_size` | 10gb | PyTorch多进程需要大共享内存 |
| `depends_on` | - | 服务启动顺序 |

---

## 2. 后端配置文件

### monai-label/Dockerfile

**构建阶段详解**:

```dockerfile
# ========== 基础镜像 ==========
FROM nvidia/cuda:12.1.1-devel-ubuntu22.04
# 使用CUDA 12.1开发版镜像，包含nvcc编译器

# ========== 系统依赖安装 ==========
RUN apt update -y && apt install -y \
    openslide-tools \       # 病理切片支持
    build-essential \       # 编译工具
    python3-pip \           # Python包管理
    python3-dev \           # Python开发头文件
    python-is-python3 \     # python命令指向python3
    ffmpeg \                # 视频处理
    libsm6 libxext6 \       # OpenCV依赖
    libxcb-xinerama0 \      # Qt依赖
    wget git                # 下载工具

# ========== 工作目录 ==========
WORKDIR /code

# ========== 复制SAM2/SAM3代码 ==========
COPY ./sam2/. /install/     # SAM2源代码
COPY ./sam3/. /install-3/   # SAM3源代码

# ========== 复制assets ==========
RUN mkdir -p /code/assets
COPY ./sam3/assets/bpe_simple_vocab_16e6.txt.gz /code/assets/
# BPE词汇表，用于SAM3的文本编码

# ========== 环境变量 ==========
ENV LD_LIBRARY_PATH=$LD_LIBRARY_PATH:"/usr/local/nvidia/lib64"
# 添加NVIDIA库路径

# ========== Python依赖安装 ==========
COPY ./monai-label/requirements.txt ./
RUN pip install --upgrade pip setuptools wheel
RUN pip install -e /install/.       # 安装SAM2
RUN pip install -e /install-3/.     # 安装SAM3
RUN pip install --no-cache-dir -r requirements.txt

# ========== 模型权重下载 ==========
ENV WEIGHTS_DIR="/code/checkpoints/"

# SAM2.1权重
ENV WEIGHTS_URL="https://dl.fbaipublicfiles.com/segment_anything_2/092824/sam2.1_hiera_tiny.pt"
RUN wget --directory-prefix ${WEIGHTS_DIR} ${WEIGHTS_URL}

# MedSAM2权重
ENV MEDSAM2_WEIGHTS_URL="https://huggingface.co/wanglab/MedSAM2/resolve/main/MedSAM2_latest.pt"
RUN wget --directory-prefix ${WEIGHTS_DIR} ${MEDSAM2_WEIGHTS_URL}

# ========== 修复dicomweb_client ==========
RUN rm -rf /usr/local/lib/python3.10/dist-packages/dicomweb_client/web.py
COPY ./monai-label/backup/web.py /usr/local/lib/python3.10/dist-packages/dicomweb_client/
# 使用自定义修复版本

# ========== 环境配置 ==========
ENV CUDA_VISIBLE_DEVICES=1
ENV PATH=$PATH":/code/monailabel/scripts"
ENV PYTHONPATH=/code:$PYTHONPATH

# ========== 下载示例应用 ==========
RUN python -m monailabel.main apps --download --name radiology --output /code/apps

# ========== 暴露端口和启动命令 ==========
EXPOSE 8002
CMD ["python", "-m", "monailabel.main", "start_server", 
     "--app", "/code/apps/radiology",
     "--studies", "http://orthanc:8042/dicom-web",
     "--conf", "models", "segmentation",
     "--conf", "use_pretrained_model", "false",
     "-p", "8002"]
```

**关键环境变量**:

| 变量 | 值 | 说明 |
|-----|-----|------|
| `CUDA_VISIBLE_DEVICES` | 0 | 指定使用的GPU |
| `WEIGHTS_DIR` | /code/checkpoints/ | 模型权重存储路径 |
| `PYTHONPATH` | /code | Python模块搜索路径 |

---

### monai-label/requirements.txt

**依赖列表详解**:

```txt
# ========== Web框架 ==========
fastapi>=0.103.1          # FastAPI主框架
uvicorn[standard]         # ASGI服务器，[standard]包含高性能依赖
python-multipart          # 处理multipart/form-data (文件上传)
requests                  # HTTP客户端
httpx                     # 异步HTTP客户端

# ========== 数据验证和配置 ==========
pydantic>=2.0             # 数据验证 (V2版本)
pydantic-settings         # 配置管理
email-validator           # 邮箱验证

# ========== 医学影像处理 ==========
simpleitk                 # 医学影像处理 (SimpleITK)
pydicom                   # DICOM文件处理
python-gdcm               # GDCM DICOM解码器
pylibjpeg                 # JPEG解码器
pylibjpeg-openjpeg        # JPEG 2000解码器
pylibjpeg-rle             # RLE解码器
nibabel                     # NIfTI/ANALYZE处理

# ========== DICOM Web ==========
dicomweb-client           # DICOM Web客户端

# ========== 深度学习 ==========
torch                     # PyTorch核心
monai>=1.3.0              # MONAI医学AI框架
transformers              # HuggingFace Transformers
huggingface_hub           # HuggingFace Hub客户端

# ========== 其他工具 ==========
numpy                     # 数值计算
scipy                     # 科学计算
Pillow                    # 图像处理
scikit-image              # 图像处理
opencv-python             # OpenCV
matplotlib                # 绘图

# ========== 异步和并发 ==========
asyncio                   # 异步IO
schedule                  # 定时任务
timeloop                  # 定时循环

# ========== 文件和路径 ==========
pathlib                   # 路径处理
tempfile                  # 临时文件
shutil                    # 文件操作
```

---

### monai-label/setup.cfg

**Python包配置**:

```ini
[metadata]
name = monailabel
version = attr: monailabel._version.__version__
author = MONAI Consortium
author_email = monai.label@example.com
url = https://github.com/Project-MONAI/MONAILabel
description = MONAI Label Server and Client
long_description = file: README.md
long_description_content_type = text/markdown
license = Apache-2.0
classifiers =
    Development Status :: 4 - Beta
    Intended Audience :: Developers
    Intended Audience :: Healthcare Industry
    License :: OSI Approved :: Apache Software License
    Programming Language :: Python :: 3.9
    Programming Language :: Python :: 3.10
    Topic :: Scientific/Engineering :: Artificial Intelligence
    Topic :: Scientific/Engineering :: Medical Science Apps

[options]
python_requires = >=3.9
packages = find:
include_package_data = True
install_requires = 
    fastapi
    uvicorn
    pydantic>=2.0
    pydantic-settings
    python-multipart
    requests
    simpleitk
    pydicom
    dicomweb-client
    torch
    monai>=1.3.0
    # ... 其他依赖

[options.extras_require]
dev = 
    pytest
    pytest-cov
    black
    flake8
    mypy

[options.entry_points]
console_scripts = 
    monailabel = monailabel.main:main

[flake8]
max-line-length = 120
ignore = E203, E501, W503
```

---

## 3. 前端配置文件

### Viewers/package.json

**关键配置详解**:

```json
{
  "name": "ohif-viewer",
  "version": "3.0.0",
  "description": "OHIF Medical Imaging Viewer",
  "private": true,
  
  "workspaces": [
    "extensions/*",
    "modes/*",
    "platform/*"
  ],
  
  "scripts": {
    "build": "rsbuild build",
    "dev": "rsbuild dev --open",
    "test": "jest",
    "test:e2e": "playwright test",
    "lint": "eslint . --ext .ts,.tsx",
    "format": "prettier --write ."
  },
  
  "dependencies": {
    "@cornerstonejs/core": "^1.0.0",
    "@cornerstonejs/tools": "^1.0.0",
    "@cornerstonejs/adapters": "^1.0.0",
    "react": "^18.2.0",
    "react-dom": "^18.2.0",
    "zustand": "^4.4.0",
    "react-router-dom": "^6.0.0",
    "i18next": "^23.0.0",
    "react-i18next": "^13.0.0",
    "tailwindcss": "^3.3.0",
    "@radix-ui/react-dialog": "^1.0.0",
    "@radix-ui/react-dropdown-menu": "^2.0.0",
    "class-variance-authority": "^0.7.0",
    "clsx": "^2.0.0",
    "tailwind-merge": "^2.0.0"
  },
  
  "devDependencies": {
    "@rsbuild/core": "^0.3.0",
    "@rsbuild/plugin-react": "^0.3.0",
    "@rsbuild/plugin-svgr": "^0.3.0",
    "@types/react": "^18.2.0",
    "@types/react-dom": "^18.2.0",
    "typescript": "^5.2.0",
    "jest": "^29.0.0",
    "playwright": "^1.40.0",
    "eslint": "^8.0.0",
    "prettier": "^3.0.0"
  }
}
```

### Viewers/rsbuild.config.ts

**构建配置详解**:

```typescript
import { defineConfig } from '@rsbuild/core';
import { pluginReact } from '@rsbuild/plugin-react';

export default defineConfig({
  source: {
    entry: {
      index: './platform/app/src/index.tsx',
    },
    // 路径别名
    alias: {
      '@': './',
      '@ohif/core': './platform/core/src',
      '@ohif/ui': './platform/ui/src',
    },
  },
  
  server: {
    port: 3000,
    proxy: {
      '/monai': {
        target: 'http://localhost:8002',
        changeOrigin: true,
      },
    },
  },
  
  output: {
    distPath: {
      root: 'dist',
      js: 'static/js',
      css: 'static/css',
      svg: 'static/media',
    },
    assetPrefix: '/',
  },
  
  tools: {
    // SWC编译器配置
    swc: {
      jsc: {
        target: 'es2020',
        transform: {
          react: {
            runtime: 'automatic',
          },
        },
      },
    },
  },
  
  // 模块联邦配置 (扩展机制)
  moduleFederation: {
    options: {
      name: 'ohif_viewer',
      shared: {
        react: { singleton: true, requiredVersion: '^18.2.0' },
        'react-dom': { singleton: true },
      },
    },
  },
  
  plugins: [pluginReact()],
});
```

### Viewers/tsconfig.json

**TypeScript配置详解**:

```json
{
  "compilerOptions": {
    "target": "ES2020",
    "lib": ["ES2020", "DOM", "DOM.Iterable"],
    "module": "ESNext",
    "moduleResolution": "bundler",
    "jsx": "react-jsx",
    "strict": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "noImplicitReturns": true,
    "noFallthroughCasesInSwitch": true,
    "allowSyntheticDefaultImports": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "resolveJsonModule": true,
    "isolatedModules": true,
    "declaration": true,
    "declarationMap": true,
    "sourceMap": true,
    "baseUrl": ".",
    "paths": {
      "@/*": ["./*"],
      "@ohif/core": ["./platform/core/src"],
      "@ohif/core/*": ["./platform/core/src/*"],
      "@ohif/ui": ["./platform/ui/src"],
      "@ohif/ui/*": ["./platform/ui/src/*"],
    },
  },
  "include": [
    "extensions/**/*",
    "modes/**/*",
    "platform/**/*",
  ],
  "exclude": [
    "node_modules",
    "dist",
    "**/*.test.ts",
  ],
}
```

---

## 4. Nginx配置

### Viewers/platform/app/.recipes/Nginx-Orthanc/config/nginx.conf

```nginx
worker_processes auto;
error_log /var/logs/nginx/error.log warn;

events {
    worker_connections 1024;
}

http {
    include /etc/nginx/mime.types;
    default_type application/octet-stream;

    # 日志格式
    log_format main '$remote_addr - $remote_user [$time_local] "$request" '
                    '$status $body_bytes_sent "$http_referer" '
                    '"$http_user_agent" "$http_x_forwarded_for"';

    access_log /var/logs/nginx/access.log main;

    # Gzip压缩
    gzip on;
    gzip_vary on;
    gzip_types text/plain text/css application/json application/javascript;

    # 上游服务
    upstream orthanc_server {
        server orthanc:8042;
    }

    upstream monai_server {
        server monai_sam2:8002;
    }

    # HTTP服务器
    server {
        listen 1026;
        server_name localhost;
        root /usr/share/nginx/html;
        index index.html;

        # 前端静态文件
        location / {
            try_files $uri $uri/ /index.html;
            add_header Cache-Control "no-cache";
        }

        # DICOM Web代理 (Orthanc)
        location /dicom-web/ {
            proxy_pass http://orthanc_server;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            
            # CORS头
            add_header 'Access-Control-Allow-Origin' '*' always;
            add_header 'Access-Control-Allow-Methods' 'GET, POST, OPTIONS' always;
        }

        # MONAI Label API代理
        location /monai/ {
            proxy_pass http://monai_server/;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_read_timeout 300s;  # 长超时（AI推理可能慢）
        }

        # 静态资源缓存
        location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg)$ {
            expires 1y;
            add_header Cache-Control "public, immutable";
        }
    }
}
```

---

## 5. Orthanc配置

### Viewers/platform/app/.recipes/Nginx-Orthanc/config/orthanc.json

```json
{
  "Name": "OHIF-Orthanc",
  "StorageDirectory": "/var/lib/orthanc/db",
  "IndexDirectory": "/var/lib/orthanc/db",
  
  "DicomServerEnabled": true,
  "DicomPort": 4242,
  "DicomAet": "ORTHANC",
  
  "HttpPort": 8042,
  "HttpServerEnabled": true,
  "RemoteAccessAllowed": true,
  "AuthenticationEnabled": false,
  
  "DicomWeb": {
    "Enable": true,
    "Root": "/dicom-web/",
    "EnableWado": true,
    "WadoRoot": "/wado",
    "Host": "0.0.0.0",
    "Ssl": false
  },
  
  "Plugins": [
    "/usr/share/orthanc/plugins/libOrthancDicomWeb.so"
  ]
}
```

---

## 6. 模式配置

### Viewers/modes/longitudinal/src/toolbarButtons.ts

**AI工具配置详解**:

```typescript
// ========== AI工具箱容器 ==========
export const aiToolBoxContainer = {
  id: 'aiToolBoxContainer',
  label: 'AI Toolbox',
  icon: 'icon-tool-ai',
  type: 'splitButton',
  buttons: [{
    id: 'aiToolBox',
    label: 'AI Tools',
    icon: 'icon-tool-ai',
    type: 'ohif.splitButton',
    props: {
      groupId: 'aiToolBox',
      isRadio: true,
      primary: 'Probe2',
      secondary: {
        icon: 'icon-chevron-down',
        label: 'AI Tools',
        items: [
          { label: 'Point', value: 'Probe2' },
          { label: 'Scribble', value: 'PlanarFreehandROI2' },
          { label: 'Lasso', value: 'PlanarFreehandROI3' },
          { label: 'Box', value: 'RectangleROI2' },
        ],
      },
    },
  }],
};

// ========== 点提示工具 ==========
export const Probe2 = {
  id: 'Probe2',
  label: 'Point Prompt',
  icon: 'icon-tool-point',
  type: 'tool',
  commandName: 'setToolActive',
  commandOptions: {
    toolName: 'Probe',
    toolGroupId: 'default',
  },
  // 提示收集回调
  listeners: {
    [EVENTS.MOUSE_CLICK]: {
      callback: (event) => {
        const { currentPoints } = event.detail;
        // 发送点坐标到AI服务
        aiService.addPoint(currentPoints.image);
      },
    },
  },
};

// ========== 模型选择按钮 ==========
export const nninterButton = {
  id: 'nninter',
  label: 'nnInteractive',
  icon: 'icon-ai-nninteractive',
  type: 'toggle',
  commandName: 'setAIModel',
  commandOptions: {
    model: 'nninter',
    params: {
      type: 'segmentation',
      dimension: 3,
    },
  },
};

// ========== 文本提示输入 ==========
export const textPromptSegmentation = {
  id: 'textPromptSegmentation',
  label: 'Text Prompt',
  icon: 'icon-text-prompt',
  type: 'custom',
  component: TextPromptInput,
  props: {
    placeholder: 'e.g., liver, tumor, kidney',
    onSubmit: (text) => aiService.runTextPrompt(text),
  },
};

// ========== MedGemma面板 ==========
export const testMedgemma = {
  id: 'testMedgemma',
  label: 'Generate Report',
  icon: 'icon-report',
  type: 'panel',
  component: MedGemmaPanel,
  props: {
    defaultInstruction: 'You are a radiology assistant...',
    onGenerate: (params) => aiService.generateReport(params),
  },
};
```

---

## 7. 环境变量配置

### 完整环境变量列表

**后端环境变量** (monai-label):

| 变量名 | 默认值 | 说明 |
|-------|-------|------|
| `MONAI_LABEL_APP_DIR` | - | 应用目录 |
| `MONAI_LABEL_STUDIES` | - | 研究数据路径 |
| `MONAI_LABEL_SERVER_PORT` | 8000 | 服务器端口 |
| `MONAI_LABEL_CORS_ORIGINS` | ["*"] | CORS来源 |
| `MONAI_LABEL_DICOMWEB_USERNAME` | - | DICOM Web用户名 |
| `MONAI_LABEL_DICOMWEB_PASSWORD` | - | DICOM Web密码 |
| `MONAI_LABEL_INFER_TIMEOUT` | 300 | 推理超时(秒) |
| `MONAI_LABEL_INFER_CONCURRENCY` | -1 | 推理并发数 |
| `MONAI_LABEL_SESSION_EXPIRY` | 3600 | 会话过期时间(秒) |
| `CUDA_VISIBLE_DEVICES` | - | 可见GPU设备 |
| `HF_TOKEN` | - | HuggingFace令牌 |

**Docker环境变量**:

| 变量名 | 值 | 说明 |
|-------|-----|------|
| `NVIDIA_VISIBLE_DEVICES` | all | 所有GPU |
| `NVIDIA_DRIVER_CAPABILITIES` | all | 所有GPU能力 |

---

## 8. 配置文件修改指南

### 修改服务端口

**docker-compose.yml**:
```yaml
services:
  ohif_viewer:
    ports:
      - '8080:1026'  # 修改主机端口为8080
  
  monai_sam2:
    ports:
      - '9000:8002'  # 修改主机端口为9000
```

**nginx.conf**:
```nginx
server {
    listen 1026;  # 保持容器内端口不变
    # ...
}
```

### 添加新模型

**Dockerfile**:
```dockerfile
# 添加新模型权重下载
ENV NEW_MODEL_URL="https://example.com/model.pt"
RUN wget --directory-prefix ${WEIGHTS_DIR} ${NEW_MODEL_URL}
```

**basic_infer.py**:
```python
# 添加新模型初始化
new_model = load_new_model("/code/checkpoints/model.pt")

# 在__call__中添加分支
if data['model_type'] == 'new_model':
    result = new_model.predict(...)
```

### 修改CORS设置

**monailabel/config.py**:
```python
MONAI_LABEL_CORS_ORIGINS: List[str] = [
    "https://your-domain.com",
    "https://app.your-domain.com",
]
```

---

*本指南详细说明了项目中的所有配置文件，更多细节请参考具体配置文件*
