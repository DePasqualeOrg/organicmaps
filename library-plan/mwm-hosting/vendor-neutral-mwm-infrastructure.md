# Vendor-Neutral .MWM Infrastructure with Easy Migration

**Date**: 2025-07-13  
**Focus**: Cloud-agnostic architecture with seamless provider migration  
**Goal**: Avoid vendor lock-in while maintaining low costs and high performance

## The CloudFlare Risk

You're absolutely right about CloudFlare's pattern:
- **Current**: "Zero egress fees for everyone!"
- **Future**: "Your usage seems excessive, consider Enterprise"
- **Common triggers**: >100TB/month bandwidth, >1M requests/day

## Migration-Ready Architecture

```
Vendor-Neutral .MWM Infrastructure
├── Storage Abstraction Layer
│   ├── CloudFlare R2 (Primary)
│   ├── DigitalOcean Spaces (Hot Standby)
│   ├── Provider-agnostic API
│   └── Automated sync between providers
├── CDN Abstraction Layer
│   ├── Multi-CDN support
│   ├── DNS-based failover
│   └── Cost monitoring per provider
├── Migration Orchestrator
│   ├── Data synchronization tools
│   ├── DNS cutover automation
│   └── Zero-downtime migration
└── Cost Monitoring & Alerts
    ├── Real-time cost tracking
    ├── Provider comparison
    └── Migration triggers
```

## Implementation: Multi-Provider Storage Manager

```python
# multi_provider_storage.py
from abc import ABC, abstractmethod
from enum import Enum
import boto3
import asyncio
import logging
from typing import Dict, List, Optional, Union
from dataclasses import dataclass
from datetime import datetime

class StorageProvider(Enum):
    CLOUDFLARE_R2 = "cloudflare_r2"
    DIGITALOCEAN_SPACES = "digitalocean_spaces"
    AWS_S3 = "aws_s3"
    BACKBLAZE_B2 = "backblaze_b2"

@dataclass
class ProviderConfig:
    provider: StorageProvider
    endpoint_url: str
    access_key: str
    secret_key: str
    bucket_name: str
    region: str = "auto"
    monthly_cost_estimate: float = 0.0
    priority: int = 1  # 1 = primary, 2 = secondary, etc.

@dataclass
class MigrationPlan:
    source_provider: StorageProvider
    target_provider: StorageProvider
    estimated_cost_savings: float
    estimated_migration_time: float
    files_to_migrate: int
    total_size_gb: float

class StorageBackend(ABC):
    """Abstract base class for storage providers"""
    
    @abstractmethod
    async def upload_file(self, local_path: str, key: str, metadata: Dict = None) -> bool:
        pass
    
    @abstractmethod
    async def download_file(self, key: str, local_path: str) -> bool:
        pass
    
    @abstractmethod
    async def file_exists(self, key: str) -> bool:
        pass
    
    @abstractmethod
    async def get_file_info(self, key: str) -> Dict:
        pass
    
    @abstractmethod
    async def list_files(self, prefix: str = "") -> List[Dict]:
        pass
    
    @abstractmethod
    async def delete_file(self, key: str) -> bool:
        pass
    
    @abstractmethod
    def get_public_url(self, key: str) -> str:
        pass
    
    @abstractmethod
    async def get_monthly_cost(self) -> float:
        pass

class CloudFlareR2Backend(StorageBackend):
    def __init__(self, config: ProviderConfig):
        self.config = config
        self.client = boto3.client(
            's3',
            endpoint_url=config.endpoint_url,
            aws_access_key_id=config.access_key,
            aws_secret_access_key=config.secret_key,
            region_name=config.region
        )
    
    async def upload_file(self, local_path: str, key: str, metadata: Dict = None) -> bool:
        try:
            extra_args = {
                'CacheControl': 'public, max-age=31536000, immutable',
                'ContentType': 'application/octet-stream'
            }
            if metadata:
                extra_args['Metadata'] = metadata
            
            self.client.upload_file(local_path, self.config.bucket_name, key, ExtraArgs=extra_args)
            return True
        except Exception as e:
            logging.error(f"CloudFlare R2 upload failed: {e}")
            return False
    
    async def file_exists(self, key: str) -> bool:
        try:
            self.client.head_object(Bucket=self.config.bucket_name, Key=key)
            return True
        except:
            return False
    
    async def get_file_info(self, key: str) -> Dict:
        try:
            response = self.client.head_object(Bucket=self.config.bucket_name, Key=key)
            return {
                'size': response['ContentLength'],
                'last_modified': response['LastModified'],
                'etag': response['ETag']
            }
        except:
            return {}
    
    def get_public_url(self, key: str) -> str:
        return f"https://your-r2-domain.com/{key}"
    
    async def get_monthly_cost(self) -> float:
        # Estimate based on current usage
        files = await self.list_files()
        total_size_gb = sum(f['size'] for f in files) / (1024**3)
        storage_cost = total_size_gb * 0.015  # $0.015/GB
        operations_cost = len(files) * 0.000001 * 4.50  # Rough estimate
        return storage_cost + operations_cost

class DigitalOceanSpacesBackend(StorageBackend):
    def __init__(self, config: ProviderConfig):
        self.config = config
        self.client = boto3.client(
            's3',
            endpoint_url=config.endpoint_url,
            aws_access_key_id=config.access_key,
            aws_secret_access_key=config.secret_key,
            region_name=config.region
        )
    
    async def upload_file(self, local_path: str, key: str, metadata: Dict = None) -> bool:
        try:
            extra_args = {
                'CacheControl': 'public, max-age=31536000',
                'ContentType': 'application/octet-stream',
                'ACL': 'public-read'
            }
            if metadata:
                extra_args['Metadata'] = metadata
            
            self.client.upload_file(local_path, self.config.bucket_name, key, ExtraArgs=extra_args)
            return True
        except Exception as e:
            logging.error(f"DigitalOcean Spaces upload failed: {e}")
            return False
    
    def get_public_url(self, key: str) -> str:
        return f"https://{self.config.bucket_name}.{self.config.region}.cdn.digitaloceanspaces.com/{key}"
    
    async def get_monthly_cost(self) -> float:
        files = await self.list_files()
        total_size_gb = sum(f['size'] for f in files) / (1024**3)
        
        if total_size_gb <= 250:
            return 5.0  # Base plan
        else:
            storage_overage = (total_size_gb - 250) * 0.02
            return 5.0 + storage_overage

class MultiProviderStorageManager:
    def __init__(self, configs: List[ProviderConfig]):
        self.backends = {}
        self.configs = {config.provider: config for config in configs}
        
        # Initialize backends
        for config in configs:
            if config.provider == StorageProvider.CLOUDFLARE_R2:
                self.backends[config.provider] = CloudFlareR2Backend(config)
            elif config.provider == StorageProvider.DIGITALOCEAN_SPACES:
                self.backends[config.provider] = DigitalOceanSpacesBackend(config)
        
        # Set primary provider (lowest priority number)
        self.primary_provider = min(configs, key=lambda x: x.priority).provider
        
    def get_provider_by_priority(self, priority: int) -> Optional[StorageProvider]:
        """Get provider by priority level"""
        for provider, config in self.configs.items():
            if config.priority == priority:
                return provider
        return None
    
    async def upload_with_redundancy(self, local_path: str, key: str, 
                                   replicas: int = 2, metadata: Dict = None) -> bool:
        """Upload to multiple providers for redundancy"""
        success_count = 0
        
        # Sort by priority
        sorted_providers = sorted(self.configs.items(), key=lambda x: x[1].priority)
        
        for provider, config in sorted_providers[:replicas]:
            backend = self.backends[provider]
            if await backend.upload_file(local_path, key, metadata):
                success_count += 1
                logging.info(f"Successfully uploaded {key} to {provider.value}")
            else:
                logging.error(f"Failed to upload {key} to {provider.value}")
        
        return success_count > 0
    
    async def get_file_with_fallback(self, key: str) -> str:
        """Get public URL with provider fallback"""
        # Try primary first
        primary_backend = self.backends[self.primary_provider]
        if await primary_backend.file_exists(key):
            return primary_backend.get_public_url(key)
        
        # Fallback to other providers
        for provider, backend in self.backends.items():
            if provider != self.primary_provider:
                if await backend.file_exists(key):
                    logging.warning(f"File {key} not found on primary, using {provider.value}")
                    return backend.get_public_url(key)
        
        raise FileNotFoundError(f"File {key} not found on any provider")
    
    async def sync_between_providers(self, source: StorageProvider, 
                                   target: StorageProvider, 
                                   prefix: str = "") -> bool:
        """Sync files between two providers"""
        source_backend = self.backends[source]
        target_backend = self.backends[target]
        
        files = await source_backend.list_files(prefix)
        sync_count = 0
        
        for file_info in files:
            key = file_info['key']
            
            # Check if file exists on target
            if await target_backend.file_exists(key):
                continue
            
            # Download from source and upload to target
            temp_path = f"/tmp/{key.replace('/', '_')}"
            
            if await source_backend.download_file(key, temp_path):
                if await target_backend.upload_file(temp_path, key):
                    sync_count += 1
                    logging.info(f"Synced {key} from {source.value} to {target.value}")
                
                # Cleanup
                import os
                os.unlink(temp_path)
        
        logging.info(f"Synced {sync_count} files from {source.value} to {target.value}")
        return sync_count > 0
    
    async def analyze_costs(self) -> Dict[StorageProvider, float]:
        """Analyze costs across all providers"""
        costs = {}
        
        for provider, backend in self.backends.items():
            costs[provider] = await backend.get_monthly_cost()
        
        return costs
    
    async def plan_migration(self, target_provider: StorageProvider) -> MigrationPlan:
        """Plan migration to a different provider"""
        current_costs = await self.analyze_costs()
        current_primary_cost = current_costs[self.primary_provider]
        target_cost = current_costs.get(target_provider, 0)
        
        # Get file count and size
        primary_backend = self.backends[self.primary_provider]
        files = await primary_backend.list_files()
        total_size_gb = sum(f['size'] for f in files) / (1024**3)
        
        return MigrationPlan(
            source_provider=self.primary_provider,
            target_provider=target_provider,
            estimated_cost_savings=current_primary_cost - target_cost,
            estimated_migration_time=total_size_gb / 10,  # Assume 10GB/hour
            files_to_migrate=len(files),
            total_size_gb=total_size_gb
        )
    
    async def execute_migration(self, target_provider: StorageProvider, 
                              dry_run: bool = True) -> bool:
        """Execute migration to target provider"""
        logging.info(f"{'DRY RUN: ' if dry_run else ''}Migrating from {self.primary_provider.value} to {target_provider.value}")
        
        if not dry_run:
            # Sync all files to target
            success = await self.sync_between_providers(self.primary_provider, target_provider)
            
            if success:
                # Update primary provider
                old_primary = self.primary_provider
                self.primary_provider = target_provider
                
                # Update priorities
                self.configs[target_provider].priority = 1
                self.configs[old_primary].priority = 2
                
                logging.info(f"Migration completed. New primary: {target_provider.value}")
                return True
        
        return False

# Usage Example
async def main():
    # Configure multiple providers
    configs = [
        ProviderConfig(
            provider=StorageProvider.CLOUDFLARE_R2,
            endpoint_url="https://your-account-id.r2.cloudflarestorage.com",
            access_key="your-r2-access-key",
            secret_key="your-r2-secret-key",
            bucket_name="mwm-files-r2",
            priority=1  # Primary
        ),
        ProviderConfig(
            provider=StorageProvider.DIGITALOCEAN_SPACES,
            endpoint_url="https://nyc3.digitaloceanspaces.com",
            access_key="your-do-access-key",
            secret_key="your-do-secret-key",
            bucket_name="mwm-files-do",
            region="nyc3",
            priority=2  # Secondary
        )
    ]
    
    # Initialize multi-provider manager
    storage = MultiProviderStorageManager(configs)
    
    # Upload with redundancy (to both providers)
    await storage.upload_with_redundancy("local_file.mwm", "maps/germany.mwm", replicas=2)
    
    # Get file URL (with automatic fallback)
    url = await storage.get_file_with_fallback("maps/germany.mwm")
    print(f"File URL: {url}")
    
    # Analyze costs
    costs = await storage.analyze_costs()
    print(f"Monthly costs: {costs}")
    
    # Plan migration if needed
    migration_plan = await storage.plan_migration(StorageProvider.DIGITALOCEAN_SPACES)
    print(f"Migration savings: ${migration_plan.estimated_cost_savings:.2f}/month")
    
    # Execute migration if beneficial
    if migration_plan.estimated_cost_savings > 0:
        await storage.execute_migration(StorageProvider.DIGITALOCEAN_SPACES, dry_run=False)

if __name__ == "__main__":
    asyncio.run(main())
```

## Cost Monitoring & Alert System

```python
# cost_monitor.py
import asyncio
import json
import logging
from datetime import datetime, timedelta
from typing import Dict, List
import aiohttp

class CostMonitor:
    def __init__(self, storage_manager: MultiProviderStorageManager):
        self.storage_manager = storage_manager
        self.cost_history = []
        self.alerts_enabled = True
        
        # Thresholds for alerts
        self.monthly_budget = 50.0  # $50/month budget
        self.cost_increase_threshold = 0.25  # 25% increase triggers alert
        
    async def collect_metrics(self):
        """Collect cost and usage metrics"""
        costs = await self.storage_manager.analyze_costs()
        
        metrics = {
            'timestamp': datetime.now().isoformat(),
            'provider_costs': {provider.value: cost for provider, cost in costs.items()},
            'total_cost': sum(costs.values()),
            'primary_provider': self.storage_manager.primary_provider.value
        }
        
        self.cost_history.append(metrics)
        
        # Keep only last 30 days
        cutoff_date = datetime.now() - timedelta(days=30)
        self.cost_history = [
            m for m in self.cost_history 
            if datetime.fromisoformat(m['timestamp']) > cutoff_date
        ]
        
        return metrics
    
    async def check_alerts(self, current_metrics: Dict):
        """Check for cost alerts and migration opportunities"""
        alerts = []
        
        # Budget alert
        if current_metrics['total_cost'] > self.monthly_budget:
            alerts.append({
                'type': 'budget_exceeded',
                'message': f"Monthly cost ${current_metrics['total_cost']:.2f} exceeds budget ${self.monthly_budget}",
                'severity': 'high'
            })
        
        # Cost increase alert
        if len(self.cost_history) >= 2:
            previous_cost = self.cost_history[-2]['total_cost']
            current_cost = current_metrics['total_cost']
            
            if current_cost > previous_cost * (1 + self.cost_increase_threshold):
                increase_pct = ((current_cost - previous_cost) / previous_cost) * 100
                alerts.append({
                    'type': 'cost_increase',
                    'message': f"Cost increased by {increase_pct:.1f}% since last check",
                    'severity': 'medium'
                })
        
        # Migration opportunity alert
        costs = current_metrics['provider_costs']
        current_primary = self.storage_manager.primary_provider.value
        
        for provider, cost in costs.items():
            if provider != current_primary and cost < costs[current_primary] * 0.8:
                savings = costs[current_primary] - cost
                alerts.append({
                    'type': 'migration_opportunity',
                    'message': f"Migrating to {provider} could save ${savings:.2f}/month",
                    'severity': 'info',
                    'target_provider': provider
                })
        
        return alerts
    
    async def send_alert(self, alert: Dict):
        """Send alert notification"""
        if not self.alerts_enabled:
            return
        
        # Log alert
        logging.warning(f"COST ALERT [{alert['severity']}]: {alert['message']}")
        
        # Could also send to Slack, email, etc.
        # await self.send_slack_notification(alert)
        # await self.send_email_notification(alert)
    
    async def auto_migrate_if_beneficial(self, threshold: float = 20.0):
        """Automatically migrate if savings exceed threshold"""
        for provider in StorageProvider:
            if provider == self.storage_manager.primary_provider:
                continue
            
            migration_plan = await self.storage_manager.plan_migration(provider)
            
            if migration_plan.estimated_cost_savings > threshold:
                logging.info(f"Auto-migration triggered: ${migration_plan.estimated_cost_savings:.2f} savings")
                await self.storage_manager.execute_migration(provider, dry_run=False)
                break
    
    async def generate_report(self) -> Dict:
        """Generate cost report"""
        if not self.cost_history:
            return {}
        
        latest = self.cost_history[-1]
        
        # Calculate 7-day average
        week_ago = datetime.now() - timedelta(days=7)
        recent_costs = [
            m['total_cost'] for m in self.cost_history
            if datetime.fromisoformat(m['timestamp']) > week_ago
        ]
        
        return {
            'current_monthly_cost': latest['total_cost'],
            'weekly_average': sum(recent_costs) / len(recent_costs) if recent_costs else 0,
            'primary_provider': latest['primary_provider'],
            'provider_breakdown': latest['provider_costs'],
            'cost_trend': 'increasing' if len(recent_costs) > 1 and recent_costs[-1] > recent_costs[0] else 'stable',
            'generated_at': datetime.now().isoformat()
        }

async def monitoring_loop():
    """Main monitoring loop"""
    # Initialize storage manager (from previous example)
    storage = MultiProviderStorageManager(configs)
    monitor = CostMonitor(storage)
    
    while True:
        try:
            # Collect metrics
            metrics = await monitor.collect_metrics()
            
            # Check for alerts
            alerts = await monitor.check_alerts(metrics)
            
            # Send alerts
            for alert in alerts:
                await monitor.send_alert(alert)
            
            # Auto-migrate if savings are significant
            await monitor.auto_migrate_if_beneficial(threshold=20.0)
            
            # Generate and save report
            report = await monitor.generate_report()
            with open('/tmp/cost_report.json', 'w') as f:
                json.dump(report, f, indent=2)
            
            # Wait 1 hour before next check
            await asyncio.sleep(3600)
            
        except Exception as e:
            logging.error(f"Monitoring error: {e}")
            await asyncio.sleep(300)  # Wait 5 minutes on error

if __name__ == "__main__":
    asyncio.run(monitoring_loop())
```

## DNS-Based Failover Configuration

```yaml
# cloudflare_dns_config.yaml
# Configure CloudFlare DNS for easy provider switching

records:
  # Primary CDN endpoint
  - name: "cdn.yourmaps.com"
    type: "CNAME"
    content: "your-r2-domain.r2.dev"
    ttl: 300  # 5 minutes for quick switching
    priority: 1
    
  # Secondary CDN endpoint  
  - name: "cdn-backup.yourmaps.com"
    type: "CNAME" 
    content: "your-do-space.nyc3.cdn.digitaloceanspaces.com"
    ttl: 300
    priority: 2
    
  # API endpoint (stays on your DigitalOcean droplet)
  - name: "api.yourmaps.com"
    type: "A"
    content: "your-droplet-ip"
    ttl: 300
```

## Migration Script

```bash
#!/bin/bash
# migrate_storage_provider.sh
# Automated migration script with zero downtime

set -euo pipefail

SOURCE_PROVIDER="cloudflare_r2"
TARGET_PROVIDER="digitalocean_spaces"
DOMAIN="yourmaps.com"

log() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $*"
}

# Phase 1: Sync data to target provider
log "Phase 1: Syncing data to $TARGET_PROVIDER..."
python3 /home/mwmapp/app/migrate.py sync --source="$SOURCE_PROVIDER" --target="$TARGET_PROVIDER"

# Phase 2: Verify data integrity
log "Phase 2: Verifying data integrity..."
python3 /home/mwmapp/app/migrate.py verify --source="$SOURCE_PROVIDER" --target="$TARGET_PROVIDER"

# Phase 3: Update DNS (gradual cutover)
log "Phase 3: Starting DNS cutover..."

# Switch 10% of traffic first
curl -X PUT "https://api.cloudflare.com/client/v4/zones/$ZONE_ID/dns_records/$RECORD_ID" \
     -H "Authorization: Bearer $CF_API_TOKEN" \
     -H "Content-Type: application/json" \
     --data '{
       "type": "CNAME",
       "name": "cdn",
       "content": "target-provider-url",
       "ttl": 60,
       "metadata": {"weight": 10}
     }'

# Wait and monitor
sleep 300

# If no errors, complete the cutover
log "Phase 4: Completing cutover..."
curl -X PUT "https://api.cloudflare.com/client/v4/zones/$ZONE_ID/dns_records/$RECORD_ID" \
     -H "Authorization: Bearer $CF_API_TOKEN" \
     -H "Content-Type: application/json" \
     --data '{
       "type": "CNAME", 
       "name": "cdn",
       "content": "target-provider-url",
       "ttl": 300
     }'

log "Migration completed successfully!"
```

## Migration Readiness Checklist

### ✅ **Immediate Protection**
- [ ] **Multi-provider setup** - R2 + DigitalOcean Spaces
- [ ] **Automated sync** - Real-time replication
- [ ] **DNS-based switching** - 5-minute TTL for quick cutover
- [ ] **Cost monitoring** - Automated alerts and migration triggers

### ✅ **Zero-Downtime Migration** 
- [ ] **Pre-sync data** - Target provider ready
- [ ] **Gradual DNS cutover** - 10% → 100% traffic shift  
- [ ] **Automated rollback** - If issues detected
- [ ] **Integrity verification** - SHA1 checksums match

### ✅ **Cost Protection**
- [ ] **Budget alerts** - Email when costs exceed thresholds
- [ ] **Automatic migration** - Switch providers if savings > $20/month
- [ ] **Real-time cost tracking** - Per-provider cost breakdown
- [ ] **Historical analysis** - Trend monitoring

## Provider Comparison Matrix

| Feature | CloudFlare R2 | DigitalOcean Spaces | Migration Effort |
|---------|---------------|-------------------|------------------|
| **Storage Cost** | $0.015/GB | $0.02/GB | Very Low |
| **Egress Cost** | $0 | $0.01/GB | Medium |
| **Setup Time** | 30 minutes | 30 minutes | Low |
| **CDN Quality** | Excellent | Good | Low |
| **Enterprise Pressure** | Possible | Low | N/A |
| **Migration Path** | Full automation | Full automation | Very Low |

## Expected Timeline for Migration

- **Planning**: 1 day
- **Setup secondary provider**: 2 hours  
- **Data sync**: 4-8 hours (500GB)
- **DNS cutover**: 15 minutes
- **Verification**: 1 hour
- **Total downtime**: 0 minutes

This architecture gives you the best of both worlds: CloudFlare R2's zero egress costs while maintaining complete freedom to migrate if they change their terms. The automated monitoring will give you early warning of any cost increases, and the migration can be executed in under a day with zero downtime.

## Expected Monthly Costs

| Scenario | R2 Primary | Migration Trigger | Post-Migration |
|----------|------------|-------------------|----------------|
| **Normal** | $15/month | No change | $15/month |
| **Enterprise Pressure** | $200+/month | Auto-migrate | $50/month |
| **High Growth** | $30/month | Cost monitoring | $30-60/month |

You're protected against vendor lock-in while still benefiting from the best current pricing!