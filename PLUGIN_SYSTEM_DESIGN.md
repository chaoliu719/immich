# Immich 插件系统重构方案

## 📋 项目概览

**目标**: 将 Immich 现有的外部存储功能重构为插件系统，支持多种数据源，并实现 overlay 机制

**核心理念**: 
- 外部只读数据源 → 插件适配器 → 统一数据接口 → Immich 核心 → Overlay 存储
- 渐进式重构，保持功能完整性
- 测试驱动开发，确保零回退

## 🔍 现状分析

### ✅ 现有优势
1. **完整的外部存储基础**: `LibraryService` 已支持文件扫描、监控、元数据提取
2. **优秀的测试覆盖**: 1294行单元测试 + E2E测试，覆盖全面
3. **模块化架构**: Service层高度解耦，易于重构
4. **异步任务系统**: 基于队列的处理机制适合插件化
5. **完整的权限系统**: 用户隔离和访问控制已就绪

### ⚠️ 当前挑战
1. **数据库耦合**: 所有数据存于单一PostgreSQL
2. **文件系统绑定**: 仅支持本地文件系统
3. **缺乏真正的overlay**: 用户自定义数据与外部数据混合存储
4. **扩展性限制**: 新增数据源需修改核心代码

## 🏗️ 系统架构设计

### 核心组件层次
```
┌─────────────────────────────────────────┐
│              Web/Mobile UI              │
└─────────────────┬───────────────────────┘
                  │
┌─────────────────┴───────────────────────┐
│         Immich API Layer                │
└─────────────────┬───────────────────────┘
                  │
┌─────────────────┴───────────────────────┐
│        Service Layer                    │
│ ┌─────────────┐ ┌─────────────┐        │
│ │AssetService │ │AlbumService │   ...  │
│ └─────────────┘ └─────────────┘        │
└─────────────────┬───────────────────────┘
                  │
┌─────────────────┴───────────────────────┐
│     Plugin Management Layer             │
│ ┌─────────────────────────────────────┐ │
│ │        Plugin Registry              │ │
│ │ ┌─────────┐ ┌─────────┐ ┌─────────┐ │ │
│ │ │LocalFile│ │Google   │ │Synology │ │ │
│ │ │Plugin   │ │Photos   │ │Photos   │ │ │
│ │ └─────────┘ └─────────┘ └─────────┘ │ │
│ └─────────────────────────────────────┘ │
└─────────────────┬───────────────────────┘
                  │
┌─────────────────┴───────────────────────┐
│     Data Abstraction Layer              │
│ ┌─────────────────────────────────────┐ │
│ │       Unified Repository            │ │
│ │ ┌─────────────┐ ┌─────────────────┐ │ │
│ │ │External Data│ │Overlay Repository│ │ │
│ │ │Repository   │ │                 │ │ │
│ │ └─────────────┘ └─────────────────┘ │ │
│ └─────────────────────────────────────┘ │
└─────────────────┬───────────────────────┘
                  │
┌─────────────────┴───────────────────────┐
│           Storage Layer                 │
│ ┌─────────────┐     ┌─────────────────┐ │
│ │External     │     │  PostgreSQL     │ │
│ │Data Sources │     │ (Overlay Data)  │ │
│ └─────────────┘     └─────────────────┘ │
└─────────────────────────────────────────┘
```

### 插件接口设计
```typescript
interface DataSourcePlugin extends ImmichPlugin {
  // 媒体文件获取
  getAssets(options: AssetQueryOptions): AsyncIterable<ExternalAsset>;
  getAssetStream(assetId: string): Promise<ReadableStream>;
  
  // 元数据获取
  getMetadata(assetId: string): Promise<AssetMetadata>;
  
  // 相册获取
  getAlbums(options: AlbumQueryOptions): AsyncIterable<ExternalAlbum>;
  getAlbumAssets(albumId: string): AsyncIterable<string>;
  
  // 标签获取
  getTags(options: TagQueryOptions): AsyncIterable<ExternalTag>;
  getAssetTags(assetId: string): Promise<string[]>;
  
  // 变更检测
  getChanges(since: Date): AsyncIterable<ChangeEvent>;
}
```

### Overlay 数据模型
```typescript
// 扩展现有 Asset 表
interface Asset {
  // ... 现有字段
  dataSourceId?: string;      // 数据源插件标识
  externalId: string;         // 外部系统中的 ID
  dataSourceMeta?: any;       // 插件特定元数据
}

// 新增 Overlay 表
interface AssetOverlay {
  assetId: string;           // 关联 Asset
  userId: string;            // 用户 ID
  isFavorite?: boolean;      // 用户收藏
  customTags?: string[];     // 用户自定义标签
  customDescription?: string; // 用户描述
  rating?: number;           // 用户评分
  createdAt: Date;
  updatedAt: Date;
}
```

## 📋 详细实施计划

### 🚨 阶段 0: 测试基线建立
- [ ] **运行现有测试建立基线**
  - 1294行 LibraryService 单元测试
  - E2E API 集成测试
  - 生成覆盖率报告作为基准
- [ ] **创建回归测试套件**
  - 复制关键测试用例作为金标准
  - 设置CI/CD测试门控
  - 任何测试失败都阻断进展

### 阶段 1: 插件系统基础 (高优先级)
- [ ] **设计插件系统核心架构**
  - 定义 `IDataSourcePlugin` 接口
  - 设计插件生命周期管理
  - 创建插件注册表机制
- [ ] **为插件接口编写单元测试** (TDD方式)
- [ ] **实现 Plugin Manager 和注册机制**
  - 插件加载/卸载/配置管理
  - 插件隔离和安全机制
  - 完整的单元测试覆盖

### 阶段 2: LocalFilePlugin 重构 (高优先级)
- [ ] **重构现有 Library Service 为 LocalFilePlugin**
  - **关键要求**: 保持所有现有测试通过 ✅
  - 将文件扫描逻辑迁移到插件
  - 保持API完全兼容
- [ ] **创建统一的数据抽象层**
  - `UnifiedRepository` 统一外部和本地数据访问
  - 数据源路由和缓存机制
  - 完整的单元和集成测试

### 阶段 3: Overlay 系统实现 (高优先级)
- [ ] **设计和实现 Overlay 数据模型**
  - 新增数据表: `asset_overlay`, `album_overlay`, `tag_overlay`
  - 数据库迁移脚本
  - Repository 层实现
- [ ] **实现 Overlay 机制核心逻辑**
  - 数据合并策略 (外部数据 + Overlay 数据)
  - 冲突解决机制
  - 权限和用户隔离
  - 完整的集成测试

### 阶段 4: 测试和验证 (高优先级)
- [ ] **开发 DummyPlugin 用于测试**
  - 模拟外部数据源
  - 验证插件系统完整性
  - 端到端测试套件
- [ ] **运行完整测试验证**
  - 所有原有测试通过
  - 新功能测试通过
  - 性能基准无回退

### 阶段 5: 管理和配置 (中优先级)
- [ ] **创建插件配置和管理 API**
  - 插件安装/卸载/配置接口
  - 数据源连接管理
  - 同步状态监控
  - API 测试覆盖
- [ ] **性能测试和压力测试**
  - 插件调用性能基准
  - 大数据量处理测试
  - 内存使用和泄漏检测

### 阶段 6: 迁移和文档 (中优先级)
- [ ] **编写数据迁移脚本**
  - 现有 Library 数据迁移到插件模式
  - 平滑升级脚本
  - 回滚机制和验证测试
- [ ] **插件开发文档和示例**
  - 插件开发指南
  - API 文档更新
  - 示例插件实现

## 🔬 测试策略

### 测试驱动原则
1. **红-绿-重构循环**:
   - 🔴 确保现有测试通过
   - 🟢 编写新功能测试（先失败）
   - 🔵 实现功能使测试通过
   - 🟡 重构优化（保持测试通过）

2. **测试覆盖要求**:
   - **单元测试**: 每个插件接口方法
   - **集成测试**: 插件与核心系统交互
   - **E2E测试**: 完整用户场景
   - **回归测试**: 现有功能零影响

3. **性能基准**:
   - 插件调用延迟 < 10ms
   - 内存使用不超过现有基准 +20%
   - 数据库查询性能不下降

## 🎯 关键成功指标

### 功能完整性
- ✅ 所有现有 Library 功能保持完整
- ✅ API 向后兼容 100%
- ✅ 数据完整性无损失
- ✅ 用户体验无感知

### 可扩展性
- ✅ 新增数据源无需修改核心代码
- ✅ 插件热插拔支持
- ✅ 多数据源并发处理
- ✅ Overlay 机制完全隔离用户数据

### 技术质量
- ✅ 测试覆盖率保持 >90%
- ✅ 性能无明显回退
- ✅ 代码质量提升 (解耦、模块化)
- ✅ 文档完整清晰

## 🚦 风险和缓解策略

### 主要风险
1. **重构引入回归**: 通过严格的测试保护缓解
2. **性能下降**: 性能基准测试和优化
3. **数据一致性**: 事务性操作和数据验证
4. **用户体验影响**: 渐进式发布和回滚机制

### 缓解措施
- 分阶段发布，每阶段充分验证
- 保持现有 API 完全兼容
- 提供详细的迁移指南和工具
- 建立完整的监控和告警机制

## 📁 关键文件位置

### 现有核心文件
- **LibraryService**: `server/src/services/library.service.ts` (主要重构目标)
- **单元测试**: `server/src/services/library.service.spec.ts` (1294行，必须保持通过)
- **E2E测试**: `e2e/src/api/specs/library.e2e-spec.ts`
- **数据模型**: `server/src/schema/tables/library.table.ts`
- **DTO定义**: `server/src/dtos/library.dto.ts`

### 新增文件计划
- **插件接口**: `server/src/plugins/interfaces/data-source-plugin.interface.ts`
- **插件管理器**: `server/src/plugins/plugin-manager.service.ts`
- **本地文件插件**: `server/src/plugins/local-file/local-file.plugin.ts`
- **统一Repository**: `server/src/repositories/unified.repository.ts`
- **Overlay数据表**: `server/src/schema/tables/asset-overlay.table.ts`
- **DummyPlugin**: `server/src/plugins/dummy/dummy.plugin.ts`

## 🚀 下一步行动

1. **立即开始**: 运行现有测试建立基线
2. **第一周**: 完成插件接口设计和测试
3. **第二周**: 实现 Plugin Manager (TDD方式)
4. **第三周**: 重构 LibraryService 为 LocalFilePlugin
5. **第四周**: 实现 Overlay 系统核心逻辑

---

**重要提醒**: 这个方案确保了**零风险的渐进式重构**，既实现了插件化目标，又保证了系统的稳定性和可靠性。每一步都有完整的测试保护，确保不会破坏现有功能。