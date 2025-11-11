---
name: walrus-code-generator
description: 自动生成 Walrus TypeScript SDK 集成代码和最佳实践模板
parameters:
  - name: use-case
    type: string
    description: 使用场景 (basic-upload/advanced-storage/browser-app/batch-processing)
    required: true
  - name: framework
    type: string
    description: 开发框架 (react/vue/nodejs/nextjs/express)
    default: nodejs
  - name: features
    type: array
    description: 需要的功能特性
    default: []
  - name: environment
    type: string
    description: 部署环境 (development/production)
    default: development
---

# Walrus 代码生成技能

## 技能概述

`walrus-code-generator` 是一个专业的代码生成助手，根据您的具体需求自动生成 Walrus TypeScript SDK 集成代码。支持多种使用场景、开发框架和部署环境，确保生成的代码符合最佳实践。

## 支持的使用场景

### 1. 基础文件上传 (basic-upload)

```typescript
// 自动生成的基础上传服务
skill: "walrus-code-generator"
// use-case: basic-upload
// framework: nodejs
// features: ["error-handling", "progress-tracking"]

export class BasicWalrusUploader {
  private client: any;
  private signer: any;

  constructor(network: 'testnet' | 'mainnet' = 'testnet') {
    const suiClient = new SuiJsonRpcClient({
      url: getFullnodeUrl(network),
      network,
    }).$extend(walrus());

    this.client = suiClient;
    this.signer = this.createSigner();
  }

  async uploadFile(
    filePath: string,
    options: UploadOptions = {}
  ): Promise<UploadResult> {
    const {
      epochs = 10,
      deletable = true,
      tags = {},
      onProgress = () => {}
    } = options;

    try {
      onProgress(25, '读取文件...');

      const fs = await import('fs/promises');
      const fileContent = await fs.readFile(filePath);
      const fileName = filePath.split('/').pop() || 'unknown';

      onProgress(50, '创建 Walrus 文件...');

      const file = WalrusFile.from({
        contents: fileContent,
        identifier: `uploads/${Date.now()}_${fileName}`,
        mimeType: this.getMimeType(filePath),
        attributes: {
          'original-path': filePath,
          'file-size': fileContent.length.toString(),
          'upload-time': new Date().toISOString(),
          ...tags
        }
      });

      onProgress(75, '上传到网络...');

      const result = await this.client.walrus.writeFiles({
        files: [file],
        epochs,
        deletable,
        signer: this.signer
      });

      onProgress(100, '上传完成');

      return {
        success: true,
        blobId: result[0].blobId,
        fileName,
        size: fileContent.length,
        storedUntil: result[0].storedUntil
      };

    } catch (error) {
      console.error('上传失败:', error.message);
      throw new Error(`上传失败: ${error.message}`);
    }
  }

  private createSigner(): Ed25519Keypair {
    const privateKey = process.env.WALRUS_PRIVATE_KEY;
    if (!privateKey) {
      throw new Error('未找到私钥环境变量 WALRUS_PRIVATE_KEY');
    }
    return Ed25519Keypair.fromSecretKey(privateKey);
  }

  private getMimeType(filePath: string): string {
    const ext = filePath.split('.').pop()?.toLowerCase();
    const mimeTypes: Record<string, string> = {
      'txt': 'text/plain',
      'pdf': 'application/pdf',
      'jpg': 'image/jpeg',
      'png': 'image/png',
      'json': 'application/json'
    };
    return mimeTypes[ext || ''] || 'application/octet-stream';
  }
}

interface UploadOptions {
  epochs?: number;
  deletable?: boolean;
  tags?: Record<string, string>;
  onProgress?: (progress: number, message: string) => void;
}

interface UploadResult {
  success: boolean;
  blobId: string;
  fileName: string;
  size: number;
  storedUntil: Date;
}
```

### 2. 高级存储管理 (advanced-storage)

```typescript
// 自动生成的高级存储管理服务
skill: "walrus-code-generator"
// use-case: advanced-storage
// framework: nodejs
// features: ["batch-operations", "retry-logic", "cache-management", "metadata-indexing"]

export class AdvancedWalrusStorage {
  private client: any;
  private signer: any;
  private cache: Map<string, CachedFile> = new Map();
  private retryConfig: RetryConfig;

  constructor(options: AdvancedStorageOptions = {}) {
    const {
      network = 'testnet',
      retryAttempts = 3,
      retryDelay = 1000,
      cacheEnabled = true,
      cacheTTL = 300000 // 5分钟
    } = options;

    const suiClient = new SuiJsonRpcClient({
      url: getFullnodeUrl(network),
      network,
    }).$extend(walrus({
      storageNodeClientOptions: {
        timeout: 60000,
        onError: (error) => console.warn('存储节点错误:', error)
      }
    }));

    this.client = suiClient;
    this.signer = this.createSigner();
    this.retryConfig = { attempts: retryAttempts, delay: retryDelay };
    this.cacheEnabled = cacheEnabled;
    this.cacheTTL = cacheTTL;
  }

  async batchUpload(
    files: FileUploadRequest[],
    options: BatchUploadOptions = {}
  ): Promise<BatchUploadResult> {
    const {
      epochs = 10,
      deletable = true,
      maxConcurrency = 3,
      commonTags = {}
    } = options;

    const results: UploadResult[] = [];
    const errors: UploadError[] = [];

    // 分批处理
    for (let i = 0; i < files.length; i += maxConcurrency) {
      const batch = files.slice(i, i + maxConcurrency);

      const batchPromises = batch.map(async (fileRequest, index) => {
        try {
          const result = await this.uploadWithRetry(fileRequest, {
            epochs,
            deletable,
            tags: { ...commonTags, ...fileRequest.tags }
          });

          results.push({
            ...result,
            originalPath: fileRequest.path,
            batchIndex: i + index
          });

        } catch (error) {
          errors.push({
            originalPath: fileRequest.path,
            error: error.message,
            batchIndex: i + index
          });
        }
      });

      await Promise.all(batchPromises);

      // 批次间延迟
      if (i + maxConcurrency < files.length) {
        await new Promise(resolve => setTimeout(resolve, 500));
      }
    }

    return {
      totalFiles: files.length,
      successful: results.length,
      failed: errors.length,
      results,
      errors
    };
  }

  async smartDownload(
    blobId: string,
    options: SmartDownloadOptions = {}
  ): Promise<SmartDownloadResult> {
    const {
      forceRefresh = false,
      verifyIntegrity = true,
      cacheKey = blobId
    } = options;

    // 检查缓存
    if (!forceRefresh && this.cacheEnabled) {
      const cached = this.getFromCache(cacheKey);
      if (cached) {
        console.log('从缓存返回文件:', blobId);
        return {
          ...cached,
          fromCache: true
        };
      }
    }

    try {
      const result = await this.downloadWithVerification(blobId, verifyIntegrity);

      // 缓存结果
      if (this.cacheEnabled) {
        this.setToCache(cacheKey, result);
      }

      return {
        ...result,
        fromCache: false
      };

    } catch (error) {
      throw new Error(`智能下载失败: ${error.message}`);
    }
  }

  async searchByTags(tags: Record<string, string>): Promise<SearchResult[]> {
    // 这是一个概念性实现
    // 实际搜索需要基于您自己的元数据索引系统
    const indexedFiles = await this.getIndexedFiles();

    return indexedFiles.filter(file => {
      return Object.entries(tags).every(([key, value]) =>
        file.tags[key] === value
      );
    });
  }

  private async uploadWithRetry(
    fileRequest: FileUploadRequest,
    options: UploadOptions
  ): Promise<UploadResult> {
    const { attempts, delay } = this.retryConfig;

    for (let attempt = 1; attempt <= attempts; attempt++) {
      try {
        return await this.uploadSingleFile(fileRequest, options);
      } catch (error) {
        if (attempt < attempts && this.isRetryableError(error)) {
          console.log(`上传失败，第 ${attempt} 次重试...`);
          await new Promise(resolve => setTimeout(resolve, delay * attempt));
          continue;
        }
        throw error;
      }
    }

    throw new Error('上传失败，已达到最大重试次数');
  }

  private async uploadSingleFile(
    fileRequest: FileUploadRequest,
    options: UploadOptions
  ): Promise<UploadResult> {
    const fs = await import('fs/promises');
    const fileContent = await fs.readFile(fileRequest.path);

    const file = WalrusFile.from({
      contents: fileContent,
      identifier: fileRequest.identifier || `files/${Date.now()}_${fileRequest.path}`,
      mimeType: fileRequest.mimeType || this.getMimeType(fileRequest.path),
      attributes: {
        'original-path': fileRequest.path,
        'file-size': fileContent.length.toString(),
        'upload-time': new Date().toISOString(),
        ...options.tags
      }
    });

    const result = await this.client.walrus.writeFiles({
      files: [file],
      epochs: options.epochs,
      deletable: options.deletable,
      signer: this.signer
    });

    // 索引文件元数据
    await this.indexFile({
      blobId: result[0].blobId,
      identifier: file.identifier,
      path: fileRequest.path,
      size: fileContent.length,
      tags: options.tags,
      uploadTime: new Date()
    });

    return {
      success: true,
      blobId: result[0].blobId,
      identifier: file.identifier,
      size: fileContent.length,
      storedUntil: result[0].storedUntil
    };
  }

  private async downloadWithVerification(
    blobId: string,
    verify: boolean
  ): Promise<SmartDownloadResult> {
    const [file] = await this.client.walrus.getFiles({ ids: [blobId] });

    const exists = await file.exists();
    if (!exists) {
      throw new Error(`文件不存在: ${blobId}`);
    }

    const content = await file.bytes();
    let verified = true;

    if (verify) {
      const verification = await this.verifyFileIntegrity(blobId, content);
      verified = verification.valid;
    }

    return {
      blobId,
      content: Buffer.from(content),
      size: content.length,
      storedUntil: await file.storedUntil(),
      verified,
      metadata: await this.getFileMetadata(file)
    };
  }

  private async verifyFileIntegrity(
    blobId: string,
    content: Uint8Array
  ): Promise<{ valid: boolean; error?: string }> {
    try {
      // 基本完整性检查
      if (content.length === 0) {
        return { valid: false, error: '文件内容为空' };
      }

      // 这里可以添加更复杂的验证逻辑
      return { valid: true };
    } catch (error) {
      return { valid: false, error: error.message };
    }
  }

  // 缓存管理
  private getFromCache(key: string): CachedFile | null {
    const cached = this.cache.get(key);
    if (cached && Date.now() - cached.timestamp < this.cacheTTL) {
      return cached;
    }
    this.cache.delete(key);
    return null;
  }

  private setToCache(key: string, file: SmartDownloadResult): void {
    this.cache.set(key, {
      ...file,
      timestamp: Date.now()
    });
  }

  private isRetryableError(error: any): boolean {
    return error instanceof RetryableWalrusClientError ||
           error.message.includes('timeout') ||
           error.message.includes('network');
  }
}

// 类型定义
interface FileUploadRequest {
  path: string;
  identifier?: string;
  mimeType?: string;
  tags?: Record<string, string>;
}

interface BatchUploadOptions {
  epochs?: number;
  deletable?: boolean;
  maxConcurrency?: number;
  commonTags?: Record<string, string>;
}

interface BatchUploadResult {
  totalFiles: number;
  successful: number;
  failed: number;
  results: (UploadResult & { originalPath: string; batchIndex: number })[];
  errors: UploadError[];
}

interface SmartDownloadOptions {
  forceRefresh?: boolean;
  verifyIntegrity?: boolean;
  cacheKey?: string;
}

interface SmartDownloadResult {
  blobId: string;
  content: Buffer;
  size: number;
  storedUntil: Date;
  verified: boolean;
  metadata: any;
  fromCache: boolean;
}

interface CachedFile extends SmartDownloadResult {
  timestamp: number;
}

interface RetryConfig {
  attempts: number;
  delay: number;
}
```

### 3. React 浏览器应用 (browser-app)

```typescript
// 自动生成的 React Hook 和组件
skill: "walrus-code-generator"
// use-case: browser-app
// framework: react
// features: ["wallet-integration", "progress-tracking", "drag-drop"]

import React, { useState, useCallback, createContext, useContext } from 'react';
import { useWallet } from '@suiet/wallet-kit';

// Walrus Context
const WalrusContext = createContext<{
  upload: (files: FileList) => Promise<UploadResult[]>;
  download: (blobId: string, filename?: string) => Promise<void>;
  isUploading: boolean;
} | null>(null);

export const WalrusProvider: React.FC<{ children: React.ReactNode }> = ({ children }) => {
  const { connected, signAndExecuteTransactionBlock } = useWallet();
  const [isUploading, setIsUploading] = useState(false);

  const upload = useCallback(async (files: FileList): Promise<UploadResult[]> => {
    if (!connected) {
      throw new Error('请先连接钱包');
    }

    setIsUploading(true);
    const results: UploadResult[] = [];

    try {
      for (let i = 0; i < files.length; i++) {
        const file = files[i];

        try {
          const result = await uploadFile(file, signAndExecuteTransactionBlock);
          results.push({ ...result, originalName: file.name, index: i });
        } catch (error) {
          results.push({
            success: false,
            originalName: file.name,
            index: i,
            error: error.message
          });
        }
      }
    } finally {
      setIsUploading(false);
    }

    return results;
  }, [connected, signAndExecuteTransactionBlock]);

  const download = useCallback(async (blobId: string, filename?: string) => {
    await downloadFile(blobId, filename);
  }, []);

  return (
    <WalrusContext.Provider value={{ upload, download, isUploading }}>
      {children}
    </WalrusContext.Provider>
  );
};

export const useWalrus = () => {
  const context = useContext(WalrusContext);
  if (!context) {
    throw new Error('useWalrus must be used within WalrusProvider');
  }
  return context;
};

// 文件上传组件
export const WalrusUpload: React.FC<{
  onUploadComplete?: (results: UploadResult[]) => void;
  maxFiles?: number;
  acceptedTypes?: string;
}> = ({ onUploadComplete, maxFiles = 10, acceptedTypes = '*' }) => {
  const { upload, isUploading } = useWalrus();
  const [dragActive, setDragActive] = useState(false);
  const [progress, setProgress] = useState(0);
  const fileInputRef = React.useRef<HTMLInputElement>(null);

  const handleFiles = useCallback(async (files: FileList) => {
    if (files.length > maxFiles) {
      alert(`最多只能上传 ${maxFiles} 个文件`);
      return;
    }

    try {
      setProgress(10);
      const results = await upload(files);
      setProgress(100);
      onUploadComplete?.(results);
    } catch (error) {
      alert(`上传失败: ${error.message}`);
      setProgress(0);
    }
  }, [upload, maxFiles, onUploadComplete]);

  const handleDrop = useCallback((e: React.DragEvent) => {
    e.preventDefault();
    e.stopPropagation();
    setDragActive(false);

    const files = e.dataTransfer.files;
    if (files.length > 0) {
      handleFiles(files);
    }
  }, [handleFiles]);

  const handleChange = useCallback((e: React.ChangeEvent<HTMLInputElement>) => {
    const files = e.target.files;
    if (files && files.length > 0) {
      handleFiles(files);
    }
  }, [handleFiles]);

  return (
    <div className="walrus-upload">
      <div
        className={`upload-area ${dragActive ? 'active' : ''} ${isUploading ? 'uploading' : ''}`}
        onDragEnter={(e) => {
          e.preventDefault();
          e.stopPropagation();
          setDragActive(true);
        }}
        onDragLeave={(e) => {
          e.preventDefault();
          e.stopPropagation();
          setDragActive(false);
        }}
        onDragOver={(e) => {
          e.preventDefault();
          e.stopPropagation();
        }}
        onDrop={handleDrop}
        onClick={() => fileInputRef.current?.click()}
      >
        <input
          ref={fileInputRef}
          type="file"
          multiple
          accept={acceptedTypes}
          onChange={handleChange}
          style={{ display: 'none' }}
        />

        {isUploading ? (
          <div className="upload-progress">
            <div className="progress-bar">
              <div
                className="progress-fill"
                style={{ width: `${progress}%` }}
              />
            </div>
            <p>上传中... {progress}%</p>
          </div>
        ) : (
          <div className="upload-prompt">
            <div className="upload-icon">📁</div>
            <h3>拖拽文件到此处或点击上传</h3>
            <p>支持最多 {maxFiles} 个文件</p>
          </div>
        )}
      </div>
    </div>
  );
};

// 文件列表组件
export const WalrusFileList: React.FC<{
  files: UploadResult[];
  onDownload?: (blobId: string, filename: string) => void;
}> = ({ files, onDownload }) => {
  const { download } = useWalrus();

  const handleDownload = useCallback(async (blobId: string, filename: string) => {
    try {
      await onDownload ? onDownload(blobId, filename) : download(blobId, filename);
    } catch (error) {
      alert(`下载失败: ${error.message}`);
    }
  }, [download, onDownload]);

  return (
    <div className="file-list">
      <h3>文件列表</h3>
      {files.map((file, index) => (
        <div key={index} className={`file-item ${file.success ? 'success' : 'error'}`}>
          <div className="file-info">
            <span className="file-name">{file.originalName}</span>
            {file.success ? (
              <span className="file-success">✓ 上传成功</span>
            ) : (
              <span className="file-error">✗ {file.error}</span>
            )}
          </div>
          {file.success && file.blobId && (
            <div className="file-actions">
              <button
                onClick={() => handleDownload(file.blobId!, file.originalName)}
                className="download-btn"
              >
                下载
              </button>
              <span className="blob-id">{file.blobId.slice(0, 16)}...</span>
            </div>
          )}
        </div>
      ))}
    </div>
  );
};

// 辅助函数
async function uploadFile(
  file: File,
  signAndExecuteTransactionBlock: any
): Promise<UploadResult> {
  const arrayBuffer = await file.arrayBuffer();
  const walrusFile = WalrusFile.from({
    contents: new Uint8Array(arrayBuffer),
    identifier: `browser/${Date.now()}_${file.name}`,
    mimeType: file.type,
    attributes: {
      'original-name': file.name,
      'file-size': file.size.toString(),
      'upload-source': 'browser'
    }
  });

  const suiClient = new SuiClient({
    url: getFullnodeUrl('testnet'),
  });

  const walrusClient = new WalrusClient({
    network: 'testnet',
    suiClient,
  });

  const writeFlow = walrusClient.walrus.writeFilesFlow({
    files: [walrusFile],
    epochs: 10,
    deletable: true
  });

  const result = await writeFlow.executeWithSigner({
    signAndExecuteTransactionBlock
  });

  return {
    success: true,
    blobId: result[0].blobId,
    size: file.size
  };
}

async function downloadFile(blobId: string, filename?: string) {
  // 实现文件下载逻辑
  const link = document.createElement('a');
  link.href = `#download-${blobId}`;
  link.download = filename || 'downloaded-file';
  link.click();
}

interface UploadResult {
  success: boolean;
  blobId?: string;
  size?: number;
  originalName: string;
  index: number;
  error?: string;
}
```

## 使用方法

### 基础代码生成
```typescript
skill: "walrus-code-generator"
// use-case: basic-upload
// framework: nodejs
// features: ["error-handling", "logging"]
```

### React 应用生成
```typescript
skill: "walrus-code-generator"
// use-case: browser-app
// framework: react
// features: ["wallet-integration", "drag-drop", "progress-bar"]
```

### 高级存储系统
```typescript
skill: "walrus-code-generator"
// use-case: advanced-storage
// framework: nodejs
// features: ["batch-operations", "caching", "retry-logic", "metadata-indexing"]
```

### 批量处理系统
```typescript
skill: "walrus-code-generator"
// use-case: batch-processing
// framework: express
// features: ["api-endpoints", "queue-system", "monitoring"]
```

## 生成的代码特性

### 🛡️ 错误处理
- 智能重试机制
- 详细错误信息
- 优雅降级策略

### 📊 进度跟踪
- 实时上传进度
- 批量操作状态
- 性能监控

### 🔧 配置管理
- 灵活的配置选项
- 环境适配
- 类型安全

### 🚀 性能优化
- 缓存机制
- 并发控制
- 内存管理

### 🔒 安全考虑
- 输入验证
- 文件完整性检查
- 访问控制

## 最佳实践

1. **错误处理** - 生成的代码包含完整的错误处理逻辑
2. **类型安全** - 使用 TypeScript 提供完整的类型定义
3. **性能优化** - 包含缓存、批量处理和并发控制
4. **可扩展性** - 模块化设计，易于扩展和维护
5. **监控日志** - 集成日志记录和性能监控

---

*更新时间：2025-11-11*