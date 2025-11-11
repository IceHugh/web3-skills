---
name: walrus-performance-optimizer
description: Walrus 性能优化专家 - 分析和优化 Walrus 应用的性能瓶颈
parameters:
  - name: analysis-type
    type: string
    description: 分析类型 (upload/download/batch/memory/network)
    required: true
  - name: target-metric
    type: string
    description: 目标指标 (speed/throughput/cost/latency)
    default: speed
  - name: code-context
    type: string
    description: 代码上下文或问题描述
    required: true
---

# Walrus 性能优化技能

## 技能概述

`walrus-performance-optimizer` 是专业的性能优化助手，专门分析 Walrus 应用的性能瓶颈，提供数据驱动的优化建议和具体的实施方案。

## 性能分析维度

### 1. 上传性能优化

#### 批量操作优化
```typescript
// 优化前：逐个上传文件
async function uploadFilesSlow(files: File[]) {
  const results = [];
  for (const file of files) {
    const result = await uploadSingleFile(file);
    results.push(result);
  }
  return results;
}

// 优化后：批量并发上传
class OptimizedUploader {
  private readonly MAX_CONCURRENT = 5;
  private readonly BATCH_DELAY = 100;

  async uploadFilesOptimized(files: File[]): Promise<UploadResult[]> {
    const results: UploadResult[] = [];
    const chunks = this.chunkArray(files, this.MAX_CONCURRENT);

    for (const chunk of chunks) {
      const chunkPromises = chunk.map(file => this.uploadWithRetry(file));
      const chunkResults = await Promise.allSettled(chunkPromises);

      results.push(...this.processChunkResults(chunkResults));

      // 批次间延迟，避免过载
      if (chunks.indexOf(chunk) < chunks.length - 1) {
        await this.delay(this.BATCH_DELAY);
      }
    }

    return results;
  }

  private async uploadWithRetry(file: File, maxRetries = 3): Promise<UploadResult> {
    for (let attempt = 1; attempt <= maxRetries; attempt++) {
      try {
        return await this.uploadSingleFile(file);
      } catch (error) {
        if (attempt === maxRetries || !this.isRetryableError(error)) {
          throw error;
        }

        // 指数退避
        const delay = Math.min(1000 * Math.pow(2, attempt - 1), 10000);
        await this.delay(delay);
      }
    }
  }

  private chunkArray<T>(array: T[], size: number): T[][] {
    const chunks: T[][] = [];
    for (let i = 0; i < array.length; i += size) {
      chunks.push(array.slice(i, i + size));
    }
    return chunks;
  }

  private isRetryableError(error: any): boolean {
    return error instanceof RetryableWalrusClientError ||
           error.message.includes('timeout') ||
           error.message.includes('network');
  }
}
```

#### 文件压缩优化
```typescript
class CompressionOptimizer {
  async optimizeUpload(file: File): Promise<OptimizedFile> {
    const fileBuffer = await file.arrayBuffer();
    const originalSize = fileBuffer.byteLength;

    // 文本文件压缩
    if (this.isTextFile(file)) {
      const compressed = await this.compressText(fileBuffer);
      if (compressed.size < originalSize * 0.8) {
        return {
          data: compressed.data,
          originalSize,
          compressedSize: compressed.size,
          compressionRatio: compressed.size / originalSize,
          method: 'gzip'
        };
      }
    }

    // 图片文件优化
    if (this.isImageFile(file)) {
      const optimized = await this.optimizeImage(fileBuffer);
      if (optimized.size < originalSize * 0.9) {
        return {
          data: optimized.data,
          originalSize,
          compressedSize: optimized.size,
          compressionRatio: optimized.size / originalSize,
          method: 'image-optimization'
        };
      }
    }

    return {
      data: new Uint8Array(fileBuffer),
      originalSize,
      compressedSize: originalSize,
      compressionRatio: 1,
      method: 'none'
    };
  }

  private async compressText(data: ArrayBuffer): Promise<CompressedData> {
    const compressionStream = new CompressionStream('gzip');
    const writer = compressionStream.writable.getWriter();
    const reader = compressionStream.readable.getReader();

    writer.write(new Uint8Array(data));
    writer.close();

    const chunks: Uint8Array[] = [];
    let totalSize = 0;

    while (true) {
      const { done, value } = await reader.read();
      if (done) break;

      chunks.push(value);
      totalSize += value.length;
    }

    const combined = new Uint8Array(totalSize);
    let offset = 0;
    for (const chunk of chunks) {
      combined.set(chunk, offset);
      offset += chunk.length;
    }

    return { data: combined, size: totalSize };
  }

  private async optimizeImage(data: ArrayBuffer): Promise<CompressedData> {
    // 使用 WebAssembly 进行图片优化
    // 这里可以集成像 sharp.js 或其他图片处理库
    const blob = new Blob([data], { type: 'image/jpeg' });

    return new Promise((resolve) => {
      const img = new Image();
      img.onload = () => {
        const canvas = document.createElement('canvas');
        const ctx = canvas.getContext('2d')!;

        // 降低质量但保持合理尺寸
        const scale = Math.min(1, 1920 / Math.max(img.width, img.height));
        canvas.width = img.width * scale;
        canvas.height = img.height * scale;

        ctx.drawImage(img, 0, 0, canvas.width, canvas.height);

        canvas.toBlob((blob) => {
          if (blob) {
            blob.arrayBuffer().then(buffer => {
              const data = new Uint8Array(buffer);
              resolve({ data, size: data.length });
            });
          }
        }, 'image/jpeg', 0.85);
      };
      img.src = URL.createObjectURL(blob);
    });
  }
}
```

### 2. 下载性能优化

#### 智能缓存系统
```typescript
class IntelligentCache {
  private cache = new Map<string, CacheEntry>();
  private accessOrder = new Map<string, number>();
  private accessCounter = 0;
  private readonly MAX_SIZE = 100; // 最大缓存条目
  private readonly TTL = 300000; // 5分钟过期

  async get(blobId: string): Promise<Uint8Array | null> {
    const entry = this.cache.get(blobId);

    if (!entry) {
      return null;
    }

    // 检查是否过期
    if (Date.now() - entry.timestamp > this.TTL) {
      this.delete(blobId);
      return null;
    }

    // 更新访问顺序 (LRU)
    this.accessOrder.set(blobId, ++this.accessCounter);

    return entry.data;
  }

  async set(blobId: string, data: Uint8Array): Promise<void> {
    // 检查缓存大小，必要时清理
    if (this.cache.size >= this.MAX_SIZE) {
      this.evictLRU();
    }

    this.cache.set(blobId, {
      data,
      timestamp: Date.now(),
      size: data.length
    });

    this.accessOrder.set(blobId, ++this.accessCounter);
  }

  private evictLRU(): void {
    let oldestKey = '';
    let oldestAccess = Infinity;

    for (const [key, accessTime] of this.accessOrder) {
      if (accessTime < oldestAccess) {
        oldestAccess = accessTime;
        oldestKey = key;
      }
    }

    if (oldestKey) {
      this.delete(oldestKey);
    }
  }

  private delete(blobId: string): void {
    this.cache.delete(blobId);
    this.accessOrder.delete(blobId);
  }

  getStats(): CacheStats {
    return {
      size: this.cache.size,
      totalMemoryUsage: Array.from(this.cache.values())
        .reduce((sum, entry) => sum + entry.size, 0),
      hitRate: this.hitRate,
      evictions: this.evictions
    };
  }
}
```

#### 预加载策略
```typescript
class PreloadingStrategy {
  private preloadQueue = new Set<string>();
  private preloadCache = new Map<string, PreloadEntry>();

  async preloadFiles(blobIds: string[], priority: 'high' | 'medium' | 'low' = 'medium') {
    const delay = priority === 'high' ? 0 : priority === 'medium' ? 100 : 500;

    for (const blobId of blobIds) {
      if (!this.preloadQueue.has(blobId)) {
        this.preloadQueue.add(blobId);

        // 延迟执行，避免阻塞主要操作
        setTimeout(() => this.preloadFile(blobId), delay);
      }
    }
  }

  private async preloadFile(blobId: string) {
    try {
      const startTime = Date.now();

      // 使用低优先级下载
      const [file] = await client.walrus.getFiles({
        ids: [blobId],
        options: { priority: 'low' }
      });

      const content = await file.bytes();
      const loadTime = Date.now() - startTime;

      this.preloadCache.set(blobId, {
        data: content,
        timestamp: Date.now(),
        loadTime
      });

      console.log(`预加载完成: ${blobId} (${loadTime}ms)`);

    } catch (error) {
      console.warn(`预加载失败: ${blobId}`, error.message);
    } finally {
      this.preloadQueue.delete(blobId);
    }
  }

  async getPreloaded(blobId: string): Promise<Uint8Array | null> {
    const entry = this.preloadCache.get(blobId);

    if (!entry) {
      return null;
    }

    // 检查是否过期
    if (Date.now() - entry.timestamp > 600000) { // 10分钟
      this.preloadCache.delete(blobId);
      return null;
    }

    return entry.data;
  }
}
```

### 3. 网络性能优化

#### 连接池管理
```typescript
class ConnectionPool {
  private connections: Map<string, Connection[]> = new Map();
  private readonly MAX_CONNECTIONS_PER_HOST = 5;
  private readonly CONNECTION_TIMEOUT = 30000;

  async getConnection(host: string): Promise<Connection> {
    const pool = this.connections.get(host) || [];

    // 尝试复用现有连接
    for (const conn of pool) {
      if (!conn.inUse && this.isConnectionHealthy(conn)) {
        conn.inUse = true;
        conn.lastUsed = Date.now();
        return conn;
      }
    }

    // 创建新连接
    if (pool.length < this.MAX_CONNECTIONS_PER_HOST) {
      const newConn = await this.createConnection(host);
      newConn.inUse = true;
      pool.push(newConn);
      this.connections.set(host, pool);
      return newConn;
    }

    // 等待连接可用
    return this.waitForAvailableConnection(host);
  }

  releaseConnection(connection: Connection) {
    connection.inUse = false;
    connection.lastUsed = Date.now();
  }

  private async createConnection(host: string): Promise<Connection> {
    return new Promise((resolve, reject) => {
      const ws = new WebSocket(host);

      const timeout = setTimeout(() => {
        reject(new Error('Connection timeout'));
      }, this.CONNECTION_TIMEOUT);

      ws.onopen = () => {
        clearTimeout(timeout);
        resolve({
          socket: ws,
          created: Date.now(),
          lastUsed: Date.now(),
          inUse: true
        });
      };

      ws.onerror = () => {
        clearTimeout(timeout);
        reject(new Error('Connection failed'));
      };
    });
  }

  private isConnectionHealthy(conn: Connection): boolean {
    return conn.socket.readyState === WebSocket.OPEN &&
           Date.now() - conn.lastUsed < 300000; // 5分钟未使用则关闭
  }

  private async waitForAvailableConnection(host: string): Promise<Connection> {
    const pool = this.connections.get(host) || [];

    return new Promise((resolve) => {
      const checkInterval = setInterval(() => {
        const available = pool.find(conn => !conn.inUse && this.isConnectionHealthy(conn));
        if (available) {
          clearInterval(checkInterval);
          available.inUse = true;
          available.lastUsed = Date.now();
          resolve(available);
        }
      }, 100);
    });
  }
}
```

#### 请求优化
```typescript
class RequestOptimizer {
  private pendingRequests = new Map<string, Promise<any>>();
  private requestQueue: QueuedRequest[] = [];
  private processing = false;

  async batchRequest<T>(requests: BatchRequest[]): Promise<T[]> {
    // 合并相同的请求
    const uniqueRequests = this.deduplicateRequests(requests);

    // 按优先级排序
    uniqueRequests.sort((a, b) => b.priority - a.priority);

    const results: T[] = [];

    for (const request of uniqueRequests) {
      try {
        const result = await this.executeRequest(request);
        results.push(result);
      } catch (error) {
        console.error(`批量请求失败: ${request.id}`, error);
        results.push(null as T);
      }
    }

    return results;
  }

  private deduplicateRequests(requests: BatchRequest[]): BatchRequest[] {
    const seen = new Set<string>();
    return requests.filter(req => {
      const key = this.getRequestKey(req);
      if (seen.has(key)) {
        return false;
      }
      seen.add(key);
      return true;
    });
  }

  private async executeRequest(request: BatchRequest): Promise<any> {
    const key = this.getRequestKey(request);

    // 检查是否有相同请求正在进行
    if (this.pendingRequests.has(key)) {
      return await this.pendingRequests.get(key);
    }

    const requestPromise = this.performRequest(request);
    this.pendingRequests.set(key, requestPromise);

    try {
      const result = await requestPromise;
      return result;
    } finally {
      this.pendingRequests.delete(key);
    }
  }

  private async performRequest(request: BatchRequest): Promise<any> {
    // 实现实际的请求逻辑
    const controller = new AbortController();
    const timeoutId = setTimeout(() => controller.abort(), request.timeout);

    try {
      const response = await fetch(request.url, {
        ...request.options,
        signal: controller.signal
      });

      if (!response.ok) {
        throw new Error(`HTTP ${response.status}: ${response.statusText}`);
      }

      return await response.json();
    } finally {
      clearTimeout(timeoutId);
    }
  }
}
```

### 4. 内存优化

#### 内存池管理
```typescript
class MemoryPool {
  private pools = new Map<number, ArrayBuffer[]>();
  private readonly MAX_POOL_SIZE = 50;
  private readonly ALLOC_INCREMENT = 1024; // 1KB increments

  allocate(size: number): ArrayBuffer {
    const actualSize = Math.ceil(size / this.ALLOC_INCREMENT) * this.ALLOC_INCREMENT;
    const pool = this.pools.get(actualSize) || [];

    if (pool.length > 0) {
      return pool.pop()!;
    }

    return new ArrayBuffer(actualSize);
  }

  release(buffer: ArrayBuffer): void {
    const size = buffer.byteLength;
    const pool = this.pools.get(size) || [];

    if (pool.length < this.MAX_POOL_SIZE) {
      // 清零缓冲区（安全考虑）
      new Uint8Array(buffer).fill(0);
      pool.push(buffer);
      this.pools.set(size, pool);
    }
  }

  getStats(): MemoryPoolStats {
    const stats: MemoryPoolStats = {
      totalBuffers: 0,
      totalMemory: 0,
      poolSizes: {}
    };

    for (const [size, pool] of this.pools) {
      stats.totalBuffers += pool.length;
      stats.totalMemory += size * pool.length;
      stats.poolSizes[size] = pool.length;
    }

    return stats;
  }
}
```

#### 流式处理
```typescript
class StreamProcessor {
  async processLargeFile(
    file: File,
    processor: (chunk: Uint8Array, index: number) => Promise<void>,
    chunkSize = 1024 * 1024 // 1MB chunks
  ): Promise<void> {
    return new Promise((resolve, reject) => {
      const reader = new FileReader();
      let offset = 0;
      let index = 0;

      const processChunk = () => {
        if (offset >= file.size) {
          resolve();
          return;
        }

        const chunk = file.slice(offset, offset + chunkSize);
        reader.onload = async (event) => {
          if (event.target?.result) {
            const arrayBuffer = event.target.result as ArrayBuffer;
            const uint8Array = new Uint8Array(arrayBuffer);

            try {
              await processor(uint8Array, index);
              offset += chunkSize;
              index++;
              processChunk();
            } catch (error) {
              reject(error);
            }
          }
        };

        reader.readAsArrayBuffer(chunk);
      };

      processChunk();
    });
  }

  async uploadStream(
    file: File,
    onProgress?: (progress: number) => void
  ): Promise<string> {
    const chunks: Uint8Array[] = [];
    const totalSize = file.size;

    await this.processLargeFile(file, async (chunk, index) => {
      chunks.push(chunk);

      const progress = ((index + 1) * chunk.length) / totalSize;
      onProgress?.(Math.min(progress, 1));
    });

    // 合并所有块
    const totalLength = chunks.reduce((sum, chunk) => sum + chunk.length, 0);
    const combined = new Uint8Array(totalLength);
    let offset = 0;

    for (const chunk of chunks) {
      combined.set(chunk, offset);
      offset += chunk.length;
    }

    // 上传合并后的数据
    const walrusFile = WalrusFile.from({
      contents: combined,
      identifier: `stream/${Date.now()}_${file.name}`
    });

    const result = await client.walrus.writeFiles({
      files: [walrusFile],
      epochs: 10,
      deletable: true
    });

    return result[0].blobId;
  }
}
```

## 性能监控

### 实时性能指标
```typescript
class PerformanceMonitor {
  private metrics = new Map<string, MetricEntry[]>();
  private readonly MAX_METRICS = 1000;

  recordMetric(name: string, value: number, tags: Record<string, string> = {}) {
    const entry: MetricEntry = {
      timestamp: Date.now(),
      value,
      tags
    };

    const existing = this.metrics.get(name) || [];
    existing.push(entry);

    // 限制历史数据数量
    if (existing.length > this.MAX_METRICS) {
      existing.splice(0, existing.length - this.MAX_METRICS);
    }

    this.metrics.set(name, existing);
  }

  getMetrics(name: string, timeRange?: number): MetricEntry[] {
    const allMetrics = this.metrics.get(name) || [];

    if (!timeRange) {
      return allMetrics;
    }

    const cutoff = Date.now() - timeRange;
    return allMetrics.filter(m => m.timestamp >= cutoff);
  }

  getAggregatedMetrics(name: string, timeRange = 300000): AggregatedMetrics {
    const metrics = this.getMetrics(name, timeRange);

    if (metrics.length === 0) {
      return {
        count: 0,
        sum: 0,
        avg: 0,
        min: 0,
        max: 0,
        p50: 0,
        p95: 0,
        p99: 0
      };
    }

    const values = metrics.map(m => m.value).sort((a, b) => a - b);
    const sum = values.reduce((s, v) => s + v, 0);

    return {
      count: values.length,
      sum,
      avg: sum / values.length,
      min: values[0],
      max: values[values.length - 1],
      p50: this.percentile(values, 0.5),
      p95: this.percentile(values, 0.95),
      p99: this.percentile(values, 0.99)
    };
  }

  private percentile(sorted: number[], p: number): number {
    const index = Math.ceil(sorted.length * p) - 1;
    return sorted[Math.max(0, index)];
  }
}
```

## 使用方法

### 基础性能分析
```typescript
skill: "walrus-performance-optimizer"
// analysis-type: upload
// target-metric: speed
// code-context: 我的文件上传速度很慢，单个10MB文件需要2分钟，如何优化？
```

### 批量操作优化
```typescript
skill: "walrus-performance-optimizer"
// analysis-type: batch
// target-metric: throughput
// code-context: 批量上传100个文件时经常超时，需要优化并发处理
```

### 内存使用优化
```typescript
skill: "walrus-performance-optimizer"
// analysis-type: memory
// target-metric: cost
// code-context: 大文件处理时内存占用过高，浏览器经常崩溃
```

### 网络优化
```typescript
skill: "walrus-performance-optimizer"
// analysis-type: network
// target-metric: latency
// code-context: 网络不稳定环境下频繁上传失败，需要优化网络策略
```

---

*更新时间：2025-11-11*