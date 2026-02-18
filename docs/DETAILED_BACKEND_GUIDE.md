# OHIF-AI 后端文件详细指南

本文档详细解释monai-label目录下每个重要文件的具体作用和实现细节。

---

## 目录结构

```
monai-label/
├── 配置文件
├── monailabel/          # Python包主目录
│   ├── 根文件          # app.py, main.py, config.py
│   ├── endpoints/      # API端点
│   ├── interfaces/     # 接口定义
│   ├── tasks/          # 任务实现
│   ├── datastore/      # 数据存储
│   ├── transform/      # 数据变换
│   ├── utils/          # 工具函数
│   └── ...
├── sample-apps/        # 示例应用
│   └── radiology/      # 放射学应用
├── sam2/               # SAM2模型配置
├── sam3/               # SAM3模型配置
└── tests/              # 测试
```

---

## 根目录配置文件

### Dockerfile
**路径**: `monai-label/Dockerfile`

**作用**: 构建后端Docker镜像

**构建流程**:
```dockerfile
# 1. 基础镜像
FROM nvidia/cuda:12.1.1-devel-ubuntu22.04

# 2. 系统依赖安装
RUN apt install openslide-tools build-essential python3-pip ...

# 3. 复制代码
COPY ./monai-label/. ./
COPY ./sam2/. /install/
COPY ./sam3/. /install-3/

# 4. 安装Python包
RUN pip install -e /install/.      # SAM2
RUN pip install -e /install-3/.    # SAM3
RUN pip install -r requirements.txt

# 5. 下载模型权重
RUN wget ${WEIGHTS_URL}            # SAM2权重
RUN wget ${MEDSAM2_WEIGHTS_URL}    # MedSAM2权重

# 6. 启动命令
CMD ["python", "-m", "monailabel.main", "start_server", ...]
```

**关键配置**:
- `ENV WEIGHTS_DIR="/code/checkpoints/"` - 模型权重目录
- `EXPOSE 8002` - 服务端口
- `NVIDIA_VISIBLE_DEVICES=all` - GPU访问

### requirements.txt
**路径**: `monai-label/requirements.txt`

**主要依赖**:
```
fastapi>=0.103.1       # Web框架
uvicorn[standard]      # ASGI服务器
pydantic>=2.0          # 数据验证
pydantic-settings      # 配置管理
python-multipart       # 文件上传
requests               # HTTP请求
simpleitk              # 医学影像处理
pydicom                # DICOM处理
nibabel                  # NIfTI处理
dicomweb-client        # DICOM Web客户端
torch                  # PyTorch
monai>=1.3.0           # MONAI框架
transformers           # HuggingFace模型
```

### setup.cfg
**路径**: `monai-label/setup.cfg`

**作用**: Python包元数据配置

```ini
[metadata]
name = monailabel
version = attr: monailabel._version.__version__
description = MONAI Label Server

[options]
packages = find:
install_requires = 
    fastapi
    uvicorn
    ...
```

---

## monailabel/ 主包目录

### __init__.py
**路径**: `monailabel/__init__.py`

**作用**: 包入口

```python
from ._version import __version__

# 导出主要类和函数
from .interfaces.app import MONAILabelApp
from .interfaces.datastore import Datastore

# 打印配置函数
def print_config():
    """打印MONAI Label配置信息"""
    print(f"MONAI Label version: {__version__}")
    ...
```

### _version.py
**路径**: `monailabel/_version.py`

**作用**: 版本号定义

```python
__version__ = "0.7.0"
```

### app.py
**路径**: `monailabel/app.py`

**作用**: **FastAPI应用主入口**

**完整代码结构**:
```python
import os
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware
from contextlib import asynccontextmanager

from monailabel.config import settings
from monailabel.endpoints import (
    datastore, infer, info, model, 
    ohif, proxy, scoring, session
)
from monailabel.interfaces.utils.app import app_instance

# CORS配置
origins = [str(origin) for origin in settings.MONAI_LABEL_CORS_ORIGINS]

@asynccontextmanager
async def lifespan(app: FastAPI):
    """应用生命周期管理"""
    print("App Init...")
    instance = app_instance()
    instance.server_mode(True)
    instance.on_init_complete()
    
    yield  # 应用运行期间
    
    print("App Shutdown...")

# 创建FastAPI应用
app = FastAPI(
    title=settings.MONAI_LABEL_PROJECT_NAME,
    root_path="/monai",           # 根路径前缀
    openapi_url="/openapi.json",  # OpenAPI文档
    docs_url=None,                # 禁用默认Swagger
    redoc_url="/docs",            # ReDoc文档
    lifespan=lifespan,
    middleware=[
        Middleware(
            CORSMiddleware,
            allow_origins=origins,  # 允许的跨域来源
            allow_credentials=True,
            allow_methods=["*"],
            allow_headers=["*"],
        )
    ],
)

# 静态文件挂载
app.mount("/static", StaticFiles(directory=...), name="static")

# 注册路由
app.include_router(info.router, prefix=settings.MONAI_LABEL_API_STR)
app.include_router(model.router, prefix=settings.MONAI_LABEL_API_STR)
app.include_router(infer.router, prefix=settings.MONAI_LABEL_API_STR)  # 推理API
app.include_router(datastore.router, prefix=settings.MONAI_LABEL_API_STR)
app.include_router(scoring.router, prefix=settings.MONAI_LABEL_API_STR)
app.include_router(ohif.router, prefix=settings.MONAI_LABEL_API_STR)
app.include_router(session.router, prefix=settings.MONAI_LABEL_API_STR)

# 自定义Swagger UI
@app.get("/", include_in_schema=False)
async def custom_swagger_ui_html():
    return HTMLResponse(...)
```

**关键配置项**:
- `root_path="/monai"`: 所有API路由前缀
- `MONAI_LABEL_API_STR`: 默认为"/"
- `StaticFiles`: 挂载静态文件目录

### main.py
**路径**: `monailabel/main.py`

**作用**: **命令行入口**

**Main类结构**:
```python
class Main:
    def __init__(self, loglevel=logging.INFO, actions=...):
        self.actions = set(actions)
        logging.basicConfig(...)
    
    # 参数定义方法
    def args_start_server(self, parser):
        """定义start_server子命令参数"""
        parser.add_argument("-a", "--app", help="App Directory")
        parser.add_argument("-s", "--studies", help="Studies Directory")
        parser.add_argument("-i", "--host", default="0.0.0.0")
        parser.add_argument("-p", "--port", default=8000, type=int)
        parser.add_argument("--conf", nargs=2, action="append")
        # ... 更多参数
    
    def args_apps(self, parser):
        """定义apps子命令参数"""
        parser.add_argument("-d", "--download", action="store_true")
        parser.add_argument("-n", "--name", help="App name")
    
    def run(self):
        """主运行方法"""
        parser = self.args_parser()
        args = parser.parse_args()
        
        if args.action == "start_server":
            self.action_start_server(args)
        elif args.action == "apps":
            self.action_apps(args)
        # ...
    
    def action_start_server(self, args):
        """启动服务器"""
        self.start_server_validate_args(args)
        self.start_server_init_settings(args)
        
        uvicorn.run(
            args.uvicorn_app,      # "monailabel.app:app"
            host=args.host,
            port=args.port,
            root_path=args.root_path,
            ...
        )
```

**启动命令示例**:
```bash
python -m monailabel.main start_server \
    --app /code/apps/radiology \
    --studies http://orthanc:8042/dicom-web \
    --conf models segmentation \
    --conf use_pretrained_model false \
    -p 8002
```

### config.py
**路径**: `monailabel/config.py`

**作用**: **全局配置管理**

```python
from pydantic_settings import BaseSettings
from typing import List, Optional

class Settings(BaseSettings):
    """MONAI Label配置类"""
    
    # 应用配置
    MONAI_LABEL_PROJECT_NAME: str = "MONAI Label"
    MONAI_LABEL_APP_DIR: str = ""
    MONAI_LABEL_STUDIES: str = ""
    MONAI_LABEL_APP_CONF: Dict = {}
    
    # 服务器配置
    MONAI_LABEL_SERVER_PORT: int = 8000
    MONAI_LABEL_CORS_ORIGINS: List[str] = ["*"]
    
    # 数据存储配置
    MONAI_LABEL_DATASTORE: str = ""
    MONAI_LABEL_DATASTORE_FILE_EXT: List[str] = [".nii.gz", ".nii", ".nrrd"]
    MONAI_LABEL_DATASTORE_AUTO_RELOAD: bool = True
    
    # DICOM Web配置
    MONAI_LABEL_DICOMWEB_USERNAME: str = ""
    MONAI_LABEL_DICOMWEB_PASSWORD: str = ""
    MONAI_LABEL_QIDO_PREFIX: str = ""
    MONAI_LABEL_WADO_PREFIX: str = ""
    MONAI_LABEL_STOW_PREFIX: str = ""
    
    # 推理配置
    MONAI_LABEL_INFER_TIMEOUT: int = 300
    MONAI_LABEL_INFER_CONCURRENCY: int = -1
    
    # 会话配置
    MONAI_LABEL_SESSIONS: bool = True
    MONAI_LABEL_SESSION_PATH: str = ""
    MONAI_LABEL_SESSION_EXPIRY: int = 3600
    
    # 任务开关
    MONAI_LABEL_TASKS_TRAIN: bool = True
    MONAI_LABEL_TASKS_STRATEGY: bool = True
    MONAI_LABEL_TASKS_SCORING: bool = True
    
    class Config:
        env_prefix = ""  # 环境变量前缀
        case_sensitive = True

# 全局设置实例
settings = Settings()
```

**环境变量示例**:
```bash
export MONAI_LABEL_APP_DIR=/code/apps/radiology
export MONAI_LABEL_STUDIES=http://orthanc:8042/dicom-web
export MONAI_LABEL_DICOMWEB_USERNAME=orthanc
export MONAI_LABEL_DICOMWEB_PASSWORD=orthanc
```

---

## endpoints/ API端点目录

### __init__.py
```python
# 导出所有端点路由器
from . import infer
from . import datastore
from . import info
from . import model
from . import session
from . import scoring
from . import ohif
```

### infer.py
**路径**: `monailabel/endpoints/infer.py`

**作用**: **推理API端点 - 核心文件**

**完整结构**:
```python
import json
import logging
from fastapi import APIRouter, File, Form, UploadFile, BackgroundTasks
from fastapi.responses import FileResponse, Response
from monailabel.interfaces.utils.app import app_instance

router = APIRouter(
    prefix="/infer",
    tags=["Infer"],
    responses={404: {"description": "Not found"}},
)

class ResultType(str, Enum):
    """返回结果类型枚举"""
    image = "image"       # 返回图像文件
    json = "json"         # 返回JSON
    all = "all"           # 返回所有
    dicom_seg = "dicom_seg"  # 返回DICOM-SEG

def send_response(datastore, result, output, background_tasks):
    """
    格式化并返回推理结果
    
    参数:
        result: {file: 结果文件路径, params: 元数据}
        output: 输出类型
    """
    res_img = result.get("file")
    res_json = result.get("params")
    
    if output == "json":
        return res_json
    elif output == "image":
        return FileResponse(res_img, ...)
    elif output == "dicom_seg":
        # 构建多部分响应
        fields = {
            "prompt_info": json.dumps(res_json.get("prompt_info")),
            "flipped": json.dumps(res_json.get("flipped")),
            ...
        }
        return stream_multipart(fields, res_img)

def run_inference(
    background_tasks: BackgroundTasks,
    model: str,                    # 模型名称
    image: str = "",               # 图像ID
    session_id: str = "",
    params: str = Form("{}"),      # JSON参数字符串
    file: UploadFile = File(None), # 上传的图像文件
    label: UploadFile = File(None),# 上传的标签文件
    output: Optional[ResultType] = None,
):
    """
    执行推理的主函数
    
    流程:
    1. 解析请求参数
    2. 获取图像路径
    3. 调用MONAILabelApp.infer()
    4. 处理返回结果
    """
    request = {"model": model, "image": image}
    
    # 处理上传的文件
    if file:
        file_ext = pathlib.Path(file.filename).suffixes
        image_file = tempfile.NamedTemporaryFile(suffix=file_ext).name
        shutil.copyfileobj(file.file, open(image_file, "wb"))
        request["image"] = image_file
        background_tasks.add_task(remove_file, image_file)
    
    # 解析params JSON
    p = json.loads(params) if params else {}
    request.update(p)
    
    # 获取应用实例并执行推理
    instance = app_instance()
    result = instance.infer(request)
    
    # 处理DICOM-SEG输出
    if output == "dicom_seg":
        # 转换为DICOM-SEG格式
        return convert_to_dicom_seg(result)
    
    return send_response(instance.datastore(), result, output, background_tasks)

@router.post("/{model}")
async def api_run_inference(...):
    """API端点包装器"""
    return run_inference(...)
```

**请求参数详解**:
```python
{
    "model": "segmentation",      # 模型名称
    "image": "series_uid",        # 图像ID
    "nninter": True,              # 使用nnInteractive
    "medsam2": "",                # SAM2变体 ("", "medsam2", "sam3")
    "pos_points": [[x,y,z], ...], # 正点提示
    "neg_points": [[x,y,z], ...], # 负点提示
    "pos_boxes": [[[x1,y1,z1],[x2,y2,z2]], ...],  # 正边界框
    "pos_scribbles": [[x,y,z], ...],  # 正涂鸦
    "texts": ["liver"],           # 文本提示 (VoxTell)
    "instruction": "...",         # MedGemma指令
    "startSlice": 10,             # MedGemma起始切片
    "endSlice": 50,               # MedGemma结束切片
}
```

**响应格式**:
```python
# image输出:
FileResponse("/path/to/result.nii.gz")

# json输出:
{
    "prompt_info": {...},        # 提示信息
    "flipped": False,            # 是否翻转
    "nninter_elapsed": 1.23,     # 推理耗时
    "label_name": "nninter_pred_20240101"
}

# dicom_seg输出 (multipart/form-data):
--boundary
Content-Type: application/json
{metadata}
--boundary
Content-Type: application/octet-stream
[binary data]
```

### datastore.py
**路径**: `monailabel/endpoints/datastore.py`

**作用**: 数据存储管理API

```python
router = APIRouter(prefix="/datastore", tags=["Datastore"])

@router.get("/", summary="获取数据存储状态")
async def api_datastore_status(user: User = Depends(...)):
    instance = app_instance()
    return instance.datastore().status()

@router.get("/images", summary="获取图像列表")
async def api_datastore_images(...):
    ...

@router.put("/label", summary="保存标签")
async def api_save_label(...):
    ...
```

### info.py
**路径**: `monailabel/endpoints/info.py`

**作用**: 应用信息API

```python
@router.get("/", summary="获取应用信息")
async def api_info(user: User = Depends(...)):
    instance = app_instance()
    return instance.info()
    
# 返回示例:
{
    "name": "Radiology App",
    "description": "...",
    "version": "2.0",
    "labels": ["liver", "spleen", ...],
    "models": {...},
    "trainers": {...},
    "datastore": {...}
}
```

### session.py
**路径**: `monailabel/endpoints/session.py`

**作用**: 会话管理API

```python
@router.post("/", summary="创建会话")
async def api_create_session(...) -> Session:
    """创建新的推理会话"""
    
@router.get("/{session_id}", summary="获取会话")
async def api_get_session(session_id: str) -> Session:
    """获取会话信息"""
```

---

## interfaces/ 接口定义目录

### app.py
**路径**: `monailabel/interfaces/app.py`

**作用**: **MONAILabelApp基类 - 应用框架核心**

**类定义**:
```python
class MONAILabelApp:
    """
    MONAI Label应用基类
    所有自定义应用都应继承此类
    """
    
    PRE_TRAINED_PATH: str = "https://github.com/Project-MONAI/MONAILabel/releases/download/data"
    
    def __init__(
        self,
        app_dir: str,              # 应用目录
        studies: str,              # 研究数据路径
        conf: Dict[str, str],      # 配置字典
        name: str = "",
        description: str = "",
        version: str = "2.0",
        labels: Union[Sequence[str], Dict] = None,
    ):
        self.app_dir = app_dir
        self.studies = studies
        self.conf = conf
        
        # 初始化各组件
        self._datastore = self.init_datastore()
        self._infers = self.init_infers()
        self._trainers = self.init_trainers()
        self._strategies = self.init_strategies()
        self._scoring_methods = self.init_scoring_methods()
        self._sessions = self._load_sessions()
        self._infers_threadpool = ThreadPoolExecutor(...)
    
    # ===== 抽象方法，子类必须实现 =====
    def init_infers(self) -> Dict[str, InferTask]:
        """初始化推理任务，返回任务字典"""
        return {}
    
    def init_trainers(self) -> Dict[str, TrainTask]:
        """初始化训练任务"""
        return {}
    
    def init_strategies(self) -> Dict[str, Strategy]:
        """初始化主动学习策略"""
        return {"random": Random()}
    
    def init_scoring_methods(self) -> Dict[str, ScoringMethod]:
        """初始化评分方法"""
        return {}
    
    def init_datastore(self) -> Datastore:
        """初始化数据存储"""
        if self.studies.startswith("http://"):
            return self.init_remote_datastore()
        return LocalDatastore(self.studies, ...)
    
    # ===== 核心方法 =====
    def infer(self, request, datastore=None) -> Dict:
        """
        执行推理
        
        参数:
            request: {
                "model": "model_name",
                "image": "image_id",
                "param1": value1,
                ...
            }
        
        返回:
            {"file": 结果文件, "params": 元数据}
        """
        model = request.get("model")
        task = self._infers.get(model)
        
        # 深拷贝并添加描述
        request = copy.deepcopy(request)
        request["description"] = task.description
        
        # 获取图像URI
        image_id = request["image"]
        request["image"] = datastore.get_image_uri(image_id)
        
        # 执行推理
        if self._infers_threadpool:
            future = self._infers_threadpool.submit(task, request)
            result_file, result_json = future.result(timeout=...)
        else:
            result_file, result_json = task(request)
        
        return {"file": result_file, "params": result_json}
    
    def train(self, request) -> Dict:
        """执行训练"""
        model = request.get("model")
        task = self._trainers.get(model)
        result = task(request, self.datastore())
        return result
    
    def info(self) -> Dict:
        """获取应用信息"""
        return {
            "name": self.name,
            "description": self.description,
            "version": self.version,
            "models": {k: v.info() for k, v in self._infers.items()},
            "trainers": {k: v.info() for k, v in self._trainers.items()},
            "datastore": self._datastore.status(),
        }
    
    def on_init_complete(self):
        """初始化完成回调"""
        logger.info("App Init - completed")
        # 启动定时任务
        self._start_cleanup_scheduler()
```

### datastore.py
**路径**: `monailabel/interfaces/datastore.py`

**作用**: 数据存储接口定义

```python
class Datastore(ABC):
    """数据存储抽象基类"""
    
    @abstractmethod
    def get_image_uri(self, image_id: str) -> str:
        """获取图像文件路径"""
        pass
    
    @abstractmethod
    def get_label_uri(self, label_id: str, tag: str) -> str:
        """获取标签文件路径"""
        pass
    
    @abstractmethod
    def list_images(self) -> List[str]:
        """列出所有图像"""
        pass
    
    @abstractmethod
    def save_label(self, image_id: str, label_filename: str, label_tag: str, info: Dict) -> str:
        """保存标签"""
        pass
    
    def status(self) -> Dict:
        """获取存储状态"""
        return {
            "total": len(self.list_images()),
            "completed": len(self.list_labels()),
        }
```

---

## tasks/infer/ 推理任务目录

### basic_infer.py
**路径**: `monailabel/tasks/infer/basic_infer.py`

**作用**: **基础推理任务 - 最重要的实现文件**

**文件结构**:
```python
# ===== 全局模型初始化 (模块级别) =====

# nnInteractive
REPO_ID = "nnInteractive/nnInteractive"
MODEL_NAME = "nnInteractive_v1.0"
session = nnInteractiveInferenceSession(device=torch.device("cuda:0"), ...)
session.initialize_from_trained_model_folder(model_path)

# SAM2
predictor_sam2 = build_sam2_video_predictor(model_cfg, sam2_checkpoint)

# MedSAM2
predictor_med = build_sam2_video_predictor_npz(medsam2_model_cfg, medsam2_checkpoint)

# SAM3 (条件初始化)
if os.path.exists(sam3_checkpoint):
    predictor_sam3 = build_sam3_video_model(...)

# VoxTell
vox_predictor = VoxTellPredictor(model_dir=vox_model_path, ...)

# MedGemma
gem_processor = transformers.AutoProcessor.from_pretrained(gem_model_id, ...)
gem_model = transformers.AutoModelForImageTextToText.from_pretrained(...)

# ===== BasicInferTask类定义 =====

class BasicInferTask(InferTask):
    """基础推理任务实现"""
    
    def __init__(self, path, network, type, labels, dimension, description, ...):
        super().__init__(type, labels, dimension, description, config)
        self.path = path
        self.network = network
        self._session_used_interactions = {...}  # 提示去重记录
    
    def __call__(self, request, callbacks=None):
        """
        主推理入口
        
        参数:
            request: 包含所有提示参数的字典
            callbacks: 回调函数字典
        
        返回:
            (result_file, result_json)
        """
        # 合并配置
        req = copy.deepcopy(self._config)
        req.update(request)
        device = name_to_device(req.get("device", "cuda"))
        
        # 解析DICOM信息
        dicom_dir = data['image'].split('.nii.gz')[0]
        dcm_img_sample = dcmread(dicom_filenames[0])
        instanceNumber = dcm_img_sample[0x00200013].value
        modality = dcm_img_sample[0x00080060].value
        contrast_center = dcm_img_sample[0x00281050].value
        contrast_window = dcm_img_sample[0x00281051].value
        
        # 加载图像
        reader = sitk.ImageSeriesReader()
        reader.SetFileNames(dicom_filenames)
        img = reader.Execute()
        img_np = sitk.GetArrayFromImage(img)[None]  # (1, Z, Y, X)
        
        # ===== 模型分支选择 =====
        
        if nnInter == "reset":
            # 重置nnInteractive会话
            session.reset_interactions()
            return f'/code/predictions/reset.nii.gz', {}
        
        if nnInter == "init":
            # 初始化nnInteractive
            session.set_image(img_np)
            session.set_target_buffer(torch.zeros(img_np.shape[1:], dtype=torch.uint8))
            return f'/code/predictions/init.nii.gz', {}
        
        if nnInter == "medGemma":
            # MedGemma报告生成分支
            return self._medgemma_infer(img_np, data, modality_type)
        
        if nnInter and len(data['texts']) == 1 and data['texts'][0]:
            # VoxTell文本提示分支
            return self._voxtell_infer(img, data)
        
        if nnInter:
            # nnInteractive推理分支
            return self._nninteractive_infer(img_np, data, instanceNumber, instanceNumber2)
        
        if not nnInter:
            # SAM2/3推理分支
            return self._sam_infer(img, data, contrast_center, contrast_window)
    
    def _nninteractive_infer(self, img_np, data, instanceNumber, instanceNumber2):
        """nnInteractive推理实现"""
        
        # 检查并添加点提示
        for point in data['pos_points']:
            if not self.is_prompt_used(point, "pos_points"):
                self.add_prompt(point, "pos_points")
                # 处理Z轴翻转
                if instanceNumber > instanceNumber2:
                    point[2] = img_np.shape[1] - 1 - point[2]
                # 坐标转换: [x,y,z] -> [z,y,x]
                session.add_point_interaction(tuple(point[::-1]), include_interaction=True)
        
        # 检查并添加涂鸦提示
        for scribble in data['pos_scribbles']:
            if not self.is_prompt_used(scribble, "pos_scribbles"):
                self.add_prompt(scribble, "pos_scribbles")
                scribble = clean_and_densify_polyline(scribble)
                scribbleMask = self._create_scribble_mask(scribble, img_np.shape[1:])
                session.add_scribble_interaction(scribbleMask, include_interaction=True)
        
        # 检查并添加套索提示
        for lasso in data['pos_lassos']:
            if not self.is_prompt_used(lasso, "pos_lassos"):
                self.add_prompt(lasso, "pos_lassos")
                lasso = get_scanline_filled_points_3d(clean_and_densify_polyline(lasso))
                lassoMask = self._create_lasso_mask(lasso, img_np.shape[1:])
                session.add_lasso_interaction(lassoMask, include_interaction=True)
        
        # 检查并添加边界框提示
        for box in data['pos_boxes']:
            if not self.is_prompt_used(box, "pos_boxes"):
                self.add_prompt(box, "pos_boxes")
                # 坐标转换
                box[0] = box[0][::-1]
                box[1] = box[1][::-1]
                session.add_bbox_interaction([[box[0][0], box[1][0]], ...], include_interaction=True)
        
        # 获取结果
        results = session.target_buffer.clone()
        pred = results.numpy()
        
        return pred, {
            "prompt_info": {...},
            "nninter_elapsed": elapsed_time,
            "flipped": instanceNumber > instanceNumber2
        }
    
    def _sam_infer(self, img, data, contrast_center, contrast_window):
        """SAM2/3推理实现"""
        
        # 选择预测器
        if data['medsam2'] == 'medsam2':
            predictor = predictor_med
        elif data['medsam2'] == 'sam3':
            predictor = predictor_sam3
        else:
            predictor = predictor_sam2
        
        # 初始化状态
        with torch.inference_mode(), torch.autocast("cuda", dtype=torch.bfloat16):
            inference_state = predictor.init_state(
                video_path=img,
                clip_low=contrast_center - contrast_window/2,
                clip_high=contrast_center + contrast_window/2
            )
        
        # 收集唯一帧索引
        ann_frame_list = np.unique([p[2] for p in data['pos_points']])
        
        # 在每帧添加提示
        for frame_idx in ann_frame_list:
            # 收集该帧的点
            pos_points = np.array([p[:2] for p in data['pos_points'] if p[2] == frame_idx])
            neg_points = np.array([p[:2] for p in data['neg_points'] if p[2] == frame_idx])
            
            points = np.concatenate((pos_points, neg_points))
            labels = np.array([1]*len(pos_points) + [0]*len(neg_points))
            
            with torch.inference_mode():
                predictor.add_new_points_or_box(
                    inference_state=inference_state,
                    frame_idx=frame_idx,
                    obj_id=1,
                    points=points,
                    labels=labels
                )
        
        # 在视频中传播
        video_segments = {}
        with torch.inference_mode():
            for out_frame_idx, out_obj_ids, out_mask_logits in predictor.propagate_in_video(
                inference_state, start_frame_idx=0
            ):
                video_segments[out_frame_idx] = {
                    obj_id: (mask_logits > 0.0).cpu().numpy()
                    for obj_id, mask_logits in zip(out_obj_ids, out_mask_logits)
                }
        
        # 组合结果
        pred = np.zeros((len_z, len_y, len_x))
        for i in video_segments.keys():
            pred[i] = video_segments[i][1][0].astype(int)
        
        return pred, {"sam_elapsed": elapsed_time, ...}
    
    def _voxtell_infer(self, img, data):
        """VoxTell文本提示推理"""
        
        # 转换到RAS方向
        orig_orient = sitk.DICOMOrientImageFilter_GetOrientationFromDirectionCosines(
            img.GetDirection()
        )
        img_ras = sitk.DICOMOrient(img, "RAS")
        img_np = sitk.GetArrayFromImage(img_ras)[None]
        
        # 执行推理
        voxtell_seg_np_ras = vox_predictor.predict_single_image(
            img_np, data['texts'][0]
        )
        
        # 转回原始方向
        voxtell_seg_sitk_ras = sitk.GetImageFromArray(voxtell_seg_np_ras[0])
        voxtell_seg_sitk_ras.CopyInformation(img_ras)
        voxtell_seg_sitk = sitk.DICOMOrient(voxtell_seg_sitk_ras, orig_orient)
        voxtell_seg_np = sitk.GetArrayFromImage(voxtell_seg_sitk)
        
        return voxtell_seg_np, {"voxtell_elapsed": elapsed_time, ...}
    
    def _medgemma_infer(self, img_np, data, modality_type):
        """MedGemma报告生成"""
        
        # 获取切片范围
        num_slices = img_np.shape[0]
        start_idx = data.get('startSlice', 1) - 1
        end_idx = data.get('endSlice', num_slices)
        slice_indices = list(range(start_idx, end_idx))
        
        # 窗宽窗位处理
        if modality_type == "CT":
            normalized_slices = [window(slice_) for slice_ in slices]
        else:
            normalized_slices = [window_mri(slice_) for slice_ in slices]
        
        # 构建消息
        instruction = data.get('instruction', default_instruction)
        query = data['texts'][0]
        
        content = [{"type": "text", "text": instruction}]
        for slice_idx, slice_img in zip(slice_indices, normalized_slices):
            content.append({"type": "image", "image": _encode(slice_img)})
            content.append({"type": "text", "text": f"SLICE {slice_idx + 1}"})
        content.append({"type": "text", "text": query})
        
        messages = [{"role": "user", "content": content}]
        
        # 生成
        inputs = gem_processor.apply_chat_template(messages, ...)
        with torch.inference_mode():
            generated = gem_model.generate(**inputs, max_new_tokens=2000)
        
        response = gem_processor.post_process_image_text_to_text(generated)
        
        return response[0], {}
```

---

## utils/others/ 工具函数目录

### helper.py
**路径**: `monailabel/utils/others/helper.py`

**作用**: **关键辅助函数**

```python
def get_scanline_filled_points_3d(polyline):
    """
    使用扫描线算法填充3D多边形
    
    参数:
        polyline: Nx3数组，多边形顶点
    
    返回:
        filled_points: Mx3数组，填充后的点
    """
    # 投影到XY平面
    xy_points = polyline[:, :2]
    z = int(polyline[0, 2])
    
    # 计算边界框
    min_x, min_y = xy_points.min(axis=0)
    max_x, max_y = xy_points.max(axis=0)
    
    filled_points = []
    for y in range(int(min_y), int(max_y) + 1):
        # 找到扫描线与多边形边的交点
        intersections = []
        for i in range(len(xy_points)):
            p1 = xy_points[i]
            p2 = xy_points[(i + 1) % len(xy_points)]
            
            if (p1[1] <= y < p2[1]) or (p2[1] <= y < p1[1]):
                x = p1[0] + (y - p1[1]) * (p2[0] - p1[0]) / (p2[1] - p1[1])
                intersections.append(x)
        
        # 排序并填充
        intersections.sort()
        for i in range(0, len(intersections), 2):
            if i + 1 < len(intersections):
                for x in range(int(intersections[i]), int(intersections[i + 1]) + 1):
                    filled_points.append([x, y, z])
    
    return np.array(filled_points)

def clean_and_densify_polyline(points, spacing=1.0):
    """
    清理并致密化多边形路径
    
    1. 去重
    2. 线性插值填充间隙
    """
    points = np.array(points)
    
    # 去重
    _, unique_indices = np.unique(points, axis=0, return_index=True)
    points = points[np.sort(unique_indices)]
    
    # 插值
    dense_points = [points[0]]
    for i in range(1, len(points)):
        prev, curr = points[i-1], points[i]
        dist = np.linalg.norm(curr - prev)
        
        if dist > spacing:
            num_points = int(dist / spacing)
            for t in np.linspace(0, 1, num_points + 1)[1:]:
                dense_points.append(prev + t * (curr - prev))
        else:
            dense_points.append(curr)
    
    return np.array(dense_points)

def spherical_kernel(radius=1):
    """生成球形结构元素"""
    size = 2 * radius + 1
    kernel = np.zeros((size, size, size), dtype=np.uint8)
    center = radius
    
    for z in range(size):
        for y in range(size):
            for x in range(size):
                dist = np.sqrt((x-center)**2 + (y-center)**2 + (z-center)**2)
                if dist <= radius:
                    kernel[z, y, x] = 1
    
    return kernel
```

### medgemma.py
**路径**: `monailabel/utils/others/medgemma.py`

**作用**: **MedGemma工具函数**

```python
def window(ct_slice, window_center=40, window_width=400):
    """
    CT窗宽窗位处理
    
    默认: 腹部窗 (WL=40, WW=400)
    """
    min_val = window_center - window_width // 2
    max_val = window_center + window_width // 2
    
    windowed = np.clip(ct_slice, min_val, max_val)
    windowed = (windowed - min_val) / window_width * 255
    
    return windowed.astype(np.uint8)

def window_mri(mr_slice, min_val=None, max_val=None):
    """
    MRI窗宽窗位处理
    """
    if min_val is None:
        min_val = mr_slice.min()
        max_val = mr_slice.max()
    
    windowed = np.clip(mr_slice, min_val, max_val)
    windowed = (windowed - min_val) / (max_val - min_val) * 255
    
    return windowed.astype(np.uint8)

def _encode(image_array):
    """
    将numpy数组编码为base64字符串
    
    用于MedGemma的图像输入
    """
    import base64
    from PIL import Image
    import io
    
    pil_image = Image.fromarray(image_array)
    buffer = io.BytesIO()
    pil_image.save(buffer, format="PNG")
    img_str = base64.b64encode(buffer.getvalue()).decode()
    
    return img_str
```

---

## sample-apps/radiology/ 示例应用

### lib/configs/segmentation.py
**路径**: `sample-apps/radiology/lib/configs/segmentation.py`

**作用**: 分割任务配置

```python
class Segmentation(MyInferTask):
    """放射学分割任务"""
    
    def __init__(self):
        super().__init__(
            path=None,
            network=None,
            type=InferType.SEGMENTATION,
            labels=None,
            dimension=3,
            description="Radiology Segmentation using AI models",
        )
```

### main.py
**路径**: `sample-apps/radiology/main.py`

**作用**: 放射学应用入口

```python
from monailabel.interfaces.app import MONAILabelApp

class MyApp(MONAILabelApp):
    def __init__(self, app_dir, studies, conf):
        super().__init__(app_dir, studies, conf, ...)
    
    def init_infers(self):
        """初始化推理任务"""
        return {
            "segmentation": Segmentation(),
        }
```

---

*本指南详细说明了后端各文件的作用和实现细节，更多内容请参考具体源码*
