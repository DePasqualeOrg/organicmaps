# Low-Cost .MWM Infrastructure with CloudFlare R2 + DigitalOcean

**Date**: 2025-07-13  
**Focus**: Optimized low-cost infrastructure using CloudFlare R2 and DigitalOcean  
**Goal**: Host .mwm files at <$20/month with global CDN performance

## Architecture Overview

```
Low-Cost .MWM Infrastructure
├── CloudFlare R2 Object Storage
│   ├── Primary .mwm file storage (~500GB)
│   ├── Zero egress fees (unlimited downloads)
│   ├── S3-compatible API
│   └── Built-in global CDN (300+ PoPs)
├── DigitalOcean Droplet ($6/month)
│   ├── Sync script automation
│   ├── API backend (Node.js/Python)
│   ├── Monitoring & health checks
│   └── Admin dashboard
├── CloudFlare Free Tier
│   ├── DNS management
│   ├── SSL certificates
│   ├── DDoS protection
│   └── Analytics
└── Optional: DigitalOcean Spaces (backup)
    └── Secondary storage for redundancy
```

## Cost Breakdown

### Monthly Operating Costs

| Component | Service | Cost | Description |
|-----------|---------|------|-------------|
| **Primary Storage** | CloudFlare R2 | $7.50 | 500GB @ $0.015/GB |
| **Egress** | CloudFlare R2 | $0.00 | Unlimited free |
| **Operations** | CloudFlare R2 | $2.00 | Read/write operations |
| **Compute** | DigitalOcean Droplet | $6.00 | Basic plan (1GB RAM) |
| **Backup** | DigitalOcean Spaces | $5.00 | Optional redundancy |
| **Total** | | **$15.50-20.50** | Full global infrastructure |

**Comparison**: AWS equivalent would cost ~$850/month!

## Implementation Guide

### 1. CloudFlare R2 Setup

```python
# cloudflare_r2_setup.py
import boto3
import os
from botocore.config import Config

class CloudFlareR2Manager:
    def __init__(self):
        self.client = boto3.client(
            's3',
            endpoint_url='https://your-account-id.r2.cloudflarestorage.com',
            aws_access_key_id=os.getenv('R2_ACCESS_KEY'),
            aws_secret_access_key=os.getenv('R2_SECRET_KEY'),
            config=Config(signature_version='s3v4'),
            region_name='auto'
        )
        self.bucket_name = 'your-mwm-files'
    
    def create_bucket(self):
        """Create R2 bucket for .mwm files"""
        try:
            self.client.create_bucket(Bucket=self.bucket_name)
            print(f"Created bucket: {self.bucket_name}")
        except Exception as e:
            print(f"Bucket creation error (may already exist): {e}")
    
    def upload_mwm_file(self, local_path, key, metadata=None):
        """Upload .mwm file with proper caching headers"""
        extra_args = {
            'CacheControl': 'public, max-age=31536000, immutable',  # 1 year cache
            'ContentType': 'application/octet-stream'
        }
        
        if metadata:
            extra_args['Metadata'] = metadata
        
        try:
            self.client.upload_file(local_path, self.bucket_name, key, ExtraArgs=extra_args)
            print(f"Uploaded: {key}")
            return True
        except Exception as e:
            print(f"Upload failed for {key}: {e}")
            return False
    
    def sync_directory(self, local_dir, prefix="maps/"):
        """Sync entire directory to R2"""
        import os
        for root, dirs, files in os.walk(local_dir):
            for file in files:
                if file.endswith('.mwm'):
                    local_path = os.path.join(root, file)
                    s3_key = f"{prefix}{file}"
                    
                    # Check if file exists and has same size
                    try:
                        response = self.client.head_object(Bucket=self.bucket_name, Key=s3_key)
                        local_size = os.path.getsize(local_path)
                        if response['ContentLength'] == local_size:
                            print(f"Skipping {file} (already exists)")
                            continue
                    except:
                        pass  # File doesn't exist, upload it
                    
                    self.upload_mwm_file(local_path, s3_key)
    
    def generate_presigned_url(self, key, expiration=3600):
        """Generate presigned URL for direct downloads"""
        try:
            url = self.client.generate_presigned_url(
                'get_object',
                Params={'Bucket': self.bucket_name, 'Key': key},
                ExpiresIn=expiration
            )
            return url
        except Exception as e:
            print(f"Failed to generate URL for {key}: {e}")
            return None
    
    def list_maps(self, prefix="maps/"):
        """List all available .mwm files"""
        try:
            response = self.client.list_objects_v2(
                Bucket=self.bucket_name,
                Prefix=prefix
            )
            
            maps = []
            for obj in response.get('Contents', []):
                maps.append({
                    'key': obj['Key'],
                    'size': obj['Size'],
                    'last_modified': obj['LastModified'],
                    'download_url': f"https://your-domain.com/{obj['Key']}"
                })
            
            return maps
        except Exception as e:
            print(f"Failed to list maps: {e}")
            return []

# Usage example
if __name__ == "__main__":
    r2 = CloudFlareR2Manager()
    r2.create_bucket()
    
    # Sync local .mwm files to R2
    r2.sync_directory('/path/to/local/mwm/files')
    
    # List available maps
    maps = r2.list_maps()
    print(f"Available maps: {len(maps)}")
```

### 2. DigitalOcean Droplet Configuration

```bash
#!/bin/bash
# setup_droplet.sh - Configure DigitalOcean droplet for .mwm infrastructure

# Update system
apt update && apt upgrade -y

# Install required packages
apt install -y python3 python3-pip nginx certbot python3-certbot-nginx htop curl jq

# Install Python packages
pip3 install boto3 fastapi uvicorn aiofiles asyncio schedule prometheus_client

# Create application user
useradd -m -s /bin/bash mwmapp
sudo -u mwmapp mkdir -p /home/mwmapp/{app,logs,data}

# Setup nginx configuration
cat > /etc/nginx/sites-available/mwm-api << 'EOF'
server {
    listen 80;
    server_name your-domain.com;
    
    # Redirect HTTP to HTTPS
    return 301 https://$server_name$request_uri;
}

server {
    listen 443 ssl http2;
    server_name your-domain.com;
    
    # SSL certificates (will be configured by certbot)
    ssl_certificate /etc/letsencrypt/live/your-domain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/your-domain.com/privkey.pem;
    
    # Security headers
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
    add_header X-Content-Type-Options nosniff always;
    add_header X-Frame-Options DENY always;
    
    # API endpoints
    location /api/ {
        proxy_pass http://127.0.0.1:8000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        
        # API rate limiting
        limit_req zone=api burst=20 nodelay;
    }
    
    # Health check
    location /health {
        access_log off;
        return 200 "healthy\n";
        add_header Content-Type text/plain;
    }
    
    # Admin dashboard
    location / {
        root /home/mwmapp/app/static;
        try_files $uri $uri/ /index.html;
    }
}

# Rate limiting
http {
    limit_req_zone $binary_remote_addr zone=api:10m rate=10r/s;
}
EOF

# Enable site
ln -s /etc/nginx/sites-available/mwm-api /etc/nginx/sites-enabled/
rm /etc/nginx/sites-enabled/default
nginx -t && systemctl reload nginx

# Setup SSL with Let's Encrypt
certbot --nginx -d your-domain.com --non-interactive --agree-tos --email your-email@domain.com

# Create systemd service for API
cat > /etc/systemd/system/mwm-api.service << 'EOF'
[Unit]
Description=MWM API Server
After=network.target

[Service]
Type=simple
User=mwmapp
WorkingDirectory=/home/mwmapp/app
Environment=PATH=/usr/local/bin:/usr/bin:/bin
ExecStart=/usr/local/bin/uvicorn main:app --host 127.0.0.1 --port 8000
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
systemctl enable mwm-api

echo "Droplet setup complete! Configure your API application in /home/mwmapp/app/"
```

### 3. FastAPI Backend for Map Discovery

```python
# /home/mwmapp/app/main.py
from fastapi import FastAPI, HTTPException, Query, BackgroundTasks
from fastapi.middleware.cors import CORSMiddleware
from fastapi.responses import JSONResponse
import asyncio
import aiofiles
import json
import logging
from typing import List, Optional
from datetime import datetime, timezone
from pydantic import BaseModel
import os

# Import CloudFlare R2 manager
from cloudflare_r2 import CloudFlareR2Manager

app = FastAPI(
    title="MWM Maps API",
    description="Lightweight API for .mwm map file discovery and download",
    version="1.0.0"
)

# CORS configuration
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],  # Configure properly for production
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Initialize R2 manager
r2_manager = CloudFlareR2Manager()

# Data models
class MapInfo(BaseModel):
    id: str
    name: str
    size_bytes: int
    download_url: str
    last_updated: datetime
    version: int
    parent_region: Optional[str] = None

class VersionInfo(BaseModel):
    current_version: int
    total_maps: int
    total_size_bytes: int
    last_sync: datetime

class DownloadStats(BaseModel):
    total_downloads: int
    popular_maps: List[str]
    bandwidth_used_gb: float

# In-memory cache
cache = {
    'maps': [],
    'version_info': None,
    'last_updated': None
}

async def refresh_cache():
    """Refresh map cache from R2 and local metadata"""
    try:
        # Get current maps from R2
        maps_data = r2_manager.list_maps()
        
        # Load local metadata if available
        metadata_path = '/home/mwmapp/data/maps_metadata.json'
        metadata = {}
        if os.path.exists(metadata_path):
            async with aiofiles.open(metadata_path, 'r') as f:
                content = await f.read()
                metadata = json.loads(content)
        
        # Transform to MapInfo objects
        maps = []
        for map_data in maps_data:
            map_id = os.path.splitext(os.path.basename(map_data['key']))[0]
            
            map_info = MapInfo(
                id=map_id,
                name=metadata.get(map_id, {}).get('name', map_id.replace('_', ' ')),
                size_bytes=map_data['size'],
                download_url=f"https://your-r2-domain.com/{map_data['key']}",
                last_updated=map_data['last_modified'],
                version=metadata.get('version', 250608),
                parent_region=metadata.get(map_id, {}).get('parent')
            )
            maps.append(map_info)
        
        # Update cache
        cache['maps'] = maps
        cache['version_info'] = VersionInfo(
            current_version=metadata.get('version', 250608),
            total_maps=len(maps),
            total_size_bytes=sum(m.size_bytes for m in maps),
            last_sync=datetime.now(timezone.utc)
        )
        cache['last_updated'] = datetime.now(timezone.utc)
        
        logging.info(f"Cache refreshed: {len(maps)} maps loaded")
        
    except Exception as e:
        logging.error(f"Failed to refresh cache: {e}")

# Background task to refresh cache periodically
async def periodic_cache_refresh():
    while True:
        await refresh_cache()
        await asyncio.sleep(3600)  # Refresh every hour

@app.on_event("startup")
async def startup_event():
    # Initial cache load
    await refresh_cache()
    # Start background refresh task
    asyncio.create_task(periodic_cache_refresh())

@app.get("/api/v1/version", response_model=VersionInfo)
async def get_version_info():
    """Get current version and statistics"""
    if not cache['version_info']:
        raise HTTPException(status_code=503, detail="Service initializing")
    return cache['version_info']

@app.get("/api/v1/maps", response_model=List[MapInfo])
async def get_maps(
    search: Optional[str] = Query(None, description="Search term"),
    limit: int = Query(100, le=1000, description="Maximum results"),
    region: Optional[str] = Query(None, description="Filter by parent region")
):
    """Get list of available maps with optional filtering"""
    if not cache['maps']:
        raise HTTPException(status_code=503, detail="Service initializing")
    
    maps = cache['maps']
    
    # Apply filters
    if search:
        maps = [m for m in maps if search.lower() in m.name.lower() or search.lower() in m.id.lower()]
    
    if region:
        maps = [m for m in maps if m.parent_region == region]
    
    # Apply limit
    maps = maps[:limit]
    
    return maps

@app.get("/api/v1/maps/{map_id}", response_model=MapInfo)
async def get_map_info(map_id: str):
    """Get detailed information about a specific map"""
    if not cache['maps']:
        raise HTTPException(status_code=503, detail="Service initializing")
    
    map_info = next((m for m in cache['maps'] if m.id == map_id), None)
    if not map_info:
        raise HTTPException(status_code=404, detail="Map not found")
    
    return map_info

@app.get("/api/v1/maps/{map_id}/download")
async def get_download_info(map_id: str):
    """Get download information for a map"""
    map_info = await get_map_info(map_id)
    
    # Generate presigned URL for direct download (optional security)
    presigned_url = r2_manager.generate_presigned_url(f"maps/{map_id}.mwm")
    
    return {
        "map_id": map_id,
        "download_url": presigned_url or map_info.download_url,
        "size_bytes": map_info.size_bytes,
        "recommended_timeout": 300  # 5 minutes for large files
    }

@app.get("/api/v1/regions")
async def get_regions():
    """Get list of available regions"""
    if not cache['maps']:
        raise HTTPException(status_code=503, detail="Service initializing")
    
    regions = set()
    for map_info in cache['maps']:
        if map_info.parent_region:
            regions.add(map_info.parent_region)
    
    return {"regions": sorted(list(regions))}

@app.post("/api/v1/refresh")
async def refresh_maps_cache(background_tasks: BackgroundTasks):
    """Manually trigger cache refresh (admin endpoint)"""
    background_tasks.add_task(refresh_cache)
    return {"message": "Cache refresh triggered"}

@app.get("/health")
async def health_check():
    """Health check endpoint"""
    return {
        "status": "healthy",
        "timestamp": datetime.now(timezone.utc),
        "cache_updated": cache.get('last_updated'),
        "maps_count": len(cache.get('maps', []))
    }

if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="127.0.0.1", port=8000, log_level="info")
```

### 4. Automated Sync Script

```python
# /home/mwmapp/app/sync_mwm_files.py
import asyncio
import aiohttp
import aiofiles
import json
import logging
import schedule
import time
from pathlib import Path
from cloudflare_r2 import CloudFlareR2Manager
from datetime import datetime

logging.basicConfig(level=logging.INFO, format='%(asctime)s - %(levelname)s - %(message)s')
logger = logging.getLogger(__name__)

class MWMSyncService:
    def __init__(self):
        self.r2_manager = CloudFlareR2Manager()
        self.data_dir = Path('/home/mwmapp/data')
        self.data_dir.mkdir(exist_ok=True)
        
        # Organic Maps source servers
        self.source_servers = [
            'https://cdn-de1.organicmaps.app/',
            'https://cdn-us3.organicmaps.app/',
            'https://cdn-nl1.organicmaps.app/'
        ]
    
    async def download_countries_catalog(self):
        """Download countries.txt from Organic Maps"""
        logger.info("Downloading countries catalog...")
        
        for server in self.source_servers:
            try:
                async with aiohttp.ClientSession() as session:
                    url = f"{server}maps/countries.txt"
                    async with session.get(url, timeout=30) as response:
                        if response.status == 200:
                            content = await response.text()
                            countries_data = json.loads(content)
                            
                            # Save catalog
                            catalog_path = self.data_dir / 'countries.txt'
                            async with aiofiles.open(catalog_path, 'w') as f:
                                await f.write(content)
                            
                            logger.info(f"Downloaded catalog version {countries_data.get('v')}")
                            return countries_data
                            
            except Exception as e:
                logger.warning(f"Failed to download from {server}: {e}")
                continue
        
        raise Exception("Failed to download countries catalog")
    
    def extract_changed_files(self, countries_data, current_version):
        """Extract files that need updating"""
        new_files = []
        
        def extract_recursive(obj):
            if isinstance(obj, list):
                for item in obj:
                    extract_recursive(item)
            elif isinstance(obj, dict):
                if 's' in obj and 'id' in obj:
                    new_files.append({
                        'id': obj['id'],
                        'size': obj['s'],
                        'sha1': obj.get('sha1_base64', '')
                    })
                if 'g' in obj:
                    extract_recursive(obj['g'])
        
        extract_recursive(countries_data)
        
        # Filter files that need updating
        changed_files = []
        for file_info in new_files:
            # Check if file exists in R2 with same size
            try:
                existing = self.r2_manager.client.head_object(
                    Bucket=self.r2_manager.bucket_name,
                    Key=f"maps/{file_info['id']}.mwm"
                )
                if existing['ContentLength'] == file_info['size']:
                    continue  # File unchanged
            except:
                pass  # File doesn't exist
            
            changed_files.append(file_info)
        
        return changed_files
    
    async def sync_file_to_r2(self, session, file_info, version):
        """Download file from source and upload to R2"""
        filename = f"{file_info['id']}.mwm"
        logger.info(f"Syncing {filename}...")
        
        # Download from source
        for server in self.source_servers:
            try:
                url = f"{server}maps/{version}/{filename}"
                async with session.get(url, timeout=600) as response:
                    if response.status == 200:
                        # Stream directly to temporary file
                        temp_path = self.data_dir / f"temp_{filename}"
                        
                        async with aiofiles.open(temp_path, 'wb') as f:
                            async for chunk in response.content.iter_chunked(8192):
                                await f.write(chunk)
                        
                        # Upload to R2
                        success = self.r2_manager.upload_mwm_file(
                            str(temp_path),
                            f"maps/{filename}",
                            metadata={
                                'version': str(version),
                                'source': 'organic-maps',
                                'sync_date': datetime.now().isoformat()
                            }
                        )
                        
                        # Cleanup
                        temp_path.unlink(missing_ok=True)
                        
                        if success:
                            logger.info(f"✓ Synced {filename}")
                            return True
                        
            except Exception as e:
                logger.warning(f"Failed to sync {filename} from {server}: {e}")
                continue
        
        logger.error(f"✗ Failed to sync {filename}")
        return False
    
    async def full_sync(self):
        """Perform full synchronization"""
        try:
            logger.info("Starting MWM sync...")
            
            # Download catalog
            countries_data = await self.download_countries_catalog()
            version = countries_data.get('v', 0)
            
            # Find changed files
            changed_files = self.extract_changed_files(countries_data, version)
            logger.info(f"Found {len(changed_files)} files to sync")
            
            if not changed_files:
                logger.info("No files need updating")
                return
            
            # Sync files with limited concurrency
            semaphore = asyncio.Semaphore(3)  # Limit concurrent downloads
            
            async def sync_with_semaphore(file_info):
                async with semaphore:
                    async with aiohttp.ClientSession() as session:
                        return await self.sync_file_to_r2(session, file_info, version)
            
            tasks = [sync_with_semaphore(file_info) for file_info in changed_files]
            results = await asyncio.gather(*tasks, return_exceptions=True)
            
            # Update metadata
            metadata = {
                'version': version,
                'last_sync': datetime.now().isoformat(),
                'total_files': len(changed_files),
                'successful_syncs': sum(1 for r in results if r is True)
            }
            
            metadata_path = self.data_dir / 'sync_metadata.json'
            async with aiofiles.open(metadata_path, 'w') as f:
                await f.write(json.dumps(metadata, indent=2))
            
            logger.info(f"Sync completed: {metadata['successful_syncs']}/{metadata['total_files']} successful")
            
        except Exception as e:
            logger.error(f"Sync failed: {e}")

# Scheduling
def run_sync():
    sync_service = MWMSyncService()
    asyncio.run(sync_service.full_sync())

# Schedule daily sync at 2 AM
schedule.every().day.at("02:00").do(run_sync)

if __name__ == "__main__":
    # Run initial sync
    logger.info("Starting initial sync...")
    run_sync()
    
    # Start scheduler
    logger.info("Starting scheduler...")
    while True:
        schedule.run_pending()
        time.sleep(60)
```

### 5. Admin Dashboard

```html
<!-- /home/mwmapp/app/static/index.html -->
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>MWM Infrastructure Dashboard</title>
    <script src="https://cdn.jsdelivr.net/npm/chart.js"></script>
    <style>
        body {
            font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif;
            margin: 0;
            padding: 20px;
            background-color: #f5f5f5;
        }
        .dashboard {
            max-width: 1200px;
            margin: 0 auto;
        }
        .card {
            background: white;
            border-radius: 8px;
            padding: 20px;
            margin-bottom: 20px;
            box-shadow: 0 2px 4px rgba(0,0,0,0.1);
        }
        .stats-grid {
            display: grid;
            grid-template-columns: repeat(auto-fit, minmax(250px, 1fr));
            gap: 20px;
        }
        .stat-card {
            text-align: center;
        }
        .stat-number {
            font-size: 2em;
            font-weight: bold;
            color: #2196F3;
        }
        .stat-label {
            color: #666;
            margin-top: 5px;
        }
        .maps-table {
            width: 100%;
            border-collapse: collapse;
        }
        .maps-table th, .maps-table td {
            padding: 12px;
            text-align: left;
            border-bottom: 1px solid #ddd;
        }
        .maps-table th {
            background-color: #f8f9fa;
        }
        .search-box {
            width: 100%;
            padding: 10px;
            border: 1px solid #ddd;
            border-radius: 4px;
            margin-bottom: 20px;
        }
        .status-indicator {
            width: 12px;
            height: 12px;
            border-radius: 50%;
            display: inline-block;
            margin-right: 8px;
        }
        .status-healthy { background-color: #4CAF50; }
        .status-warning { background-color: #FF9800; }
        .status-error { background-color: #F44336; }
    </style>
</head>
<body>
    <div class="dashboard">
        <h1>🗺️ MWM Infrastructure Dashboard</h1>
        
        <!-- Status Cards -->
        <div class="stats-grid">
            <div class="card stat-card">
                <div class="stat-number" id="total-maps">-</div>
                <div class="stat-label">Total Maps</div>
            </div>
            <div class="card stat-card">
                <div class="stat-number" id="total-size">-</div>
                <div class="stat-label">Total Size</div>
            </div>
            <div class="card stat-card">
                <div class="stat-number" id="current-version">-</div>
                <div class="stat-label">Current Version</div>
            </div>
            <div class="card stat-card">
                <div class="stat-number" id="last-sync">-</div>
                <div class="stat-label">Last Sync</div>
            </div>
        </div>
        
        <!-- System Status -->
        <div class="card">
            <h2>System Status</h2>
            <div id="system-status">
                <div><span class="status-indicator status-healthy"></span>API Server: Healthy</div>
                <div><span class="status-indicator status-healthy"></span>CloudFlare R2: Connected</div>
                <div><span class="status-indicator status-healthy"></span>Sync Service: Running</div>
            </div>
        </div>
        
        <!-- Maps Browser -->
        <div class="card">
            <h2>Available Maps</h2>
            <input type="text" class="search-box" id="search-maps" placeholder="Search maps...">
            <table class="maps-table" id="maps-table">
                <thead>
                    <tr>
                        <th>Map ID</th>
                        <th>Name</th>
                        <th>Size</th>
                        <th>Last Updated</th>
                        <th>Download URL</th>
                    </tr>
                </thead>
                <tbody id="maps-tbody">
                    <tr><td colspan="5">Loading...</td></tr>
                </tbody>
            </table>
        </div>
        
        <!-- Cost Monitoring -->
        <div class="card">
            <h2>Cost Estimation</h2>
            <canvas id="cost-chart"></canvas>
        </div>
    </div>

    <script>
        // API base URL
        const API_BASE = '/api/v1';
        
        // Load dashboard data
        async function loadDashboard() {
            try {
                // Load version info
                const versionResponse = await fetch(`${API_BASE}/version`);
                const versionData = await versionResponse.json();
                
                document.getElementById('total-maps').textContent = versionData.total_maps.toLocaleString();
                document.getElementById('total-size').textContent = formatBytes(versionData.total_size_bytes);
                document.getElementById('current-version').textContent = versionData.current_version;
                document.getElementById('last-sync').textContent = formatDate(versionData.last_sync);
                
                // Load maps
                const mapsResponse = await fetch(`${API_BASE}/maps?limit=1000`);
                const mapsData = await mapsResponse.json();
                
                renderMapsTable(mapsData);
                renderCostChart(versionData);
                
            } catch (error) {
                console.error('Failed to load dashboard:', error);
            }
        }
        
        function renderMapsTable(maps) {
            const tbody = document.getElementById('maps-tbody');
            tbody.innerHTML = '';
            
            maps.forEach(map => {
                const row = document.createElement('tr');
                row.innerHTML = `
                    <td>${map.id}</td>
                    <td>${map.name}</td>
                    <td>${formatBytes(map.size_bytes)}</td>
                    <td>${formatDate(map.last_updated)}</td>
                    <td><a href="${map.download_url}" target="_blank">Download</a></td>
                `;
                tbody.appendChild(row);
            });
            
            // Add search functionality
            const searchBox = document.getElementById('search-maps');
            searchBox.addEventListener('input', (e) => {
                const query = e.target.value.toLowerCase();
                const rows = tbody.querySelectorAll('tr');
                
                rows.forEach(row => {
                    const text = row.textContent.toLowerCase();
                    row.style.display = text.includes(query) ? '' : 'none';
                });
            });
        }
        
        function renderCostChart(versionData) {
            const ctx = document.getElementById('cost-chart').getContext('2d');
            
            // Calculate costs
            const storageGB = Math.ceil(versionData.total_size_bytes / (1024 * 1024 * 1024));
            const monthlyCosts = {
                storage: storageGB * 0.015,
                compute: 6,
                domain: 1,
                total: function() { return this.storage + this.compute + this.domain; }
            };
            
            new Chart(ctx, {
                type: 'doughnut',
                data: {
                    labels: [
                        `Storage ($${monthlyCosts.storage.toFixed(2)})`,
                        `Compute ($${monthlyCosts.compute.toFixed(2)})`,
                        `Domain ($${monthlyCosts.domain.toFixed(2)})`
                    ],
                    datasets: [{
                        data: [monthlyCosts.storage, monthlyCosts.compute, monthlyCosts.domain],
                        backgroundColor: ['#FF6384', '#36A2EB', '#FFCE56']
                    }]
                },
                options: {
                    responsive: true,
                    plugins: {
                        title: {
                            display: true,
                            text: `Monthly Cost: $${monthlyCosts.total().toFixed(2)}`
                        }
                    }
                }
            });
        }
        
        function formatBytes(bytes) {
            if (bytes === 0) return '0 Bytes';
            const k = 1024;
            const sizes = ['Bytes', 'KB', 'MB', 'GB', 'TB'];
            const i = Math.floor(Math.log(bytes) / Math.log(k));
            return parseFloat((bytes / Math.pow(k, i)).toFixed(2)) + ' ' + sizes[i];
        }
        
        function formatDate(dateString) {
            return new Date(dateString).toLocaleDateString();
        }
        
        // Auto-refresh dashboard every 5 minutes
        setInterval(loadDashboard, 5 * 60 * 1000);
        
        // Initial load
        loadDashboard();
    </script>
</body>
</html>
```

## Deployment Commands

```bash
# 1. Setup DigitalOcean Droplet
doctl compute droplet create mwm-server \
  --size s-1vcpu-1gb \
  --image ubuntu-22-04-x64 \
  --region nyc1 \
  --ssh-keys your-ssh-key-id

# 2. Configure CloudFlare R2
# Create bucket and get credentials from CloudFlare dashboard

# 3. Setup environment variables
cat > /home/mwmapp/app/.env << 'EOF'
R2_ACCESS_KEY=your-access-key
R2_SECRET_KEY=your-secret-key
R2_ACCOUNT_ID=your-account-id
EOF

# 4. Start services
systemctl start mwm-api
systemctl enable mwm-api

# 5. Setup cron for sync service
echo "0 2 * * * /usr/bin/python3 /home/mwmapp/app/sync_mwm_files.py" | crontab -
```

## Performance Optimizations

### 1. CloudFlare Settings

```javascript
// CloudFlare Page Rules for optimal caching
// Pattern: your-domain.com/maps/*
Cache Level: Cache Everything
Edge Cache TTL: 1 month
Browser Cache TTL: 1 month

// Pattern: your-domain.com/api/*
Cache Level: Bypass
```

### 2. Nginx Optimizations

```nginx
# Add to nginx configuration for better .mwm delivery
location ~* \.mwm$ {
    # Cache headers for .mwm files
    expires 1y;
    add_header Cache-Control "public, immutable";
    
    # Enable range requests for resumable downloads
    add_header Accept-Ranges bytes;
    
    # Optimize for large files
    proxy_buffering off;
    proxy_request_buffering off;
    
    # Proxy to CloudFlare R2
    proxy_pass https://your-r2-domain.com;
}
```

## Monitoring & Alerts

### CloudFlare Analytics Integration

```javascript
// Add to your web pages for usage tracking
window.addEventListener('load', function() {
    // Track .mwm downloads
    document.querySelectorAll('a[href$=".mwm"]').forEach(link => {
        link.addEventListener('click', function() {
            // Send analytics event
            fetch('/api/v1/analytics/download', {
                method: 'POST',
                headers: {'Content-Type': 'application/json'},
                body: JSON.stringify({
                    map_id: this.href.split('/').pop().replace('.mwm', ''),
                    timestamp: Date.now()
                })
            });
        });
    });
});
```

## Expected Results

### Performance Metrics
- **Global CDN**: Sub-100ms response times worldwide
- **Download Speed**: Full CloudFlare edge network performance
- **Availability**: 99.9%+ uptime with redundancy

### Cost Comparison
| Scale | Your Setup | AWS Equivalent | Savings |
|-------|------------|----------------|---------|
| **500GB + 5TB/month** | $15/month | $850/month | 98% |
| **1TB + 10TB/month** | $25/month | $1,650/month | 98% |
| **2TB + 20TB/month** | $45/month | $3,250/month | 99% |

This architecture provides enterprise-grade .mwm hosting at consumer-friendly prices while maintaining full control and legal clarity.