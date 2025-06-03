# User Story 03.4: Backup and Recovery

## Story
As a **system administrator**, I want to **implement automated backup and recovery procedures** so that **certificate data can be restored in case of disasters**.

## Acceptance Criteria
```gherkin
Feature: Backup and Recovery Procedures
  As a system administrator
  I want automated backup and recovery capabilities
  So that I can restore data after failures

  Scenario: Automated daily backups
    Given I have a running Cassandra cluster
    When the scheduled backup time arrives
    Then a snapshot should be created automatically
    And the snapshot should be uploaded to object storage
    And old backups should be pruned according to retention policy
    And backup completion should be verified

  Scenario: Point-in-time recovery
    Given I have regular backups and commit logs
    When I need to restore to a specific timestamp
    Then I should be able to restore the nearest snapshot
    And replay commit logs to the exact time
    And verify data integrity after restore
    And the recovery time should be under 30 minutes

  Scenario: Disaster recovery
    Given a catastrophic cluster failure
    When I provision a new cluster
    Then I should be able to restore from remote backups
    And the cluster should be operational within RTO
    And no data loss should occur beyond RPO
    And all certificates should be accessible

  Scenario: Selective restoration
    Given I need to restore specific data
    When I restore individual tables or keyspaces
    Then only the selected data should be restored
    And existing data should not be affected
    And the restore should complete quickly
```

## Technical Details

### Backup Strategy Implementation
```python
import os
import subprocess
import boto3
from datetime import datetime, timedelta
from cassandra.cluster import Cluster
import hashlib
import json
import logging

class CassandraBackupManager:
    def __init__(self, config):
        self.config = config
        self.cluster = Cluster(config['contact_points'])
        self.session = self.cluster.connect()
        self.s3_client = boto3.client('s3',
            aws_access_key_id=config['aws_access_key_id'],
            aws_secret_access_key=config['aws_secret_access_key']
        )
        self.logger = logging.getLogger(__name__)
    
    def create_snapshot(self, keyspaces=None, tag=None):
        """Create consistent snapshot across all nodes"""
        if not tag:
            tag = f"backup_{datetime.utcnow().strftime('%Y%m%d_%H%M%S')}"
        
        # Get all nodes in cluster
        nodes = self._get_cluster_nodes()
        
        # Create snapshot on each node
        snapshot_results = []
        for node in nodes:
            result = self._create_node_snapshot(node, keyspaces, tag)
            snapshot_results.append(result)
        
        # Verify snapshots
        if all(r['success'] for r in snapshot_results):
            self.logger.info(f"Snapshot {tag} created successfully on all nodes")
            return {
                'tag': tag,
                'timestamp': datetime.utcnow(),
                'nodes': snapshot_results,
                'keyspaces': keyspaces or 'all'
            }
        else:
            raise Exception(f"Snapshot {tag} failed on some nodes")
    
    def _create_node_snapshot(self, node, keyspaces, tag):
        """Create snapshot on individual node"""
        try:
            # Use nodetool to create snapshot
            cmd = ['nodetool', '-h', node['address'], 'snapshot', '-t', tag]
            if keyspaces:
                cmd.extend(keyspaces)
            
            result = subprocess.run(cmd, capture_output=True, text=True)
            
            if result.returncode == 0:
                return {
                    'node': node['address'],
                    'success': True,
                    'tag': tag,
                    'message': 'Snapshot created successfully'
                }
            else:
                return {
                    'node': node['address'],
                    'success': False,
                    'error': result.stderr
                }
        except Exception as e:
            return {
                'node': node['address'],
                'success': False,
                'error': str(e)
            }
    
    def backup_to_s3(self, snapshot_tag):
        """Upload snapshot to S3 with metadata"""
        backup_manifest = {
            'snapshot_tag': snapshot_tag,
            'timestamp': datetime.utcnow().isoformat(),
            'cluster_name': self.config['cluster_name'],
            'keyspaces': {},
            'files': []
        }
        
        # Get snapshot directories from all nodes
        for node in self._get_cluster_nodes():
            node_snapshots = self._get_node_snapshots(node, snapshot_tag)
            
            for snapshot in node_snapshots:
                # Upload each SSTable file
                for sstable_file in snapshot['files']:
                    s3_key = self._generate_s3_key(
                        snapshot_tag,
                        node['address'],
                        snapshot['keyspace'],
                        snapshot['table'],
                        sstable_file
                    )
                    
                    # Upload with checksum
                    checksum = self._upload_file_to_s3(
                        sstable_file['path'],
                        self.config['backup_bucket'],
                        s3_key
                    )
                    
                    backup_manifest['files'].append({
                        'node': node['address'],
                        'keyspace': snapshot['keyspace'],
                        'table': snapshot['table'],
                        'file': sstable_file['name'],
                        's3_key': s3_key,
                        'size': sstable_file['size'],
                        'checksum': checksum
                    })
        
        # Upload manifest
        manifest_key = f"backups/{snapshot_tag}/manifest.json"
        self.s3_client.put_object(
            Bucket=self.config['backup_bucket'],
            Key=manifest_key,
            Body=json.dumps(backup_manifest, indent=2),
            ContentType='application/json'
        )
        
        self.logger.info(f"Backup {snapshot_tag} uploaded to S3")
        return backup_manifest
    
    def restore_from_backup(self, snapshot_tag, target_time=None):
        """Restore cluster from backup"""
        # Download manifest
        manifest = self._download_manifest(snapshot_tag)
        
        # Verify target cluster is empty or confirm overwrite
        if not self._verify_restore_target():
            raise Exception("Target cluster not ready for restore")
        
        # Stop Cassandra on all nodes
        self._stop_cluster()
        
        try:
            # Clear existing data
            self._clear_data_directories()
            
            # Download and restore SSTable files
            for file_info in manifest['files']:
                self._restore_sstable_file(file_info)
            
            # Start Cassandra
            self._start_cluster()
            
            # If point-in-time recovery requested
            if target_time:
                self._replay_commit_logs(manifest['timestamp'], target_time)
            
            # Verify restoration
            self._verify_restoration(manifest)
            
            self.logger.info(f"Restore from {snapshot_tag} completed successfully")
            
        except Exception as e:
            self.logger.error(f"Restore failed: {str(e)}")
            # Attempt to restart cluster in original state
            self._start_cluster()
            raise
    
    def _replay_commit_logs(self, backup_time, target_time):
        """Replay commit logs for point-in-time recovery"""
        # Download commit logs from archive
        commit_logs = self._download_commit_logs(backup_time, target_time)
        
        for log_file in commit_logs:
            # Use commitlog replay tool
            cmd = [
                'cassandra-commitlog-replay',
                '--keyspace', 'certificate_manager',
                '--target-time', target_time.isoformat(),
                log_file
            ]
            subprocess.run(cmd, check=True)
    
    def setup_continuous_archiving(self):
        """Configure commit log archiving for PITR"""
        archive_script = """#!/bin/bash
        # Commit log archive script
        
        ARCHIVE_DIR="/var/lib/cassandra/archive"
        S3_BUCKET="{bucket}"
        
        # Create archive directory
        mkdir -p $ARCHIVE_DIR
        
        # Copy commit log to archive
        cp "$1" "$ARCHIVE_DIR/"
        
        # Upload to S3
        aws s3 cp "$1" "s3://$S3_BUCKET/commitlogs/$(hostname)/$(basename $1)"
        
        # Compress local archive
        gzip "$ARCHIVE_DIR/$(basename $1)"
        """.format(bucket=self.config['backup_bucket'])
        
        # Write archive script
        script_path = '/etc/cassandra/commitlog_archiving.sh'
        with open(script_path, 'w') as f:
            f.write(archive_script)
        os.chmod(script_path, 0o755)
        
        # Update cassandra configuration
        self._update_cassandra_config({
            'commitlog_archiving': {
                'archive_command': f'{script_path} %path',
                'restore_command': 'cp %from %to',
                'restore_directories': '/var/lib/cassandra/restore'
            }
        })
```

### Backup Scheduling
```python
from apscheduler.schedulers.background import BackgroundScheduler
from apscheduler.triggers.cron import CronTrigger

class BackupScheduler:
    def __init__(self, backup_manager):
        self.backup_manager = backup_manager
        self.scheduler = BackgroundScheduler()
        self.scheduler.start()
    
    def schedule_daily_backup(self, hour=2, minute=0):
        """Schedule daily backup at specified time"""
        self.scheduler.add_job(
            func=self._perform_daily_backup,
            trigger=CronTrigger(hour=hour, minute=minute),
            id='daily_backup',
            name='Daily Cassandra Backup',
            replace_existing=True
        )
    
    def schedule_incremental_backup(self, interval_hours=6):
        """Schedule incremental commit log backups"""
        self.scheduler.add_job(
            func=self._perform_incremental_backup,
            trigger='interval',
            hours=interval_hours,
            id='incremental_backup',
            name='Incremental Commit Log Backup'
        )
    
    def _perform_daily_backup(self):
        """Execute daily backup routine"""
        try:
            # Create snapshot
            snapshot = self.backup_manager.create_snapshot()
            
            # Upload to S3
            self.backup_manager.backup_to_s3(snapshot['tag'])
            
            # Clean up old local snapshots
            self._cleanup_old_snapshots(days_to_keep=7)
            
            # Prune old S3 backups
            self._prune_s3_backups(retention_days=30)
            
            # Send success notification
            self._notify_backup_success(snapshot)
            
        except Exception as e:
            self._notify_backup_failure(str(e))
            raise
```

### Disaster Recovery Procedures
```yaml
# disaster-recovery-playbook.yml
---
- name: Cassandra Disaster Recovery
  hosts: cassandra_nodes
  vars:
    backup_tag: "{{ restore_point }}"
    new_cluster: "{{ target_cluster | default(false) }}"
  
  tasks:
    - name: Verify backup availability
      aws_s3:
        bucket: "{{ backup_bucket }}"
        prefix: "backups/{{ backup_tag }}"
        mode: list
      register: backup_files
      run_once: true
    
    - name: Stop Cassandra service
      systemd:
        name: cassandra
        state: stopped
      when: not new_cluster
    
    - name: Clear data directories
      file:
        path: "{{ item }}"
        state: absent
      loop:
        - /var/lib/cassandra/data
        - /var/lib/cassandra/commitlog
        - /var/lib/cassandra/saved_caches
    
    - name: Create data directories
      file:
        path: "{{ item }}"
        state: directory
        owner: cassandra
        group: cassandra
        mode: '0750'
      loop:
        - /var/lib/cassandra/data
        - /var/lib/cassandra/commitlog
        - /var/lib/cassandra/saved_caches
    
    - name: Download backup files
      aws_s3:
        bucket: "{{ backup_bucket }}"
        object: "{{ item.key }}"
        dest: "/tmp/restore/{{ item.key | basename }}"
        mode: get
      loop: "{{ backup_files.s3_keys }}"
      when: item.key is match(".*\\.db$")
    
    - name: Restore SSTable files
      unarchive:
        src: "/tmp/restore/{{ item }}"
        dest: /var/lib/cassandra/data/
        owner: cassandra
        group: cassandra
      loop: "{{ restored_files }}"
    
    - name: Start Cassandra service
      systemd:
        name: cassandra
        state: started
        enabled: yes
    
    - name: Wait for Cassandra to be ready
      wait_for:
        port: 9042
        host: "{{ inventory_hostname }}"
        delay: 30
        timeout: 300
    
    - name: Verify cluster status
      command: nodetool status
      register: cluster_status
      until: "'UN' in cluster_status.stdout"
      retries: 10
      delay: 30
    
    - name: Run data verification
      command: |
        cqlsh -e "SELECT COUNT(*) FROM certificate_manager.certificates"
      register: data_check
    
    - name: Report recovery status
      debug:
        msg: "Recovery completed. Certificate count: {{ data_check.stdout }}"
```

### Backup Monitoring and Alerting
```python
class BackupMonitor:
    def __init__(self, metrics_client, alert_client):
        self.metrics = metrics_client
        self.alerts = alert_client
    
    def check_backup_health(self):
        """Monitor backup health metrics"""
        metrics = {
            'last_successful_backup': self._get_last_backup_time(),
            'backup_size_gb': self._get_backup_size(),
            'backup_duration_minutes': self._get_last_backup_duration(),
            'failed_backups_24h': self._count_failed_backups(hours=24),
            'recovery_test_status': self._get_last_recovery_test()
        }
        
        # Check for issues
        issues = []
        
        # Alert if no backup in 25 hours
        if metrics['last_successful_backup'] < datetime.utcnow() - timedelta(hours=25):
            issues.append({
                'severity': 'critical',
                'message': 'No successful backup in over 24 hours'
            })
        
        # Alert if backup size anomaly
        avg_size = self._get_average_backup_size()
        if abs(metrics['backup_size_gb'] - avg_size) / avg_size > 0.5:
            issues.append({
                'severity': 'warning',
                'message': f'Backup size anomaly detected: {metrics["backup_size_gb"]}GB'
            })
        
        # Send alerts
        for issue in issues:
            self.alerts.send_alert(
                severity=issue['severity'],
                service='cassandra-backup',
                message=issue['message'],
                metrics=metrics
            )
        
        # Record metrics
        for key, value in metrics.items():
            self.metrics.gauge(f'cassandra.backup.{key}', value)
        
        return len(issues) == 0, metrics
```

## Testing Requirements
1. **Backup Tests**
   - Automated backup execution
   - S3 upload verification
   - Backup integrity checks
   - Retention policy enforcement

2. **Recovery Tests**
   - Full cluster restoration
   - Point-in-time recovery
   - Single table restoration
   - Cross-region recovery

3. **Performance Tests**
   - Backup impact on cluster performance
   - Recovery time objectives (RTO)
   - Recovery point objectives (RPO)
   - Network bandwidth usage

## Definition of Done
- [ ] Automated daily backups configured
- [ ] S3 backup storage implemented
- [ ] Point-in-time recovery tested
- [ ] Disaster recovery playbook created
- [ ] Backup monitoring alerts configured
- [ ] Recovery procedures documented
- [ ] RTO/RPO targets validated
- [ ] Backup encryption implemented