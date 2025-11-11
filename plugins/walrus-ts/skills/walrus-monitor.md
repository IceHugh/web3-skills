---
name: walrus-monitor
description: Walrus 监控分析专家 - 存储使用监控、成本分析和性能指标跟踪
parameters:
  - name: monitor-type
    type: string
    description: 监控类型 (usage/cost/performance/errors/analytics)
    required: true
  - name: time-range
    type: string
    description: 时间范围 (1h/24h/7d/30d/custom)
    default: 24h
  - name: metrics
    type: array
    description: 需要监控的具体指标
    default: []
---

# Walrus 监控分析技能

## 技能概述

`walrus-monitor` 是专业的监控分析助手，提供全面的 Walrus 存储系统监控、成本分析和性能指标跟踪功能，帮助开发者优化资源使用和控制成本。

## 监控维度

### 1. 存储使用监控

#### 实时存储指标
```typescript
class StorageMonitor {
  private metrics: StorageMetrics[] = [];
  private alerts: Alert[] = [];

  async collectStorageMetrics(): Promise<StorageMetrics> {
    const client = await this.getWalrusClient();
    const accountAddress = this.getAccountAddress();

    // 获取账户存储信息
    const storageInfo = await client.getStorageInfo(accountAddress);
    const blobIds = storageInfo.storedBlobs || [];

    // 计算存储统计
    const metrics: StorageMetrics = {
      timestamp: Date.now(),
      totalFiles: blobIds.length,
      totalSize: 0,
      totalCost: 0,
      storageDistribution: {},
      fileAges: [],
      oldestFile: null,
      newestFile: null,
      expiringSoon: [],
      expired: []
    };

    // 分析每个文件
    for (const blobId of blobIds) {
      try {
        const [file] = await client.walrus.getFiles({ ids: [blobId] });
        const metadata = await this.getFileMetadata(file);
        const storedUntil = await file.storedUntil();

        metrics.totalSize += metadata.size;
        metrics.totalCost += metadata.estimatedCost;

        // 文件类型分布
        const fileType = this.getFileType(metadata.mimeType);
        metrics.storageDistribution[fileType] =
          (metrics.storageDistribution[fileType] || 0) + 1;

        // 文件年龄分析
        const age = Date.now() - metadata.uploadTime;
        metrics.fileAges.push(age);

        // 检查过期状态
        const daysToExpiry = Math.floor((storedUntil.getTime() - Date.now()) / (1000 * 60 * 60 * 24));
        if (daysToExpiry < 7) {
          metrics.expiringSoon.push({
            blobId,
            fileName: metadata.identifier,
            daysToExpiry,
            size: metadata.size
          });
        }

        if (daysToExpiry < 0) {
          metrics.expired.push({
            blobId,
            fileName: metadata.identifier,
            daysExpired: Math.abs(daysToExpiry),
            size: metadata.size
          });
        }

        // 更新最新和最旧文件
        if (!metrics.oldestFile || age > metrics.oldestFile.age) {
          metrics.oldestFile = {
            blobId,
            fileName: metadata.identifier,
            age,
            uploadDate: new Date(metadata.uploadTime)
          };
        }

        if (!metrics.newestFile || age < metrics.newestFile.age) {
          metrics.newestFile = {
            blobId,
            fileName: metadata.identifier,
            age,
            uploadDate: new Date(metadata.uploadTime)
          };
        }

      } catch (error) {
        console.warn(`Failed to analyze blob ${blobId}:`, error.message);
      }
    }

    // 存储指标历史
    this.metrics.push(metrics);
    if (this.metrics.length > 1000) {
      this.metrics.shift(); // 保留最近1000条记录
    }

    // 检查告警条件
    this.checkStorageAlerts(metrics);

    return metrics;
  }

  private checkStorageAlerts(metrics: StorageMetrics): void {
    // 存储空间告警
    if (metrics.totalSize > 100 * 1024 * 1024 * 1024) { // 100GB
      this.createAlert({
        type: 'STORAGE_LIMIT',
        severity: 'WARNING',
        message: `存储使用量达到 ${this.formatBytes(metrics.totalSize)}`,
        recommendation: '考虑清理过期文件或升级存储计划'
      });
    }

    // 文件数量告警
    if (metrics.totalFiles > 10000) {
      this.createAlert({
        type: 'FILE_COUNT_LIMIT',
        severity: 'INFO',
        message: `文件数量达到 ${metrics.totalFiles}`,
        recommendation: '考虑实施文件归档策略'
      });
    }

    // 即将过期告警
    if (metrics.expiringSoon.length > 0) {
      this.createAlert({
        type: 'EXPIRING_FILES',
        severity: 'WARNING',
        message: `${metrics.expiringSoon.length} 个文件将在7天内过期`,
        recommendation: '续期或备份重要文件'
      });
    }
  }

  getStorageTrends(timeRange: number = 7 * 24 * 60 * 60 * 1000): StorageTrends {
    const cutoff = Date.now() - timeRange;
    const relevantMetrics = this.metrics.filter(m => m.timestamp >= cutoff);

    if (relevantMetrics.length < 2) {
      return {
        growthRate: 0,
        costGrowthRate: 0,
        fileGrowthRate: 0,
        projectedUsage: null,
        projectedCost: null
      };
    }

    const oldest = relevantMetrics[0];
    const newest = relevantMetrics[relevantMetrics.length - 1];
    const timeDiff = newest.timestamp - oldest.timestamp;

    return {
      growthRate: this.calculateGrowthRate(oldest.totalSize, newest.totalSize, timeDiff),
      costGrowthRate: this.calculateGrowthRate(oldest.totalCost, newest.totalCost, timeDiff),
      fileGrowthRate: this.calculateGrowthRate(oldest.totalFiles, newest.totalFiles, timeDiff),
      projectedUsage: this.projectUsage(newest.totalSize, this.calculateGrowthRate(oldest.totalSize, newest.totalSize, timeDiff)),
      projectedCost: this.projectCost(newest.totalCost, this.calculateGrowthRate(oldest.totalCost, newest.totalCost, timeDiff))
    };
  }
}
```

### 2. 成本分析

#### 成本监控和优化
```typescript
class CostAnalyzer {
  private costHistory: CostRecord[] = [];
  private pricingModel: PricingModel;

  constructor() {
    this.pricingModel = {
      storagePerGBPerEpoch: 0.001,
      writeFeePerMB: 0.0001,
      readFeePerMB: 0.00001,
      gasMultiplier: 1.0
    };
  }

  async analyzeCosts(timeRange: number = 30 * 24 * 60 * 60 * 1000): Promise<CostAnalysis> {
    const records = this.getRelevantCostRecords(timeRange);
    const storageMetrics = await this.getStorageMetrics(timeRange);

    return {
      totalCost: this.calculateTotalCost(records),
      storageCost: this.calculateStorageCost(storageMetrics),
      transactionCost: this.calculateTransactionCost(records),
      breakdown: this.getCostBreakdown(records),
      optimization: this.getOptimizationSuggestions(storageMetrics, records),
      forecast: this.forecastCosts(records, storageMetrics)
    };
  }

  private calculateStorageCost(metrics: StorageMetrics[]): number {
    return metrics.reduce((total, metric) => {
      const storageEpochs = 30; // 假设30天的轮数
      const sizeGB = metric.totalSize / (1024 * 1024 * 1024);
      return total + (sizeGB * this.pricingModel.storagePerGBPerEpoch * storageEpochs);
    }, 0);
  }

  private calculateTransactionCost(records: CostRecord[]): number {
    return records
      .filter(r => r.type === 'transaction')
      .reduce((total, record) => total + record.amount, 0);
  }

  private getOptimizationSuggestions(metrics: StorageMetrics[], records: CostRecord[]): OptimizationSuggestion[] {
    const suggestions: OptimizationSuggestion[] = [];

    // 分析文件大小分布
    const sizeDistribution = this.analyzeSizeDistribution(metrics);
    if (sizeDistribution.smallFiles > sizeDistribution.largeFiles * 2) {
      suggestions.push({
        type: 'BUNDLE_SMALL_FILES',
        description: '检测到大量小文件，建议使用 Quilt 打包存储',
        potentialSavings: '20-30%',
        difficulty: 'medium',
        implementation: this.getBundleImplementation()
      });
    }

    // 分析存储时长
    const avgStorageTime = this.calculateAverageStorageTime(metrics);
    if (avgStorageTime > 90) { // 超过90天
      suggestions.push({
        type: 'OPTIMIZE_STORAGE_DURATION',
        description: `平均存储时长 ${avgStorageTime} 天，建议实施生命周期管理`,
        potentialSavings: '40-60%',
        difficulty: 'low',
        implementation: this.getLifecycleImplementation()
      });
    }

    // 分析访问模式
    const accessPattern = this.analyzeAccessPattern(records);
    if (accessPattern.readHeavy) {
      suggestions.push({
        type: 'IMPLEMENT_CACHING',
        description: '读取频繁，建议实施缓存层',
        potentialSavings: '50-70%',
        difficulty: 'medium',
        implementation: this.getCacheImplementation()
      });
    }

    return suggestions;
  }

  private getBundleImplementation(): string {
    return `
// 使用 Quilt 打包小文件
async function bundleSmallFiles(files: WalrusFile[]): Promise<string> {
  const quilt = QuiltBuilder.create()
    .addFiles(files)
    .build();

  const result = await client.walrus.writeFiles({
    files: [quilt],
    epochs: 30,
    deletable: true,
    signer: keypair
  });

  return result[0].blobId;
}

// 原本多个小文件：
// File 1: 10KB, File 2: 15KB, File 3: 8KB = 33KB 总计
// 打包后：单个 Quilt 包含所有文件，减少存储开销
    `;
  }

  private getLifecycleImplementation(): string {
    return `
// 自动生命周期管理
class StorageLifecycleManager {
  async manageExpiredFiles() {
    const expiringFiles = await this.getFilesExpiringWithin(7);

    for (const file of expiringFiles) {
      const accessFrequency = await this.getAccessFrequency(file.blobId);

      if (accessFrequency === 'never') {
        // 删除未访问的文件
        await this.deleteFile(file.blobId);
      } else if (accessFrequency === 'rare') {
        // 降低存储时长
        await this.reduceStorageEpochs(file.blobId, 10);
      } else {
        // 续期重要文件
        await this.extendStorage(file.blobId, 30);
      }
    }
  }

  private async getAccessFrequency(blobId: string): Promise<'never' | 'rare' | 'frequent'> {
    // 分析访问日志
    const accessLogs = await this.getAccessLogs(blobId, 30); // 30天内
    const accessCount = accessLogs.length;

    if (accessCount === 0) return 'never';
    if (accessCount < 5) return 'rare';
    return 'frequent';
  }
}
    `;
  }
}
```

### 3. 性能监控

#### 实时性能指标
```typescript
class PerformanceMonitor {
  private performanceMetrics: PerformanceMetric[] = [];
  private slowQueries: SlowQuery[] = [];
  private alerts: PerformanceAlert[] = [];

  async recordPerformanceMetric(
    operation: string,
    duration: number,
    success: boolean,
    metadata: any = {}
  ): Promise<void> {
    const metric: PerformanceMetric = {
      timestamp: Date.now(),
      operation,
      duration,
      success,
      metadata
    };

    this.performanceMetrics.push(metric);

    // 限制历史数据
    if (this.performanceMetrics.length > 10000) {
      this.performanceMetrics.shift();
    }

    // 检查性能告警
    this.checkPerformanceAlerts(metric);

    // 记录慢操作
    if (duration > this.getSlowThreshold(operation)) {
      this.slowQueries.push({
        ...metric,
        threshold: this.getSlowThreshold(operation),
        severity: this.getSeverityLevel(duration, this.getSlowThreshold(operation))
      });

      // 限制慢查询记录
      if (this.slowQueries.length > 1000) {
        this.slowQueries.shift();
      }
    }
  }

  getPerformanceAnalysis(timeRange: number = 24 * 60 * 60 * 1000): PerformanceAnalysis {
    const metrics = this.getRelevantMetrics(timeRange);

    return {
      overview: this.getPerformanceOverview(metrics),
      operations: this.analyzeOperations(metrics),
      trends: this.analyzeTrends(metrics),
      bottlenecks: this.identifyBottlenecks(metrics),
      recommendations: this.getPerformanceRecommendations(metrics)
    };
  }

  private analyzeOperations(metrics: PerformanceMetric[]): OperationAnalysis[] {
    const operationGroups = this.groupBy(metrics, 'operation');
    const analyses: OperationAnalysis[] = [];

    for (const [operation, operationMetrics] of Object.entries(operationGroups)) {
      const durations = operationMetrics.map(m => m.duration);
      const successRate = operationMetrics.filter(m => m.success).length / operationMetrics.length;

      analyses.push({
        operation,
        totalCalls: operationMetrics.length,
        successRate,
        avgDuration: durations.reduce((a, b) => a + b, 0) / durations.length,
        minDuration: Math.min(...durations),
        maxDuration: Math.max(...durations),
        p95: this.percentile(durations, 0.95),
        p99: this.percentile(durations, 0.99),
        errorRate: 1 - successRate,
        slowQueries: operationMetrics.filter(m => m.duration > this.getSlowThreshold(operation)).length
      });
    }

    return analyses.sort((a, b) => b.avgDuration - a.avgDuration);
  }

  private identifyBottlenecks(metrics: PerformanceMetric[]): PerformanceBottleneck[] {
    const bottlenecks: PerformanceBottleneck[] = [];

    // 识别慢操作
    const slowOperations = this.analyzeOperations(metrics)
      .filter(op => op.avgDuration > this.getSlowThreshold(op.operation));

    for (const operation of slowOperations) {
      bottlenecks.push({
        type: 'SLOW_OPERATION',
        operation: operation.operation,
        severity: this.getBottleneckSeverity(operation.avgDuration, this.getSlowThreshold(operation.operation)),
        impact: 'medium',
        description: `\${operation.operation} 平均耗时 \${operation.avgDuration}ms`,
        suggestion: this.getOptimizationSuggestion(operation.operation)
      });
    }

    // 识别错误率高的操作
    const highErrorOps = this.analyzeOperations(metrics)
      .filter(op => op.errorRate > 0.05); // 5% 错误率

    for (const operation of highErrorOps) {
      bottlenecks.push({
        type: 'HIGH_ERROR_RATE',
        operation: operation.operation,
        severity: 'high',
        impact: 'high',
        description: `\${operation.operation} 错误率 \${(operation.errorRate * 100).toFixed(1)}%`,
        suggestion: this.getErrorOptimizationSuggestion(operation.operation)
      });
    }

    return bottlenecks;
  }

  private getPerformanceRecommendations(metrics: PerformanceMetric[]): PerformanceRecommendation[] {
    const recommendations: PerformanceRecommendation[] = [];

    // 分析上传性能
    const uploadMetrics = metrics.filter(m => m.operation.startsWith('upload'));
    if (uploadMetrics.length > 0) {
      const avgUploadSize = this.getAverageUploadSize(uploadMetrics);
      const avgUploadTime = uploadMetrics.reduce((sum, m) => sum + m.duration, 0) / uploadMetrics.length;

      if (avgUploadSize > 10 * 1024 * 1024 && avgUploadTime > 10000) { // 10MB, 10s
        recommendations.push({
          type: 'UPLOAD_OPTIMIZATION',
          priority: 'high',
          title: '大文件上传优化',
          description: '检测到大文件上传性能问题',
          implementation: `
// 实施分块上传和断点续传
class ChunkedUploader {
  async uploadLargeFile(file: File, chunkSize = 5 * 1024 * 1024): Promise<string> {
    const chunks = Math.ceil(file.size / chunkSize);
    const chunkIds: string[] = [];

    for (let i = 0; i < chunks; i++) {
      const start = i * chunkSize;
      const end = Math.min(start + chunkSize, file.size);
      const chunk = file.slice(start, end);

      const result = await this.uploadChunk(chunk, i, chunks);
      chunkIds.push(result.blobId);

      // 更新进度
      this.updateProgress((i + 1) / chunks);
    }

    // 创建文件清单
    return await this.createManifest(chunkIds, file);
  }
}

// 使用压缩减少传输量
const compressedFile = await this.compressFile(file);
const uploadResult = await client.walrus.writeFiles({
  files: [compressedFile],
  epochs: 10,
  signer: keypair
});
          `,
          expectedImprovement: '50-70%'
        });
      }
    }

    // 分析批量操作性能
    const batchMetrics = metrics.filter(m => m.operation.includes('batch'));
    if (batchMetrics.length > 0) {
      const concurrencyIssues = batchMetrics.filter(m => m.duration > 60000).length; // 超过1分钟

      if (concurrencyIssues > batchMetrics.length * 0.3) { // 30%超时
        recommendations.push({
          type: 'BATCH_OPTIMIZATION',
          priority: 'medium',
          title: '批量操作并发优化',
          description: '批量操作存在并发瓶颈',
          implementation: `
// 优化并发控制
class OptimizedBatchProcessor {
  private readonly MAX_CONCURRENT = 3;
  private readonly RETRY_DELAY = 1000;

  async processBatch<T>(items: T[], processor: (item: T) => Promise<any>): Promise<any[]> {
    const results: any[] = [];
    const chunks = this.chunkArray(items, this.MAX_CONCURRENT);

    for (const chunk of chunks) {
      const chunkPromises = chunk.map(async (item, index) => {
        try {
          return await this.withRetry(() => processor(item), 3);
        } catch (error) {
          console.error(\`Batch item failed: \${error.message}\`);
          return null;
        }
      });

      const chunkResults = await Promise.all(chunkPromises);
      results.push(...chunkResults);

      // 批次间延迟
      if (chunks.indexOf(chunk) < chunks.length - 1) {
        await this.delay(this.RETRY_DELAY);
      }
    }

    return results;
  }
}
          `,
          expectedImprovement: '30-50%'
        });
      }
    }

    return recommendations;
  }
}
```

### 4. 错误监控

#### 错误统计和分析
```typescript
class ErrorMonitor {
  private errorLog: ErrorRecord[] = [];
  private errorPatterns: ErrorPattern[] = [];

  recordError(error: Error, context: ErrorContext): void {
    const errorRecord: ErrorRecord = {
      timestamp: Date.now(),
      message: error.message,
      stack: error.stack,
      type: this.classifyError(error),
      severity: this.determineSeverity(error, context),
      context,
      resolved: false
    };

    this.errorLog.push(errorRecord);

    // 分析错误模式
    this.analyzeErrorPattern(errorRecord);

    // 错误告警
    this.checkErrorAlerts(errorRecord);
  }

  getErrorAnalysis(timeRange: number = 24 * 60 * 60 * 1000): ErrorAnalysis {
    const recentErrors = this.getRecentErrors(timeRange);

    return {
      summary: this.getErrorSummary(recentErrors),
      trends: this.getErrorTrends(recentErrors),
      patterns: this.getErrorPatterns(recentErrors),
      recommendations: this.getErrorRecommendations(recentErrors),
      resolutionRate: this.calculateResolutionRate(recentErrors)
    };
  }

  private analyzeErrorPattern(errorRecord: ErrorRecord): void {
    const patternKey = this.getPatternKey(errorRecord);
    let pattern = this.errorPatterns.find(p => p.key === patternKey);

    if (!pattern) {
      pattern = {
        key: patternKey,
        type: errorRecord.type,
        count: 0,
        firstOccurrence: errorRecord.timestamp,
        lastOccurrence: errorRecord.timestamp,
        contexts: [],
        resolutions: []
      };
      this.errorPatterns.push(pattern);
    }

    pattern.count++;
    pattern.lastOccurrence = errorRecord.timestamp;

    // 记录上下文
    const contextHash = this.hashContext(errorRecord.context);
    if (!pattern.contexts.some(c => c.hash === contextHash)) {
      pattern.contexts.push({
        hash: contextHash,
        context: errorRecord.context,
        occurrences: 1
      });
    } else {
      const existingContext = pattern.contexts.find(c => c.hash === contextHash);
      if (existingContext) {
        existingContext.occurrences++;
      }
    }
  }

  private getErrorRecommendations(errors: ErrorRecord[]): ErrorRecommendation[] {
    const recommendations: ErrorRecommendation[] = [];

    // 分析最常见的错误类型
    const errorTypes = this.groupBy(errors, 'type');
    const mostCommonType = Object.entries(errorTypes)
      .sort(([, a], [, b]) => b.length - a.length)[0];

    if (mostCommonType && mostCommonType[1].length > errors.length * 0.3) { // 30%以上
      recommendations.push({
        type: 'PREVENTION',
        priority: 'high',
        title: `高频率 \${mostCommonType[0]} 错误`,
        description: `\${mostCommonType[1].length} 次错误中 \${mostCommonType[1].length} 次是 \${mostCommonType[0]} 错误`,
        implementation: this.getPreventionImplementation(mostCommonType[0]),
        expectedReduction: '60-80%'
      });
    }

    // 分析网络相关错误
    const networkErrors = errors.filter(e => e.type === 'network' || e.type === 'timeout');
    if (networkErrors.length > errors.length * 0.2) { // 20%以上
      recommendations.push({
        type: 'NETWORK_OPTIMIZATION',
        priority: 'medium',
        title: '网络连接优化',
        description: '检测到大量网络相关错误',
        implementation: `
// 实施网络优化策略
class NetworkOptimizer {
  private readonly circuitBreaker = new CircuitBreaker({
    failureThreshold: 5,
    recoveryTimeout: 60000,
    monitoringPeriod: 30000
  });

  async robustRequest(url: string, options: RequestInit = {}): Promise<Response> {
    return this.circuitBreaker.execute(async () => {
      // 使用指数退避重试
      return this.withRetry(() => fetch(url, {
        ...options,
        timeout: 30000
      }), 3);
    });
  }

  private async withRetry<T>(
    operation: () => Promise<T>,
    maxRetries: number = 3
  ): Promise<T> {
    for (let attempt = 1; attempt <= maxRetries; attempt++) {
      try {
        return await operation();
      } catch (error) {
        if (attempt === maxRetries || !this.isRetryableError(error)) {
          throw error;
        }

        const delay = Math.min(1000 * Math.pow(2, attempt - 1), 10000);
        await this.delay(delay);
      }
    }

    throw new Error('Max retries exceeded');
  }
}
        `,
        expectedReduction: '70-90%'
      });
    }

    return recommendations;
  }
}
```

## 使用方法

### 存储使用监控
```typescript
skill: "walrus-monitor"
// monitor-type: usage
// time-range: 7d
// metrics: ["total-size", "file-count", "cost", "aging"]
```

### 成本分析
```typescript
skill: "walrus-monitor"
// monitor-type: cost
// time-range: 30d
// metrics: ["storage-cost", "transaction-cost", "optimization-suggestions"]
```

### 性能监控
```typescript
skill: "walrus-monitor"
// monitor-type: performance
// time-range: 24h
// metrics: ["upload-speed", "download-speed", "error-rate", "bottlenecks"]
```

### 错误分析
```typescript
skill: "walrus-monitor"
// monitor-type: errors
// time-range: 7d
// metrics: ["error-patterns", "resolution-rate", "prevention-suggestions"]
```

---

*更新时间：2025-11-11*