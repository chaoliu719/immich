# Immich Provider System V2 - 集成式设计

## 概述

基于对现有Immich架构的深入分析，V2设计采用**扩展而非替代**的策略。Provider系统作为现有Repository模式的自然扩展，充分复用Job、Cache、Storage等成熟基础设施，实现多源照片聚合功能。

## 核心设计原则

### 1. 架构集成
- Provider作为特殊的Repository实现
- 复用现有Job、Cache、Storage基础设施
- 保持与现有API的兼容性

### 2. 渐进式演进
- 从现有代码自然演进，而非大规模重构
- 先支持外部存储Provider
- 逐步扩展到云服务支持

## 核心架构

### ImmichProviderRepository

扩展现有Repository模式，集成Provider功能：

```typescript
interface ImmichProviderRepository extends Omit<AssetRepository, 'create' | 'update'> {
  readonly providerId: string;
  readonly provider: ImmichProvider;
  
  // 利用现有Job系统的异步操作
  queueSync(): Promise<void>;
  queueDiscovery(): Promise<void>;
  queueThumbnailGeneration(assetIds: string[]): Promise<void>;
  
  // Provider特有的操作
  getProviderAsset(providerAssetId: string): Promise<AssetEntity | null>;
  syncAssetFromProvider(providerAssetId: string): Promise<AssetEntity>;
  
  // 复用现有缓存机制
  getCachedAssetMetadata(assetId: string): Promise<any>;
  
  // 与现有权限系统集成
  checkProviderAccess(userId: string, operation: ProviderOperation): Promise<boolean>;
}
```

### 简化的Provider接口

专注核心能力，避免过度设计：

```typescript
interface ImmichProvider {
  readonly id: string;
  readonly name: string;
  readonly capabilities: ProviderCapabilities;
  
  // 核心数据访问
  getAssetMetadata(id: string): Promise<ProviderAssetMetadata>;
  getAssetStream(id: string): Promise<ReadableStream>;
  listAssets(options: ListOptions): AsyncIterator<ProviderAssetMetadata>;
  
  // 可选高级功能
  searchAssets?(query: string): Promise<ProviderAssetMetadata[]>;
  onAssetChanged?(callback: (changes: AssetChange[]) => void): void;
  
  // 生命周期管理
  initialize(config: ProviderConfig): Promise<void>;
  checkHealth(): Promise<HealthStatus>;
  dispose(): Promise<void>;
}

interface ProviderCapabilities {
  canRead: boolean;
  canWrite: boolean;
  canDelete: boolean;
  supportsSearch: boolean;
  supportsWatch: boolean;
  isRemote: boolean;
  batchSize: number;
  rateLimits?: {
    requestsPerSecond?: number;
    concurrentRequests?: number;
  };
}

interface ProviderAssetMetadata {
  id: string;
  originalPath: string;
  filename: string;
  fileSize: number;
  mimeType: string;
  checksum?: string;
  createdAt: Date;
  modifiedAt: Date;
  takenAt?: Date;
  
  // 可选扩展数据
  dimensions?: { width: number; height: number };
  location?: GeolocationData;
  tags?: string[];
  albums?: string[];
}
```

## 与现有Job系统集成

### 扩展Job类型

```typescript
enum ProviderJobName {
  ProviderSync = 'provider-sync',
  ProviderDiscovery = 'provider-discovery', 
  ProviderAssetImport = 'provider-asset-import',
  ProviderThumbnailGenerate = 'provider-thumbnail-generate',
  ProviderHealthCheck = 'provider-health-check'
}

interface ProviderJobData {
  providerId: string;
  userId: string;
  batchSize?: number;
  since?: Date;
}

interface ProviderAssetImportJobData extends ProviderJobData {
  providerAssetIds: string[];
  generateThumbnails?: boolean;
}
```

### ProviderService实现

```typescript
@Injectable()
export class ProviderService extends BaseService {
  private repositories = new Map<string, ImmichProviderRepository>();
  
  @OnJob({ name: ProviderJobName.ProviderSync, queue: QueueName.BackgroundTask })
  async handleProviderSync({ providerId, userId, since }: ProviderJobData): Promise<JobStatus> {
    const repository = this.repositories.get(providerId);
    if (!repository) return JobStatus.Failed;
    
    const jobs: JobItem[] = [];
    const assets = repository.provider.listAssets({ since });
    
    for await (const asset of assets) {
      jobs.push({
        name: ProviderJobName.ProviderAssetImport,
        data: { providerId, userId, providerAssetIds: [asset.id] }
      });
      
      // 复用现有批处理逻辑
      if (jobs.length >= JOBS_ASSET_PAGINATION_SIZE) {
        await this.jobRepository.queueAll(jobs);
        jobs.length = 0;
      }
    }
    
    await this.jobRepository.queueAll(jobs);
    return JobStatus.Success;
  }
  
  @OnJob({ name: ProviderJobName.ProviderAssetImport, queue: QueueName.BackgroundTask })
  async handleAssetImport({ providerId, providerAssetIds }: ProviderAssetImportJobData): Promise<JobStatus> {
    const repository = this.repositories.get(providerId);
    if (!repository) return JobStatus.Failed;
    
    for (const providerAssetId of providerAssetIds) {
      // 同步Provider资产到本地数据库
      const asset = await repository.syncAssetFromProvider(providerAssetId);
      
      // 复用现有缩略图生成Job
      await this.jobRepository.queue({
        name: JobName.AssetGenerateThumbnails,
        data: { id: asset.id }
      });
    }
    
    return JobStatus.Success;
  }
}
```

## 与现有Storage系统集成

### 扩展StorageCore

```typescript
export class ProviderStorageCore extends StorageCore {
  
  // Provider资产的路径管理
  static getProviderAssetPath(providerId: string, providerAssetId: string): string {
    return join(StorageCore.getBaseFolder(StorageFolder.Providers), providerId, providerAssetId);
  }
  
  // 复用现有缩略图路径逻辑
  static getProviderThumbnailPath(asset: ThumbnailPathEntity, providerId: string): string {
    const basePath = StorageCore.getImagePath(asset, AssetPathType.Thumbnail, 'jpeg');
    return basePath.replace('/thumbnails/', `/thumbnails/providers/${providerId}/`);
  }
  
  // Provider资产的缓存管理
  async cacheProviderAsset(providerId: string, providerAssetId: string, stream: ReadableStream): Promise<string> {
    const cachePath = ProviderStorageCore.getProviderAssetPath(providerId, providerAssetId);
    this.ensureFolders(cachePath);
    
    // 复用现有文件操作逻辑
    await this.storageRepository.writeStream(cachePath, stream);
    return cachePath;
  }
}

// 扩展StorageFolder枚举
enum StorageFolder {
  // ... 现有值
  Providers = 'providers'
}
```

## 数据库集成

### 最小化数据库变更

```sql
-- 扩展现有assets表
ALTER TABLE assets ADD COLUMN provider_id VARCHAR;
ALTER TABLE assets ADD COLUMN provider_asset_id VARCHAR;
ALTER TABLE assets ADD COLUMN provider_metadata JSONB DEFAULT '{}';
ALTER TABLE assets ADD COLUMN sync_status VARCHAR DEFAULT 'synced'; -- 'synced', 'pending', 'error'

-- Provider配置表
CREATE TABLE provider_configs (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID REFERENCES users(id) NOT NULL,
  provider_id VARCHAR NOT NULL,
  config JSONB NOT NULL,
  enabled BOOLEAN DEFAULT true,
  last_sync_at TIMESTAMPTZ,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW(),
  
  UNIQUE(user_id, provider_id)
);

-- 高效索引
CREATE INDEX idx_assets_provider ON assets(provider_id, provider_asset_id) WHERE provider_id IS NOT NULL;
CREATE INDEX idx_assets_sync_status ON assets(sync_status, provider_id) WHERE provider_id IS NOT NULL;
CREATE INDEX idx_provider_configs_user ON provider_configs(user_id, enabled);
```

## ProviderManager

```typescript
@Injectable()
export class ProviderManager {
  private providers = new Map<string, ImmichProvider>();
  private repositories = new Map<string, ImmichProviderRepository>();
  
  constructor(
    private assetRepository: AssetRepository,
    private jobRepository: JobRepository,
    private configRepository: ConfigRepository,
    private storageCore: ProviderStorageCore,
  ) {}
  
  registerProvider(provider: ImmichProvider): void {
    this.providers.set(provider.id, provider);
    
    // 创建集成Repository
    const repository = new ProviderRepositoryImpl(
      provider,
      this.assetRepository,
      this.jobRepository,
      this.storageCore
    );
    this.repositories.set(provider.id, repository);
  }
  
  async enableUserProvider(userId: string, providerId: string, config: ProviderConfig): Promise<void> {
    const provider = this.providers.get(providerId);
    if (!provider) throw new Error(`Provider ${providerId} not found`);
    
    await provider.initialize(config);
    
    // 保存配置到数据库
    await this.configRepository.upsert({
      userId,
      providerId,
      config,
      enabled: true
    });
    
    // 启动初始同步Job
    await this.jobRepository.queue({
      name: ProviderJobName.ProviderSync,
      data: { providerId, userId }
    });
  }
  
  async searchAcrossProviders(userId: string, query: string): Promise<AssetEntity[]> {
    const userConfigs = await this.configRepository.getByUserId(userId);
    const results: AssetEntity[] = [];
    
    for (const config of userConfigs) {
      if (!config.enabled) continue;
      
      const repository = this.repositories.get(config.providerId);
      if (repository?.provider.capabilities.supportsSearch) {
        // 复用现有搜索结果映射逻辑
        const providerResults = await repository.provider.searchAssets!(query);
        // 转换为统一的AssetEntity格式
        for (const result of providerResults) {
          const asset = await repository.getProviderAsset(result.id);
          if (asset) results.push(asset);
        }
      }
    }
    
    return results;
  }
  
  async checkAllProvidersHealth(userId: string): Promise<Map<string, HealthStatus>> {
    const userConfigs = await this.configRepository.getByUserId(userId);
    const healthStatuses = new Map<string, HealthStatus>();
    
    for (const config of userConfigs) {
      if (!config.enabled) continue;
      
      const provider = this.providers.get(config.providerId);
      if (provider) {
        const health = await provider.checkHealth();
        healthStatuses.set(config.providerId, health);
      }
    }
    
    return healthStatuses;
  }
}
```

## Provider实现示例

### LocalFileProvider

基于现有Library系统重构：

```typescript
export class LocalFileProvider implements ImmichProvider {
  id = 'local-files';
  name = 'Local File System';
  capabilities: ProviderCapabilities = {
    canRead: true,
    canWrite: false, // 只读，避免复杂的文件操作
    canDelete: false,
    supportsSearch: true,
    supportsWatch: true,
    isRemote: false,
    batchSize: 1000
  };
  
  private watcher?: FSWatcher;
  private config!: LocalFileProviderConfig;
  private onChange?: (changes: AssetChange[]) => void;
  
  async initialize(config: LocalFileProviderConfig): Promise<void> {
    this.config = config;
    
    if (this.capabilities.supportsWatch) {
      this.setupFileWatcher();
    }
  }
  
  async getAssetMetadata(id: string): Promise<ProviderAssetMetadata> {
    const filePath = this.resolveAssetPath(id);
    const stats = await fs.stat(filePath);
    
    return {
      id,
      originalPath: filePath,
      filename: path.basename(filePath),
      fileSize: stats.size,
      mimeType: getMimeType(filePath),
      createdAt: stats.birthtime,
      modifiedAt: stats.mtime,
      checksum: await this.calculateChecksum(filePath)
    };
  }
  
  async *listAssets(options: ListOptions): AsyncIterator<ProviderAssetMetadata> {
    // 复用现有的文件扫描逻辑
    const walker = walk(this.config.paths, {
      extensions: this.config.allowedExtensions,
      ...options
    });
    
    for await (const filePath of walker) {
      const stats = await fs.stat(filePath);
      const asset: ProviderAssetMetadata = {
        id: this.generateAssetId(filePath),
        originalPath: filePath,
        filename: path.basename(filePath),
        fileSize: stats.size,
        mimeType: getMimeType(filePath),
        createdAt: stats.birthtime,
        modifiedAt: stats.mtime,
        checksum: await this.calculateChecksum(filePath)
      };
      
      yield asset;
    }
  }
  
  async getAssetStream(id: string): Promise<ReadableStream> {
    const filePath = this.resolveAssetPath(id);
    return fs.createReadStream(filePath);
  }
  
  async searchAssets(query: string): Promise<ProviderAssetMetadata[]> {
    // 实现基于文件名的简单搜索
    const results: ProviderAssetMetadata[] = [];
    
    for await (const asset of this.listAssets({})) {
      if (asset.filename.toLowerCase().includes(query.toLowerCase())) {
        results.push(asset);
      }
    }
    
    return results;
  }
  
  async checkHealth(): Promise<HealthStatus> {
    try {
      // 检查配置的路径是否可访问
      for (const path of this.config.paths) {
        await fs.access(path, fs.constants.R_OK);
      }
      return { status: 'healthy' };
    } catch (error) {
      return { status: 'unhealthy', error: error.message };
    }
  }
  
  async dispose(): Promise<void> {
    this.watcher?.close();
  }
  
  private setupFileWatcher(): void {
    // 复用现有的chokidar监控逻辑
    this.watcher = chokidar.watch(this.config.paths, {
      ignoreInitial: true,
      ignorePermissionErrors: true
    });
    
    this.watcher.on('add', (path) => {
      this.onChange?.([{ type: 'added', assetId: this.generateAssetId(path) }]);
    });
    
    this.watcher.on('unlink', (path) => {
      this.onChange?.([{ type: 'deleted', assetId: this.generateAssetId(path) }]);
    });
  }
  
  private generateAssetId(filePath: string): string {
    // 基于文件路径生成稳定的ID
    return crypto.createHash('sha256').update(filePath).digest('hex');
  }
  
  private resolveAssetPath(id: string): string {
    // 从ID反向解析文件路径
    // 实现依赖于具体的ID生成策略
    return this.pathCache.get(id) || '';
  }
  
  private async calculateChecksum(filePath: string): Promise<string> {
    // 复用现有的校验和计算逻辑
    return crypto.createHash('sha256').update(await fs.readFile(filePath)).digest('hex');
  }
}

interface LocalFileProviderConfig extends ProviderConfig {
  paths: string[];
  allowedExtensions: string[];
  followSymlinks?: boolean;
  ignorePatterns?: string[];
}
```

### S3Provider

云存储集成示例：

```typescript
export class S3Provider implements ImmichProvider {
  id = 's3-storage';
  name = 'S3 Compatible Storage';
  capabilities: ProviderCapabilities = {
    canRead: true,
    canWrite: true,
    canDelete: true,
    supportsSearch: false,
    supportsWatch: true, // S3 Event Notifications
    isRemote: true,
    batchSize: 100,
    rateLimits: {
      requestsPerSecond: 10,
      concurrentRequests: 5
    }
  };
  
  private s3Client!: S3Client;
  private bucket!: string;
  private config!: S3ProviderConfig;
  
  async initialize(config: S3ProviderConfig): Promise<void> {
    this.config = config;
    this.s3Client = new S3Client({
      region: config.region,
      credentials: {
        accessKeyId: config.accessKeyId,
        secretAccessKey: config.secretAccessKey
      }
    });
    this.bucket = config.bucket;
  }
  
  async getAssetMetadata(id: string): Promise<ProviderAssetMetadata> {
    const response = await this.s3Client.send(new HeadObjectCommand({
      Bucket: this.bucket,
      Key: id
    }));
    
    return {
      id,
      originalPath: `s3://${this.bucket}/${id}`,
      filename: path.basename(id),
      fileSize: response.ContentLength || 0,
      mimeType: response.ContentType || getMimeType(id),
      createdAt: response.LastModified || new Date(),
      modifiedAt: response.LastModified || new Date(),
      checksum: response.ETag?.replace(/"/g, '')
    };
  }
  
  async *listAssets(options: ListOptions): AsyncIterator<ProviderAssetMetadata> {
    let continuationToken: string | undefined;
    
    do {
      const response = await this.rateLimitedRequest(() =>
        this.s3Client.send(new ListObjectsV2Command({
          Bucket: this.bucket,
          MaxKeys: this.capabilities.batchSize,
          ContinuationToken: continuationToken
        }))
      );
      
      for (const object of response.Contents || []) {
        if (!object.Key || !this.isImageOrVideo(object.Key)) continue;
        
        const asset: ProviderAssetMetadata = {
          id: object.Key,
          originalPath: `s3://${this.bucket}/${object.Key}`,
          filename: path.basename(object.Key),
          fileSize: object.Size || 0,
          mimeType: getMimeType(object.Key),
          createdAt: object.LastModified || new Date(),
          modifiedAt: object.LastModified || new Date(),
          checksum: object.ETag?.replace(/"/g, '')
        };
        
        yield asset;
      }
      
      continuationToken = response.NextContinuationToken;
    } while (continuationToken);
  }
  
  async getAssetStream(id: string): Promise<ReadableStream> {
    const response = await this.rateLimitedRequest(() =>
      this.s3Client.send(new GetObjectCommand({
        Bucket: this.bucket,
        Key: id
      }))
    );
    
    return response.Body as ReadableStream;
  }
  
  async checkHealth(): Promise<HealthStatus> {
    try {
      await this.s3Client.send(new HeadBucketCommand({
        Bucket: this.bucket
      }));
      return { status: 'healthy' };
    } catch (error) {
      return { status: 'unhealthy', error: error.message };
    }
  }
  
  async dispose(): Promise<void> {
    // 清理资源
  }
  
  private async rateLimitedRequest<T>(operation: () => Promise<T>): Promise<T> {
    // 实现基于capabilities.rateLimits的速率控制
    return operation();
  }
  
  private isImageOrVideo(key: string): boolean {
    const ext = path.extname(key).toLowerCase();
    return ['.jpg', '.jpeg', '.png', '.gif', '.mp4', '.mov', '.avi'].includes(ext);
  }
}

interface S3ProviderConfig extends ProviderConfig {
  region: string;
  bucket: string;
  accessKeyId: string;
  secretAccessKey: string;
  endpoint?: string; // 支持MinIO等S3兼容服务
}
```

## 扩展现有API

### AssetService集成

```typescript
// 扩展现有AssetService以支持Provider
@Injectable()
export class AssetService extends BaseService {
  constructor(
    // ... 现有依赖
    private providerManager: ProviderManager,
  ) {
    super();
  }
  
  async getWithProvider(auth: AuthDto, id: string): Promise<AssetResponseDto> {
    // 首先尝试从本地数据库获取
    let asset = await this.assetRepository.getById(id);
    
    // 如果是Provider资产且本地不存在，从Provider同步
    if (!asset) {
      // 解析ID确定Provider
      const { providerId, providerAssetId } = this.parseProviderAssetId(id);
      if (providerId) {
        const repository = this.providerManager.getRepository(providerId);
        asset = await repository?.syncAssetFromProvider(providerAssetId);
      }
    }
    
    if (!asset) {
      throw new BadRequestException('Asset not found');
    }
    
    return mapAsset(asset, { auth });
  }
  
  async searchAcrossAllSources(auth: AuthDto, query: string): Promise<AssetResponseDto[]> {
    // 搜索本地资产
    const localResults = await this.assetRepository.search(auth.user.id, query);
    
    // 搜索Provider资产
    const providerResults = await this.providerManager.searchAcrossProviders(auth.user.id, query);
    
    // 合并和去重结果
    const allResults = [...localResults, ...providerResults];
    return allResults.map(asset => mapAsset(asset, { auth }));
  }
}
```

### 前端Cache集成

```typescript
// 扩展现有Service Worker缓存以支持Provider资产
export async function getCachedOrFetchProvider(request: URL | Request | string, providerId?: string) {
  // 如果是Provider资产，使用特殊的缓存策略
  if (providerId) {
    const cacheKey = `provider-${providerId}-${getCacheKey(request)}`;
    const cached = await checkCache(cacheKey);
    if (cached) return cached;
    
    // Provider资产可能需要特殊的获取逻辑
    const response = await fetchProviderAsset(request, providerId);
    await setCached(response, cacheKey);
    return response;
  }
  
  // 回退到原有逻辑
  return getCachedOrFetch(request);
}

async function fetchProviderAsset(request: URL | Request | string, providerId: string): Promise<Response> {
  // 通过API获取Provider资产，可能涉及认证等
  const apiUrl = `/api/providers/${providerId}/assets/${getCacheKey(request)}`;
  return fetch(apiUrl);
}
```

## 权限系统扩展

```typescript
// 扩展现有Permission枚举
enum Permission {
  // ... 现有权限
  ProviderRead = 'provider.read',
  ProviderWrite = 'provider.write',
  ProviderConfig = 'provider.config',
  ProviderDelete = 'provider.delete'
}

// 扩展现有AccessService
@Injectable()
export class AccessService {
  async checkProviderAccess(
    auth: AuthDto,
    permission: Permission,
    providerId: string
  ): Promise<boolean> {
    // 检查用户是否有Provider的访问权限
    const config = await this.configRepository.getByUserAndProvider(auth.user.id, providerId);
    return config?.enabled && this.checkPermission(auth, permission);
  }
}
```