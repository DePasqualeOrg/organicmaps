# Creating Your Own .MWM File Mirror Infrastructure

**Date**: 2025-07-13  
**Focus**: Building independent infrastructure to host Organic Maps .mwm files  
**Goal**: Create a scalable, reliable CDN for serving .mwm files to your library users

## Overview

This guide provides a comprehensive approach to creating your own mirror infrastructure for hosting Organic Maps .mwm files. Since using Organic Maps' sponsored infrastructure without permission could impact their sustainability, creating your own hosting solution ensures legal clarity and gives you complete control over availability and performance.

## Why Create Your Own Infrastructure?

### ✅ **Complete Independence**
- No dependency on third-party infrastructure availability
- Full control over hosting costs and scaling
- Custom caching and optimization strategies
- Ability to add your own metadata and services

### ✅ **Legal Clarity**  
- No ambiguity about usage rights
- Clear ownership of hosting infrastructure
- Ability to offer commercial services without restrictions
- Compliance with sponsorship terms of original infrastructure

### ✅ **Custom Features**
- Add analytics and usage tracking
- Implement custom rate limiting and quotas
- Add additional metadata and search capabilities
- Support for custom map regions or data sources

## Infrastructure Architecture

```
Your .MWM CDN Infrastructure
├── Data Synchronization Layer
│   ├── Mirror Script (Python/Go)
│   ├── Countries.txt Sync
│   ├── Version Tracking
│   └── Integrity Verification
├── Storage Layer
│   ├── Object Storage (S3/GCS/Azure)
│   ├── File Organization Structure  
│   ├── Redundancy & Backup
│   └── Archive Management
├── CDN Distribution Layer
│   ├── Global CDN (CloudFlare/AWS CloudFront)
│   ├── Regional Edge Locations
│   ├── Caching Strategies
│   └── Load Balancing
├── API & Metadata Layer
│   ├── Countries API Endpoint
│   ├── Version Management API
│   ├── Search & Discovery API
│   └── Download Progress Tracking
└── Monitoring & Management
    ├── Health Checks
    ├── Usage Analytics
    ├── Cost Monitoring
    └── Alerting
```

## Step 1: Data Synchronization System

### Enhanced Python Mirroring Script

Based on the existing `mwm_downloader.py`, here's a production-ready mirroring solution:

```python
#!/usr/bin/env python3
"""
Enhanced .MWM Mirror Synchronization Script
Downloads and maintains a complete mirror of Organic Maps .mwm files
"""

import argparse
import asyncio
import aiohttp
import aiofiles
import json
import logging
import hashlib
import time
import sys
from pathlib import Path
from typing import Dict, List, Optional, Tuple
from dataclasses import dataclass
from datetime import datetime, timezone

# Configuration
ORGANIC_MAPS_SERVERS = [
    'https://cdn-de1.organicmaps.app/',
    'https://cdn-us3.organicmaps.app/',
    'https://cdn-nl1.organicmaps.app/',
    'https://cdn-uk1.organicmaps.app/',
    'https://cdn-fi1.organicmaps.app/',
    'https://cdn.organicmaps.app/'
]

@dataclass
class MapFile:
    id: str
    size: int
    sha1_base64: str
    version: int
    parent_id: Optional[str] = None
    local_path: Optional[Path] = None

@dataclass
class SyncStats:
    total_files: int = 0
    downloaded_files: int = 0
    updated_files: int = 0
    failed_files: int = 0
    total_bytes: int = 0
    downloaded_bytes: int = 0
    start_time: float = 0
    
class MWMSynchronizer:
    def __init__(self, output_dir: Path, concurrent_downloads: int = 8, 
                 verify_checksums: bool = True):
        self.output_dir = Path(output_dir)
        self.concurrent_downloads = concurrent_downloads
        self.verify_checksums = verify_checksums
        self.logger = logging.getLogger(__name__)
        self.stats = SyncStats()
        
        # Create directory structure
        self.maps_dir = self.output_dir / 'maps'
        self.metadata_dir = self.output_dir / 'metadata'
        self.maps_dir.mkdir(parents=True, exist_ok=True)
        self.metadata_dir.mkdir(parents=True, exist_ok=True)
        
    async def download_countries_catalog(self) -> Tuple[Dict, int]:
        """Download and parse the countries.txt catalog"""
        self.logger.info("Downloading countries catalog...")
        
        for server in ORGANIC_MAPS_SERVERS:
            try:
                async with aiohttp.ClientSession() as session:
                    # Try to get current version first
                    version_url = f"{server}maps/countries.txt"
                    async with session.get(version_url, timeout=30) as response:
                        if response.status == 200:
                            content = await response.text()
                            countries_data = json.loads(content)
                            version = countries_data.get('v', 0)
                            
                            # Save catalog locally
                            catalog_path = self.metadata_dir / f'countries_{version}.txt'
                            async with aiofiles.open(catalog_path, 'w') as f:
                                await f.write(content)
                            
                            self.logger.info(f"Downloaded catalog version {version} from {server}")
                            return countries_data, version
                            
            except Exception as e:
                self.logger.warning(f"Failed to download from {server}: {e}")
                continue
                
        raise Exception("Failed to download countries catalog from all servers")
    
    def extract_map_files(self, countries_data: Dict, version: int) -> List[MapFile]:
        """Extract all map files from countries hierarchy"""
        map_files = []
        
        def extract_recursive(obj, parent_id=None):
            if isinstance(obj, list):
                for item in obj:
                    extract_recursive(item, parent_id)
            elif isinstance(obj, dict):
                if 's' in obj and 'id' in obj:  # This is a map file
                    map_file = MapFile(
                        id=obj['id'],
                        size=obj['s'],
                        sha1_base64=obj.get('sha1_base64', ''),
                        version=version,
                        parent_id=parent_id
                    )
                    map_files.append(map_file)
                    
                if 'g' in obj:  # Has children
                    current_id = obj.get('id')
                    extract_recursive(obj['g'], current_id)
        
        extract_recursive(countries_data)
        return map_files
    
    async def verify_file_integrity(self, file_path: Path, expected_sha1: str) -> bool:
        """Verify file SHA1 checksum"""
        if not self.verify_checksums or not expected_sha1:
            return True
            
        try:
            import base64
            sha1_hash = hashlib.sha1()
            async with aiofiles.open(file_path, 'rb') as f:
                async for chunk in f:
                    sha1_hash.update(chunk)
            
            actual_sha1 = base64.b64encode(sha1_hash.digest()).decode('utf-8')
            return actual_sha1 == expected_sha1
            
        except Exception as e:
            self.logger.error(f"Failed to verify {file_path}: {e}")
            return False
    
    async def download_map_file(self, session: aiohttp.ClientSession, 
                               map_file: MapFile, version: int) -> bool:
        """Download a single .mwm file"""
        filename = f"{map_file.id}.mwm"
        local_path = self.maps_dir / str(version) / filename
        local_path.parent.mkdir(parents=True, exist_ok=True)
        
        # Skip if file exists and is valid
        if local_path.exists():
            if await self.verify_file_integrity(local_path, map_file.sha1_base64):
                self.logger.debug(f"Skipping {filename} (already exists and valid)")
                return True
            else:
                self.logger.info(f"Re-downloading {filename} (checksum mismatch)")
        
        # Try each server until successful
        for server in ORGANIC_MAPS_SERVERS:
            try:
                url = f"{server}maps/{version}/{filename}"
                self.logger.info(f"Downloading {filename} from {server}")
                
                async with session.get(url, timeout=300) as response:
                    if response.status == 200:
                        # Download with progress tracking
                        downloaded = 0
                        async with aiofiles.open(local_path, 'wb') as f:
                            async for chunk in response.content.iter_chunked(8192):
                                await f.write(chunk)
                                downloaded += len(chunk)
                                self.stats.downloaded_bytes += len(chunk)
                        
                        # Verify checksum
                        if await self.verify_file_integrity(local_path, map_file.sha1_base64):
                            self.stats.downloaded_files += 1
                            self.logger.info(f"✓ Downloaded {filename} ({downloaded:,} bytes)")
                            return True
                        else:
                            local_path.unlink(missing_ok=True)
                            self.logger.error(f"✗ Checksum verification failed for {filename}")
                            
            except Exception as e:
                self.logger.warning(f"Failed to download {filename} from {server}: {e}")
                continue
        
        self.stats.failed_files += 1
        self.logger.error(f"✗ Failed to download {filename} from all servers")
        return False
    
    async def sync_all_maps(self, specific_maps: Optional[List[str]] = None) -> SyncStats:
        """Synchronize all maps or specific map list"""
        self.stats.start_time = time.time()
        
        # Download countries catalog
        countries_data, version = await self.download_countries_catalog()
        
        # Extract map files
        all_map_files = self.extract_map_files(countries_data, version)
        
        # Filter specific maps if requested
        if specific_maps:
            map_files = [f for f in all_map_files if f.id in specific_maps]
        else:
            map_files = all_map_files
        
        self.stats.total_files = len(map_files)
        self.stats.total_bytes = sum(f.size for f in map_files)
        
        self.logger.info(f"Starting sync of {self.stats.total_files} maps "
                        f"({self.stats.total_bytes:,} bytes total)")
        
        # Create semaphore for concurrent downloads
        semaphore = asyncio.Semaphore(self.concurrent_downloads)
        
        async def download_with_semaphore(map_file):
            async with semaphore:
                async with aiohttp.ClientSession() as session:
                    return await self.download_map_file(session, map_file, version)
        
        # Execute downloads
        tasks = [download_with_semaphore(map_file) for map_file in map_files]
        results = await asyncio.gather(*tasks, return_exceptions=True)
        
        # Calculate final stats
        successful = sum(1 for r in results if r is True)
        failed = len(results) - successful
        
        elapsed_time = time.time() - self.stats.start_time
        
        self.logger.info(f"Sync completed in {elapsed_time:.1f}s:")
        self.logger.info(f"  ✓ Downloaded: {successful} files")
        self.logger.info(f"  ✗ Failed: {failed} files")
        self.logger.info(f"  📊 Data transferred: {self.stats.downloaded_bytes:,} bytes")
        
        return self.stats
    
    def generate_metadata(self, countries_data: Dict, version: int):
        """Generate additional metadata files for your CDN"""
        
        # Generate simplified countries list
        simplified_catalog = {
            'version': version,
            'generated_at': datetime.now(timezone.utc).isoformat(),
            'countries': []
        }
        
        def build_simplified_tree(obj, parent_path=""):
            result = []
            if isinstance(obj, list):
                for item in obj:
                    result.extend(build_simplified_tree(item, parent_path))
            elif isinstance(obj, dict):
                current_path = f"{parent_path}/{obj['id']}" if parent_path else obj['id']
                
                if 's' in obj:  # Leaf node (actual map file)
                    result.append({
                        'id': obj['id'],
                        'path': current_path,
                        'size_bytes': obj['s'],
                        'download_url': f"maps/{version}/{obj['id']}.mwm"
                    })
                
                if 'g' in obj:  # Has children
                    result.extend(build_simplified_tree(obj['g'], current_path))
            
            return result
        
        simplified_catalog['countries'] = build_simplified_tree(countries_data.get('g', []))
        
        # Save simplified catalog
        catalog_path = self.metadata_dir / f'simplified_catalog_{version}.json'
        with open(catalog_path, 'w') as f:
            json.dump(simplified_catalog, f, indent=2)
        
        # Generate API endpoints data
        api_data = {
            'current_version': version,
            'base_url': 'https://your-cdn-domain.com/',
            'endpoints': {
                'countries': f'api/countries/{version}',
                'download': f'maps/{version}/{{country_id}}.mwm',
                'metadata': f'api/metadata/{version}/{{country_id}}'
            },
            'total_countries': len(simplified_catalog['countries']),
            'total_size_bytes': sum(c['size_bytes'] for c in simplified_catalog['countries'])
        }
        
        api_path = self.metadata_dir / f'api_info_{version}.json'
        with open(api_path, 'w') as f:
            json.dump(api_data, f, indent=2)

async def main():
    parser = argparse.ArgumentParser(description='Synchronize Organic Maps .mwm files')
    parser.add_argument('-o', '--output', required=True, help='Output directory')
    parser.add_argument('-c', '--concurrent', type=int, default=8, help='Concurrent downloads')
    parser.add_argument('-m', '--maps', nargs='+', help='Specific maps to download')
    parser.add_argument('--no-verify', action='store_true', help='Skip checksum verification')
    parser.add_argument('--verbose', action='store_true', help='Verbose logging')
    
    args = parser.parse_args()
    
    # Setup logging
    level = logging.DEBUG if args.verbose else logging.INFO
    logging.basicConfig(level=level, format='%(asctime)s - %(levelname)s - %(message)s')
    
    # Initialize synchronizer
    synchronizer = MWMSynchronizer(
        output_dir=args.output,
        concurrent_downloads=args.concurrent,
        verify_checksums=not args.no_verify
    )
    
    # Run synchronization
    try:
        stats = await synchronizer.sync_all_maps(args.maps)
        sys.exit(0 if stats.failed_files == 0 else 1)
    except Exception as e:
        logging.error(f"Synchronization failed: {e}")
        sys.exit(1)

if __name__ == '__main__':
    asyncio.run(main())
```

### Usage Examples

```bash
# Download all maps
python3 enhanced_mwm_sync.py -o /data/mwm-mirror --concurrent 16

# Download specific countries
python3 enhanced_mwm_sync.py -o /data/mwm-mirror -m "Germany" "France" "Italy"

# Incremental sync (skips existing valid files)
python3 enhanced_mwm_sync.py -o /data/mwm-mirror --concurrent 8

# Fast sync without checksum verification
python3 enhanced_mwm_sync.py -o /data/mwm-mirror --no-verify
```

## Step 2: Storage Architecture

### Cloud Storage Options

#### Option 1: AWS S3 + CloudFront

```yaml
# Terraform configuration for AWS infrastructure
resource "aws_s3_bucket" "mwm_storage" {
  bucket = "your-company-mwm-files"
  
  versioning {
    enabled = true
  }
  
  lifecycle_configuration {
    rule {
      enabled = true
      
      transition {
        days          = 30
        storage_class = "STANDARD_IA"
      }
      
      transition {
        days          = 90
        storage_class = "GLACIER"
      }
    }
  }
}

resource "aws_cloudfront_distribution" "mwm_cdn" {
  origin {
    domain_name = aws_s3_bucket.mwm_storage.bucket_regional_domain_name
    origin_id   = "mwm-s3-origin"
    
    s3_origin_config {
      origin_access_identity = aws_cloudfront_origin_access_identity.mwm_oai.cloudfront_access_identity_path
    }
  }
  
  enabled = true
  
  default_cache_behavior {
    allowed_methods  = ["GET", "HEAD", "OPTIONS"]
    cached_methods   = ["GET", "HEAD"]
    target_origin_id = "mwm-s3-origin"
    compress         = true
    
    cache_policy_id = data.aws_cloudfront_cache_policy.caching_optimized.id
    
    ttl {
      default_ttl = 86400  # 1 day
      max_ttl     = 31536000  # 1 year
    }
  }
  
  restrictions {
    geo_restriction {
      restriction_type = "none"
    }
  }
  
  viewer_certificate {
    acm_certificate_arn = aws_acm_certificate.mwm_cert.arn
    ssl_support_method  = "sni-only"
  }
}
```

#### Option 2: Google Cloud Storage + CDN

```yaml
# Google Cloud infrastructure
resources:
- name: mwm-storage-bucket
  type: storage.v1.bucket
  properties:
    name: your-company-mwm-files
    location: US
    storageClass: STANDARD
    lifecycle:
      rule:
      - action:
          type: SetStorageClass
          storageClass: NEARLINE
        condition:
          age: 30
      - action:
          type: SetStorageClass
          storageClass: COLDLINE
        condition:
          age: 90

- name: mwm-cdn
  type: compute.v1.backendBucket
  properties:
    bucketName: $(ref.mwm-storage-bucket.name)
    enableCdn: true
    cdnPolicy:
      cacheMode: CACHE_ALL_STATIC
      defaultTtl: 86400
      maxTtl: 31536000
```

### Directory Structure

```
your-mwm-storage/
├── maps/
│   ├── 250608/                 # Version-based organization
│   │   ├── World.mwm
│   │   ├── Germany.mwm
│   │   ├── Germany_Berlin.mwm
│   │   └── ...
│   ├── 250515/                 # Previous version
│   │   └── ...
│   └── latest/                 # Symlinks to current version
├── metadata/
│   ├── countries_250608.txt    # Original countries file
│   ├── simplified_catalog_250608.json
│   ├── api_info_250608.json
│   └── checksums/
│       ├── 250608_checksums.json
│       └── ...
├── api/
│   ├── v1/
│   │   ├── countries/
│   │   ├── search/
│   │   └── status/
│   └── docs/
└── logs/
    ├── sync/
    ├── access/
    └── errors/
```

## Step 3: CDN Distribution Setup

### CloudFlare Configuration

```javascript
// CloudFlare Worker for enhanced .mwm delivery
export default {
  async fetch(request, env) {
    const url = new URL(request.url);
    
    // Handle .mwm file requests
    if (url.pathname.endsWith('.mwm')) {
      return handleMWMRequest(request, env);
    }
    
    // Handle API requests
    if (url.pathname.startsWith('/api/')) {
      return handleAPIRequest(request, env);
    }
    
    return new Response('Not Found', { status: 404 });
  }
};

async function handleMWMRequest(request, env) {
  const cache = caches.default;
  const cacheKey = new Request(request.url, request);
  
  // Check cache first
  let response = await cache.match(cacheKey);
  if (response) {
    return response;
  }
  
  // Forward to origin
  response = await fetch(request);
  
  if (response.ok) {
    // Cache for 30 days
    const headers = new Headers(response.headers);
    headers.set('Cache-Control', 'public, max-age=2592000');
    headers.set('CDN-Cache-Control', 'max-age=2592000');
    
    response = new Response(response.body, {
      status: response.status,
      statusText: response.statusText,
      headers: headers
    });
    
    // Store in cache
    await cache.put(cacheKey, response.clone());
  }
  
  return response;
}
```

### Nginx Configuration for Origin Server

```nginx
# /etc/nginx/sites-available/mwm-origin
server {
    listen 443 ssl http2;
    server_name origin.your-mwm-cdn.com;
    
    ssl_certificate /path/to/certificate.pem;
    ssl_certificate_key /path/to/private-key.pem;
    
    # Security headers
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
    add_header X-Content-Type-Options nosniff always;
    add_header X-Frame-Options DENY always;
    
    # Root directory
    root /var/www/mwm-storage;
    
    # .mwm files
    location ~* \.mwm$ {
        expires 30d;
        add_header Cache-Control "public, immutable";
        
        # Enable range requests for large files
        add_header Accept-Ranges bytes;
        
        # Security: prevent direct access to certain files
        location ~* (countries\.txt|\.json)$ {
            deny all;
        }
    }
    
    # API endpoints
    location /api/ {
        proxy_pass http://api-backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        
        # Cache API responses
        proxy_cache api_cache;
        proxy_cache_valid 200 1h;
    }
    
    # Health check
    location /health {
        access_log off;
        return 200 "healthy\n";
        add_header Content-Type text/plain;
    }
    
    # Logging
    access_log /var/log/nginx/mwm-access.log combined;
    error_log /var/log/nginx/mwm-error.log warn;
}
```

## Step 4: API Layer Implementation

### RESTful API for Map Discovery

```python
# FastAPI implementation for .mwm API
from fastapi import FastAPI, HTTPException, Query
from fastapi.responses import JSONResponse
from typing import List, Optional
import json
from pathlib import Path

app = FastAPI(title="MWM Maps API", version="1.0.0")

class MWMMetadataAPI:
    def __init__(self, metadata_dir: Path):
        self.metadata_dir = Path(metadata_dir)
        self.current_version = self._get_current_version()
    
    def _get_current_version(self) -> int:
        # Find the latest version from metadata files
        version_files = list(self.metadata_dir.glob('api_info_*.json'))
        if not version_files:
            raise ValueError("No version metadata found")
        
        versions = []
        for file in version_files:
            try:
                version = int(file.stem.split('_')[-1])
                versions.append(version)
            except ValueError:
                continue
        
        return max(versions) if versions else 0

@app.get("/api/v1/version")
async def get_current_version():
    """Get current map data version"""
    return {"version": metadata_api.current_version}

@app.get("/api/v1/countries")
async def get_countries(
    version: Optional[int] = None,
    search: Optional[str] = Query(None, description="Search term")
):
    """Get list of available countries/regions"""
    if version is None:
        version = metadata_api.current_version
    
    catalog_path = metadata_api.metadata_dir / f'simplified_catalog_{version}.json'
    if not catalog_path.exists():
        raise HTTPException(status_code=404, detail="Version not found")
    
    with open(catalog_path) as f:
        catalog = json.load(f)
    
    countries = catalog['countries']
    
    # Apply search filter
    if search:
        countries = [c for c in countries if search.lower() in c['id'].lower()]
    
    return {
        'version': version,
        'total_count': len(countries),
        'countries': countries
    }

@app.get("/api/v1/countries/{country_id}")
async def get_country_info(country_id: str, version: Optional[int] = None):
    """Get detailed information about a specific country/region"""
    if version is None:
        version = metadata_api.current_version
    
    catalog_path = metadata_api.metadata_dir / f'simplified_catalog_{version}.json'
    if not catalog_path.exists():
        raise HTTPException(status_code=404, detail="Version not found")
    
    with open(catalog_path) as f:
        catalog = json.load(f)
    
    country = next((c for c in catalog['countries'] if c['id'] == country_id), None)
    if not country:
        raise HTTPException(status_code=404, detail="Country not found")
    
    return country

@app.get("/api/v1/download/{country_id}")
async def get_download_info(country_id: str, version: Optional[int] = None):
    """Get download information for a country"""
    if version is None:
        version = metadata_api.current_version
    
    country_info = await get_country_info(country_id, version)
    
    return {
        'country_id': country_id,
        'version': version,
        'download_url': f"https://your-cdn-domain.com/maps/{version}/{country_id}.mwm",
        'size_bytes': country_info['size_bytes'],
        'checksum_url': f"https://your-cdn-domain.com/api/v1/checksum/{country_id}?version={version}"
    }

@app.get("/api/v1/search")
async def search_countries(
    q: str = Query(..., description="Search query"),
    limit: int = Query(50, le=100),
    version: Optional[int] = None
):
    """Search for countries/regions"""
    countries_response = await get_countries(version=version, search=q)
    countries = countries_response['countries'][:limit]
    
    return {
        'query': q,
        'total_matches': len(countries),
        'results': countries
    }

# Initialize metadata API
metadata_api = MWMMetadataAPI('/path/to/metadata')

if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=8000)
```

## Step 5: Monitoring and Management

### Prometheus Monitoring

```yaml
# prometheus.yml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'mwm-api'
    static_configs:
      - targets: ['api-server:8000']
  
  - job_name: 'mwm-nginx'
    static_configs:
      - targets: ['origin-server:9113']  # nginx-prometheus-exporter

  - job_name: 'mwm-sync'
    static_configs:
      - targets: ['sync-server:8001']

rule_files:
  - "mwm_alerts.yml"

alerting:
  alertmanagers:
    - static_configs:
        - targets:
          - alertmanager:9093
```

### Alerting Rules

```yaml
# mwm_alerts.yml
groups:
- name: mwm_alerts
  rules:
  - alert: MWMSyncFailed
    expr: mwm_sync_failed_files > 0
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: "MWM sync has failed files"
      description: "{{ $value }} files failed to sync in the last sync operation"

  - alert: MWMHighErrorRate
    expr: rate(nginx_http_requests_total{status=~"5.."}[5m]) > 0.1
    for: 2m
    labels:
      severity: critical
    annotations:
      summary: "High error rate on MWM CDN"
      description: "Error rate is {{ $value }} requests/second"

  - alert: MWMStorageSpaceLow
    expr: (node_filesystem_free_bytes{mountpoint="/data"} / node_filesystem_size_bytes{mountpoint="/data"}) < 0.1
    for: 1m
    labels:
      severity: warning
    annotations:
      summary: "MWM storage space low"
      description: "Less than 10% storage space remaining"
```

### Automated Sync Script

```bash
#!/bin/bash
# sync_mwm_daily.sh - Daily synchronization script

set -euo pipefail

SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
LOG_DIR="/var/log/mwm-sync"
DATA_DIR="/data/mwm-storage"
BACKUP_DIR="/backup/mwm"

# Create log directory
mkdir -p "$LOG_DIR"

# Log function
log() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $*" | tee -a "$LOG_DIR/sync.log"
}

# Function to send alerts
send_alert() {
    local message="$1"
    local severity="${2:-info}"
    
    # Send to monitoring system
    curl -X POST "http://alertmanager:9093/api/v1/alerts" \
         -H "Content-Type: application/json" \
         -d "[{
           \"labels\": {
             \"alertname\": \"MWMSyncStatus\",
             \"severity\": \"$severity\"
           },
           \"annotations\": {
             \"summary\": \"$message\"
           }
         }]" || true
}

# Main sync function
main() {
    log "Starting MWM synchronization"
    
    # Check available space
    available_space=$(df "$DATA_DIR" | awk 'NR==2 {print $4}')
    required_space=1048576  # 1GB in KB
    
    if [ "$available_space" -lt "$required_space" ]; then
        log "ERROR: Insufficient disk space"
        send_alert "MWM sync failed: insufficient disk space" "critical"
        exit 1
    fi
    
    # Create backup of current version
    log "Creating backup..."
    if [ -d "$DATA_DIR/maps" ]; then
        rsync -av --delete "$DATA_DIR/maps/" "$BACKUP_DIR/$(date +%Y%m%d)/" || {
            log "WARNING: Backup failed"
            send_alert "MWM backup failed" "warning"
        }
    fi
    
    # Run synchronization
    log "Running synchronization..."
    cd "$SCRIPT_DIR"
    
    if python3 enhanced_mwm_sync.py \
        --output "$DATA_DIR" \
        --concurrent 8 \
        --verbose 2>&1 | tee -a "$LOG_DIR/sync.log"; then
        
        log "Synchronization completed successfully"
        send_alert "MWM sync completed successfully" "info"
        
        # Update latest symlinks
        if [ -d "$DATA_DIR/maps" ]; then
            latest_version=$(ls "$DATA_DIR/maps" | sort -nr | head -1)
            ln -sfn "$DATA_DIR/maps/$latest_version" "$DATA_DIR/maps/latest"
            log "Updated latest symlink to version $latest_version"
        fi
        
        # Cleanup old backups (keep 7 days)
        find "$BACKUP_DIR" -maxdepth 1 -type d -mtime +7 -exec rm -rf {} \; || true
        
    else
        log "ERROR: Synchronization failed"
        send_alert "MWM sync failed" "critical"
        exit 1
    fi
}

# Cleanup function
cleanup() {
    log "Sync process completed"
}

trap cleanup EXIT

main "$@"
```

## Step 6: Cost Optimization

### Storage Cost Analysis

```python
# Cost calculation tool
def calculate_monthly_costs():
    """Calculate estimated monthly costs for different providers"""
    
    # Approximate total size of all .mwm files
    total_size_gb = 500  # ~500GB for all world maps
    monthly_downloads_gb = 10000  # Estimate based on usage
    
    costs = {}
    
    # AWS S3 + CloudFront
    s3_storage_cost = total_size_gb * 0.023  # $0.023/GB for Standard
    cloudfront_cost = monthly_downloads_gb * 0.085  # $0.085/GB for first 10TB
    aws_total = s3_storage_cost + cloudfront_cost
    costs['AWS'] = aws_total
    
    # Google Cloud Storage + CDN
    gcs_storage_cost = total_size_gb * 0.020  # $0.020/GB for Standard
    gcs_cdn_cost = monthly_downloads_gb * 0.08  # $0.08/GB
    gcs_total = gcs_storage_cost + gcs_cdn_cost
    costs['GCP'] = gcs_total
    
    # CloudFlare + Origin Server
    cf_bandwidth_cost = 0  # Free tier: 100GB/month
    origin_server_cost = 50  # $50/month for VPS
    cf_total = cf_bandwidth_cost + origin_server_cost
    costs['CloudFlare'] = cf_total
    
    return costs

# Example output:
# AWS: ~$850/month
# GCP: ~$810/month  
# CloudFlare: ~$50/month (if under free tier limits)
```

### Traffic Optimization

```nginx
# Advanced caching configuration
location ~* \.mwm$ {
    # Serve from local cache if available
    try_files $uri @backend;
    
    # Enable compression for .mwm files (they're already compressed, but metadata can benefit)
    gzip on;
    gzip_vary on;
    gzip_comp_level 6;
    
    # Cache for 1 year (files are immutable)
    expires 1y;
    add_header Cache-Control "public, immutable";
    
    # Enable range requests for resumable downloads
    add_header Accept-Ranges bytes;
    
    # Rate limiting
    limit_req zone=download burst=10 nodelay;
}

# Rate limiting zones
http {
    limit_req_zone $binary_remote_addr zone=download:10m rate=5r/s;
    limit_req_zone $binary_remote_addr zone=api:10m rate=10r/s;
}
```

## Step 7: Legal and Attribution

### Proper Attribution Implementation

```html
<!-- Add to your application's about/credits page -->
<div class="attribution">
    <h3>Map Data Attribution</h3>
    <p>
        Map data © <a href="https://www.openstreetmap.org/copyright">OpenStreetMap contributors</a><br>
        Map rendering engine © <a href="https://organicmaps.app/">Organic Maps</a><br>
        Licensed under <a href="https://www.apache.org/licenses/LICENSE-2.0">Apache License 2.0</a>
    </p>
    
    <p>
        This application uses map data from OpenStreetMap, which is available under the 
        <a href="https://opendatacommons.org/licenses/odbl/">Open Database License (ODbL)</a>.
        The rendering engine is based on <a href="https://github.com/organicmaps/organicmaps">Organic Maps</a>,
        licensed under Apache 2.0.
    </p>
</div>
```

### Usage Compliance

```python
# Add to your API responses
def add_attribution_headers(response):
    """Add required attribution headers to API responses"""
    response.headers['X-Map-Data-License'] = 'ODbL-1.0'
    response.headers['X-Map-Data-Source'] = 'OpenStreetMap contributors'
    response.headers['X-Rendering-Engine'] = 'Organic Maps (Apache-2.0)'
    response.headers['X-Data-Attribution'] = 'https://www.openstreetmap.org/copyright'
    return response
```

## Deployment Checklist

### Pre-Launch Verification

- [ ] **Legal Compliance**
  - [ ] OpenStreetMap attribution implemented
  - [ ] Apache 2.0 license compliance verified
  - [ ] Terms of service created

- [ ] **Infrastructure Setup**
  - [ ] Storage backend configured
  - [ ] CDN distribution setup
  - [ ] SSL certificates installed
  - [ ] Monitoring and alerting configured

- [ ] **Data Synchronization**
  - [ ] Initial mirror sync completed
  - [ ] Checksum verification working
  - [ ] Automated sync schedule configured
  - [ ] Backup strategy implemented

- [ ] **API Implementation**
  - [ ] Countries discovery API working
  - [ ] Download endpoints functional
  - [ ] Rate limiting configured
  - [ ] Error handling implemented

- [ ] **Performance Optimization**
  - [ ] Caching strategies configured
  - [ ] Compression enabled
  - [ ] Range request support enabled
  - [ ] Load testing completed

- [ ] **Security**
  - [ ] HTTPS enforced
  - [ ] Security headers configured
  - [ ] Access logs enabled
  - [ ] Rate limiting active

## Expected Costs and Timeline

### Development Timeline

- **Week 1-2**: Infrastructure setup and basic sync script
- **Week 3-4**: API development and CDN configuration  
- **Week 5-6**: Monitoring, optimization, and testing
- **Week 7-8**: Documentation and launch preparation

### Monthly Operating Costs

- **Small Scale** (< 1TB/month): $50-100
- **Medium Scale** (1-10TB/month): $200-800
- **Large Scale** (10-100TB/month): $800-5000

### Benefits Over Using Organic Maps Infrastructure

1. **Legal Clarity**: No ambiguity about usage rights
2. **Customization**: Add your own metadata and features
3. **Reliability**: Complete control over uptime and performance
4. **Scaling**: Handle unlimited traffic without restrictions
5. **Analytics**: Full visibility into usage patterns
6. **Commercial Use**: Clear path for commercial applications

This approach provides a robust, scalable foundation for hosting .mwm files while respecting the Organic Maps project and ensuring long-term sustainability of your library.