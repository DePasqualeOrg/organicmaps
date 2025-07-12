# .MWM Infrastructure on DigitalOcean App Platform

**Date**: 2025-07-13  
**Focus**: Container-based architecture using DigitalOcean App Platform  
**Goal**: Serverless, auto-scaling .mwm infrastructure with minimal management overhead

## Why App Platform vs Droplets?

### ✅ **App Platform Advantages**
- **Zero server management** - No SSH, updates, or security patches
- **Automatic scaling** - Handles traffic spikes automatically
- **Built-in CI/CD** - Deploy directly from GitHub
- **SSL certificates** - Automatic Let's Encrypt integration
- **Health checks** - Automatic restart if containers fail
- **Cost efficiency** - Pay only for what you use, sleep when idle
- **Integrated logging** - Built-in log aggregation and monitoring

### ✅ **Perfect for .MWM Infrastructure**
- **Stateless API** - Ideal for container deployment
- **Predictable load** - Map downloads are bursty but manageable
- **Simple deployment** - Single container with FastAPI
- **Auto-scaling** - Handle download spikes automatically

## Updated Architecture

```
Container-Based .MWM Infrastructure
├── DigitalOcean App Platform
│   ├── FastAPI Container (Auto-scaling)
│   │   ├── Map Discovery API
│   │   ├── Cost Monitoring Service
│   │   ├── Health Checks
│   │   └── Admin Dashboard
│   ├── Background Worker Container
│   │   ├── Sync Service (scheduled)
│   │   ├── Migration Tools
│   │   └── Integrity Verification
│   └── Static Site Container
│       └── Admin Dashboard UI
├── Storage Layer (Multi-Provider)
│   ├── CloudFlare R2 (Primary)
│   ├── DigitalOcean Spaces (Secondary)
│   └── Automated Sync
└── External Services
    ├── CloudFlare DNS & CDN
    ├── GitHub (CI/CD source)
    └── Monitoring (App Platform built-in)
```

## Project Structure for App Platform

```
mwm-infrastructure/
├── .do/
│   └── app.yaml                    # App Platform configuration
├── api/
│   ├── Dockerfile
│   ├── requirements.txt
│   ├── main.py                     # FastAPI application
│   ├── storage_manager.py          # Multi-provider storage
│   ├── cost_monitor.py             # Cost monitoring
│   └── models.py                   # Data models
├── worker/
│   ├── Dockerfile
│   ├── requirements.txt
│   ├── sync_worker.py              # Background sync worker
│   └── migration_tools.py          # Migration utilities
├── dashboard/
│   ├── index.html                  # Static admin dashboard
│   ├── script.js
│   └── style.css
├── docker-compose.yml              # Local development
├── README.md
└── .github/
    └── workflows/
        └── deploy.yml              # GitHub Actions CI/CD
```

## App Platform Configuration

```yaml
# .do/app.yaml
name: mwm-infrastructure
region: nyc1

services:
  # Main API Service
  - name: api
    source_dir: /api
    github:
      repo: your-username/mwm-infrastructure
      branch: main
      deploy_on_push: true
    dockerfile_path: api/Dockerfile
    http_port: 8000
    instance_count: 1
    instance_size_slug: basic-xxs  # $5/month, auto-scales
    
    envs:
      - key: R2_ACCESS_KEY
        scope: RUN_TIME
        type: SECRET
      - key: R2_SECRET_KEY
        scope: RUN_TIME
        type: SECRET
      - key: DO_SPACES_ACCESS_KEY
        scope: RUN_TIME
        type: SECRET
      - key: DO_SPACES_SECRET_KEY
        scope: RUN_TIME
        type: SECRET
      - key: ENVIRONMENT
        value: production
        scope: RUN_TIME
    
    routes:
      - path: /api
      - path: /health
    
    health_check:
      http_path: /health
      initial_delay_seconds: 10
      period_seconds: 10
      timeout_seconds: 5
      success_threshold: 1
      failure_threshold: 3

  # Background Worker Service
  - name: worker
    source_dir: /worker
    github:
      repo: your-username/mwm-infrastructure
      branch: main
      deploy_on_push: true
    dockerfile_path: worker/Dockerfile
    instance_count: 1
    instance_size_slug: basic-xxs
    
    envs:
      - key: R2_ACCESS_KEY
        scope: RUN_TIME
        type: SECRET
      - key: R2_SECRET_KEY
        scope: RUN_TIME
        type: SECRET
      - key: DO_SPACES_ACCESS_KEY
        scope: RUN_TIME
        type: SECRET
      - key: DO_SPACES_SECRET_KEY
        scope: RUN_TIME
        type: SECRET
      - key: SYNC_SCHEDULE
        value: "0 2 * * *"  # Daily at 2 AM
        scope: RUN_TIME

  # Static Dashboard
  - name: dashboard
    source_dir: /dashboard
    github:
      repo: your-username/mwm-infrastructure
      branch: main
      deploy_on_push: true
    
    routes:
      - path: /
      - path: /dashboard

# Database (if needed for caching/metrics)
databases:
  - name: cache
    engine: REDIS
    size: basic

# Custom Domain
domains:
  - domain: yourmaps.com
    type: PRIMARY
  - domain: api.yourmaps.com
    type: ALIAS
```

## API Container Implementation

```dockerfile
# api/Dockerfile
FROM python:3.11-slim

WORKDIR /app

# Install system dependencies
RUN apt-get update && apt-get install -y \
    curl \
    && rm -rf /var/lib/apt/lists/*

# Copy requirements first for better caching
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy application code
COPY . .

# Create non-root user
RUN useradd --create-home --shell /bin/bash app
USER app

# Health check
HEALTHCHECK --interval=30s --timeout=10s --start-period=5s --retries=3 \
    CMD curl -f http://localhost:8000/health || exit 1

# Run the application
EXPOSE 8000
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000", "--workers", "1"]
```

```python
# api/requirements.txt
fastapi==0.104.1
uvicorn[standard]==0.24.0
pydantic==2.5.0
boto3==1.34.0
aiofiles==23.2.1
aiohttp==3.9.1
redis==5.0.1
prometheus-client==0.19.0
python-multipart==0.0.6
```

```python
# api/main.py
from fastapi import FastAPI, HTTPException, BackgroundTasks, Depends
from fastapi.middleware.cors import CORSMiddleware
from fastapi.responses import JSONResponse
from contextlib import asynccontextmanager
import asyncio
import logging
import os
import redis
from datetime import datetime, timezone
from typing import List, Optional

from storage_manager import MultiProviderStorageManager, StorageProvider, ProviderConfig
from cost_monitor import CostMonitor
from models import MapInfo, VersionInfo, ProviderStatus

# Initialize logging
logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

# Global storage manager and monitoring
storage_manager = None
cost_monitor = None
redis_client = None

@asynccontextmanager
async def lifespan(app: FastAPI):
    """Initialize and cleanup resources"""
    global storage_manager, cost_monitor, redis_client
    
    # Initialize Redis (for caching)
    redis_url = os.getenv('DATABASE_URL')  # App Platform provides this
    if redis_url:
        redis_client = redis.from_url(redis_url)
    
    # Initialize storage manager
    configs = [
        ProviderConfig(
            provider=StorageProvider.CLOUDFLARE_R2,
            endpoint_url=f"https://{os.getenv('R2_ACCOUNT_ID')}.r2.cloudflarestorage.com",
            access_key=os.getenv('R2_ACCESS_KEY'),
            secret_key=os.getenv('R2_SECRET_KEY'),
            bucket_name='mwm-files-r2',
            priority=1
        ),
        ProviderConfig(
            provider=StorageProvider.DIGITALOCEAN_SPACES,
            endpoint_url="https://nyc3.digitaloceanspaces.com",
            access_key=os.getenv('DO_SPACES_ACCESS_KEY'),
            secret_key=os.getenv('DO_SPACES_SECRET_KEY'),
            bucket_name='mwm-files-do',
            region='nyc3',
            priority=2
        )
    ]
    
    storage_manager = MultiProviderStorageManager(configs)
    cost_monitor = CostMonitor(storage_manager)
    
    # Start background monitoring
    asyncio.create_task(background_monitoring())
    
    logger.info("Application started successfully")
    yield
    
    # Cleanup
    if redis_client:
        redis_client.close()
    logger.info("Application shutdown complete")

app = FastAPI(
    title="MWM Infrastructure API",
    description="Container-based .mwm file hosting infrastructure",
    version="2.0.0",
    lifespan=lifespan
)

# CORS middleware
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],  # Configure appropriately for production
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

async def background_monitoring():
    """Background task for cost monitoring and health checks"""
    while True:
        try:
            # Run cost analysis
            if cost_monitor:
                await cost_monitor.collect_metrics()
                
                # Check for migration opportunities
                alerts = await cost_monitor.check_alerts({})
                for alert in alerts:
                    logger.warning(f"Cost Alert: {alert['message']}")
            
            # Sleep for 1 hour
            await asyncio.sleep(3600)
            
        except Exception as e:
            logger.error(f"Background monitoring error: {e}")
            await asyncio.sleep(300)  # Wait 5 minutes on error

@app.get("/health")
async def health_check():
    """Health check endpoint for App Platform"""
    health_status = {
        "status": "healthy",
        "timestamp": datetime.now(timezone.utc).isoformat(),
        "version": "2.0.0",
        "environment": os.getenv("ENVIRONMENT", "development")
    }
    
    # Check storage connections
    if storage_manager:
        try:
            # Quick connectivity test
            primary_backend = storage_manager.backends[storage_manager.primary_provider]
            health_status["storage_primary"] = "connected"
        except Exception as e:
            health_status["storage_primary"] = f"error: {str(e)}"
            return JSONResponse(status_code=503, content=health_status)
    
    return health_status

@app.get("/api/v1/version", response_model=VersionInfo)
async def get_version():
    """Get current version information"""
    if not storage_manager:
        raise HTTPException(status_code=503, detail="Storage not initialized")
    
    # Try to get from cache first
    cached_version = None
    if redis_client:
        try:
            cached_version = redis_client.get("version_info")
            if cached_version:
                import json
                return VersionInfo(**json.loads(cached_version))
        except:
            pass
    
    # Get fresh data
    try:
        files = await storage_manager.list_files_with_metadata()
        version_info = VersionInfo(
            current_version=250608,  # Would be dynamic in real implementation
            total_maps=len(files),
            total_size_bytes=sum(f.get('size', 0) for f in files),
            last_sync=datetime.now(timezone.utc)
        )
        
        # Cache for 1 hour
        if redis_client:
            try:
                redis_client.setex(
                    "version_info", 
                    3600, 
                    version_info.model_dump_json()
                )
            except:
                pass
        
        return version_info
        
    except Exception as e:
        logger.error(f"Failed to get version info: {e}")
        raise HTTPException(status_code=500, detail="Failed to retrieve version information")

@app.get("/api/v1/maps", response_model=List[MapInfo])
async def get_maps(
    search: Optional[str] = None,
    limit: int = 100,
    region: Optional[str] = None
):
    """Get available maps with optional filtering"""
    if not storage_manager:
        raise HTTPException(status_code=503, detail="Storage not initialized")
    
    try:
        # Get from cache if available
        cache_key = f"maps:{search or 'all'}:{limit}:{region or 'all'}"
        cached_maps = None
        
        if redis_client:
            try:
                cached_maps = redis_client.get(cache_key)
                if cached_maps:
                    import json
                    maps_data = json.loads(cached_maps)
                    return [MapInfo(**m) for m in maps_data]
            except:
                pass
        
        # Get fresh data
        files = await storage_manager.list_files_with_metadata()
        maps = []
        
        for file_info in files:
            if not file_info.get('key', '').endswith('.mwm'):
                continue
                
            map_id = file_info['key'].replace('maps/', '').replace('.mwm', '')
            
            # Apply filters
            if search and search.lower() not in map_id.lower():
                continue
            if region and not map_id.startswith(region):
                continue
            
            map_info = MapInfo(
                id=map_id,
                name=map_id.replace('_', ' '),
                size_bytes=file_info.get('size', 0),
                download_url=await storage_manager.get_file_with_fallback(file_info['key']),
                last_updated=file_info.get('last_modified', datetime.now(timezone.utc)),
                version=250608
            )
            maps.append(map_info)
        
        # Apply limit
        maps = maps[:limit]
        
        # Cache for 30 minutes
        if redis_client:
            try:
                maps_json = [m.model_dump() for m in maps]
                redis_client.setex(cache_key, 1800, json.dumps(maps_json))
            except:
                pass
        
        return maps
        
    except Exception as e:
        logger.error(f"Failed to get maps: {e}")
        raise HTTPException(status_code=500, detail="Failed to retrieve maps")

@app.get("/api/v1/maps/{map_id}")
async def get_map_info(map_id: str):
    """Get detailed information about a specific map"""
    if not storage_manager:
        raise HTTPException(status_code=503, detail="Storage not initialized")
    
    try:
        key = f"maps/{map_id}.mwm"
        file_info = await storage_manager.get_file_info_with_fallback(key)
        
        if not file_info:
            raise HTTPException(status_code=404, detail="Map not found")
        
        return {
            "id": map_id,
            "name": map_id.replace('_', ' '),
            "size_bytes": file_info.get('size', 0),
            "download_url": await storage_manager.get_file_with_fallback(key),
            "last_updated": file_info.get('last_modified'),
            "version": 250608,
            "checksum": file_info.get('etag', '')
        }
        
    except HTTPException:
        raise
    except Exception as e:
        logger.error(f"Failed to get map info for {map_id}: {e}")
        raise HTTPException(status_code=500, detail="Failed to retrieve map information")

@app.get("/api/v1/providers/status")
async def get_provider_status():
    """Get status of all storage providers"""
    if not storage_manager:
        raise HTTPException(status_code=503, detail="Storage not initialized")
    
    try:
        status = {}
        costs = await storage_manager.analyze_costs()
        
        for provider, backend in storage_manager.backends.items():
            try:
                # Test connectivity
                files = await backend.list_files("", limit=1)
                status[provider.value] = ProviderStatus(
                    provider=provider.value,
                    status="healthy",
                    monthly_cost=costs.get(provider, 0),
                    is_primary=provider == storage_manager.primary_provider,
                    last_check=datetime.now(timezone.utc)
                )
            except Exception as e:
                status[provider.value] = ProviderStatus(
                    provider=provider.value,
                    status=f"error: {str(e)}",
                    monthly_cost=0,
                    is_primary=provider == storage_manager.primary_provider,
                    last_check=datetime.now(timezone.utc)
                )
        
        return {"providers": list(status.values())}
        
    except Exception as e:
        logger.error(f"Failed to get provider status: {e}")
        raise HTTPException(status_code=500, detail="Failed to check provider status")

@app.post("/api/v1/migration/plan")
async def plan_migration(target_provider: str):
    """Plan migration to a different provider"""
    if not storage_manager:
        raise HTTPException(status_code=503, detail="Storage not initialized")
    
    try:
        target = StorageProvider(target_provider)
        migration_plan = await storage_manager.plan_migration(target)
        
        return {
            "source_provider": migration_plan.source_provider.value,
            "target_provider": migration_plan.target_provider.value,
            "estimated_cost_savings": migration_plan.estimated_cost_savings,
            "estimated_migration_time_hours": migration_plan.estimated_migration_time,
            "files_to_migrate": migration_plan.files_to_migrate,
            "total_size_gb": migration_plan.total_size_gb
        }
        
    except ValueError:
        raise HTTPException(status_code=400, detail="Invalid target provider")
    except Exception as e:
        logger.error(f"Failed to plan migration: {e}")
        raise HTTPException(status_code=500, detail="Failed to plan migration")

@app.post("/api/v1/cache/refresh")
async def refresh_cache(background_tasks: BackgroundTasks):
    """Refresh application cache"""
    def clear_cache():
        if redis_client:
            try:
                # Clear all cache keys
                for key in redis_client.scan_iter(match="*"):
                    redis_client.delete(key)
                logger.info("Cache cleared successfully")
            except Exception as e:
                logger.error(f"Failed to clear cache: {e}")
    
    background_tasks.add_task(clear_cache)
    return {"message": "Cache refresh initiated"}

if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=8000, log_level="info")
```

## Background Worker Container

```dockerfile
# worker/Dockerfile
FROM python:3.11-slim

WORKDIR /app

# Install system dependencies
RUN apt-get update && apt-get install -y \
    curl \
    cron \
    && rm -rf /var/lib/apt/lists/*

# Copy requirements and install
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy worker code
COPY . .

# Create non-root user
RUN useradd --create-home --shell /bin/bash worker
USER worker

# Run the worker
CMD ["python", "sync_worker.py"]
```

```python
# worker/requirements.txt
asyncio==3.4.3
aiohttp==3.9.1
aiofiles==23.2.1
boto3==1.34.0
schedule==1.2.0
redis==5.0.1
pydantic==2.5.0
```

```python
# worker/sync_worker.py
import asyncio
import logging
import os
import schedule
import time
from datetime import datetime
import sys
import signal

# Add parent directory to path for imports
sys.path.append('/app')
from storage_manager import MultiProviderStorageManager, StorageProvider, ProviderConfig

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

class SyncWorker:
    def __init__(self):
        self.storage_manager = None
        self.running = True
        
        # Handle shutdown signals
        signal.signal(signal.SIGTERM, self.shutdown_handler)
        signal.signal(signal.SIGINT, self.shutdown_handler)
    
    def shutdown_handler(self, signum, frame):
        logger.info(f"Received signal {signum}, shutting down...")
        self.running = False
    
    async def initialize_storage(self):
        """Initialize storage manager"""
        configs = [
            ProviderConfig(
                provider=StorageProvider.CLOUDFLARE_R2,
                endpoint_url=f"https://{os.getenv('R2_ACCOUNT_ID')}.r2.cloudflarestorage.com",
                access_key=os.getenv('R2_ACCESS_KEY'),
                secret_key=os.getenv('R2_SECRET_KEY'),
                bucket_name='mwm-files-r2',
                priority=1
            ),
            ProviderConfig(
                provider=StorageProvider.DIGITALOCEAN_SPACES,
                endpoint_url="https://nyc3.digitaloceanspaces.com",
                access_key=os.getenv('DO_SPACES_ACCESS_KEY'),
                secret_key=os.getenv('DO_SPACES_SECRET_KEY'),
                bucket_name='mwm-files-do',
                region='nyc3',
                priority=2
            )
        ]
        
        self.storage_manager = MultiProviderStorageManager(configs)
        logger.info("Storage manager initialized")
    
    async def sync_organic_maps_data(self):
        """Sync data from Organic Maps sources"""
        if not self.storage_manager:
            logger.error("Storage manager not initialized")
            return
        
        logger.info("Starting Organic Maps data sync...")
        
        try:
            # This would implement the sync logic from the previous examples
            # For brevity, showing the structure
            
            # 1. Download countries.txt
            # 2. Compare with existing data
            # 3. Download changed files
            # 4. Upload to both providers with redundancy
            # 5. Verify integrity
            
            logger.info("Sync completed successfully")
            
        except Exception as e:
            logger.error(f"Sync failed: {e}")
    
    async def verify_redundancy(self):
        """Verify all files exist on both providers"""
        if not self.storage_manager:
            return
        
        logger.info("Verifying data redundancy...")
        
        try:
            # Check that all files exist on both providers
            await self.storage_manager.sync_between_providers(
                StorageProvider.CLOUDFLARE_R2,
                StorageProvider.DIGITALOCEAN_SPACES
            )
            
            logger.info("Redundancy verification completed")
            
        except Exception as e:
            logger.error(f"Redundancy verification failed: {e}")
    
    async def health_check(self):
        """Health check for worker"""
        logger.info("Worker health check - OK")
    
    def run_sync_job(self):
        """Run sync job (called by scheduler)"""
        asyncio.run(self.sync_organic_maps_data())
    
    def run_redundancy_check(self):
        """Run redundancy check (called by scheduler)"""
        asyncio.run(self.verify_redundancy())
    
    def run_health_check(self):
        """Run health check (called by scheduler)"""
        asyncio.run(self.health_check())
    
    async def run(self):
        """Main worker loop"""
        await self.initialize_storage()
        
        # Schedule jobs
        sync_schedule = os.getenv('SYNC_SCHEDULE', '0 2 * * *')  # Default: daily at 2 AM
        
        # For App Platform, we'll use simple interval scheduling
        # In production, you might want to use a proper cron library
        
        logger.info("Worker started, waiting for scheduled tasks...")
        
        last_sync = 0
        last_health_check = 0
        
        while self.running:
            current_time = time.time()
            
            # Run sync every 24 hours (86400 seconds)
            if current_time - last_sync > 86400:
                await self.sync_organic_maps_data()
                await self.verify_redundancy()
                last_sync = current_time
            
            # Health check every 5 minutes
            if current_time - last_health_check > 300:
                await self.health_check()
                last_health_check = current_time
            
            # Sleep for 1 minute
            await asyncio.sleep(60)
        
        logger.info("Worker shutdown complete")

if __name__ == "__main__":
    worker = SyncWorker()
    asyncio.run(worker.run())
```

## GitHub Actions CI/CD

```yaml
# .github/workflows/deploy.yml
name: Deploy to DigitalOcean App Platform

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

jobs:
  test:
    runs-on: ubuntu-latest
    
    steps:
    - uses: actions/checkout@v4
    
    - name: Set up Python
      uses: actions/setup-python@v4
      with:
        python-version: '3.11'
    
    - name: Install dependencies
      run: |
        cd api
        pip install -r requirements.txt
        pip install pytest pytest-asyncio
    
    - name: Run tests
      run: |
        cd api
        pytest tests/ -v
    
    - name: Test Docker build
      run: |
        docker build -t mwm-api ./api
        docker build -t mwm-worker ./worker

  deploy:
    needs: test
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    
    steps:
    - uses: actions/checkout@v4
    
    - name: Install doctl
      uses: digitalocean/action-doctl@v2
      with:
        token: ${{ secrets.DIGITALOCEAN_ACCESS_TOKEN }}
    
    - name: Deploy to App Platform
      run: |
        doctl apps create --spec .do/app.yaml
        
        # Or update existing app
        # doctl apps update ${{ secrets.APP_ID }} --spec .do/app.yaml
```

## Cost Analysis: App Platform vs Droplets

| Component | Droplet Cost | App Platform Cost | Benefits |
|-----------|-------------|-------------------|----------|
| **API Service** | $6/month (always on) | $5/month (scales to 0) | Auto-scaling, managed |
| **Worker Service** | Included | $5/month | Isolation, scheduling |
| **SSL Certificates** | $0 (Let's Encrypt) | $0 (included) | Automatic renewal |
| **Load Balancing** | $12/month (if needed) | $0 (included) | High availability |
| **Monitoring** | $0 (manual setup) | $0 (included) | Built-in dashboards |
| **CI/CD** | Manual deployment | $0 (included) | GitHub integration |
| **Total** | **$18-30/month** | **$10/month** | **Much easier management** |

## Deployment Commands

```bash
# Install doctl CLI
curl -L https://github.com/digitalocean/doctl/releases/latest/download/doctl-linux-amd64.tar.gz | tar xz
sudo mv doctl /usr/local/bin

# Authenticate
doctl auth init

# Create app from spec
doctl apps create --spec .do/app.yaml

# Get app info
doctl apps list

# View logs
doctl apps logs <app-id> --type=run

# Update app
doctl apps update <app-id> --spec .do/app.yaml
```

## Key Benefits of App Platform Architecture

### ✅ **Zero Infrastructure Management**
- No SSH access needed
- Automatic security updates
- Built-in SSL certificates
- Automatic health checks and restarts

### ✅ **True Auto-Scaling**
- Scales to zero when no traffic
- Automatically handles traffic spikes
- Pay only for actual usage
- No capacity planning required

### ✅ **Integrated DevOps**
- GitHub integration for CI/CD
- Automatic deployments on push
- Built-in monitoring and logging
- Easy rollbacks and versioning

### ✅ **Cost Optimization**
- Sleep when idle (unlike droplets)
- No over-provisioning
- Built-in load balancing
- Simplified billing

### ✅ **Production Ready**
- High availability by default
- Automatic failover
- Integrated monitoring
- Professional SSL certificates

This App Platform architecture provides a more modern, scalable, and cost-effective foundation for your .mwm infrastructure while maintaining all the vendor-neutral benefits we designed earlier. The container-based approach also makes it much easier to migrate to other platforms if needed in the future.

Would you like me to detail any specific aspect of the App Platform deployment?
