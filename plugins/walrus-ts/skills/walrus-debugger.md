---
name: walrus-debugger
description: Walrus 错误诊断专家 - 智能分析和解决 Walrus 应用中的问题
parameters:
  - name: error-type
    type: string
    description: 错误类型 (upload/download/network/configuration/timeout/authentication)
    required: true
  - name: error-context
    type: string
    description: 错误信息、代码片段或问题描述
    required: true
  - name: environment
    type: string
    description: 运行环境 (browser/nodejs/nextjs/production)
    default: nodejs
---

# Walrus 错误诊断技能

## 技能概述

`walrus-debugger` 是专业的错误诊断助手，专门分析 Walrus 应用中的各种问题，提供智能的解决方案和预防措施。

## 常见错误类型及解决方案

### 1. 上传失败错误

#### 错误类型: Upload Failed
```typescript
// 错误示例
Error: Upload failed: insufficient storage balance
Error: Upload failed: blob size exceeds maximum limit
Error: Upload failed: transaction rejected

// 诊断和解决方案
class UploadErrorDiagnostics {
  async diagnoseUploadError(error: Error, context: UploadContext): Promise<Diagnosis> {
    const analysis = this.analyzeErrorMessage(error.message);

    switch (analysis.type) {
      case 'INSUFFICIENT_BALANCE':
        return this.diagnoseBalanceError(context);
      case 'SIZE_LIMIT':
        return this.diagnoseSizeError(context);
      case 'TRANSACTION_REJECTED':
        return this.diagnoseTransactionError(error, context);
      case 'TIMEOUT':
        return this.diagnoseTimeoutError(context);
      default:
        return this.diagnoseGenericUploadError(error, context);
    }
  }

  private async diagnoseBalanceError(context: UploadContext): Promise<Diagnosis> {
    const walletBalance = await this.checkWalletBalance(context.signerAddress);

    return {
      errorType: 'INSUFFICIENT_BALANCE',
      description: '钱包余额不足，无法支付存储费用',
      causes: [
        'WAL 代币余额不足',
        'SUI 代币余额不足以支付 gas 费',
        '存储成本计算错误'
      ],
      solutions: [
        {
          title: '检查并充值钱包',
          code: `
// 检查钱包余额
async function checkBalances(address: string) {
  const suiClient = new SuiClient({ url: getFullnodeUrl('testnet') });
  const balance = await suiClient.getBalance({ owner: address });

  console.log('SUI Balance:', balance.totalBalance);

  // 检查 WAL 余额（如果支持）
  const walBalance = await suiClient.getBalance({
    owner: address,
    coinType: '0x2::sui::SUI' // 替换为 WAL coin type
  });

  return { sui: balance.totalBalance, wal: walBalance.totalBalance };
}`
        },
        {
          title: '优化存储策略',
          code: `
// 使用更短的存储周期
const result = await client.walrus.writeFiles({
  files: [file],
  epochs: 5, // 减少存储周期
  deletable: true,
  signer: keypair
});

// 或者使用更小的文件
function compressFile(file: File): Promise<File> {
  return new Promise((resolve) => {
    if (file.size > 10 * 1024 * 1024) { // 10MB
      // 压缩或分割文件
      compressImage(file).then(resolve);
    } else {
      resolve(file);
    }
  });
}`
        }
      ],
      prevention: [
        '定期检查钱包余额',
        '设置余额预警',
        '实施费用估算功能'
      ]
    };
  }

  private async diagnoseSizeError(context: UploadContext): Promise<Diagnosis> {
    const fileStats = await this.analyzeFileSize(context.file);

    return {
      errorType: 'SIZE_LIMIT',
      description: '文件大小超出限制',
      causes: [
        `文件大小 ${fileStats.size}MB 超出限制`,
        '网络环境不支持大文件传输',
        '客户端内存不足'
      ],
      solutions: [
        {
          title: '文件压缩',
          code: `
// 图片压缩
async function compressImage(file: File, quality = 0.8): Promise<File> {
  return new Promise((resolve) => {
    const canvas = document.createElement('canvas');
    const ctx = canvas.getContext('2d')!;
    const img = new Image();

    img.onload = () => {
      // 计算压缩后的尺寸
      const scale = Math.min(1, 1920 / Math.max(img.width, img.height));
      canvas.width = img.width * scale;
      canvas.height = img.height * scale;

      ctx.drawImage(img, 0, 0, canvas.width, canvas.height);

      canvas.toBlob((blob) => {
        resolve(new File([blob!], file.name, { type: file.type }));
      }, file.type, quality);
    };

    img.src = URL.createObjectURL(file);
  });
}

// 文本文件压缩
async function compressTextFile(file: File): Promise<File> {
  const text = await file.text();
  const compressed = await this.compressString(text);
  return new File([compressed], file.name + '.gz', { type: 'application/gzip' });
}`
        },
        {
          title: '文件分割上传',
          code: `
// 分割大文件
async function splitAndUpload(file: File, chunkSize = 5 * 1024 * 1024): Promise<string[]> {
  const chunks: string[] = [];
  const totalChunks = Math.ceil(file.size / chunkSize);

  for (let i = 0; i < totalChunks; i++) {
    const start = i * chunkSize;
    const end = Math.min(start + chunkSize, file.size);
    const chunk = file.slice(start, end);

    const chunkFile = new File([chunk], \`\${file.name}.part\${i}\`);
    const result = await uploadSingleFile(chunkFile);
    chunks.push(result.blobId);
  }

  return chunks;
}

// 创建清单文件记录分割信息
async function createManifest(chunks: string[], originalFile: File): Promise<string> {
  const manifest = {
    originalName: originalFile.name,
    originalSize: originalFile.size,
    chunks: chunks.map((blobId, index) => ({
      blobId,
      index,
      size: originalFile.size / chunks.length
    }))
  };

  const manifestBlob = new Blob([JSON.stringify(manifest)], { type: 'application/json' });
  const manifestFile = new File([manifestBlob], \`\${originalFile.name}.manifest\`);

  const result = await uploadSingleFile(manifestFile);
  return result.blobId;
}`
        }
      ],
      prevention: [
        '预先检查文件大小',
        '实施文件大小限制',
        '提供文件压缩工具'
      ]
    };
  }
}
```

### 2. 网络连接错误

#### 错误类型: Network Error
```typescript
// 错误示例
Error: Network timeout
Error: Connection refused
Error: Failed to fetch

class NetworkErrorDiagnostics {
  async diagnoseNetworkError(error: Error, context: NetworkContext): Promise<Diagnosis> {
    const networkStatus = await this.checkNetworkStatus();

    return {
      errorType: 'NETWORK_ERROR',
      description: '网络连接问题',
      causes: [
        '网络连接不稳定',
        'RPC 节点不可用',
        '防火墙或代理阻止连接',
        'DNS 解析失败'
      ],
      solutions: [
        {
          title: '实施自动重试机制',
          code: `
class RobustWalrusClient {
  private readonly MAX_RETRIES = 3;
  private readonly RETRY_DELAY_BASE = 1000;

  async withRetry<T>(
    operation: () => Promise<T>,
    operationName: string
  ): Promise<T> {
    for (let attempt = 1; attempt <= this.MAX_RETRIES; attempt++) {
      try {
        return await operation();
      } catch (error) {
        if (!this.isRetryableError(error) || attempt === this.MAX_RETRIES) {
          throw new Error(\`\${operationName} failed after \${attempt} attempts: \${error.message}\`);
        }

        const delay = this.RETRY_DELAY_BASE * Math.pow(2, attempt - 1);
        console.warn(\`\${operationName} failed (attempt \${attempt}/\${this.MAX_RETRIES}), retrying in \${delay}ms\`);

        await this.sleep(delay);
      }
    }

    throw new Error('Should not reach here');
  }

  private isRetryableError(error: any): boolean {
    return error instanceof RetryableWalrusClientError ||
           error.message.includes('timeout') ||
           error.message.includes('network') ||
           error.message.includes('connection') ||
           error.status >= 500; // Server errors
  }

  private sleep(ms: number): Promise<void> {
    return new Promise(resolve => setTimeout(resolve, ms));
  }

  async uploadFiles(files: WalrusFile[]): Promise<any> {
    return this.withRetry(
      () => this.client.walrus.writeFiles({ files, epochs: 10, signer: this.signer }),
      'File upload'
    );
  }
}`
        },
        {
          title: '实施节点健康检查',
          code: `
// 节点健康检查
class NodeHealthChecker {
  private nodes: string[] = [
    'https://fullnode.testnet.sui.io',
    'https://fullnode.testnet.sui.io:443',
    // 添加备用节点
  ];

  async getHealthyNode(): Promise<string> {
    for (const node of this.nodes) {
      try {
        const response = await fetch(\`\${node}/\`, {
          method: 'HEAD',
          timeout: 5000
        });

        if (response.ok) {
          console.log(\`Node \${node} is healthy\`);
          return node;
        }
      } catch (error) {
        console.warn(\`Node \${node} is unhealthy: \${error.message}\`);
      }
    }

    throw new Error('No healthy nodes available');
  }

  async testAllNodes(): Promise<NodeHealth[]> {
    const results = await Promise.allSettled(
      this.nodes.map(async (node) => {
        const startTime = Date.now();
        try {
          const response = await fetch(\`\${node}/\`, { method: 'HEAD', timeout: 5000 });
          return {
            node,
            healthy: response.ok,
            responseTime: Date.now() - startTime,
            error: null
          };
        } catch (error) {
          return {
            node,
            healthy: false,
            responseTime: Date.now() - startTime,
            error: error.message
          };
        }
      })
    );

    return results.map(result =>
      result.status === 'fulfilled' ? result.value : null
    ).filter(Boolean) as NodeHealth[];
  }
}`
        },
        {
          title: '配置超时和重试',
          code: `
// 配置自定义超时和重试
const client = new SuiJsonRpcClient({
  url: getFullnodeUrl('testnet'),
  network: 'testnet',
}).$extend(walrus({
  storageNodeClientOptions: {
    timeout: 60000, // 60秒超时
    retryAttempts: 5,
    retryDelay: 2000,
    onError: (error) => {
      console.error('Storage node error:', error);
      // 发送错误到监控系统
      this.trackError('storage_node_error', error);
    }
  }
}));`
        }
      ],
      prevention: [
        '实施网络监控',
        '使用多个备用节点',
        '设置合理的超时时间',
        '实施断路器模式'
      ]
    };
  }

  private async checkNetworkStatus(): Promise<NetworkStatus> {
    try {
      const response = await fetch('https://httpbin.org/ip');
      const ipInfo = await response.json();

      return {
        online: true,
        ip: ipInfo.ip,
        latency: Date.now() - this.startTime
      };
    } catch (error) {
      return {
        online: false,
        error: error.message
      };
    }
  }
}
```

### 3. 配置错误

#### 错误类型: Configuration Error
```typescript
// 错误示例
Error: Invalid network configuration
Error: WASM module failed to load
Error: Invalid signer configuration

class ConfigurationErrorDiagnostics {
  async diagnoseConfigError(error: Error, context: ConfigContext): Promise<Diagnosis> {
    return {
      errorType: 'CONFIGURATION_ERROR',
      description: '配置问题',
      causes: [
        '网络配置错误',
        'WASM 模块路径错误',
        '签名者配置无效',
        '环境变量缺失'
      ],
      solutions: [
        {
          title: '验证配置完整性',
          code: `
// 配置验证器
class ConfigValidator {
  static validateWalrusConfig(config: any): ValidationResult {
    const errors: string[] = [];

    // 验证必需字段
    if (!config.network) {
      errors.push('network is required (testnet/mainnet)');
    }

    if (!config.rpcUrl) {
      errors.push('rpcUrl is required');
    }

    // 验证网络配置
    if (config.network && !['testnet', 'mainnet'].includes(config.network)) {
      errors.push('network must be either "testnet" or "mainnet"');
    }

    // 验证 URL 格式
    if (config.rpcUrl && !this.isValidUrl(config.rpcUrl)) {
      errors.push('rpcUrl must be a valid URL');
    }

    // 验证超时配置
    if (config.timeout && (config.timeout < 1000 || config.timeout > 300000)) {
      errors.push('timeout must be between 1000ms and 300000ms');
    }

    return {
      valid: errors.length === 0,
      errors
    };
  }

  static async validateEnvironment(): Promise<ValidationResult> {
    const errors: string[] = [];

    // 检查环境变量
    const requiredEnvVars = ['WALRUS_NETWORK', 'SUI_RPC_URL'];
    for (const envVar of requiredEnvVars) {
      if (!process.env[envVar]) {
        errors.push(\`Environment variable \${envVar} is required\`);
      }
    }

    // 检查 WASM 支持
    if (typeof window !== 'undefined' && !window.WebAssembly) {
      errors.push('WebAssembly is not supported in this browser');
    }

    // 验证网络连接
    try {
      const response = await fetch(process.env.SUI_RPC_URL!, {
        method: 'HEAD',
        timeout: 5000
      });
      if (!response.ok) {
        errors.push('Cannot connect to SUI RPC endpoint');
      }
    } catch (error) {
      errors.push(\`Network connection failed: \${error.message}\`);
    }

    return {
      valid: errors.length === 0,
      errors
    };
  }

  private static isValidUrl(string: string): boolean {
    try {
      new URL(string);
      return true;
    } catch {
      return false;
    }
  }
}

// 使用配置验证
const config = {
  network: 'testnet',
  rpcUrl: getFullnodeUrl('testnet'),
  timeout: 30000
};

const validation = ConfigValidator.validateWalrusConfig(config);
if (!validation.valid) {
  console.error('Configuration validation failed:', validation.errors);
  throw new Error('Invalid configuration');
}`
        },
        {
          title: 'WASM 模块配置',
          code: `
// WASM 配置验证和加载
class WasmLoader {
  static async loadWasmWithFallback(): Promise<any> {
    const strategies = [
      () => this.loadFromBundle(),
      () => this.loadFromCDN(),
      () => this.loadFromLocal()
    ];

    for (const strategy of strategies) {
      try {
        const wasm = await strategy();
        console.log('WASM loaded successfully');
        return wasm;
      } catch (error) {
        console.warn('WASM loading strategy failed:', error.message);
      }
    }

    throw new Error('All WASM loading strategies failed');
  }

  private static async loadFromBundle(): Promise<any> {
    // Vite 示例
    if (typeof import.meta !== 'undefined' && import.meta.url) {
      const wasmUrl = new URL('@mysten/walrus-wasm/web/walrus_wasm_bg.wasm', import.meta.url);
      const wasmModule = await import(wasmUrl);
      return wasmModule.default;
    }
    throw new Error('Bundle loading not supported');
  }

  private static async loadFromCDN(): Promise<any> {
    const cdnUrls = [
      'https://unpkg.com/@mysten/walrus-wasm@latest/web/walrus_wasm_bg.wasm',
      'https://cdn.jsdelivr.net/npm/@mysten/walrus-wasm@latest/web/walrus_wasm_bg.wasm'
    ];

    for (const url of cdnUrls) {
      try {
        const response = await fetch(url);
        if (response.ok) {
          const wasmBuffer = await response.arrayBuffer();
          return WebAssembly.compile(wasmBuffer);
        }
      } catch (error) {
        console.warn(\`CDN loading failed for \${url}:\`, error.message);
      }
    }

    throw new Error('All CDN URLs failed');
  }

  private static async loadFromLocal(): Promise<any> {
    // 尝试从本地路径加载
    const localPaths = [
      '/walrus/walrus_wasm_bg.wasm',
      './node_modules/@mysten/walrus-wasm/web/walrus_wasm_bg.wasm'
    ];

    for (const path of localPaths) {
      try {
        const response = await fetch(path);
        if (response.ok) {
          const wasmBuffer = await response.arrayBuffer();
          return WebAssembly.compile(wasmBuffer);
        }
      } catch (error) {
        console.warn(\`Local loading failed for \${path}:\`, error.message);
      }
    }

    throw new Error('Local loading failed');
  }
}

// 在客户端配置中使用 WASM
const client = new SuiJsonRpcClient({
  url: getFullnodeUrl('testnet'),
  network: 'testnet',
}).$extend(
  walrus({
    wasmUrl: await WasmLoader.loadWasmWithFallback()
  })
);`
        }
      ],
      prevention: [
        '实施配置验证',
        '提供配置模板',
        '设置默认值',
        '实施配置监控'
      ]
    };
  }
}
```

### 4. 浏览器环境特定错误

#### 错误类型: Browser Environment Error
```typescript
class BrowserErrorDiagnostics {
  async diagnoseBrowserError(error: Error, context: BrowserContext): Promise<Diagnosis> {
    return {
      errorType: 'BROWSER_ERROR',
      description: '浏览器环境特定问题',
      causes: [
        'WASM 不支持',
        'CORS 策略阻止',
        '内存不足',
        '钱包连接问题'
      ],
      solutions: [
        {
          title: '浏览器兼容性检查',
          code: `
// 浏览器兼容性检查
class BrowserCompatibility {
  static checkCompatibility(): BrowserCompatibilityResult {
    const checks = {
      webAssembly: this.checkWebAssembly(),
      fetch: this.checkFetch(),
      blob: this.checkBlob(),
      fileAPI: this.checkFileAPI(),
      webWorkers: this.checkWebWorkers(),
      localStorage: this.checkLocalStorage()
    };

    const supported = Object.values(checks).every(check => check.supported);

    return {
      supported,
      checks,
      recommendations: this.getRecommendations(checks)
    };
  }

  private static checkWebAssembly(): CompatibilityCheck {
    const supported = typeof WebAssembly === 'object' && WebAssembly !== null;
    return {
      supported,
      message: supported ? 'WebAssembly is supported' : 'WebAssembly is not supported',
      fix: !supported ? 'Please use a modern browser that supports WebAssembly' : null
    };
  }

  private static getRecommendations(checks: Record<string, CompatibilityCheck>): string[] {
    const recommendations: string[] = [];

    if (!checks.webAssembly.supported) {
      recommendations.push('Upgrade to a modern browser (Chrome 57+, Firefox 52+, Safari 11+)');
    }

    if (!checks.webWorkers.supported) {
      recommendations.push('Consider enabling Web Workers for better performance');
    }

    return recommendations;
  }
}

// 使用兼容性检查
const compatibility = BrowserCompatibility.checkCompatibility();
if (!compatibility.supported) {
  console.error('Browser compatibility issues:', compatibility.recommendations);
  // 显示用户友好的错误信息
  this.showCompatibilityError(compatibility);
}`
        },
        {
          title: '内存管理',
          code: `
// 浏览器内存管理
class BrowserMemoryManager {
  private memoryThreshold = 100 * 1024 * 1024; // 100MB

  async checkMemoryUsage(): Promise<MemoryUsage> {
    if ('memory' in performance) {
      const memory = (performance as any).memory;
      return {
        used: memory.usedJSHeapSize,
        total: memory.totalJSHeapSize,
        limit: memory.jsHeapSizeLimit,
        usageRatio: memory.usedJSHeapSize / memory.jsHeapSizeLimit
      };
    }
    return { used: 0, total: 0, limit: 0, usageRatio: 0 };
  }

  async manageMemoryForLargeFile(file: File): Promise<boolean> {
    const memoryUsage = await this.checkMemoryUsage();

    // 检查是否有足够内存处理文件
    const estimatedUsage = file.size * 3; // 估算处理过程中需要的内存
    const availableMemory = memoryUsage.limit - memoryUsage.used;

    if (estimatedUsage > availableMemory) {
      console.warn('Insufficient memory for file processing');

      // 触发垃圾回收（如果可能）
      if (window.gc) {
        window.gc();
      }

      // 重新检查内存
      const newUsage = await this.checkMemoryUsage();
      return (file.size * 3) < (newUsage.limit - newUsage.used);
    }

    return true;
  }

  // 流式处理大文件
  async processLargeFileInChunks(
    file: File,
    processor: (chunk: Uint8Array) => Promise<void>,
    chunkSize = 1024 * 1024 // 1MB
  ): Promise<void> {
    const reader = new FileReader();
    let offset = 0;

    const processChunk = () => {
      if (offset >= file.size) {
        return Promise.resolve();
      }

      return new Promise<void>((resolve, reject) => {
        const chunk = file.slice(offset, offset + chunkSize);

        reader.onload = async (event) => {
          try {
            if (event.target?.result) {
              const arrayBuffer = event.target.result as ArrayBuffer;
              const uint8Array = new Uint8Array(arrayBuffer);

              await processor(uint8Array);
              offset += chunkSize;

              // 检查内存使用
              const memoryUsage = await this.checkMemoryUsage();
              if (memoryUsage.usageRatio > 0.8) {
                console.warn('High memory usage, forcing garbage collection');
                if (window.gc) window.gc();
              }

              await processChunk();
            }
            resolve();
          } catch (error) {
            reject(error);
          }
        };

        reader.onerror = () => reject(new Error('File reading failed'));
        reader.readAsArrayBuffer(chunk);
      });
    };

    return processChunk();
  }
}`
        }
      ],
      prevention: [
        '实施浏览器兼容性检查',
        '监控内存使用',
        '提供降级方案',
        '实施优雅错误处理'
      ]
    };
  }
}
```

## 使用方法

### 上传错误诊断
```typescript
skill: "walrus-debugger"
// error-type: upload
// error-context: Upload failed: insufficient storage balance, trying to upload 10MB file to testnet
// environment: browser
```

### 网络连接问题
```typescript
skill: "walrus-debugger"
// error-type: network
// error-context: Network timeout when connecting to storage nodes, intermittent connection issues
// environment: production
```

### 配置验证
```typescript
skill: "walrus-debugger"
// error-type: configuration
// error-context: WASM module failed to load in Vite production build, error: "Failed to resolve module"
// environment: nodejs
```

### 性能问题
```typescript
skill: "walrus-debugger"
// error-type: performance
// error-context: Memory usage spikes when uploading multiple large files, browser crashes
// environment: browser
```

---

*更新时间：2025-11-11*