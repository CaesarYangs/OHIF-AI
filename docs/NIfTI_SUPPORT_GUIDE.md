# OHIF-AI 支持MSD(NIfTI)数据格式实现指南

本文档详细说明如何让OHIF-AI支持直接上传和加载MSD(Medical Segmentation Decathlon)格式的`.nii.gz`文件作为3D数据输入。

---

## 📋 目录

1. [需求分析](#1-需求分析)
2. [架构方案](#2-架构方案)
3. [前端实现](#3-前端实现)
4. [后端实现](#4-后端实现)
5. [数据流改造](#5-数据流改造)
6. [完整代码实现](#6-完整代码实现)
7. [测试验证](#7-测试验证)

---

## 1. 需求分析

### 当前系统支持
- ✅ DICOM格式 (通过Orthanc PACS)
- ✅ DICOM Web (WADO-RS)
- ❌ NIfTI (.nii, .nii.gz)

### MSD数据集格式
```
Task03_Liver/
├── imagesTr/           # 训练图像
│   ├── liver_001.nii.gz
│   ├── liver_002.nii.gz
│   └── ...
├── labelsTr/           # 训练标签
│   ├── liver_001.nii.gz
│   ├── liver_002.nii.gz
│   └── ...
├── imagesTs/           # 测试图像
└── dataset.json        # 数据集描述
```

### 需要实现的功能
1. 直接上传 `.nii.gz` 文件
2. 自动识别NIfTI格式并加载为3D体积
3. 在Cornerstone3D中显示
4. AI模型推理支持
5. 保存分割结果为NIfTI

---

## 2. 架构方案

### 数据流对比

```
当前DICOM流程:
用户上传DICOM → Orthanc存储 → DICOM Web请求 → 前端显示 → AI推理

新的NIfTI流程:
用户上传nii.gz → 后端直接存储 → NIfTI加载器 → 前端显示 → AI推理
```

### 修改模块清单

| 层级 | 模块 | 修改内容 |
|-----|------|---------|
| 前端 | DicomUpload组件 | 支持nii.gz文件选择 |
| 前端 | DataSource | 添加NIfTI数据源 |
| 前端 | SOP Class Handler | 添加NIfTI文件处理器 |
| 前端 | Cornerstone | 配置NIfTI体积加载器 |
| 后端 | Datastore | 支持nii.gz文件存储 |
| 后端 | File Parser | 添加NIfTI元数据提取 |
| 后端 | Infer Task | 支持NIfTI文件路径输入 |
| 后端 | Writer | 支持NIfTI格式输出 |

---

## 3. 前端实现

### 3.1 修改文件上传组件

**文件**: `Viewers/extensions/cornerstone/src/components/DicomUpload/DicomUpload.tsx`

```typescript
// ========== 修改后的上传组件 ==========
import React, { useState, useCallback } from 'react';
import { useDropzone } from 'react-dropzone';

interface UploadFile {
  file: File;
  progress: number;
  status: 'pending' | 'uploading' | 'success' | 'error';
}

// 支持的文件扩展名
const SUPPORTED_EXTENSIONS = {
  DICOM: ['.dcm', '.DCM'],
  NIFTI: ['.nii', '.nii.gz', '.nii.gz'],
  NRRD: ['.nrrd', '.nhdr'],
};

const NIfTIUploadComponent: React.FC<{
  servicesManager: ServicesManager;
  onUploadComplete: (displaySet: any) => void;
}> = ({ servicesManager, onUploadComplete }) => {
  const { uiNotificationService } = servicesManager.services;
  const [uploads, setUploads] = useState<UploadFile[]>([]);

  // 检测文件类型
  const detectFileType = (file: File): 'dicom' | 'nifti' | 'nrrd' | 'unknown' => {
    const name = file.name.toLowerCase();
    
    if (name.endsWith('.dcm')) return 'dicom';
    if (name.endsWith('.nii') || name.endsWith('.nii.gz')) return 'nifti';
    if (name.endsWith('.nrrd') || name.endsWith('.nhdr')) return 'nrrd';
    
    return 'unknown';
  };

  // 处理NIfTI文件上传
  const uploadNIfTIFile = async (file: File) => {
    const formData = new FormData();
    formData.append('file', file);
    formData.append('contentType', 'application/gzip');
    
    try {
      const response = await fetch('/monai/datastore/nifti', {
        method: 'POST',
        body: formData,
      });
      
      if (!response.ok) {
        throw new Error(`Upload failed: ${response.statusText}`);
      }
      
      const result = await response.json();
      
      // 创建显示集
      const displaySet = await createNIfTIDisplaySet(result, file.name);
      
      onUploadComplete(displaySet);
      
      uiNotificationService.show({
        title: 'Upload Complete',
        message: `${file.name} uploaded successfully`,
        type: 'success',
      });
      
    } catch (error) {
      console.error('Upload error:', error);
      uiNotificationService.show({
        title: 'Upload Failed',
        message: error.message,
        type: 'error',
      });
    }
  };

  const onDrop = useCallback(async (acceptedFiles: File[]) => {
    for (const file of acceptedFiles) {
      const fileType = detectFileType(file);
      
      switch (fileType) {
        case 'dicom':
          // 现有DICOM处理逻辑
          await uploadDICOMFile(file);
          break;
          
        case 'nifti':
          // 新的NIfTI处理逻辑
          await uploadNIfTIFile(file);
          break;
          
        case 'nrrd':
          // 可选：NRRD支持
          await uploadNRRDFile(file);
          break;
          
        default:
          uiNotificationService.show({
            title: 'Unsupported File',
            message: `${file.name} is not a supported format`,
            type: 'warning',
          });
      }
    }
  }, []);

  const { getRootProps, getInputProps, isDragActive } = useDropzone({
    onDrop,
    accept: {
      'application/dicom': ['.dcm'],
      'application/gzip': ['.nii.gz'],
      'application/octet-stream': ['.nii', '.nrrd'],
    },
    multiple: true,
  });

  return (
    <div {...getRootProps()} className="upload-dropzone">
      <input {...getInputProps()} />
      <div className="upload-content">
        {isDragActive ? (
          <p>Drop the files here ...</p>
        ) : (
          <>
            <p>Drag & drop files here, or click to select</p>
            <p className="file-types">
              Supported: DICOM (.dcm), NIfTI (.nii, .nii.gz), NRRD (.nrrd)
            </p>
          </>
        )}
      </div>
      
      {/* 上传进度列表 */}
      <div className="upload-progress-list">
        {uploads.map((upload, index) => (
          <div key={index} className={`upload-item ${upload.status}`}>
            <span className="file-name">{upload.file.name}</span>
            <div className="progress-bar">
              <div 
                className="progress-fill" 
                style={{ width: `${upload.progress}%` }}
              />
            </div>
            <span className="status">{upload.status}</span>
          </div>
        ))}
      </div>
    </div>
  );
};

export default NIfTIUploadComponent;
```

---

### 3.2 创建NIfTI数据源

**新建文件**: `Viewers/extensions/default/src/NIfTIDataSource/index.ts`

```typescript
// ========== NIfTI数据源实现 ==========
import { utils, dataSources } from '@ohif/core';
import { createNIfTILoader } from './createNIfTILoader';

const { createStudyMetadata, createSeriesMetadata } = utils;

/**
 * NIfTI数据源
 * 支持直接加载.nii和.nii.gz文件
 */
function createNIfTIDataSource(config: any) {
  const { name, configuration } = config;
  
  const dataSource = {
    name,
    type: 'nifti',
    
    /**
     * 查询研究列表
     */
    queryStudies: async function(queryParams) {
      // 从后端获取已上传的NIfTI文件列表
      const response = await fetch('/monai/datastore/images?format=nifti');
      const niftiFiles = await response.json();
      
      return niftiFiles.map(file => ({
        studyInstanceUid: file.id,
        patientId: file.patientId || 'Unknown',
        patientName: file.patientName || 'Unknown',
        studyDate: file.uploadDate,
        studyDescription: file.description || `NIfTI: ${file.name}`,
        numSeries: 1,
        modalities: file.modality || 'CT',
      }));
    },
    
    /**
     * 查询序列列表
     */
    querySeries: async function(studyInstanceUid) {
      const response = await fetch(`/monai/datastore/images/${studyInstanceUid}/series`);
      const seriesData = await response.json();
      
      return [{
        studyInstanceUid,
        seriesInstanceUid: seriesData.id,
        modality: seriesData.modality || 'CT',
        seriesNumber: 1,
        seriesDescription: seriesData.description || 'NIfTI Volume',
        numInstances: 1,
      }];
    },
    
    /**
     * 获取系列详情
     */
    getSeries: async function(studyInstanceUid, seriesInstanceUid) {
      const response = await fetch(
        `/monai/datastore/images/${studyInstanceUid}/series/${seriesInstanceUid}`
      );
      const seriesData = await response.json();
      
      return createSeriesMetadata({
        studyInstanceUid,
        seriesInstanceUid,
        ...seriesData,
      });
    },
    
    /**
     * 获取图像加载器
     */
    getImageLoaders: function() {
      return {
        nifti: createNIfTILoader(),
      };
    },
  };
  
  return dataSource;
}

export { createNIfTIDataSource };
```

---

### 3.3 创建NIfTI加载器

**新建文件**: `Viewers/extensions/default/src/NIfTIDataSource/createNIfTILoader.ts`

```typescript
// ========== NIfTI图像加载器 ==========
import * as cornerstone from '@cornerstonejs/core';
import { volumeLoader, imageLoader } from '@cornerstonejs/core';
import { parseNIfTIHeader, decodeNIfTIImage } from './niftiParser';

/**
 * 创建NIfTI加载器
 */
function createNIfTILoader() {
  const loader = {
    name: 'nifti',
    
    /**
     * 检查是否支持该URI
     */
    canLoadURI: function(uri: string) {
      return uri.toLowerCase().endsWith('.nii') || 
             uri.toLowerCase().endsWith('.nii.gz');
    },
    
    /**
     * 加载为体积(Volume)
     */
    loadVolume: async function(volumeId: string) {
      // 解析volumeId: nifti:{fileId}
      const fileId = volumeId.replace('nifti:', '');
      
      // 从后端获取NIfTI文件
      const response = await fetch(`/monai/datastore/image/${fileId}`, {
        headers: { 'Accept': 'application/gzip' },
      });
      
      if (!response.ok) {
        throw new Error(`Failed to load NIfTI file: ${response.statusText}`);
      }
      
      // 获取二进制数据
      const arrayBuffer = await response.arrayBuffer();
      
      // 解析NIfTI头信息
      const header = parseNIfTIHeader(arrayBuffer);
      
      // 解码图像数据
      const imageData = decodeNIfTIImage(arrayBuffer, header);
      
      // 创建Cornerstone体积
      const volume = await cornerstone.volumeLoader.createAndCacheVolume(volumeId, {
        // 维度 [x, y, z]
        dimensions: header.dim.slice(1, 4),
        
        // 体素间距 (mm)
        spacing: header.pixdim.slice(1, 4),
        
        // 原点
        origin: header.qoffset || [0, 0, 0],
        
        // 方向矩阵 (从qform/sform获取)
        direction: header.getDirectionMatrix(),
        
        // 数据类型
        scalarData: imageData,
        
        // 其他元数据
        metadata: {
          modality: header.intentName || 'CT',
          patientName: header.description || 'Unknown',
          seriesDescription: `NIfTI: ${header.description}`,
        },
      });
      
      return volume;
    },
    
    /**
     * 加载为单个图像 (用于切片显示)
     */
    loadImage: async function(imageId: string) {
      // 解析: nifti:{fileId}?slice={z}
      const url = new URL(imageId.replace('nifti:', 'http://localhost/'));
      const fileId = url.pathname;
      const sliceIndex = parseInt(url.searchParams.get('slice') || '0');
      
      // 先加载整个体积
      const volumeId = `nifti:${fileId}`;
      let volume = cornerstone.cache.getVolume(volumeId);
      
      if (!volume) {
        volume = await loader.loadVolume(volumeId);
      }
      
      // 提取指定切片
      const sliceImage = await cornerstone.utilities.extractImageSlice(
        volume,
        sliceIndex
      );
      
      return sliceImage;
    },
  };
  
  return loader;
}

export { createNIfTILoader };
```

---

### 3.4 NIfTI解析器

**新建文件**: `Viewers/extensions/default/src/NIfTIDataSource/niftiParser.ts`

```typescript
// ========== NIfTI文件解析器 ==========
import { inflate } from 'pako';  // gzip解压

interface NIfTIHeader {
  // 必要字段
  dim: number[];           // 维度信息 [dim0, x, y, z, ...]
  pixdim: number[];        // 体素间距 [pix0, dx, dy, dz, ...]
  datatype: number;        // 数据类型代码
  bitpix: number;          // 每个体素的比特数
  
  // 空间变换
  qformCode: number;       // qform变换代码
  sformCode: number;       // sform变换代码
  quaternB: number;        // 四元数参数
  quaternC: number;
  quaternD: number;
  qoffsetX: number;        // 原点偏移
  qoffsetY: number;
  qoffsetZ: number;
  
  // 方向矩阵 (4x4)
  srowX: number[];
  srowY: number[];
  srowZ: number[];
  
  // 其他
  intentCode: number;
  intentName: string;
  description: string;
  
  // 辅助方法
  getDirectionMatrix(): number[];
  getVoxelSize(): number[];
}

/**
 * 解析NIfTI文件头
 */
function parseNIfTIHeader(buffer: ArrayBuffer): NIfTIHeader {
  const dataView = new DataView(buffer);
  const isLittleEndian = true;
  
  // 检查魔数 (348或540)
  const sizeofHdr = dataView.getInt32(0, isLittleEndian);
  
  // 检查是否为gzip压缩
  const isGzipped = (new Uint8Array(buffer, 0, 2)[0] === 0x1f && 
                     new Uint8Array(buffer, 0, 2)[1] === 0x8b);
  
  let headerBuffer: ArrayBuffer;
  let dataBuffer: ArrayBuffer;
  
  if (isGzipped) {
    // 解压gzip
    const inflated = inflate(new Uint8Array(buffer));
    headerBuffer = inflated.buffer;
    dataBuffer = inflated.buffer;
  } else {
    headerBuffer = buffer;
    dataBuffer = buffer;
  }
  
  const headerView = new DataView(headerBuffer);
  
  // 解析维度信息 (偏移40)
  const dim = [];
  for (let i = 0; i < 8; i++) {
    dim.push(headerView.getInt16(40 + i * 2, isLittleEndian));
  }
  
  // 解析体素间距 (偏移80)
  const pixdim = [];
  for (let i = 0; i < 8; i++) {
    pixdim.push(headerView.getFloat32(80 + i * 4, isLittleEndian));
  }
  
  // 解析数据类型 (偏移70)
  const datatype = headerView.getInt16(70, isLittleEndian);
  
  // 解析qform/sform
  const qformCode = headerView.getInt16(252, isLittleEndian);
  const sformCode = headerView.getInt16(254, isLittleEndian);
  
  // 解析四元数
  const quaternB = headerView.getFloat32(256, isLittleEndian);
  const quaternC = headerView.getFloat32(260, isLittleEndian);
  const quaternD = headerView.getFloat32(264, isLittleEndian);
  
  // 解析原点偏移
  const qoffsetX = headerView.getFloat32(268, isLittleEndian);
  const qoffsetY = headerView.getFloat32(272, isLittleEndian);
  const qoffsetZ = headerView.getFloat32(276, isLittleEndian);
  
  // 解析sform矩阵 (4x4, 偏移280)
  const srowX = [], srowY = [], srowZ = [];
  for (let i = 0; i < 4; i++) {
    srowX.push(headerView.getFloat32(280 + i * 4, isLittleEndian));
    srowY.push(headerView.getFloat32(296 + i * 4, isLittleEndian));
    srowZ.push(headerView.getFloat32(312 + i * 4, isLittleEndian));
  }
  
  // 解析描述
  const descriptionBytes = new Uint8Array(headerBuffer, 148, 80);
  const description = new TextDecoder().decode(descriptionBytes).replace(/\0/g, '');
  
  // 数据偏移 (通常为352或544)
  const voxOffset = headerView.getFloat32(108, isLittleEndian);
  
  return {
    dim,
    pixdim,
    datatype,
    bitpix: headerView.getInt16(72, isLittleEndian),
    qformCode,
    sformCode,
    quaternB,
    quaternC,
    quaternD,
    qoffsetX,
    qoffsetY,
    qoffsetZ,
    srowX,
    srowY,
    srowZ,
    intentCode: headerView.getInt16(68, isLittleEndian),
    intentName: '',  // 从偏移328解析
    description,
    
    // 辅助方法
    getDirectionMatrix() {
      if (sformCode > 0) {
        // 使用sform
        return [
          srowX[0], srowX[1], srowX[2],
          srowY[0], srowY[1], srowY[2],
          srowZ[0], srowZ[1], srowZ[2],
        ];
      } else if (qformCode > 0) {
        // 使用qform计算方向矩阵
        return computeQFormMatrix(quaternB, quaternC, quaternD);
      }
      // 默认单位矩阵
      return [1, 0, 0, 0, 1, 0, 0, 0, 1];
    },
    
    getVoxelSize() {
      return [pixdim[1], pixdim[2], pixdim[3]];
    },
  };
}

/**
 * 从四元数计算方向矩阵
 */
function computeQFormMatrix(b: number, c: number, d: number): number[] {
  // 计算a (四元数归一化)
  const a = Math.sqrt(1.0 - (b * b + c * c + d * d));
  
  // 构建旋转矩阵
  const r11 = a * a + b * b - c * c - d * d;
  const r12 = 2 * b * c - 2 * a * d;
  const r13 = 2 * b * d + 2 * a * c;
  
  const r21 = 2 * b * c + 2 * a * d;
  const r22 = a * a + c * c - b * b - d * d;
  const r23 = 2 * c * d - 2 * a * b;
  
  const r31 = 2 * b * d - 2 * a * c;
  const r32 = 2 * c * d + 2 * a * b;
  const r33 = a * a + d * d - b * b - c * c;
  
  return [
    r11, r12, r13,
    r21, r22, r23,
    r31, r32, r33,
  ];
}

/**
 * 解码NIfTI图像数据
 */
function decodeNIfTIImage(buffer: ArrayBuffer, header: NIfTIHeader): Float32Array {
  // 数据偏移 (跳过头部)
  const dataOffset = 352;  // 标准NIfTI-1头部大小
  
  // 计算数据大小
  const nx = header.dim[1];
  const ny = header.dim[2];
  const nz = header.dim[3];
  const dataSize = nx * ny * nz;
  
  // 根据数据类型解码
  let data: Float32Array;
  
  switch (header.datatype) {
    case 2:  // DT_UNSIGNED_CHAR (8-bit)
      const u8 = new Uint8Array(buffer, dataOffset, dataSize);
      data = new Float32Array(dataSize);
      for (let i = 0; i < dataSize; i++) {
        data[i] = u8[i];
      }
      break;
      
    case 4:  // DT_SIGNED_SHORT (16-bit)
      const s16 = new Int16Array(buffer, dataOffset, dataSize);
      data = new Float32Array(dataSize);
      for (let i = 0; i < dataSize; i++) {
        data[i] = s16[i];
      }
      break;
      
    case 8:  // DT_SIGNED_INT (32-bit)
      const s32 = new Int32Array(buffer, dataOffset, dataSize);
      data = new Float32Array(dataSize);
      for (let i = 0; i < dataSize; i++) {
        data[i] = s32[i];
      }
      break;
      
    case 16:  // DT_FLOAT (32-bit float)
      data = new Float32Array(buffer, dataOffset, dataSize);
      break;
      
    case 64:  // DT_DOUBLE (64-bit double)
      const f64 = new Float64Array(buffer, dataOffset, dataSize);
      data = new Float32Array(dataSize);
      for (let i = 0; i < dataSize; i++) {
        data[i] = f64[i];
      }
      break;
      
    default:
      throw new Error(`Unsupported NIfTI datatype: ${header.datatype}`);
  }
  
  return data;
}

export { parseNIfTIHeader, decodeNIfTIImage, NIfTIHeader };
```

---

### 3.5 注册NIfTI数据源

**修改文件**: `Viewers/platform/app/src/App.tsx`

```typescript
// 在数据源注册部分添加NIfTI支持
import { createNIfTIDataSource } from '@ohif/extension-default/src/NIfTIDataSource';

function App() {
  useEffect(() => {
    // 注册数据源
    dataSourceManager.registerDataSource('dicomweb', createDicomWebDataSource(config));
    
    // 注册NIfTI数据源
    dataSourceManager.registerDataSource('nifti', createNIfTIDataSource({
      name: 'nifti',
      configuration: {
        wadoRoot: '/monai/datastore',
      },
    }));
  }, []);
}
```

---

## 4. 后端实现

### 4.1 添加NIfTI数据存储端点

**修改文件**: `monai-label/monailabel/endpoints/datastore.py`

```python
# ========== 添加NIfTI文件处理端点 ==========
import os
import uuid
import gzip
import shutil
from pathlib import Path
from fastapi import APIRouter, File, UploadFile, HTTPException
from fastapi.responses import FileResponse
import nibabel as nib
import numpy as np

router = APIRouter(
    prefix="/datastore",
    tags=["Datastore"],
)

# NIfTI文件存储目录
NIFTI_STORAGE_PATH = Path(os.environ.get("MONAI_LABEL_NIFTI_PATH", "/code/predictions/nifti"))
NIFTI_STORAGE_PATH.mkdir(parents=True, exist_ok=True)


@router.post("/nifti", summary="上传NIfTI文件")
async def upload_nifti_file(
    file: UploadFile = File(...),
    description: str = "",
):
    """
    上传NIfTI (.nii, .nii.gz) 文件
    
    参数:
        file: NIfTI文件
        description: 文件描述
    
    返回:
        {
            "id": 文件ID,
            "name": 文件名,
            "path": 存储路径,
            "dimensions": [x, y, z],
            "spacing": [dx, dy, dz],
            "modality": 模态类型
        }
    """
    # 验证文件扩展名
    filename = file.filename.lower()
    if not (filename.endswith('.nii') or filename.endswith('.nii.gz')):
        raise HTTPException(
            status_code=400,
            detail=f"Invalid file format. Expected .nii or .nii.gz, got {filename}"
        )
    
    # 生成唯一ID
    file_id = str(uuid.uuid4())
    file_name = f"{file_id}.nii.gz"
    file_path = NIFTI_STORAGE_PATH / file_name
    
    try:
        # 保存上传的文件
        with open(file_path, 'wb') as f:
            shutil.copyfileobj(file.file, f)
        
        # 解析NIfTI头信息
        header_info = parse_nifti_header(file_path)
        
        # 保存元数据
        metadata = {
            "id": file_id,
            "name": file.filename,
            "path": str(file_path),
            "upload_date": datetime.now().isoformat(),
            "description": description,
            **header_info,
        }
        
        # 保存元数据到JSON
        meta_path = NIFTI_STORAGE_PATH / f"{file_id}.json"
        with open(meta_path, 'w') as f:
            json.dump(metadata, f, indent=2)
        
        return metadata
        
    except Exception as e:
        # 清理失败的文件
        if file_path.exists():
            file_path.unlink()
        raise HTTPException(status_code=500, detail=f"Failed to process NIfTI file: {str(e)}")


@router.get("/image/{file_id}", summary="获取NIfTI文件")
async def get_nifti_file(file_id: str):
    """获取NIfTI文件数据"""
    file_path = NIFTI_STORAGE_PATH / f"{file_id}.nii.gz"
    
    if not file_path.exists():
        # 尝试非压缩版本
        file_path = NIFTI_STORAGE_PATH / f"{file_id}.nii"
        if not file_path.exists():
            raise HTTPException(status_code=404, detail="File not found")
    
    return FileResponse(
        file_path,
        media_type='application/gzip',
        filename=file_path.name,
    )


@router.get("/images", summary="列出所有NIfTI文件")
async def list_nifti_files(
    format: str = "nifti",
    skip: int = 0,
    limit: int = 100,
):
    """列出已上传的NIfTI文件"""
    files = []
    
    for meta_file in NIFTI_STORAGE_PATH.glob("*.json"):
        with open(meta_file, 'r') as f:
            metadata = json.load(f)
            files.append(metadata)
    
    # 按上传日期排序
    files.sort(key=lambda x: x.get("upload_date", ""), reverse=True)
    
    return files[skip:skip + limit]


def parse_nifti_header(file_path: Path) -> dict:
    """
    解析NIfTI文件头信息
    
    返回:
        {
            "dimensions": [x, y, z],
            "spacing": [dx, dy, dz],
            "origin": [ox, oy, oz],
            "direction": [...],
            "datatype": 数据类型,
            "modality": 模态,
            "size_mb": 文件大小(MB)
        }
    """
    try:
        # 使用nibabel加载
        img = nib.load(str(file_path))
        header = img.header
        
        # 获取维度
        dimensions = list(img.shape[:3])  # 取前3维
        
        # 获取体素间距
        spacing = header.get_zooms()[:3]
        
        # 获取仿射矩阵
        affine = img.affine
        
        # 提取原点和方向
        origin = affine[:3, 3].tolist()
        
        # 提取方向矩阵 (3x3)
        direction = [
            affine[0, 0], affine[0, 1], affine[0, 2],
            affine[1, 0], affine[1, 1], affine[1, 2],
            affine[2, 0], affine[2, 1], affine[2, 2],
        ]
        
        # 数据类型
        datatype = int(header['datatype'])
        
        # 尝试从描述中提取模态
        descrip = header.get('descrip', b'').decode('utf-8', errors='ignore').strip()
        modality = guess_modality(descrip, dimensions)
        
        # 文件大小
        size_mb = file_path.stat().st_size / (1024 * 1024)
        
        return {
            "dimensions": dimensions,
            "spacing": list(spacing),
            "origin": origin,
            "direction": direction,
            "datatype": datatype,
            "modality": modality,
            "size_mb": round(size_mb, 2),
        }
        
    except Exception as e:
        raise ValueError(f"Failed to parse NIfTI header: {str(e)}")


def guess_modality(description: str, dimensions: list) -> str:
    """根据描述和维度猜测模态"""
    desc_lower = description.lower()
    
    if 'ct' in desc_lower or 'computerized' in desc_lower:
        return 'CT'
    elif 'mr' in desc_lower or 'mri' in desc_lower or 'magnetic' in desc_lower:
        return 'MR'
    elif 'pet' in desc_lower:
        return 'PT'
    elif 'us' in desc_lower or 'ultrasound' in desc_lower:
        return 'US'
    
    # 根据维度猜测 (CT/MR通常是512x512或更高)
    if dimensions[0] >= 256 and dimensions[1] >= 256:
        return 'CT'
    
    return 'OT'  # 其他
```

---

### 4.2 修改推理任务支持NIfTI

**修改文件**: `monai-label/monailabel/tasks/infer/basic_infer.py`

```python
# ========== 在 __call__ 方法中添加NIfTI支持 ==========

def __call__(self, request, callbacks=None):
    """
    主推理入口，支持DICOM和NIfTI输入
    """
    req = copy.deepcopy(self._config)
    req.update(request)
    
    # 检查输入类型
    image_path = req.get("image", "")
    
    if image_path.endswith('.nii') or image_path.endswith('.nii.gz'):
        # NIfTI输入处理
        return self._process_nifti(req)
    else:
        # DICOM输入处理 (原有逻辑)
        return self._process_dicom(req)


def _process_nifti(self, request: Dict) -> Tuple:
    """
    处理NIfTI文件推理
    
    优势:
    - 无需DICOM到NIfTI转换
    - 直接加载numpy数组
    - 更快的预处理
    """
    import nibabel as nib
    
    nifti_path = request["image"]
    
    # 1. 加载NIfTI文件
    img = nib.load(nifti_path)
    img_np = img.get_fdata()
    
    # 2. 确保维度为 (C, H, W, D)
    if img_np.ndim == 3:
        img_np = img_np[np.newaxis, ...]  # 添加通道维
    elif img_np.ndim == 4:
        # 已经是4D，选择第一个通道或取平均
        if img_np.shape[3] > 1:
            img_np = img_np[..., 0:1]  # 取第一个时间帧
        img_np = np.transpose(img_np, (3, 0, 1, 2))  # (H,W,D,C) -> (C,H,W,D)
    
    # 3. 标准化数据类型
    if img_np.dtype != np.float32:
        img_np = img_np.astype(np.float32)
    
    # 4. 获取元数据
    header = img.header
    spacing = header.get_zooms()[:3]
    origin = img.affine[:3, 3]
    
    logger.info(f"NIfTI loaded: shape={img_np.shape}, spacing={spacing}")
    
    # 5. 根据选择的模型执行推理
    model_type = request.get("nninter")
    
    if model_type == True or model_type == "nninter":
        return self._nninteractive_infer_nifti(img_np, request, spacing)
    elif not model_type:
        return self._sam_infer_nifti(img_np, request, spacing)
    else:
        return self._voxtell_infer_nifti(img_np, request)


def _nninteractive_infer_nifti(self, img_np: np.ndarray, request: Dict, spacing: tuple) -> Tuple:
    """
    使用nnInteractive处理NIfTI数据
    """
    import nibabel as nib
    
    # nnInteractive期望的输入格式: (1, Z, Y, X)
    if img_np.shape[0] != 1:
        # 如果是多通道，取第一个或使用特定通道
        img_input = img_np[0:1, ...]
    else:
        img_input = img_np
    
    # 调整维度顺序 (C, Z, Y, X) -> (1, Z, Y, X)
    # nibabel加载的是 (X, Y, Z) 或 (X, Y, Z, C)
    # 我们已经转换为 (C, Z, Y, X)
    
    start_time = time.time()
    
    # 初始化nnInteractive (如果图像变化)
    series_id = request.get("image", "")
    if series_id != getattr(self, '_current_nifti_id', None):
        session.set_image(img_input)
        session.set_target_buffer(torch.zeros(img_input.shape[1:], dtype=torch.uint8))
        self._current_nifti_id = series_id
        self._session_used_interactions = {k: set() for k in self._session_used_interactions}
    
    # 处理提示 (与DICOM版本类似，但无需处理DICOM方向)
    data = request
    result_json = {}
    
    # 坐标转换: NIfTI使用数组索引，需要注意Z轴方向
    # NIfTI通常是RAS方向，而nnInteractive可能有特定要求
    
    # 正点提示
    if len(data.get('pos_points', [])) != 0:
        result_json["pos_points"] = copy.deepcopy(data["pos_points"])
        for point in data['pos_points']:
            if not self.is_prompt_used(point, "pos_points"):
                self.add_prompt(point, "pos_points")
                # NIfTI坐标通常是 [x, y, z]，需要转换为 [z, y, x] (numpy到nnInteractive)
                # 注意: NIfTI的Z轴方向可能需要翻转
                nninteractive_point = (point[2], point[1], point[0])
                session.add_point_interaction(nninteractive_point, include_interaction=True)
    
    # 涂鸦提示
    if len(data.get('pos_scribbles', [])) != 0:
        result_json["pos_scribbles"] = copy.deepcopy(data["pos_scribbles"])
        for scribble in data['pos_scribbles']:
            if not self.is_prompt_used(scribble, "pos_scribbles"):
                self.add_prompt(scribble, "pos_scribbles")
                scribble = clean_and_densify_polyline(scribble)
                scribbleMask = np.zeros(img_input.shape[1:], dtype=np.uint8)
                
                # NIfTI坐标转换
                filled_indices = np.round(np.asarray(scribble)).astype(int)
                z = filled_indices[:, 2]
                y = filled_indices[:, 1]
                x = filled_indices[:, 0]
                
                valid = (
                    (z >= 0) & (z < img_input.shape[1]) &
                    (y >= 0) & (y < img_input.shape[2]) &
                    (x >= 0) & (x < img_input.shape[3])
                )
                scribbleMask[z[valid], y[valid], x[valid]] = 1
                
                session.add_scribble_interaction(scribbleMask, include_interaction=True)
    
    # 边界框提示
    if len(data.get('pos_boxes', [])) != 0:
        result_json["pos_boxes"] = copy.deepcopy(data["pos_boxes"])
        for box in data['pos_boxes']:
            if not self.is_prompt_used(box, "pos_boxes"):
                self.add_prompt(box, "pos_boxes")
                # 坐标转换: [[x1,y1,z1], [x2,y2,z2]] -> [[z1,z2], [y1,y2], [x1,x2]]
                nninteractive_box = [
                    [box[0][2], box[1][2]],  # z
                    [box[0][1], box[1][1]],  # y
                    [box[0][0], box[1][0]],  # x
                ]
                session.add_bbox_interaction(nninteractive_box, include_interaction=True)
    
    # 获取结果
    results = session.target_buffer.clone()
    pred = results.numpy()
    
    elapsed = time.time() - start_time
    logger.info(f"nnInteractive NIfTI inference completed in {elapsed:.2f}s")
    
    # 构建返回结果
    final_result = {
        "prompt_info": result_json,
        "nninter_elapsed": elapsed,
        "spacing": list(spacing),
    }
    
    # 如果需要，保存为NIfTI格式
    if request.get("output_format") == "nifti":
        output_path = self._save_as_nifti(pred, request["image"], spacing)
        return output_path, final_result
    
    return pred, final_result


def _save_as_nifti(self, data: np.ndarray, reference_path: str, spacing: tuple) -> str:
    """保存分割结果为NIfTI格式"""
    import nibabel as nib
    from datetime import datetime
    
    # 加载参考图像获取元数据
    ref_img = nib.load(reference_path)
    
    # 创建新的NIfTI图像
    # 确保数据是uint8
    if data.dtype != np.uint8:
        data = data.astype(np.uint8)
    
    # 创建NIfTI图像
    seg_img = nib.Nifti1Image(data, ref_img.affine, ref_img.header)
    
    # 更新描述
    seg_img.header['descrip'] = f'AI Segmentation {datetime.now().isoformat()}'
    
    # 保存
    output_dir = Path("/code/predictions")
    output_dir.mkdir(parents=True, exist_ok=True)
    
    timestamp = datetime.now().strftime("%Y%m%d%H%M%S")
    output_path = output_dir / f"segmentation_{timestamp}.nii.gz"
    
    nib.save(seg_img, str(output_path))
    
    logger.info(f"Segmentation saved to {output_path}")
    
    return str(output_path)
```

---

### 4.3 添加NIfTI到数据存储接口

**修改文件**: `monai-label/monailabel/datastore/local.py`

```python
# ========== 扩展LocalDatastore支持NIfTI ==========

class LocalDatastore(Datastore):
    """支持DICOM和NIfTI的本地数据存储"""
    
    def __init__(self, path, extensions=None, **kwargs):
        # 添加NIfTI扩展名
        if extensions is None:
            extensions = [".nii.gz", ".nii", ".nrrd", ".dcm"]
        
        super().__init__(path, extensions=extensions, **kwargs)
        
        # NIfTI专用存储路径
        self._nifti_path = Path(path) / "nifti"
        self._nifti_path.mkdir(parents=True, exist_ok=True)
    
    def get_image_uri(self, image_id: str) -> str:
        """
        获取图像文件路径
        支持DICOM目录或NIfTI文件
        """
        # 检查是否为NIfTI ID (UUID格式)
        if self._is_nifti_id(image_id):
            nifti_file = self._nifti_path / f"{image_id}.nii.gz"
            if nifti_file.exists():
                return str(nifti_file)
            
            # 尝试非压缩版本
            nifti_file = self._nifti_path / f"{image_id}.nii"
            if nifti_file.exists():
                return str(nifti_file)
        
        # 原有DICOM逻辑
        return super().get_image_uri(image_id)
    
    def _is_nifti_id(self, image_id: str) -> bool:
        """检查ID是否为NIfTI文件ID (UUID格式)"""
        try:
            uuid.UUID(image_id)
            return True
        except ValueError:
            return False
    
    def add_nifti_image(self, file_path: Path, metadata: dict) -> str:
        """添加NIfTI图像到存储"""
        import shutil
        
        # 生成唯一ID
        image_id = str(uuid.uuid4())
        
        # 目标路径
        target_path = self._nifti_path / f"{image_id}.nii.gz"
        
        # 复制文件
        shutil.copy(file_path, target_path)
        
        # 保存元数据
        meta_path = self._nifti_path / f"{image_id}.json"
        with open(meta_path, 'w') as f:
            json.dump(metadata, f)
        
        return image_id
```

---

## 5. 数据流改造

### 5.1 上传流程对比

```
DICOM上传流程:
┌─────────┐    ┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│ 用户选择 │───▶│  前端上传   │───▶│  Orthanc    │───▶│  DICOM Web  │
│ DICOM文件│    │             │    │   PACS      │    │   加载显示  │
└─────────┘    └─────────────┘    └─────────────┘    └─────────────┘

NIfTI上传流程:
┌─────────┐    ┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│ 用户选择 │───▶│  前端上传   │───▶│ 后端直接    │───▶│  NIfTI      │
│ NIfTI文件│    │             │    │ 存储解析    │    │ 加载器显示  │
└─────────┘    └─────────────┘    └─────────────┘    └─────────────┘
     │                                                │
     │         更简洁，无需PACS中转                    │
     └────────────────────────────────────────────────┘
```

### 5.2 推理流程对比

```
DICOM推理:
1. 读取DICOM目录
2. 使用SimpleITK读取为Image
3. 转换为numpy (Z, Y, X)
4. 处理DICOM方向 (可能需要翻转)
5. 执行AI推理
6. 结果转回DICOM方向

NIfTI推理:
1. 直接读取.nii.gz文件
2. nibabel加载为numpy
3. 已经是numpy数组
4. 方向已编码在affine矩阵中
5. 执行AI推理
6. 直接保存为NIfTI (保留原affine)
```

---

## 6. 完整代码实现

### 6.1 前端NIfTI SOP Class Handler

**新建文件**: `Viewers/extensions/cornerstone/src/getNIfTIHandler.ts`

```typescript
/**
 * NIfTI文件处理器
 * 将NIfTI文件注册为显示集
 */
import { SOPClassHandlerId } from './enums';

const NIfTI_LOADER_SCHEME = 'nifti';

function getNIfTISopClassHandler(servicesManager: ServicesManager) {
  const { displaySetService } = servicesManager.services;

  return {
    name: 'NIfTI',
    sopClassUids: ['1.2.840.10008.5.1.4.1.1.77.1.5'],  // 私有SOP Class UID
    
    getDisplaySetsFromSeries: function(series) {
      const displaySet = {
        Modality: series.modality || 'CT',
        SeriesDescription: series.seriesDescription || 'NIfTI Volume',
        SeriesNumber: series.seriesNumber || 1,
        SeriesInstanceUID: series.seriesInstanceUid,
        StudyInstanceUID: series.studyInstanceUid,
        SOPClassHandlerId: 'NIfTI',
        SOPClassUID: '1.2.840.10008.5.1.4.1.1.77.1.5',
        
        // NIfTI特定属性
        url: `${NIfTI_LOADER_SCHEME}:${series.fileId}`,
        isReconstructable: true,
        is3D: true,
        
        // 维度信息
        numImageFrames: series.dimensions?.[2] || 1,
        frameRate: 0,
        
        // 加载器配置
        load: function() {
          return loadNIfTIDisplaySet(this, servicesManager);
        },
      };
      
      return [displaySet];
    },
  };
}

async function loadNIfTIDisplaySet(displaySet, servicesManager) {
  const { cornerstoneViewportService } = servicesManager.services;
  
  // 使用NIfTI加载器加载体积
  const volumeId = displaySet.url;
  
  const volume = await cornerstone.volumeLoader.createAndCacheVolume(volumeId, {
    // 体积属性
  });
  
  await volume.load();
  
  return volume;
}

export default getNIfTISopClassHandler;
```

### 6.2 后端批量NIfTI导入工具

**新建文件**: `monai-label/monailabel/scripts/import_msd.py`

```python
#!/usr/bin/env python
"""
MSD数据集导入工具
批量导入MSD格式的NIfTI文件

使用方法:
    python import_msd.py --input /path/to/Task03_Liver --modality CT
"""

import argparse
import json
import os
from pathlib import Path
from typing import List, Dict
import nibabel as nib
import requests


def parse_msd_dataset(dataset_path: Path) -> Dict:
    """解析MSD数据集结构"""
    
    dataset_info = {
        "name": dataset_path.name,
        "imagesTr": [],
        "labelsTr": [],
        "imagesTs": [],
    }
    
    # 读取dataset.json
    json_file = dataset_path / "dataset.json"
    if json_file.exists():
        with open(json_file, 'r') as f:
            msd_info = json.load(f)
            dataset_info.update(msd_info)
    
    # 扫描imagesTr目录
    images_tr_dir = dataset_path / "imagesTr"
    if images_tr_dir.exists():
        dataset_info["imagesTr"] = sorted([
            f.name for f in images_tr_dir.glob("*.nii.gz")
        ])
    
    # 扫描labelsTr目录
    labels_tr_dir = dataset_path / "labelsTr"
    if labels_tr_dir.exists():
        dataset_info["labelsTr"] = sorted([
            f.name for f in labels_tr_dir.glob("*.nii.gz")
        ])
    
    return dataset_info


def import_nifti_to_monai(
    file_path: Path,
    server_url: str = "http://localhost:8002",
    description: str = "",
) -> str:
    """
    导入单个NIfTI文件到MONAI Label
    
    返回:
        文件ID
    """
    import requests
    
    url = f"{server_url}/monai/datastore/nifti"
    
    with open(file_path, 'rb') as f:
        files = {'file': (file_path.name, f, 'application/gzip')}
        data = {'description': description}
        
        response = requests.post(url, files=files, data=data)
        response.raise_for_status()
        
        result = response.json()
        return result["id"]


def batch_import_msd(
    dataset_path: Path,
    server_url: str = "http://localhost:8002",
    import_labels: bool = True,
):
    """批量导入MSD数据集"""
    
    dataset_info = parse_msd_dataset(dataset_path)
    print(f"Importing MSD dataset: {dataset_info['name']}")
    print(f"Training images: {len(dataset_info['imagesTr'])}")
    print(f"Training labels: {len(dataset_info['labelsTr'])}")
    
    imported = []
    
    # 导入训练图像
    images_tr_dir = dataset_path / "imagesTr"
    for img_name in dataset_info['imagesTr']:
        img_path = images_tr_dir / img_name
        
        try:
            file_id = import_nifti_to_monai(
                img_path,
                server_url,
                description=f"MSD {dataset_info['name']}: {img_name}"
            )
            
            imported.append({
                "name": img_name,
                "id": file_id,
                "type": "image",
            })
            
            print(f"✓ Imported: {img_name} -> {file_id}")
            
        except Exception as e:
            print(f"✗ Failed to import {img_name}: {e}")
    
    # 导入标签 (可选)
    if import_labels and dataset_info['labelsTr']:
        labels_tr_dir = dataset_path / "labelsTr"
        for label_name in dataset_info['labelsTr']:
            label_path = labels_tr_dir / label_name
            
            try:
                # 标签作为已有图像的标签上传
                # 需要匹配图像ID
                pass
                
            except Exception as e:
                print(f"✗ Failed to import label {label_name}: {e}")
    
    # 保存导入记录
    output_file = dataset_path / "imported_monai.json"
    with open(output_file, 'w') as f:
        json.dump(imported, f, indent=2)
    
    print(f"\nImport complete! {len(imported)} files imported.")
    print(f"Import record saved to: {output_file}")


def main():
    parser = argparse.ArgumentParser(description="Import MSD dataset to MONAI Label")
    parser.add_argument("--input", "-i", required=True, help="MSD dataset path")
    parser.add_argument("--server", "-s", default="http://localhost:8002", 
                        help="MONAI Label server URL")
    parser.add_argument("--no-labels", action="store_true", 
                        help="Skip importing labels")
    
    args = parser.parse_args()
    
    dataset_path = Path(args.input)
    if not dataset_path.exists():
        print(f"Error: Dataset path does not exist: {dataset_path}")
        return
    
    batch_import_msd(
        dataset_path,
        server_url=args.server,
        import_labels=not args.no_labels,
    )


if __name__ == "__main__":
    main()
```

---

## 7. 测试验证

### 7.1 单元测试

**新建文件**: `monai-label/tests/unit/test_nifti.py`

```python
import pytest
import tempfile
import numpy as np
import nibabel as nib
from pathlib import Path

from monailabel.datastore.local import LocalDatastore
from monailabel.endpoints.datastore import parse_nifti_header


class TestNIfTI:
    """NIfTI文件处理测试"""
    
    @pytest.fixture
    def sample_nifti(self):
        """创建示例NIfTI文件"""
        with tempfile.NamedTemporaryFile(suffix='.nii.gz', delete=False) as f:
            # 创建3D数据
            data = np.random.rand(64, 64, 32).astype(np.float32)
            
            # 创建NIfTI图像
            img = nib.Nifti1Image(data, np.eye(4))
            nib.save(img, f.name)
            
            return f.name
    
    def test_parse_nifti_header(self, sample_nifti):
        """测试NIfTI头信息解析"""
        header_info = parse_nifti_header(Path(sample_nifti))
        
        assert header_info["dimensions"] == [64, 64, 32]
        assert header_info["datatype"] == 16  # DT_FLOAT
        assert "spacing" in header_info
    
    def test_nifti_datastore(self, sample_nifti, tmp_path):
        """测试NIfTI数据存储"""
        datastore = LocalDatastore(str(tmp_path))
        
        # 添加NIfTI文件
        metadata = {
            "name": "test.nii.gz",
            "modality": "CT",
        }
        image_id = datastore.add_nifti_image(Path(sample_nifti), metadata)
        
        assert image_id is not None
        
        # 获取文件路径
        uri = datastore.get_image_uri(image_id)
        assert Path(uri).exists()
    
    def test_nifti_inference(self, sample_nifti):
        """测试NIfTI推理流程"""
        from monailabel.tasks.infer.basic_infer import BasicInferTask
        
        # 创建推理任务
        task = BasicInferTask(
            path=None,
            network=None,
            type="segmentation",
            labels=["organ"],
            dimension=3,
            description="Test NIfTI inference",
        )
        
        # 准备请求
        request = {
            "model": "segmentation",
            "image": sample_nifti,
            "nninter": True,
            "pos_points": [[32, 32, 16]],
        }
        
        # 执行推理
        result_file, result_json = task(request)
        
        assert result_file is not None
        assert "nninter_elapsed" in result_json
```

### 7.2 集成测试

**测试步骤**:

```bash
# 1. 启动服务
docker compose up -d

# 2. 上传测试NIfTI文件
curl -X POST \
  http://localhost:8002/monai/datastore/nifti \
  -F "file=@Task03_Liver/imagesTr/liver_001.nii.gz" \
  -F "description=MSD Liver Test"

# 3. 查看文件列表
curl http://localhost:8002/monai/datastore/images?format=nifti

# 4. 执行推理
curl -X POST \
  http://localhost:8002/monai/infer/segmentation \
  -F "model=segmentation" \
  -F "image={file_id}" \
  -F 'params={"nninter": true, "pos_points": [[128, 128, 64]]}' \
  -F "output=dicom_seg"
```

---

## 8. 总结

### 修改文件清单

| 文件路径 | 修改类型 | 说明 |
|---------|---------|------|
| `Viewers/extensions/cornerstone/src/components/DicomUpload/DicomUpload.tsx` | 修改 | 添加NIfTI上传支持 |
| `Viewers/extensions/default/src/NIfTIDataSource/index.ts` | 新建 | NIfTI数据源 |
| `Viewers/extensions/default/src/NIfTIDataSource/createNIfTILoader.ts` | 新建 | NIfTI加载器 |
| `Viewers/extensions/default/src/NIfTIDataSource/niftiParser.ts` | 新建 | NIfTI解析器 |
| `Viewers/platform/app/src/App.tsx` | 修改 | 注册NIfTI数据源 |
| `monai-label/monailabel/endpoints/datastore.py` | 修改 | 添加NIfTI上传端点 |
| `monai-label/monailabel/tasks/infer/basic_infer.py` | 修改 | 添加NIfTI推理支持 |
| `monai-label/monailabel/datastore/local.py` | 修改 | 扩展NIfTI存储 |
| `monai-label/monailabel/scripts/import_msd.py` | 新建 | MSD批量导入工具 |

### 关键优势

1. **无需PACS**: 直接上传NIfTI，无需Orthanc中转
2. **更快加载**: 无需DICOM到NIfTI转换
3. **保留空间信息**: NIfTI的affine矩阵完整保留
4. **批量导入**: 支持MSD数据集批量导入
5. **双向支持**: 同时支持DICOM和NIfTI

---

*本文档提供了完整的NIfTI支持实现方案，更多细节请参考具体实现代码*
