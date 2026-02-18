# 自定义修改SAM2模型并集成到OHIF-AI的完整指南

本文档详细说明如何修改SAM2原始代码，并将修改后的模型集成到OHIF-AI中进行推理分割。

---

## 📋 目录

1. [SAM2代码结构概览](#1-sam2代码结构概览)
2. [常见修改场景](#2-常见修改场景)
3. [修改SAM2核心代码](#3-修改sam2核心代码)
4. [后端集成修改](#4-后端集成修改)
5. [前端参数传递](#5-前端参数传递)
6. [完整示例：添加自定义提示编码器](#6-完整示例添加自定义提示编码器)
7. [调试与验证](#7-调试与验证)

---

## 1. SAM2代码结构概览

### 1.1 目录结构

```
sam2/                          # SAM2根目录
├── sam2/                      # Python包
│   ├── __init__.py           # 包入口
│   ├── build_sam.py          # 模型构建入口 ⭐
│   ├── sam2_image_predictor.py  # 图像预测器
│   ├── sam2_video_predictor.py  # 视频预测器 ⭐⭐
│   ├── modeling/             # 模型定义
│   │   ├── __init__.py
│   │   ├── sam.py            # SAM模型主类 ⭐⭐
│   │   ├── image_encoder.py  # 图像编码器 ⭐
│   │   ├── memory_encoder.py # 记忆编码器
│   │   ├── memory_attention.py # 记忆注意力
│   │   ├── prompt_encoder.py # 提示编码器 ⭐⭐
│   │   ├── mask_decoder.py   # 掩码解码器 ⭐⭐
│   │   └── transformer.py    # Transformer模块
│   └── utils/                # 工具函数
│       ├── misc.py
│       └── transforms.py
├── checkpoints/              # 模型权重目录
├── configs/                  # 配置文件
│   ├── sam2/                 # SAM2配置
│   └── sam2.1/               # SAM2.1配置 ⭐
│       ├── sam2.1_hiera_t.yaml
│       └── sam2.1_hiera_s.yaml
└── setup.py                  # 安装配置
```

### 1.2 关键文件说明

| 文件 | 作用 | 修改频率 |
|------|------|---------|
| `sam2_video_predictor.py` | 视频/3D推理入口 | ⭐⭐⭐ 高 |
| `sam.py` | 模型前向传播 | ⭐⭐⭐ 高 |
| `prompt_encoder.py` | 提示编码(点/框) | ⭐⭐⭐ 高 |
| `mask_decoder.py` | 掩码解码输出 | ⭐⭐ 中 |
| `image_encoder.py` | 图像特征编码 | ⭐⭐ 中 |
| `build_sam.py` | 模型构建 | ⭐ 低 |
| `*.yaml` | 模型配置 | ⭐⭐ 中 |

---

## 2. 常见修改场景

### 2.1 场景分类

| 场景 | 修改内容 | 影响范围 |
|-----|---------|---------|
| **A. 修改提示类型** | 添加新的提示方式(如涂鸦编码) | `prompt_encoder.py` |
| **B. 修改网络结构** | 添加/删除层，修改通道数 | `modeling/*.py` |
| **C. 修改推理逻辑** | 改变传播方式，添加后处理 | `sam2_video_predictor.py` |
| **D. 修改损失函数** | 训练时使用的损失(如需要微调) | `modeling/sam.py` |
| **E. 修改配置参数** | 图像尺寸，内存库大小等 | `configs/*.yaml` |

### 2.2 OHIF-AI中的SAM2调用链

```
用户提示 → frontend
    ↓
POST /monai/infer/segmentation
    ↓
basic_infer.py: __call__()
    ↓
SAM分支 (nnInter=False)
    ↓
predictor.init_state()  ← 加载3D体积
    ↓
predictor.add_new_points_or_box()  ← 添加提示
    ↓
predictor.propagate_in_video()  ← 3D传播
    ↓
返回分割结果
```

---

## 3. 修改SAM2核心代码

### 3.1 场景A：添加自定义提示编码器(以涂鸦为例)

**目标**：让SAM2支持涂鸦(scribble)提示，而不仅是点和框

#### 步骤1：修改Prompt Encoder

**文件**：`sam2/modeling/prompt_encoder.py`

```python
# ========== 原始代码结构 ==========
class PromptEncoder(nn.Module):
    def __init__(self, ...):
        self.pe_layer = PositionEmbeddingRandom(...)
        self.point_embeddings = nn.ModuleList([...])  # 点嵌入
        self.not_a_point_embed = nn.Embedding(...)    # 非点嵌入
        # 只有点和框的编码
        
    def forward(self, points, boxes, masks):
        """编码提示"""
        # 处理点和框...

# ========== 修改后代码 ==========
class PromptEncoder(nn.Module):
    def __init__(self, embed_dim=256, ...):
        super().__init__()
        self.pe_layer = PositionEmbeddingRandom(num_pos_feats=embed_dim // 2)
        
        # 原有嵌入
        self.point_embeddings = nn.ModuleList([
            nn.Embedding(1, embed_dim) for _ in range(4)
        ])
        self.not_a_point_embed = nn.Embedding(1, embed_dim)
        
        # 【新增】涂鸦编码器
        self.scribble_encoder = ScribbleEncoder(embed_dim)
        
        # 【新增】涂鸦嵌入
        self.scribble_embed = nn.Embedding(1, embed_dim)
        
    def forward(self, points, boxes, masks, scribbles=None):
        """
        参数:
            points: (B, N, 2) 点坐标
            boxes: (B, M, 4) 框坐标
            masks: (B, 1, H, W) 掩码提示
            scribbles: (B, 1, H, W) 涂鸦提示 【新增】
        """
        bs = self._get_batch_size(points, boxes, masks, scribbles)
        sparse_embeddings = []
        
        # 编码点提示
        if points is not None:
            point_embeddings = self._embed_points(points)
            sparse_embeddings.append(point_embeddings)
        
        # 编码框提示
        if boxes is not None:
            box_embeddings = self._embed_boxes(boxes)
            sparse_embeddings.append(box_embeddings)
        
        # 【新增】编码涂鸦提示
        if scribbles is not None:
            scribble_embeddings = self.scribble_encoder(scribbles)
            sparse_embeddings.append(scribble_embeddings)
        
        # 合并所有稀疏提示
        if len(sparse_embeddings) == 0:
            sparse_embeddings = torch.zeros((bs, 0, self.embed_dim), ...)
        else:
            sparse_embeddings = torch.cat(sparse_embeddings, dim=1)
        
        # 编码掩码提示(密集)
        if masks is not None:
            dense_embeddings = self.mask_downscaling(masks)
        else:
            dense_embeddings = self.no_mask_embed.weight.reshape(...).expand(bs, -1, -1, -1)
        
        return sparse_embeddings, dense_embeddings


# 【新增】涂鸦编码器模块
class ScribbleEncoder(nn.Module):
    """
    将涂鸦掩码编码为稀疏嵌入
    """
    def __init__(self, embed_dim=256, num_layers=3):
        super().__init__()
        
        # 使用轻量CNN编码涂鸦
        layers = []
        in_channels = 1
        out_channels = 64
        
        for _ in range(num_layers):
            layers.extend([
                nn.Conv2d(in_channels, out_channels, 3, padding=1),
                nn.BatchNorm2d(out_channels),
                nn.ReLU(inplace=True),
                nn.MaxPool2d(2),
            ])
            in_channels = out_channels
            out_channels = min(out_channels * 2, 256)
        
        self.encoder = nn.Sequential(*layers)
        
        # 全局平均池化 + 投影到embed_dim
        self.projection = nn.Sequential(
            nn.AdaptiveAvgPool2d(1),
            nn.Flatten(),
            nn.Linear(out_channels, embed_dim),
            nn.LayerNorm(embed_dim),
        )
    
    def forward(self, scribbles):
        """
        参数:
            scribbles: (B, 1, H, W) 二值涂鸦掩码
        返回:
            embeddings: (B, 1, embed_dim) 涂鸦嵌入
        """
        features = self.encoder(scribbles)
        embeddings = self.projection(features)
        return embeddings.unsqueeze(1)  # (B, 1, embed_dim)
```

#### 步骤2：修改SAM模型类接受涂鸦输入

**文件**：`sam2/modeling/sam.py`

```python
class SAM2Base(nn.Module):
    def forward(
        self,
        image=None,
        point_coords=None,
        point_labels=None,
        boxes=None,
        mask_inputs=None,
        scribbles=None,  # 【新增】
        ...
    ):
        # ... 原有代码 ...
        
        # 编码提示
        sparse_embeddings, dense_embeddings = self.sam_prompt_encoder(
            points=(point_coords, point_labels),
            boxes=boxes,
            masks=mask_inputs,
            scribbles=scribbles,  # 【新增】传递涂鸦
        )
        
        # ... 后续代码 ...

class SAM2VideoPredictor(SAM2Base):
    def predict(
        self,
        image,
        point_coords=None,
        point_labels=None,
        boxes=None,
        mask_input=None,
        scribbles=None,  # 【新增】
        ...
    ):
        """
        单次预测
        """
        # 调用父类forward
        return super().forward(
            image=image,
            point_coords=point_coords,
            point_labels=point_labels,
            boxes=boxes,
            mask_inputs=mask_input,
            scribbles=scribbles,  # 【新增】
            ...
        )
    
    def add_new_points_or_box(
        self,
        inference_state,
        frame_idx,
        obj_id,
        points=None,
        labels=None,
        box=None,
        scribbles=None,  # 【新增】
    ):
        """
        添加新的提示(包括涂鸦)
        """
        # 存储涂鸦到inference_state
        if scribbles is not None:
            inference_state["scribbles"][frame_idx] = {
                obj_id: scribbles,
            }
        
        # 调用原有逻辑
        return self._add_points_or_boxes(
            inference_state, frame_idx, obj_id,
            points, labels, box, scribbles
        )
```

#### 步骤3：修改Video Predictor处理3D涂鸦

**文件**：`sam2/sam2_video_predictor.py`

```python
class SAM2VideoPredictor:
    def add_new_points_or_box(
        self,
        inference_state,
        frame_idx,
        obj_id,
        points=None,
        labels=None,
        box=None,
        scribbles=None,  # 【新增】(H, W) 2D涂鸦掩码
        scribbles_3d=None,  # 【新增】(D, H, W) 3D涂鸦体积
    ):
        """
        添加提示到指定帧
        
        新增参数:
            scribbles: 2D涂鸦掩码，用于单帧
            scribbles_3d: 3D涂鸦体积，用于多帧传播
        """
        # 获取当前帧的图像
        img_idx = inference_state["frame_idx"][frame_idx]
        
        # 处理3D涂鸦 - 提取当前帧的切片
        if scribbles_3d is not None:
            scribbles = scribbles_3d[frame_idx]  # 提取对应帧
        
        # 预处理涂鸦
        if scribbles is not None:
            scribbles_tensor = self._prepare_scribble_input(scribbles)
        else:
            scribbles_tensor = None
        
        # 存储提示
        if "prompts" not in inference_state:
            inference_state["prompts"] = {}
        
        frame_prompts = inference_state["prompts"].get(frame_idx, [])
        frame_prompts.append({
            "obj_id": obj_id,
            "points": points,
            "labels": labels,
            "box": box,
            "scribbles": scribbles_tensor,  # 【新增】
        })
        inference_state["prompts"][frame_idx] = frame_prompts
        
        # 如果是涂鸦，立即处理该帧
        if scribbles is not None:
            # 运行编码器
            high_res_features = self._get_image_features(inference_state, frame_idx)
            
            # 编码涂鸦
            scribble_embedding = self.model.sam_prompt_encoder.scribble_encoder(
                scribbles_tensor
            )
            
            # 与其他提示合并
            # ...
        
        return self.predict(
            inference_state=inference_state,
            frame_idx=frame_idx,
            obj_id=obj_id,
            points=points,
            labels=labels,
            box=box,
            scribbles=scribbles_tensor,  # 【新增】
        )
    
    def _prepare_scribble_input(self, scribbles):
        """
        预处理涂鸦输入
        
        参数:
            scribbles: numpy数组 (H, W) 或 list of points
        
        返回:
            tensor: (1, 1, H, W) 处理后的涂鸦
        """
        import torch
        import numpy as np
        
        if isinstance(scribbles, list):
            # 点列表转换为掩码
            # 假设scribbles是 [[x1,y1], [x2,y2], ...]
            points = np.array(scribbles)
            H, W = inference_state["image_shape"]
            mask = np.zeros((H, W), dtype=np.float32)
            
            # 绘制线条
            for i in range(len(points) - 1):
                cv2.line(mask, 
                        tuple(points[i]), 
                        tuple(points[i+1]), 
                        color=1, 
                        thickness=2)
            
            scribbles = mask
        
        # 转换为tensor
        if isinstance(scribbles, np.ndarray):
            scribbles = torch.from_numpy(scribbles).float()
        
        # 添加batch和channel维度
        if scribbles.dim() == 2:
            scribbles = scribbles.unsqueeze(0).unsqueeze(0)
        
        return scribbles
    
    def propagate_in_video_with_scribbles(
        self,
        inference_state,
        start_frame_idx=0,
        max_frame_num_to_track=None,
        scribbles_3d=None,  # 【新增】3D涂鸦引导传播
    ):
        """
        在视频中传播，支持3D涂鸦引导
        """
        # 如果提供了3D涂鸦，在每个帧使用对应的切片
        if scribbles_3d is not None:
            for frame_idx in range(start_frame_idx, num_frames):
                if frame_idx < len(scribbles_3d):
                    # 获取当前帧的涂鸦
                    frame_scribble = scribbles_3d[frame_idx]
                    
                    # 添加涂鸦提示
                    self.add_new_points_or_box(
                        inference_state,
                        frame_idx=frame_idx,
                        obj_id=1,
                        scribbles=frame_scribble,
                    )
        
        # 继续正常传播
        yield from self.propagate_in_video(
            inference_state,
            start_frame_idx,
            max_frame_num_to_track,
        )
```

### 3.2 场景B：修改图像编码器输出维度

**文件**：`sam2/modeling/image_encoder.py`

```python
class ImageEncoder(nn.Module):
    def __init__(
        self,
        trunk: nn.Module,
        neck: nn.Module,
        scalp: int = 1,
        custom_embed_dim: int = None,  # 【新增】自定义嵌入维度
    ):
        super().__init__()
        self.trunk = trunk
        self.neck = neck
        self.scalp = scalp
        
        # 【新增】添加投影层修改输出维度
        if custom_embed_dim is not None:
            self.output_projection = nn.Sequential(
                nn.Conv2d(self.neck.embed_dim, custom_embed_dim, 1),
                nn.LayerNorm(custom_embed_dim),
            )
            self.embed_dim = custom_embed_dim
        else:
            self.embed_dim = self.neck.embed_dim
    
    def forward(self, sample: torch.Tensor):
        """
        参数:
            sample: (B, 3, H, W) 输入图像
        返回:
            features: (B, embed_dim, H', W') 特征图
        """
        # 主干网络
        x = self.trunk(sample)
        
        # 颈部处理
        x = self.neck(x)
        
        # 【新增】输出投影
        if hasattr(self, 'output_projection'):
            x = self.output_projection(x)
        
        return x
```

### 3.3 场景C：修改传播逻辑(添加后处理)

**文件**：`sam2/sam2_video_predictor.py`

```python
class SAM2VideoPredictor:
    def propagate_in_video(
        self,
        inference_state,
        start_frame_idx=0,
        max_frame_num_to_track=None,
        reverse=False,
        post_process_mask=True,  # 【新增】掩码后处理
        apply_3d_morphology=True,  # 【新增】3D形态学处理
    ):
        """
        在视频中传播分割
        
        新增参数:
            post_process_mask: 是否应用后处理
            apply_3d_morphology: 是否应用3D形态学操作
        """
        # 收集所有帧的结果
        results = []
        for frame_idx, obj_ids, mask_logits in self._propagate_frames(...):
            
            # 【新增】后处理
            if post_process_mask:
                mask_logits = self._post_process_mask(mask_logits)
            
            results.append((frame_idx, obj_ids, mask_logits))
            yield frame_idx, obj_ids, mask_logits
        
        # 【新增】3D形态学处理(整个体积)
        if apply_3d_morphology:
            self._apply_3d_morphology(results)
    
    def _post_process_mask(self, mask_logits):
        """
        单帧掩码后处理
        """
        import torch.nn.functional as F
        
        # 1. 阈值化
        mask = (mask_logits > 0.0).float()
        
        # 2. 填充小孔
        from scipy import ndimage
        mask_np = mask.cpu().numpy()
        mask_np = ndimage.binary_fill_holes(mask_np)
        
        # 3. 移除小连通域
        mask_np = self._remove_small_regions(mask_np, min_area=100)
        
        return torch.from_numpy(mask_np).to(mask_logits.device)
    
    def _remove_small_regions(self, mask, min_area=100):
        """移除小的连通域"""
        from scipy import ndimage
        labeled, num_features = ndimage.label(mask)
        
        for i in range(1, num_features + 1):
            region = (labeled == i)
            if region.sum() < min_area:
                mask[region] = 0
        
        return mask
    
    def _apply_3d_morphology(self, results):
        """
        对整个3D体积应用形态学操作
        """
        import scipy.ndimage as ndi
        
        # 收集所有掩码
        masks = [r[2].cpu().numpy() for r in results]
        volume = np.stack(masks, axis=0)  # (D, H, W)
        
        # 3D闭运算(填充间隙)
        volume = ndi.binary_closing(volume, structure=np.ones((3, 3, 3)))
        
        # 3D开运算(去除噪点)
        volume = ndi.binary_opening(volume, structure=np.ones((3, 3, 3)))
        
        # 更新结果
        for i, (frame_idx, obj_ids, _) in enumerate(results):
            results[i] = (frame_idx, obj_ids, torch.from_numpy(volume[i]))
```

---

## 4. 后端集成修改

### 4.1 修改basic_infer.py支持自定义SAM2

**文件**：`monai-label/monailabel/tasks/infer/basic_infer.py`

```python
# ========== 修改后的SAM2初始化 ==========

# 判断使用哪个版本的SAM2
USE_CUSTOM_SAM2 = os.environ.get("USE_CUSTOM_SAM2", "false").lower() == "true"
CUSTOM_SAM2_PATH = os.environ.get("CUSTOM_SAM2_PATH", "/code/custom_sam2")

if USE_CUSTOM_SAM2:
    # 加载修改后的SAM2
    import sys
    sys.path.insert(0, CUSTOM_SAM2_PATH)
    
    from custom_sam2.build_sam import build_sam2_video_predictor
    from custom_sam2.sam2_video_predictor import SAM2VideoPredictor
    
    print("Using CUSTOM SAM2 from:", CUSTOM_SAM2_PATH)
else:
    # 使用原版SAM2
    from sam2.build_sam import build_sam2_video_predictor

# 初始化时添加自定义参数
sam2_config = {
    "model_cfg": "configs/sam2.1/sam2.1_hiera_t.yaml",
    "checkpoint": "/code/checkpoints/sam2.1_hiera_tiny.pt",
    # 【新增】自定义参数
    "custom_embed_dim": 256,  # 如果修改了编码器输出
    "enable_scribble": True,  # 启用涂鸦支持
    "post_process_mask": True,  # 启用后处理
}

# 使用自定义构建函数
if USE_CUSTOM_SAM2:
    predictor_sam2 = build_sam2_video_predictor(
        sam2_config["model_cfg"],
        sam2_config["checkpoint"],
        custom_embed_dim=sam2_config.get("custom_embed_dim"),
        enable_scribble=sam2_config.get("enable_scribble", False),
    )
else:
    predictor_sam2 = build_sam2_video_predictor(
        sam2_config["model_cfg"],
        sam2_config["checkpoint"],
    )


# ========== 修改推理流程传递涂鸦 ==========

def _sam_infer(self, img, data, contrast_center, contrast_window):
    """SAM2推理，支持涂鸦"""
    
    # 选择预测器(原版或自定义)
    if data.get('use_custom_sam2', False) and USE_CUSTOM_SAM2:
        predictor = predictor_sam2_custom
    else:
        predictor = predictor_sam2
    
    # 初始化状态
    with torch.inference_mode(), torch.autocast("cuda", dtype=torch.bfloat16):
        inference_state = predictor.init_state(
            video_path=img,
            clip_low=contrast_center - contrast_window/2 if contrast_center else None,
            clip_high=contrast_center + contrast_window/2 if contrast_center else None,
        )
    
    # 处理点提示
    if len(result_json.get('pos_points', [])) > 0:
        for frame_idx in ann_frame_list:
            # ... 原有代码 ...
            
            # 【新增】如果自定义SAM2支持涂鸦且前端传递了涂鸦
            if USE_CUSTOM_SAM2 and 'scribbles' in data:
                scribble_mask = self._prepare_scribble_mask(
                    data['scribbles'], 
                    img_np.shape
                )
                
                predictor.add_new_points_or_box(
                    inference_state=inference_state,
                    frame_idx=frame_idx,
                    obj_id=ann_obj_id,
                    points=points,
                    labels=labels,
                    scribbles=scribble_mask,  # 【新增】传递涂鸦
                )
            else:
                # 原版SAM2，只传递点和框
                predictor.add_new_points_or_box(
                    inference_state=inference_state,
                    frame_idx=frame_idx,
                    obj_id=ann_obj_id,
                    points=points,
                    labels=labels,
                    box=boxes if len(boxes) > 0 else None,
                )
    
    # 传播时启用后处理
    video_segments = {}
    with torch.inference_mode(), torch.autocast("cuda", dtype=torch.bfloat16):
        
        # 【新增】检查是否使用自定义传播
        if USE_CUSTOM_SAM2 and hasattr(predictor, 'propagate_in_video_with_scribbles'):
            # 使用自定义传播(支持3D涂鸦)
            generator = predictor.propagate_in_video_with_scribbles(
                inference_state,
                scribbles_3d=data.get('scribbles_3d'),  # 【新增】
                post_process_mask=True,  # 【新增】
                apply_3d_morphology=data.get('apply_3d_morphology', False),  # 【新增】
            )
        else:
            # 原版传播
            generator = predictor.propagate_in_video(inference_state)
        
        for out_frame_idx, out_obj_ids, out_mask_logits in generator:
            video_segments[out_frame_idx] = {
                obj_id: (out_mask_logits[i] > 0.0).cpu().numpy()
                for i, obj_id in enumerate(out_obj_ids)
            }
    
    return pred, {"sam_elapsed": elapsed_time, ...}


def _prepare_scribble_mask(self, scribbles_data, image_shape):
    """
    将前端传来的涂鸦数据转换为SAM2可用的掩码
    
    参数:
        scribbles_data: 从前端传来的涂鸦坐标列表
        image_shape: (H, W, D) 或 (H, W)
    
    返回:
        scribble_mask: (D, H, W) 3D涂鸦掩码或 (H, W) 2D掩码
    """
    if len(image_shape) == 3:
        H, W, D = image_shape
    else:
        H, W = image_shape
        D = 1
    
    # 创建3D掩码
    scribble_mask = np.zeros((D, H, W), dtype=np.float32)
    
    # 填充涂鸦点
    for point in scribbles_data:
        x, y, z = int(point[0]), int(point[1]), int(point[2])
        if 0 <= z < D and 0 <= y < H and 0 <= x < W:
            # 以点为中心绘制小圆
            cv2.circle(scribble_mask[z], (x, y), radius=2, color=1, thickness=-1)
    
    return torch.from_numpy(scribble_mask).unsqueeze(1)  # (D, 1, H, W)
```

### 4.2 添加自定义模型配置

**文件**：`monai-label/monailabel/config.py`

```python
class Settings(BaseSettings):
    # ... 原有配置 ...
    
    # 【新增】自定义SAM2配置
    USE_CUSTOM_SAM2: bool = False
    CUSTOM_SAM2_PATH: str = "/code/custom_sam2"
    SAM2_CUSTOM_EMBED_DIM: int = 256
    SAM2_ENABLE_SCRIBBLE: bool = False
    SAM2_POST_PROCESS: bool = True
```

---

## 5. 前端参数传递

### 5.1 修改请求参数

**文件**：`Viewers/modes/longitudinal/src/toolbarButtons.ts`

```typescript
// 添加自定义SAM2选项
const sam2ModelSelector = {
  id: 'sam2ModelSelector',
  label: 'SAM2 Version',
  type: 'radioGroup',
  options: [
    { label: 'Standard SAM2', value: 'standard' },
    { label: 'Custom SAM2 (Scribble)', value: 'custom_scribble' },
    { label: 'Custom SAM2 (3D Morph)', value: 'custom_3d' },
  ],
  commandName: 'setSAM2Version',
};

// 添加后处理选项
const postProcessToggle = {
  id: 'sam2PostProcess',
  label: 'Enable Post-processing',
  type: 'toggle',
  defaultValue: true,
};
```

### 5.2 修改API调用

**文件**：前端AI服务

```typescript
class AIService {
  async runInference(requestData: InferenceRequest) {
    const params = {
      ...requestData,
      // 传递自定义SAM2参数
      use_custom_sam2: this.selectedModel === 'custom_scribble',
      apply_3d_morphology: this.selectedModel === 'custom_3d',
      enable_post_process: this.postProcessEnabled,
      
      // 如果有涂鸦，传递涂鸦数据
      scribbles: this.collectedScribbles,
      scribbles_3d: this.convertScribblesTo3D(this.collectedScribbles),
    };
    
    const response = await fetch('/monai/infer/segmentation', {
      method: 'POST',
      body: JSON.stringify(params),
    });
    
    return response;
  }
  
  private convertScribblesTo3D(scribbles: Point2D[]): Scribble3D {
    // 将2D涂鸦列表转换为3D体积
    const volume = new Float32Array(depth * height * width);
    
    scribbles.forEach(point => {
      const idx = point.z * height * width + point.y * width + point.x;
      volume[idx] = 1.0;
    });
    
    return volume;
  }
}
```

---

## 6. 完整示例：添加自定义提示编码器

### 步骤总结

```bash
# 1. 复制SAM2代码到自定义目录
cp -r sam2 custom_sam2

# 2. 修改 custom_sam2/modeling/prompt_encoder.py
#    - 添加 ScribbleEncoder 类
#    - 修改 forward 方法接受 scribbles 参数

# 3. 修改 custom_sam2/modeling/sam.py
#    - 修改 forward 方法传递 scribbles
#    - 修改 predict 方法

# 4. 修改 custom_sam2/sam2_video_predictor.py
#    - 修改 add_new_points_or_box 接受 scribbles
#    - 添加 propagate_in_video_with_scribbles

# 5. 重新安装自定义SAM2
cd custom_sam2
pip install -e .

# 6. 修改后端 basic_infer.py
#    - 添加 USE_CUSTOM_SAM2 判断
#    - 修改 _sam_infer 传递涂鸦参数

# 7. 修改前端传递涂鸦数据

# 8. 测试
```

---

## 7. 调试与验证

### 7.1 单元测试

```python
def test_custom_prompt_encoder():
    from custom_sam2.modeling.prompt_encoder import PromptEncoder, ScribbleEncoder
    
    # 测试涂鸦编码器
    encoder = ScribbleEncoder(embed_dim=256)
    scribble = torch.randn(1, 1, 256, 256)
    
    embedding = encoder(scribble)
    assert embedding.shape == (1, 1, 256)


def test_custom_sam2_forward():
    from custom_sam2.build_sam import build_sam2_video_predictor
    
    predictor = build_sam2_video_predictor(
        "configs/sam2.1/sam2.1_hiera_t.yaml",
        "checkpoints/sam2.1_hiera_tiny.pt",
        enable_scribble=True,
    )
    
    # 测试涂鸦输入
    scribbles = torch.zeros(1, 1, 256, 256)
    scribbles[0, 0, 100:150, 100:150] = 1.0
    
    result = predictor.predict(
        image=torch.randn(1, 3, 1024, 1024),
        scribbles=scribbles,
    )
    
    assert result is not None
```

### 7.2 集成验证

```bash
# 1. 检查自定义SAM2是否正确加载
python -c "from custom_sam2.sam2_video_predictor import SAM2VideoPredictor; print('OK')"

# 2. 检查后端是否能导入
python -c "from monailabel.tasks.infer.basic_infer import predictor_sam2; print(type(predictor_sam2))"

# 3. 发送测试请求
curl -X POST http://localhost:8002/monai/infer/segmentation \
  -F 'params={"use_custom_sam2": true, "scribbles": [[100,100,50]], "pos_points": [[100,100,50]]}'
```

---

## 8. 总结

### 修改清单

| 组件 | 文件 | 修改内容 |
|------|------|---------|
| **SAM2核心** | `modeling/prompt_encoder.py` | 添加涂鸦编码器 |
| **SAM2核心** | `modeling/sam.py` | 修改forward接受涂鸦 |
| **SAM2核心** | `sam2_video_predictor.py` | 添加涂鸦处理方法 |
| **后端** | `basic_infer.py` | 支持自定义SAM2加载和调用 |
| **后端** | `config.py` | 添加自定义SAM2配置 |
| **前端** | `toolbarButtons.ts` | 添加版本选择器 |
| **前端** | AI服务 | 传递涂鸦参数 |

### 关键注意事项

1. **版本兼容性**：确保自定义SAM2与原版API兼容
2. **权重加载**：修改结构后可能需要重新训练或微调
3. **性能影响**：复杂编码器可能影响推理速度
4. **CUDA内存**：3D处理注意显存管理

---

*本文档提供了完整的SAM2自定义修改方案，更多细节请参考具体代码实现*
