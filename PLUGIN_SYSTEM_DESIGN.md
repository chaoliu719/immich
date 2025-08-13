# Immich Provider System Design Document

## Overview

This document outlines the design for implementing a Provider system in Immich. The core concept is to abstract all asset operations (similar to database CRUD operations) through a unified Provider interface, allowing Immich to aggregate photos from any data source without knowing the underlying implementation details.

## Design Philosophy

### Core Concept: Asset Operations Abstraction

Just like how applications interact with databases through SQL without caring about the underlying storage engine (MySQL, PostgreSQL, etc.), Immich will interact with asset sources through a unified Provider interface without caring whether the source is local files, Google Photos, S3, or any other storage system.

### Key Principles

1. **Unified Interface**: All Providers implement the same `ImmichProvider` interface
2. **CRUD Operations**: Create, Read, Update, Delete operations for assets
3. **Capability Declaration**: Each Provider declares what it can and cannot do
4. **Data Source Agnostic**: Immich core doesn't know or care about the underlying data source
5. **Open Ecosystem**: Third-party developers can easily create new Providers

## Core Architecture

### ImmichProvider Interface

```typescript
interface ImmichProvider {
  // Basic Information
  readonly id: string;                    // "google-photos", "s3-storage"
  readonly name: string;                  // "Google Photos Provider"
  readonly version: string;               // "1.2.3"
  readonly description: string;           // Provider description
  readonly capabilities: ProviderCapabilities;
  
  // Lifecycle Management
  initialize(config: ProviderConfig): Promise<void>;
  dispose(): Promise<void>;
  
  // Core CRUD Operations
  getAsset(id: string): Promise<Asset>;
  createAsset(data: AssetCreateData): Promise<Asset>;
  updateAsset(id: string, changes: AssetUpdateData): Promise<Asset>;
  deleteAsset(id: string): Promise<void>;
  
  // Query and Discovery
  listAssets(options: ListOptions): AsyncIterator<Asset>;
  searchAssets(query: SearchQuery): Promise<Asset[]>;
  discoverAssets(): AsyncIterator<Asset>;              // Actively discover new assets
  
  // Streaming Access
  getAssetStream(id: string): Promise<ReadableStream>;
  getAssetThumbnail(id: string, size: ThumbnailSize): Promise<Buffer>;
  
  // Change Monitoring
  onAssetChanged(callback: AssetChangeCallback): void;
  offAssetChanged(callback: AssetChangeCallback): void;
  
  // Health Check
  checkHealth(): Promise<HealthStatus>;
}
```

### Provider Capabilities System

```typescript
interface ProviderCapabilities {
  // Basic Operations
  canRead: boolean;                       // Can read assets
  canWrite: boolean;                      // Can create/modify assets
  canDelete: boolean;                     // Can delete assets
  canStream: boolean;                     // Supports streaming read
  
  // Advanced Features
  supportsSearch: boolean;                // Supports search functionality
  supportsWatch: boolean;                 // Supports change monitoring
  supportsMetadata: boolean;              // Supports metadata operations
  supportsThumbnails: boolean;            // Supports thumbnail generation
  supportsAlbums: boolean;                // Supports album concept
  
  // Performance Characteristics
  isRemote: boolean;                      // Is remote storage
  batchSize: number;                      // Batch processing size
  rateLimits?: RateLimitConfig;           // Rate limiting configuration
  
  // Custom Features
  customFeatures?: string[];              // Custom feature identifiers
}

interface RateLimitConfig {
  requestsPerMinute?: number;
  requestsPerHour?: number;
  requestsPerDay?: number;
  concurrentRequests?: number;
}
```

### Universal Asset Data Model

```typescript
interface Asset {
  id: string;
  providerId: string;                     // Which Provider provides this asset
  originalPath: string;                   // Original path/identifier
  
  // Basic Information
  filename: string;
  fileSize: number;
  mimeType: string;
  checksum?: string;
  
  // Time Information
  createdAt: Date;
  modifiedAt: Date;
  takenAt?: Date;                         // Photo/video taken time
  
  // Media Information
  type: AssetType;                        // IMAGE, VIDEO
  dimensions?: { width: number; height: number };
  duration?: number;                      // Video duration in seconds
  
  // Location Information
  location?: {
    latitude: number;
    longitude: number;
    address?: string;
    country?: string;
    state?: string;
    city?: string;
  };
  
  // Extended Metadata
  metadata?: Record<string, any>;         // Provider-specific metadata
  tags?: string[];                        // Tags/keywords
  albums?: string[];                      // Album associations
  
  // Immich-specific
  isFavorite?: boolean;
  isArchived?: boolean;
  exifData?: ExifData;
}

enum AssetType {
  IMAGE = 'IMAGE',
  VIDEO = 'VIDEO',
  AUDIO = 'AUDIO',
  DOCUMENT = 'DOCUMENT',
  OTHER = 'OTHER'
}
```

### Provider Configuration System

```typescript
interface ProviderConfig {
  // General Settings
  enabled: boolean;
  syncInterval?: number;                  // Sync interval in seconds
  
  // Authentication
  credentials?: {
    apiKey?: string;
    accessToken?: string;
    refreshToken?: string;
    username?: string;
    password?: string;
    clientId?: string;
    clientSecret?: string;
    [key: string]: any;                   // Flexible authentication fields
  };
  
  // Provider-specific Settings
  settings: Record<string, any>;
  
  // Content Filtering
  filters?: {
    includePatterns?: string[];
    excludePatterns?: string[];
    dateRange?: {
      from?: Date;
      to?: Date;
    };
    sizeRange?: {
      min?: number;                       // Minimum file size in bytes
      max?: number;                       // Maximum file size in bytes
    };
    assetTypes?: AssetType[];             // Allowed asset types
  };
  
  // Performance Settings
  batchSize?: number;
  maxConcurrentRequests?: number;
  timeout?: number;                       // Request timeout in milliseconds
}
```

## Provider Ecosystem Examples

### Local File Provider

```typescript
class LocalFileProvider implements ImmichProvider {
  id = 'local-files';
  name = 'Local File System Provider';
  version = '1.0.0';
  description = 'Provides access to local file system storage';
  
  capabilities: ProviderCapabilities = {
    canRead: true,
    canWrite: true,
    canDelete: true,
    canStream: true,
    supportsSearch: true,
    supportsWatch: true,          // File system monitoring
    supportsMetadata: true,
    supportsThumbnails: true,
    supportsAlbums: false,        // File system doesn't have album concept
    isRemote: false,
    batchSize: 10000,
  };
  
  async initialize(config: ProviderConfig): Promise<void> {
    // Initialize file system monitoring
    // Set up watched directories
  }
  
  async discoverAssets(): AsyncIterator<Asset> {
    // Scan configured directories
    // Extract file metadata
    // Yield assets one by one
  }
  
  // ... implement other interface methods
}
```

### Google Photos Provider

```typescript
class GooglePhotosProvider implements ImmichProvider {
  id = 'google-photos';
  name = 'Google Photos Provider';
  version = '1.0.0';
  description = 'Provides access to Google Photos library';
  
  capabilities: ProviderCapabilities = {
    canRead: true,
    canWrite: false,              // Google Photos API limitations
    canDelete: false,             // Google Photos API limitations
    canStream: true,
    supportsSearch: true,
    supportsWatch: false,         // No real-time notifications
    supportsMetadata: true,
    supportsThumbnails: true,
    supportsAlbums: true,
    isRemote: true,
    batchSize: 100,
    rateLimits: {
      requestsPerMinute: 1000,
      requestsPerDay: 75000,
    },
  };
  
  async initialize(config: ProviderConfig): Promise<void> {
    // Setup OAuth authentication
    // Validate API credentials
  }
  
  async searchAssets(query: SearchQuery): Promise<Asset[]> {
    // Use Google Photos search API
    // Convert Google Photos response to Asset format
  }
  
  // ... implement other interface methods
}
```

### S3 Storage Provider

```typescript
class S3StorageProvider implements ImmichProvider {
  id = 's3-storage';
  name = 'S3 Compatible Storage Provider';
  version = '1.0.0';
  description = 'Provides access to S3-compatible object storage';
  
  capabilities: ProviderCapabilities = {
    canRead: true,
    canWrite: true,
    canDelete: true,
    canStream: true,
    supportsSearch: false,        // S3 doesn't have built-in search
    supportsWatch: true,          // S3 Event Notifications
    supportsMetadata: true,
    supportsThumbnails: false,    // Need to generate locally
    supportsAlbums: false,        // Simulate with folder structure
    isRemote: true,
    batchSize: 1000,
  };
  
  async onAssetChanged(callback: AssetChangeCallback): void {
    // Setup S3 Event Notifications
    // Handle bucket events (put, delete)
  }
  
  // ... implement other interface methods
}
```

## Provider Manager Architecture

### ProviderManager

```typescript
class ProviderManager {
  private providers: Map<string, ImmichProvider> = new Map();
  private configs: Map<string, ProviderConfig> = new Map();
  
  // Provider Registration
  registerProvider(provider: ImmichProvider): void {
    this.providers.set(provider.id, provider);
  }
  
  unregisterProvider(providerId: string): void {
    const provider = this.providers.get(providerId);
    if (provider) {
      provider.dispose();
      this.providers.delete(providerId);
    }
  }
  
  // Provider Management
  async enableProvider(providerId: string, config: ProviderConfig): Promise<void> {
    const provider = this.providers.get(providerId);
    if (provider) {
      await provider.initialize(config);
      this.configs.set(providerId, config);
    }
  }
  
  async disableProvider(providerId: string): Promise<void> {
    const provider = this.providers.get(providerId);
    if (provider) {
      await provider.dispose();
      this.configs.delete(providerId);
    }
  }
  
  // Asset Operations (Proxy to Providers)
  async getAsset(providerId: string, assetId: string): Promise<Asset> {
    const provider = this.providers.get(providerId);
    return provider?.getAsset(assetId);
  }
  
  async searchAllProviders(query: SearchQuery): Promise<Asset[]> {
    const results: Asset[] = [];
    for (const [providerId, provider] of this.providers) {
      if (provider.capabilities.supportsSearch) {
        const providerResults = await provider.searchAssets(query);
        results.push(...providerResults);
      }
    }
    return results;
  }
  
  // Health Monitoring
  async checkAllProvidersHealth(): Promise<Map<string, HealthStatus>> {
    const healthStatuses = new Map<string, HealthStatus>();
    for (const [providerId, provider] of this.providers) {
      const health = await provider.checkHealth();
      healthStatuses.set(providerId, health);
    }
    return healthStatuses;
  }
}
```

## Database Schema

### Provider Configuration Table

```sql
CREATE TABLE provider_configs (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  provider_id VARCHAR NOT NULL,
  user_id UUID REFERENCES users(id),
  config JSONB NOT NULL,
  enabled BOOLEAN DEFAULT true,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW(),
  
  UNIQUE(provider_id, user_id)
);

CREATE INDEX idx_provider_configs_provider_id ON provider_configs(provider_id);
CREATE INDEX idx_provider_configs_user_id ON provider_configs(user_id);
```

### Asset Source Tracking

```sql
-- Extend existing assets table
ALTER TABLE assets ADD COLUMN provider_id VARCHAR;
ALTER TABLE assets ADD COLUMN provider_asset_id VARCHAR;
ALTER TABLE assets ADD COLUMN provider_metadata JSONB DEFAULT '{}';

-- Index for efficient provider queries
CREATE INDEX idx_assets_provider ON assets(provider_id, provider_asset_id);
```

## Implementation Strategy

### Phase 1: Foundation (2-3 weeks)
1. **Core Interface Definition**
   - Define `ImmichProvider` interface
   - Implement `ProviderManager` class
   - Create capability declaration system

2. **Database Schema Updates**
   - Add provider configuration table
   - Extend assets table with provider fields
   - Create necessary indexes

3. **Basic Provider Implementation**
   - Implement `LocalFileProvider` as the first example
   - Extract existing local file handling logic

### Phase 2: External Storage Migration (3-4 weeks)
1. **External Storage Provider**
   - Create `ExternalStorageProvider` to replace current Libraries functionality
   - Migrate existing library service logic
   - Ensure backward compatibility

2. **Provider Integration**
   - Integrate Provider system with existing asset services
   - Update asset creation/update flows
   - Implement provider-aware asset queries

### Phase 3: Cloud Providers (4-6 weeks)
1. **Google Photos Provider**
   - Implement OAuth authentication flow
   - Create Google Photos API integration
   - Handle rate limiting and pagination

2. **S3 Storage Provider**
   - Implement S3 API integration
   - Handle S3 Event Notifications for real-time updates
   - Support multiple S3-compatible services (AWS, MinIO, etc.)

### Phase 4: Management Interface (2-3 weeks)
1. **Admin Interface**
   - Provider management UI in admin panel
   - Provider configuration forms
   - Health monitoring dashboard

2. **User Interface**
   - Provider status indicators
   - Per-provider asset filtering
   - Provider-specific settings

## Security Considerations

### Authentication & Authorization
- Secure storage of provider credentials
- OAuth flow implementation for cloud providers
- Per-user provider configurations
- API key rotation mechanisms

### Data Privacy
- Provider-specific privacy controls
- Data residency considerations
- Audit logging for provider operations
- Compliance with data protection regulations

### Access Control
- Provider permission system
- Rate limiting enforcement
- Resource quota management
- Sandboxed provider execution

## Benefits of This Architecture

### For Users
1. **Unified Experience**: Access all photos from one interface
2. **Data Freedom**: Photos stay in their original locations
3. **Flexibility**: Mix and match different storage solutions
4. **Migration Ease**: Easy to move between different storage providers

### For Developers
1. **Clear Interface**: Simple, well-defined Provider interface
2. **Extensibility**: Easy to add new providers
3. **Community Contributions**: Standard way to contribute new integrations
4. **Testing**: Providers can be developed and tested independently

### For Immich
1. **Modularity**: Core remains focused on photo management
2. **Scalability**: Support for unlimited storage types
3. **Future-Proof**: Architecture adapts to new storage technologies
4. **Ecosystem Growth**: Encourage third-party development

## Future Possibilities

### Additional Provider Types
- **Social Media**: Instagram, Facebook, Twitter media
- **Cloud Storage**: Dropbox, OneDrive, iCloud Photos
- **Specialized Services**: Flickr, 500px, SmugMug
- **Enterprise**: SharePoint, Google Workspace, Office 365
- **IoT Devices**: Security cameras, IoT photo devices
- **Legacy Systems**: Old photo management software migration

### Advanced Features
- **Multi-Provider Search**: Search across all connected providers
- **Cross-Provider Sync**: Synchronize photos between providers
- **Provider Analytics**: Usage statistics and insights
- **Backup Strategies**: Automated cross-provider backup
- **AI Integration**: Provider-specific AI features

This Provider system transforms Immich from a photo management application into a photo aggregation platform, providing users with a unified interface to access and manage their photos regardless of where they are stored.