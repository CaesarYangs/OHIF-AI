# OHIF-AI 代码实现原理详解

## 目录
1. [前端交互实现](#1-前端交互实现)
2. [后端推理实现](#2-后端推理实现)
3. [模型集成原理](#3-模型集成原理)
4. [数据处理流程](#4-数据处理流程)
5. [通信协议](#5-通信协议)

---

## 1. 前端交互实现

### 1.1 工具激活系统

前端使用Cornerstone3D Tools管理交互工具。AI提示工具的实现基于标准的Cornerstone工具系统：

```typescript
// 工具配置示例 (toolbarButtons.ts)
{
  id: 'Probe2',  // 点提示工具
  label: 'Point Prompt',
  icon: 'icon-tool-point',
  commands: [
    {
      commandName: 'setToolActive',
      commandOptions: {
        toolName: 'Probe',  // 使用Cornerstone的Probe工具
        toolGroupId: 'default'
      }
    }
  ]
}
```

### 1.2 提示收集与发送

当用户在影像上交互时，系统收集提示数据并发送到后端：

```typescript
// 伪代码示例
class AIPromptTool {
  // 收集点提示
  collectPointPrompts() {
    const points = this.pointToolData.map(p => ({
      x: p.canvasCoords[0],
      y: p.canvasCoords[1],
      z: this.currentSliceIndex
    }));
    return points;
  }
  
  // 发送推理请求
  async runInference() {
    const requestData = {
      model: this.selectedModel,  // 'nninter', 'sam2', etc.
      image: this.currentImageId,
      nninter: this.selectedModel === 'nninter',
      medsam2: this.selectedModel === 'medsam2' ? 'medsam2' : 
               this.selectedModel === 'sam3' ? 'sam3' : '',
      pos_points: this.positivePoints,
      neg_points: this.negativePoints,
      pos_boxes: this.positiveBoxes,
      pos_scribbles: this.scribblePoints,
      texts: this.textPrompts
    };
    
    const response = await fetch('/monai/infer/segmentation', {
      method: 'POST',
      body: JSON.stringify(requestData)
    });
    
    // 加载返回的分割结果
    const segmentation = await response.blob();
    this.loadSegmentation(segmentation);
  }
}
```

### 1.3 分割结果显示

返回的分割数据以DICOM-SEG格式加载到Cornerstone：

```typescript
// 加载分割到视口
const loadSegmentation = async (segBlob) => {
  // 创建显示集
  const displaySet = await createSegmentationDisplaySet(segBlob);
  
  // 添加到分割服务
  segmentationService.addSegmentation(displaySet);
  
  // 在视口中显示
  await segmentationService.addSegmentationRepresentation(
    viewportId,
    displaySet.id
  );
};
```

---

## 2. 后端推理实现

### 2.1 FastAPI应用生命周期

```python
# app.py
@asynccontextmanager
async def lifespan(app: FastAPI):
    # 应用启动时初始化
    print("App Init...")
    instance = app_instance()
    instance.server_mode(True)
    instance.on_init_complete()
    
    yield
    
    # 应用关闭时清理
    print("App Shutdown...")
```

### 2.2 MONAILabelApp核心类

```python
# interfaces/app.py
class MONAILabelApp:
    def __init__(self, app_dir, studies, conf, ...):
        # 初始化数据存储
        self._datastore = self.init_datastore()
        
        # 初始化推理任务
        self._infers = self.init_infers()
        
        # 初始化训练任务
        self._trainers = self.init_trainers()
        
        # 初始化会话管理
        self._sessions = self._load_sessions()
    
    def infer(self, request, datastore=None):
        """执行推理"""
        model = request.get("model")
        task = self._infers.get(model)
        
        # 深拷贝请求并添加描述
        request = copy.deepcopy(request)
        request["description"] = task.description
        
        # 获取图像路径
        image_id = request["image"]
        request["image"] = datastore.get_image_uri(image_id)
        
        # 执行推理
        result_file_name, result_json = task(request)
        
        return {"file": result_file_name, "params": result_json}
```

### 2.3 BasicInferTask推理流程

`BasicInferTask`是所有推理任务的基类，实现了标准化的推理流水线：

```
┌─────────────────────────────────────────────────────────┐
│                  BasicInferTask.__call__                 │
└─────────────────────────────────────────────────────────┘
                           │
    ┌──────────────────────┼──────────────────────┐
    ▼                      ▼                      ▼
┌─────────┐          ┌─────────┐           ┌───────────┐
│ Pre     │          │ Infer   │           │ Post      │
│ Transforms│        │ Model   │           │ Transforms│
└────┬────┘          └────┬────┘           └─────┬─────┘
     │                    │                       │
     ▼                    ▼                       ▼
数据预处理              模型推理               结果后处理
(加载、归一化)        (神经网络前向传播)      (阈值、形态学)
```

**代码实现**:

```python
def __call__(self, request, callbacks=None):
    # 合并配置
    req = copy.deepcopy(self._config)
    req.update(request)
    
    # 设置设备
    device = name_to_device(req.get("device", "cuda"))
    req["device"] = device
    
    # ===== 预处理 =====
    data = copy.deepcopy(req)
    data.update({"image_path": req.get("image")})
    
    # 执行预处理变换
    transforms = self.pre_transforms(data)
    data = self.run_pre_transforms(data, transforms)
    
    if callback_run_pre_transforms:
        data = callback_run_pre_transforms(data)
    
    # ===== 推理 =====
    inferer = self.inferer(data)
    network = self._get_network(device, data)
    
    inputs = data[self.input_key]
    inputs = inputs if torch.is_tensor(inputs) else torch.from_numpy(inputs)
    inputs = inputs[None]  # 添加batch维度
    inputs = inputs.to(torch.device(device))
    
    with torch.no_grad():
        outputs = inferer(inputs, network)
    
    data[self.output_label_key] = outputs
    
    # ===== 后处理 =====
    post_transforms = self.post_transforms(data)
    data = self.run_post_transforms(data, post_transforms)
    
    # ===== 写入结果 =====
    file, json_data = self.writer(data)
    
    return file, json_data
```

---

## 3. 模型集成原理

### 3.1 nnInteractive集成

nnInteractive是3D交互式分割的核心模型。

**初始化**:
```python
from nnInteractive.inference.inference_session import nnInteractiveInferenceSession

session = nnInteractiveInferenceSession(
    device=torch.device("cuda:0"),
    use_torch_compile=False,
    verbose=False,
    torch_n_threads=os.cpu_count(),
    do_autozoom=True,  # 启用AutoZoom
    use_pinned_memory=True,
)

model_path = os.path.join(DOWNLOAD_DIR, "nnInteractive_v1.0")
session.initialize_from_trained_model_folder(model_path)
```

**交互处理**:
```python
# 设置图像 (仅第一次或图像变化时)
if seriesInstanceUID != self._session_image["seriesInstanceUID"]:
    session.set_image(img_np)
    session.set_target_buffer(torch.zeros(img_np.shape[1:], dtype=torch.uint8))

# 添加点提示
for point in data['pos_points']:
    # 坐标转换: [x, y, z] -> [z, y, x] (numpy到nnInteractive)
    point_coords = tuple(point[::-1])
    session.add_point_interaction(point_coords, include_interaction=True)

# 添加涂鸦提示
for scribble in data['pos_scribbles']:
    # 创建3D掩码
    scribbleMask = np.zeros(img_np.shape[1:], dtype=np.uint8)
    # 使用球形核填充涂鸦路径
    kernel = spherical_kernel(radius=1)
    for x, y, z in scribble:
        scribbleMask[z, y, x] = 1
    session.add_scribble_interaction(scribbleMask, include_interaction=True)

# 添加套索提示
for lasso in data['pos_lassos']:
    # 扫描线填充算法
    filled_points = get_scanline_filled_points_3d(lasso)
    lassoMask = np.zeros(img_np.shape[1:], dtype=np.uint8)
    lassoMask[filled_points[:, 2], filled_points[:, 1], filled_points[:, 0]] = 1
    session.add_lasso_interaction(lassoMask, include_interaction=True)

# 获取结果
results = session.target_buffer.clone()
pred = results.numpy()
```

### 3.2 SAM2/MedSAM2/SAM3集成

SAM2系列模型使用视频跟踪方式处理3D医学影像。

**初始化**:
```python
from sam2.build_sam import build_sam2_video_predictor

# SAM2
predictor_sam2 = build_sam2_video_predictor(
    "configs/sam2.1/sam2.1_hiera_t.yaml",
    "/code/checkpoints/sam2.1_hiera_tiny.pt"
)

# MedSAM2 (使用NPZ格式)
predictor_med = build_sam2_video_predictor_npz(
    "configs/sam2.1/sam2.1_hiera_t512.yaml",
    "/code/checkpoints/MedSAM2_latest.pt"
)

# SAM3 (可选)
from sam3.model_builder import build_sam3_video_model
if os.path.exists(sam3_checkpoint):
    sam3_model = build_sam3_video_model(checkpoint_path=sam3_checkpoint)
    predictor_sam3 = sam3_model.tracker
```

**推理流程**:
```python
# 1. 初始化状态
with torch.inference_mode(), torch.autocast("cuda", dtype=torch.bfloat16):
    inference_state = predictor.init_state(
        video_path=img,
        clip_low=contrast_center - contrast_window/2,
        clip_high=contrast_center + contrast_window/2
    )

# 2. 在指定帧添加提示
for frame_idx in ann_frame_list:
    # 收集该帧的所有点提示
    pos_points = np.array([p for p in result_json['pos_points'] if p[2] == frame_idx])
    neg_points = np.array([p for p in result_json['neg_points'] if p[2] == frame_idx])
    
    # 合并正负点
    points = np.concatenate((pos_points, neg_points), axis=0)
    labels = np.array([1]*len(pos_points) + [0]*len(neg_points))
    
    # 添加提示
    _, out_obj_ids, out_mask_logits = predictor.add_new_points_or_box(
        inference_state=inference_state,
        frame_idx=frame_idx,
        obj_id=1,
        points=points,
        labels=labels,
        box=boxes if len(boxes) > 0 else None
    )

# 3. 在视频中传播
video_segments = {}
with torch.inference_mode(), torch.autocast("cuda", dtype=torch.bfloat16):
    for out_frame_idx, out_obj_ids, out_mask_logits in predictor.propagate_in_video(
        inference_state,
        start_frame_idx=0,
        reverse=False
    ):
        video_segments[out_frame_idx] = {
            obj_id: (mask_logits > 0.0).cpu().numpy()
            for obj_id, mask_logits in zip(out_obj_ids, out_mask_logits)
        }

# 4. 组合结果
pred = np.zeros((len_z, len_y, len_x))
for i in video_segments.keys():
    pred[i] = video_segments[i][1][0].astype(int)
```

### 3.3 VoxTell集成

VoxTell实现文本提示分割。

**初始化**:
```python
from voxtell.inference.predictor import VoxTellPredictor

vox_predictor = VoxTellPredictor(
    model_dir=vox_model_path,
    device=torch.device("cuda:0")
)
```

**推理**:
```python
# 转换为RAS方向 (VoxTell要求)
orig_orient = sitk.DICOMOrientImageFilter_GetOrientationFromDirectionCosines(
    img.GetDirection()
)
img_ras = sitk.DICOMOrient(img, "RAS")
img_np = sitk.GetArrayFromImage(img_ras)[None]  # 添加通道维度

# 执行文本提示分割
voxtell_seg_np_ras = vox_predictor.predict_single_image(
    img_np, 
    data['texts'][0]  # 文本提示，如 "liver"
)

# 转回原始方向
voxtell_seg_sitk_ras = sitk.GetImageFromArray(voxtell_seg_np_ras[0])
voxtell_seg_sitk_ras.CopyInformation(img_ras)
voxtell_seg_sitk = sitk.DICOMOrient(voxtell_seg_sitk_ras, orig_orient)
```

### 3.4 MedGemma集成

MedGemma用于生成放射学报告。

**初始化**:
```python
import transformers

gem_model_id = "google/medgemma-1.5-4b-it"

gem_model_kwargs = dict(
    dtype=torch.bfloat16,
    device_map="auto",  # 自动分配设备
    offload_buffers=True,
)

gem_processor = transformers.AutoProcessor.from_pretrained(
    gem_model_id, 
    use_fast=True,
    **gem_model_kwargs
)

gem_model = transformers.AutoModelForImageTextToText.from_pretrained(
    gem_model_id,
    **gem_model_kwargs
)
```

**报告生成**:
```python
if nnInter == "medGemma":
    query = data['texts'][0]
    instruction = data['instruction'] or "You are a radiology assistant..."
    
    # 获取切片范围
    start_slice = data.get('startSlice', 1)
    end_slice = data.get('endSlice', num_axial_slices)
    
    # 窗宽窗位处理
    if modality_type == "CT":
        normalized_slices = [window(ct_slice) for ct_slice in slices]
    else:
        normalized_slices = [window_mri(mr_slice, min_val, max_val) for mr_slice in slices]
    
    # 构建消息
    content = [{"type": "text", "text": instruction}]
    for slice_idx, ct_slice in zip(slice_indices, normalized_slices):
        content.append({"type": "image", "image": _encode(ct_slice)})
        content.append({"type": "text", "text": f"SLICE {slice_idx + 1}"})
    content.append({"type": "text", "text": query})
    
    messages = [{"role": "user", "content": content}]
    
    # 生成报告
    inputs = gem_processor.apply_chat_template(
        messages,
        add_generation_prompt=True,
        return_tensors="pt",
        tokenize=True,
        return_dict=True,
    )
    
    with torch.inference_mode():
        inputs = inputs.to(gem_model.device, dtype=torch.bfloat16)
        generated_sequence = gem_model.generate(
            **inputs,
            do_sample=False,  # 确定性输出
            max_new_tokens=2000
        )
    
    # 解码响应
    medgemma_response = gem_processor.post_process_image_text_to_text(
        generated_sequence,
        skip_special_tokens=True
    )
    
    return medgemma_response[0]
```

---

## 4. 数据处理流程

### 4.1 DICOM到NIfTI转换

```python
import SimpleITK as sitk

# 读取DICOM序列
reader = sitk.ImageSeriesReader()
dicom_filenames = reader.GetGDCMSeriesFileNames(dicom_dir)
reader.SetFileNames(dicom_filenames)
img = reader.Execute()

# 转换为numpy数组
img_np = sitk.GetArrayFromImage(img)
# 结果形状: (Z, Y, X)
```

### 4.2 坐标系统转换

医学影像涉及多个坐标系统：

```
DICOM坐标 (LPS) → NumPy索引 (ZYX) → 模型坐标
     │                   │              │
     │     方向矩阵        │    轴交换     │
     └──────────────────▶ └─────────────▶
```

**关键代码**:
```python
# DICOM方向检查
orig_orient = sitk.DICOMOrientImageFilter_GetOrientationFromDirectionCosines(
    img.GetDirection()
)

# 对于某些模型需要RAS方向
img_ras = sitk.DICOMOrient(img, "RAS")

# InstanceNumber方向检查 (判断Z轴是否需要翻转)
dcm_img_sample = dcmread(dicom_filenames[0])
dcm_img_sample_2 = dcmread(dicom_filenames[1])
instanceNumber = dcm_img_sample[0x00200013].value
instanceNumber2 = dcm_img_sample_2[0x00200013].value

if instanceNumber > instanceNumber2:
    # Z轴需要翻转
    point[2] = img_np.shape[1] - 1 - point[2]
```

### 4.3 窗宽窗位计算

**CT影像**:
```python
def window(ct_slice, window_center=40, window_width=400):
    """标准腹部CT窗宽窗位"""
    min_val = window_center - window_width // 2
    max_val = window_center + window_width // 2
    windowed = np.clip(ct_slice, min_val, max_val)
    windowed = (windowed - min_val) / window_width * 255
    return windowed.astype(np.uint8)
```

**MRI影像**:
```python
def window_mri(mr_slice, min_val=None, max_val=None):
    """MRI使用DICOM元数据或自动窗宽窗位"""
    if min_val is None or max_val is None:
        # 自动计算
        min_val = mr_slice.min()
        max_val = mr_slice.max()
    
    windowed = np.clip(mr_slice, min_val, max_val)
    windowed = (windowed - min_val) / (max_val - min_val) * 255
    return windowed.astype(np.uint8)
```

---

## 5. 通信协议

### 5.1 API端点

**推理请求**:
```
POST /monai/infer/{model}
Content-Type: multipart/form-data

参数:
- model: 模型名称 (segmentation)
- image: 图像ID
- params: JSON字符串，包含提示信息
- output: 输出格式 (image, json, dicom_seg)
```

**请求参数示例**:
```json
{
  "nninter": true,
  "medsam2": "",
  "pos_points": [[100, 150, 50], [120, 160, 50]],
  "neg_points": [],
  "pos_boxes": [],
  "neg_boxes": [],
  "pos_lassos": [],
  "neg_lassos": [],
  "pos_scribbles": [],
  "neg_scribbles": [],
  "texts": [""],
  "instruction": "",
  "startSlice": 10,
  "endSlice": 50
}
```

**响应格式**:

对于 `output=dicom_seg`:
```
Content-Type: multipart/form-data; boundary=monai-{random}

--monai-{random}
Content-Type: application/json

{
  "prompt_info": {...},
  "flipped": false,
  "nninter_elapsed": 1.23,
  "label_name": "nninter_pred_20240101"
}

--monai-{random}
Content-Type: application/octet-stream

[Binary DICOM-SEG data]
--monai-{random}--
```

### 5.2 DICOM-SEG格式

系统使用DICOM-SEG格式传输分割结果：

```python
def read_seg_file(seg):
    """读取DICOM-SEG文件"""
    num_frames = seg.NumberOfFrames
    rows = seg.Rows
    columns = seg.Columns
    num_segments = len(seg.SegmentSequence)
    
    # 解包像素数据
    pixel_array = np.unpackbits(
        np.frombuffer(seg.PixelData, dtype=np.uint8)
    ).reshape(num_frames, rows, columns)
    
    # 获取引用实例UID
    referenced_instance_uids = [
        item.ReferencedSOPInstanceUID
        for item in seg.ReferencedSeriesSequence[0].ReferencedInstanceSequence
    ]
    
    return pixel_array, referenced_instance_uids
```

---

## 6. 关键算法实现

### 6.1 涂鸦路径处理

**路径致密化**:
```python
def clean_and_densify_polyline(points, spacing=1.0):
    """清理并致密化多边形路径"""
    points = np.array(points)
    
    # 去重
    _, unique_indices = np.unique(points, axis=0, return_index=True)
    points = points[np.sort(unique_indices)]
    
    # 线性插值填充间隙
    dense_points = [points[0]]
    for i in range(1, len(points)):
        prev = points[i-1]
        curr = points[i]
        dist = np.linalg.norm(curr - prev)
        
        if dist > spacing:
            # 插入中间点
            num_points = int(dist / spacing)
            for t in np.linspace(0, 1, num_points + 1)[1:]:
                interpolated = prev + t * (curr - prev)
                dense_points.append(interpolated)
        else:
            dense_points.append(curr)
    
    return np.array(dense_points)
```

**扫描线填充**:
```python
def get_scanline_filled_points_3d(polyline):
    """使用扫描线算法填充3D多边形"""
    # 投影到XY平面
    xy_points = polyline[:, :2]
    z = int(polyline[0, 2])
    
    # 计算边界框
    min_x, min_y = xy_points.min(axis=0)
    max_x, max_y = xy_points.max(axis=0)
    
    filled_points = []
    for y in range(int(min_y), int(max_y) + 1):
        # 找到与当前扫描线的交点
        intersections = []
        for i in range(len(xy_points)):
            p1 = xy_points[i]
            p2 = xy_points[(i + 1) % len(xy_points)]
            
            if (p1[1] <= y < p2[1]) or (p2[1] <= y < p1[1]):
                # 计算交点x坐标
                x = p1[0] + (y - p1[1]) * (p2[0] - p1[0]) / (p2[1] - p1[1])
                intersections.append(x)
        
        # 排序并填充
        intersections.sort()
        for i in range(0, len(intersections), 2):
            if i + 1 < len(intersections):
                x_start = int(intersections[i])
                x_end = int(intersections[i + 1])
                for x in range(x_start, x_end + 1):
                    filled_points.append([x, y, z])
    
    return np.array(filled_points)
```

### 6.2 球形核生成

```python
def spherical_kernel(radius=1):
    """生成球形结构元素"""
    size = 2 * radius + 1
    kernel = np.zeros((size, size, size), dtype=np.uint8)
    
    center = radius
    for z in range(size):
        for y in range(size):
            for x in range(size):
                dist = np.sqrt((x - center)**2 + (y - center)**2 + (z - center)**2)
                if dist <= radius:
                    kernel[z, y, x] = 1
    
    return kernel
```

### 6.3 提示去重机制

为避免重复计算相同的提示：

```python
def add_prompt(self, prompt, prompt_type):
    """添加提示到已使用集合"""
    # 使用MD5哈希标识提示
    prompt_hash = hashlib.md5(np.array(prompt).tobytes()).hexdigest()
    self._session_used_interactions[prompt_type].add(prompt_hash)

def is_prompt_used(self, prompt, prompt_type):
    """检查提示是否已使用"""
    prompt_hash = hashlib.md5(np.array(prompt).tobytes()).hexdigest()
    return prompt_hash in self._session_used_interactions[prompt_type]

# 使用示例
for point in data['pos_points']:
    if not self.is_prompt_used(point, "pos_points"):
        self.add_prompt(point, "pos_points")
        session.add_point_interaction(tuple(point[::-1]), include_interaction=True)
```

---

## 7. 性能优化实现

### 7.1 异步推理

```python
# 使用线程池处理并发推理
self._infers_threadpool = ThreadPoolExecutor(
    max_workers=settings.MONAI_LABEL_INFER_CONCURRENCY,
    thread_name_prefix="INFER"
)

# 提交推理任务
def run_infer_in_thread(t, r):
    handle_torch_linalg_multithread(r)
    return t(r)

f = self._infers_threadpool.submit(run_infer_in_thread, t=task, r=request)
result_file_name, result_json = f.result(timeout=request.get("timeout", 300))
```

### 7.2 模型缓存

```python
def _get_network(self, device, data):
    """获取或加载模型，带缓存机制"""
    path = self.get_path()
    
    # 检查缓存
    cached = self._networks.get(device)
    statbuf = os.stat(path) if path else None
    
    if cached:
        if statbuf and statbuf.st_mtime == cached[1]:
            # 缓存有效
            return cached[0]
        else:
            # 模型文件已更新，重新加载
            logger.warning(f"Reload model from cache")
    
    # 加载模型
    network = copy.deepcopy(self.network)
    network.to(torch.device(device))
    
    if path:
        checkpoint = torch.load(path, map_location=torch.device(device))
        model_state_dict = checkpoint.get(self.model_state_dict, checkpoint)
        network.load_state_dict(model_state_dict, strict=self.load_strict)
    
    network.eval()
    
    # 存入缓存
    self._networks[device] = (network, statbuf.st_mtime if statbuf else 0)
    
    return network
```

### 7.3 内存管理

```python
# 及时释放GPU内存
torch.cuda.empty_cache()

# 使用bfloat16降低显存占用
with torch.autocast("cuda", dtype=torch.bfloat16):
    outputs = model(inputs)

# 分离不需要梯度的张量
with torch.no_grad():
    outputs = model(inputs)
```

---

## 8. 错误处理与容错

### 8.1 会话恢复

```python
def _safe_interaction(self, perform_callable):
    """安全的交互执行，带错误恢复"""
    try:
        if session.original_image_shape is None:
            # 会话未初始化，尝试重新设置
            session.set_image(img_np)
            session.set_target_buffer(torch.zeros(img_np.shape[1:], dtype=torch.uint8))
            
            # 等待预处理完成
            max_wait_time = 5.0
            wait_interval = 0.1
            waited_time = 0.0
            
            while session.preprocessed_image is None and waited_time < max_wait_time:
                time.sleep(wait_interval)
                waited_time += wait_interval
            
            if session.preprocessed_image is None:
                # 超时，重置会话
                session.executor.shutdown(wait=False, cancel_futures=True)
                session.executor = ThreadPoolExecutor(max_workers=2)
                session._reset_session()
                return False
        
        # 执行交互
        with timeout_context(seconds=5):
            perform_callable()
        return True
        
    except Exception as e:
        logger.error(f"Error during interaction: {e}")
        # 尝试恢复
        session.executor.shutdown(wait=False, cancel_futures=True)
        session.executor = ThreadPoolExecutor(max_workers=2)
        session._reset_session()
        return False
```

---

## 9. 总结

本文档详细阐述了OHIF-AI的核心实现原理：

1. **前端**: 基于Cornerstone3D的交互系统，收集用户提示并发送到后端
2. **后端**: 基于FastAPI和MONAI的推理服务，管理多个AI模型
3. **模型集成**: 统一的接口封装不同的深度学习模型
4. **数据处理**: 复杂的DICOM方向处理和坐标变换
5. **通信协议**: 基于HTTP的多部分表单数据传输

理解这些原理有助于进行二次开发和问题排查。
