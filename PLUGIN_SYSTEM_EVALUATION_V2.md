  1. Repository模式冲突

  // V2设计
  interface ImmichProviderRepository extends Omit<AssetRepository, 'create' | 'update'>

  // 现实问题: AssetRepository有100+方法，Omit会造成严重的类型混乱
  // asset.repository.ts实际有: getById, getByIds, getByLibraryId, updateAll等核心方法
  // V2的"扩展AssetRepository"根本不可行

  2. Database Schema问题

  -- V2提议
  ALTER TABLE assets ADD COLUMN provider_id VARCHAR;

  -- 现实冲突: assets表已有libraryId字段
  -- library.service.ts:676 已有: getByLibraryIdAndOriginalPath
  -- 这会造成概念重复和查询复杂化

  3. Job处理机制不匹配

  // V2设计假设
  await this.jobRepository.queueAll(jobs);

  // 现实: job.service.ts:304使用固定的nightly任务机制
  // library.service.ts使用特定的异步流处理
  // V2的同步机制与现有异步流不兼容

  性能问题分析

  1. 批处理冲突

  - 现有: JOBS_ASSET_PAGINATION_SIZE = 1000 (assets), JOBS_LIBRARY_PAGINATION_SIZE =
  10,000 (library)
  - V2设计: batchSize: 100 (过小，会造成过多job队列)
  - 影响: 增加3-5倍的job开销

  2. 缓存策略问题

  // V2设计
  getCachedAssetMetadata(assetId: string): Promise<any>

  // 现实: storage.core.ts没有metadata缓存机制
  // 需要完全重新设计缓存层

  3. 数据库查询性能

  V2的跨Provider搜索会造成严重性能问题:
  // V2设计会产生N个Provider * M个query的笛卡尔积
  for (const repository of repositories) {
    const results = await repository.provider.searchAssets(query); // 性能灾难
  }