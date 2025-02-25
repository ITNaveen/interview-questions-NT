# Kubernetes Backup and Recovery Interview Questions and Answers

## Question 1: Explain Kubernetes backup strategies for both etcd and application state. How would you design a comprehensive backup solution for a production cluster?

**Answer:**
A comprehensive Kubernetes backup strategy must address both the control plane state (primarily etcd) and application data. Here's how I would design a production-ready backup solution:

### Etcd Backup Strategy:

**Regular Snapshots:**
- Use built-in etcdctl for consistent snapshots:
```bash
ETCDCTL_API=3 etcdctl --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  snapshot save /backup/etcd-snapshot-$(date +%Y-%m-%d-%H-%M-%S).db
```
- Run snapshots at least every 30 minutes for critical clusters
- Verify snapshot integrity after creation

**Automated Rotation:**
- Keep hourly snapshots for 24 hours
- Keep daily snapshots for 30 days
- Keep weekly snapshots for 3 months
- Implement automated cleanup jobs

**Offsite Storage:**
- Store backups in different failure domains
- Encrypt backups at rest
- Implement immutable backups to protect against ransomware

### Application State Backup:

**Volume Snapshots Using CSI:**
```yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: postgres-snapshot-20230225
spec:
  volumeSnapshotClassName: csi-hostpath-snapclass
  source:
    persistentVolumeClaimName: postgres-pvc
```

**Application-Consistent Backups:**
- Use pre-snapshot hooks for database consistency:
```yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshotClass
metadata:
  name: csi-hostpath-snapclass
driver: hostpath.csi.k8s.io
parameters:
  # Pre-snapshot command to flush database writes
  presnapshotCommand: "/bin/sh -c '/scripts/db-freeze.sh'"
  # Post-snapshot command to unfreeze database
  postsnapshotCommand: "/bin/sh -c '/scripts/db-unfreeze.sh'"
```

**Application-Specific Backup Methods:**
- For databases: Use native backup tools (pg_dump, mysqldump)
- Schedule via CronJobs with appropriate service accounts
```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: postgres-backup
spec:
  schedule: "0 2 * * *"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: postgres-backup
            image: postgres:13
            command:
            - /bin/sh
            - -c
            - pg_dump -U postgres -d mydb | gzip > /backup/mydb-$(date +%Y-%m-%d).sql.gz
            volumeMounts:
            - name: backup-volume
              mountPath: /backup
          volumes:
          - name: backup-volume
            persistentVolumeClaim:
              claimName: backup-pvc
          restartPolicy: OnFailure
```

### Comprehensive Backup Architecture:

**Multi-Layer Approach:**
- Cluster state (etcd)
- Volume data (PV snapshots)
- Application data (application-specific backups)
- Kubernetes resource definitions (exported YAML)

**Backup Controller/Operator:**
- Deploy Velero or similar backup operator:
```bash
velero backup create full-cluster-backup \
  --include-namespaces=* \
  --include-resources=* \
  --include-cluster-resources=true \
  --snapshot-volumes=true
```

**Backup Validation:**
- Regular test restores to validate backup integrity
- Automated validation scripts
- Document recovery procedures with step-by-step guides

### Implementation Strategy:

**Backup Schedule:**
- etcd: Every 30 minutes
- PV snapshots: Daily and before major changes
- Full cluster backups: Weekly
- Config exports: After configuration changes

**Storage Considerations:**
- Use separate storage systems for backups
- Implement retention policies based on data criticality
- Calculate storage needs based on change rate and retention

**Monitoring and Alerting:**
- Monitor backup job success/failure
- Alert on failed backups
- Track backup size trends
- Verify backup storage consumption

**Documentation and Testing:**
- Document recovery procedures
- Perform quarterly recovery tests
- Train multiple team members on recovery procedures
- Include backup validation in CI/CD pipeline

## Question 2: What disaster recovery strategies would you implement for a mission-critical Kubernetes application? Discuss RTO, RPO, and how you'd design for both high availability and recoverability.

**Answer:**
For mission-critical Kubernetes applications, I would implement a comprehensive disaster recovery strategy balancing RTO (Recovery Time Objective) and RPO (Recovery Point Objective) requirements with costs and operational complexity.

### Key Disaster Recovery Concepts:

**RTO (Recovery Time Objective):**
- The maximum acceptable time to restore service after a disaster
- For mission-critical apps: typically minutes to hours
- Influences infrastructure investment (shorter RTO = higher cost)

**RPO (Recovery Point Objective):**
- The maximum acceptable data loss measured in time
- For mission-critical apps: typically seconds to minutes
- Influences backup frequency and replication strategy

### Multi-Tier Disaster Recovery Strategy:

**Tier 1: High Availability (HA) Within a Cluster**
- Deploy applications across multiple availability zones
- Implement Pod Disruption Budgets to maintain minimum replicas
- Use PodAntiAffinity to spread workloads across nodes
- Configure Horizontal Pod Autoscaling for load fluctuations
- Employ liveness and readiness probes for self-healing

**Tier 2: Regional Resilience**
- Deploy clusters across multiple regions
- Implement global load balancing with health checks
- Use storage replication (synchronous or asynchronous based on RPO)
- Set up data replication for stateful applications:
  ```yaml
  apiVersion: apps/v1
  kind: StatefulSet
  metadata:
    name: postgresql-ha
  spec:
    replicas: 3
    selector:
      matchLabels:
        app: postgresql
    template:
      spec:
        containers:
        - name: postgresql
          image: postgres:13
          env:
          - name: POSTGRES_PASSWORD
            valueFrom:
              secretKeyRef:
                name: postgres-secrets
                key: password
          - name: REPLICATION_MODE
            value: "synchronous"
  ```

**Tier 3: Full Disaster Recovery**
- Maintain active-active or active-passive cluster pairs
- Implement automated failover mechanisms
- Set up regular backups with Velero or similar tools
- Create GitOps pipelines for rapid infrastructure recreation
- Document DR playbooks and conduct regular drills

### Implementation Considerations:

**Network Design:**
- Global DNS with automated failover
- Low-latency dedicated connections between regions
- Network policy replication across environments

**Data Consistency:**
- For RPO near-zero: Synchronous data replication
- For RPO minutes: Asynchronous replication with write-ahead logs
- Cross-region volume snapshots scheduled based on RPO

**Operational Readiness:**
- Automated canary deployments to verify DR environments
- Regular DR tests with automated restoration workflows
- Chaos engineering to identify resilience gaps
- Comprehensive monitoring across all regions

**Recovery Automation:**
```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: dr-readiness-test
spec:
  schedule: "0 3 * * 0"  # Weekly on Sunday at 3am
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: dr-test
            image: our-dr-test-tools:v1.2
            command:
            - /bin/sh
            - -c
            - /scripts/test-dr-recovery.sh && /scripts/verify-application-health.sh
          restartPolicy: OnFailure
```

### Cost vs. Resilience Optimization:

**For RPO < 1 minute and RTO < 15 minutes:**
- Active-active multi-region setup
- Synchronous data replication
- Automated failover with DNS and load balancer reconfiguration
- Stateless components running in all regions
- Higher cost but maximum resilience

**For RPO < 1 hour and RTO < 4 hours:**
- Active-passive setup with regular data replication
- Warm standby environment with minimum resources
- Semi-automated recovery processes
- Balanced cost and resilience

**Business Continuity Integration:**
- Ensure DR strategy aligns with business priorities
- Document communication procedures during outages
- Regularly validate recovery procedures match current architecture
- Incorporate lessons learned from each DR test

## Question 3: How would you approach backing up and restoring stateful applications with strict data consistency requirements, such as databases running in Kubernetes?

**Answer:**
Backing up and restoring stateful applications with strict consistency requirements in Kubernetes requires a methodical approach that ensures data integrity throughout the process. Here's how I would approach it:

### 1. Understanding Application Consistency Requirements

**Types of Consistency:**
- **Crash-consistent backups:** Capture data as it exists at a single point in time (like taking a snapshot of a VM)
- **Application-consistent backups:** Application is aware of the backup process and prepares for it (flushing transactions, completing writes)
- **Transaction-consistent backups:** All in-flight transactions are either committed or rolled back

**For databases specifically:**
- Most databases require application-consistent backups to prevent data corruption
- Transaction logs must be properly managed to ensure point-in-time recovery

### 2. Pre-Backup Preparations

**Quiesce the Database:**
- Use database-specific commands to temporarily pause writes
- For PostgreSQL:
  ```sql
  SELECT pg_start_backup('backup_label', true);
  -- Backup operations here
  SELECT pg_stop_backup();
  ```
- For MySQL/MariaDB:
  ```sql
  FLUSH TABLES WITH READ LOCK;
  -- Backup operations here
  UNLOCK TABLES;
  ```

**Implement via Init Containers or Sidecar Pattern:**
```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql-database
spec:
  template:
    spec:
      containers:
      - name: mysql
        image: mysql:8.0
      # Backup sidecar
      - name: backup-agent
        image: backup-tooling:v1
        volumeMounts:
        - name: mysql-data
          mountPath: /var/lib/mysql
        - name: backup-scripts
          mountPath: /scripts
        command:
        - /bin/sh
        - -c
        - /scripts/backup-coordinator.sh
```

### 3. Backup Implementation

**Operator-Based Approach:**
- Use database-specific operators that understand consistency requirements
  - PostgreSQL Operator (e.g., Zalando, Crunchy Data)
  - MySQL Operator (e.g., Oracle, Percona)

**Example with Percona Operator:**
```yaml
apiVersion: pxc.percona.com/v1
kind: PerconaXtraDBClusterBackup
metadata:
  name: backup-mysql-consistent
spec:
  pxcCluster: my-cluster-name
  storageName: s3-us-west
  schedule:
    enabled: true
    schedule: "0 0 * * *"
    keep: 7
```

**Custom Backup Implementation:**
- Execute database native backup commands via Jobs:
```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: mongodb-backup
spec:
  template:
    spec:
      containers:
      - name: mongodb-backup
        image: mongodb:4.4
        command:
        - /bin/sh
        - -c
        - |
          mongodump --host=mongodb-svc --username=$MONGO_USER --password=$MONGO_PASSWORD \
          --authenticationDatabase=admin --db=mydb --out=/backup/mydb-$(date +%Y-%m-%d) && \
          tar -czf /backup-destination/mydb-$(date +%Y-%m-%d).tar.gz /backup/mydb-$(date +%Y-%m-%d)
        env:
        - name: MONGO_USER
          valueFrom:
            secretKeyRef:
              name: mongodb-secrets
              key: username
        - name: MONGO_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mongodb-secrets
              key: password
        volumeMounts:
        - name: backup-storage
          mountPath: /backup-destination
      volumes:
      - name: backup-storage
        persistentVolumeClaim:
          claimName: backup-pvc
      restartPolicy: OnFailure
```

### 4. Consistency Verification

**Post-Backup Validation:**
- Verify backup integrity with database-specific tools
- For MySQL:
  ```bash
  mysqlcheck --all-databases --check-only --quick
  ```
- For PostgreSQL:
  ```bash
  pg_verifybackup /path/to/backup
  ```

**Automation via Post-Backup Hooks:**
```yaml
apiVersion: velero.io/v1
kind: Backup
metadata:
  name: postgres-backup
  annotations:
    post.hook.backup.velero.io/container: postgres
    post.hook.backup.velero.io/command: '["/bin/sh", "-c", "/scripts/verify-backup.sh"]'
spec:
  includedNamespaces:
  - database
  includedResources:
  - persistentvolumeclaims
  - persistentvolumes
  storageLocation: default
```

### 5. Advanced Consistency Techniques

**Transaction Log Management:**
- Implement Write-Ahead Log (WAL) archiving for PostgreSQL
- Binary log management for MySQL
- Oplog tailing for MongoDB

**Example WAL Archiving for PostgreSQL:**
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: postgres-config
data:
  postgresql.conf: |
    wal_level = replica
    archive_mode = on
    archive_command = 'test ! -f /mnt/archive/%f && cp %p /mnt/archive/%f'
    archive_timeout = 60
```

**Point-in-Time Recovery Configuration:**
- Ensure WAL segments are backed up regularly
- Use a separate persistent volume for WAL archives
- Implement automated log shipping to backup storage

### 6. Restore Strategy

**Pre-Restore Verification:**
- Verify backup integrity before restoration
- Check for available disk space
- Validate backup metadata

**Controlled Restore Process:**
- Stop the application pods to prevent concurrent writes
- Restore data using database-native tools
- Apply transaction logs to reach desired recovery point
- Validate restored data before resuming traffic

**Example PostgreSQL Restore Job:**
```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: postgres-restore
spec:
  template:
    spec:
      containers:
      - name: postgres-restore
        image: postgres:13
        command:
        - /bin/sh
        - -c
        - |
          pg_restore -U postgres -d mydatabase /backups/mydatabase-backup.dump && \
          # Apply WAL files to reach point-in-time
          pg_waldump /wal_archive/000000010000000000000001 | \
          psql -U postgres -d mydatabase
        volumeMounts:
        - name: restore-volume
          mountPath: /backups
        - name: wal-archive
          mountPath: /wal_archive
      restartPolicy: OnFailure
```

### 7. Testing and Verification

**Regular Restore Testing:**
- Schedule automated restore tests to isolated environments
- Validate data consistency after restore
- Measure actual RTOs and RPOs achieved

**Data Integrity Verification:**
- Run schema validation
- Check record counts
- Perform application-level validation tests
- Run database consistency checkers

By combining these approaches, I can ensure that stateful applications with strict consistency requirements are properly backed up and can be reliably restored when needed, meeting both data integrity and recovery time objectives.
