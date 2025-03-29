## Backup and Recovery Questions

# etcd - 
In a Kubernetes cluster (like in Amazon EKS), ensuring your data and configuration are safely backed up is critical for disaster recovery and ensuring uptime in case of failures. Here’s a simplified guide to understanding and implementing etcd and application state backups in AWS.

1. Why Back Up etcd and Application State?
etcd (Control Plane State):
What is it?:
etcd is a distributed key-value store that holds all the configuration data and the state of your Kubernetes cluster (e.g., deployments, services, secrets, config maps, etc.).
Why is it important?:
If etcd fails or gets corrupted, Kubernetes won’t know the state of the cluster, and this could lead to complete loss of your cluster's configuration.
Backing up etcd ensures you can restore the configuration of your Kubernetes cluster to a working state if something goes wrong.

Application State (Workload Data):
What is it?:
Application state includes Persistent Volumes (PVs), which hold data for running applications (such as databases like PostgreSQL or MySQL).
Backing up application data ensures that even if your cluster is recreated, your critical app data (e.g., database records) is safe.

- etcd stores infrastructure (infra) state → Things like your Pods, Services, ConfigMaps, Persistent Volume (PV) definitions, etc. (basically, all Kubernetes objects you define in YAML).
- Application state stores actual app data → Things like logs, database records, files inside PVs, etc.

✅ etcd Backup → Saves cluster config, but not app data.
✅ Persistent Volumes (PV) Backup → Storage snapshots or Velero.
✅ Database Backup → Use database tools like mysqldump, pg_dump.
✅ Logs Backup → Save ELK/Loki data or copy logs from nodes.

2. What is the etcdctl Command and Why Use It?
The etcdctl command is used to interact with the etcd database. Here's the backup command:

ETCDCTL_API=3 etcdctl snapshot save /backup/etcd-snapshot-$(date +%Y-%m-%d-%H-%M-%S).db

- ETCDCTL_API=3: This specifies the version of the etcdctl command (version 3 is the latest).
- etcdctl snapshot save: This command saves a snapshot (a backup) of the current state of etcd.
- /backup/etcd-snapshot-$(date +%Y-%m-%d-%H-%M-%S).db: This specifies the location where the snapshot file will be saved. The file path (/backup/) and file name include the current date and time to ensure the backup is unique.

Where is the /backup/ directory?
The /backup/ directory is a local folder in the system where the backup file will be stored.
In practice, you need to make sure this folder exists on the machine where you’re running the command. You can set it to any directory you want, for example: /home/username/etcd-backups/ or /mnt/backups/.

# 3.How to Automate the Backup Process with CronJob
We want to back up the etcd snapshot regularly and upload it to AWS S3 for offsite storage. This can be done by using a CronJob to schedule the backup process.

Steps to Set Up CronJob:
Create a CronJob to back up etcd and upload to S3:

A CronJob in Kubernetes allows you to run tasks on a scheduled basis (like a cron job in Linux).
Here’s an example CronJob YAML to automate the backup:

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: etcd-backup
spec:
  schedule: "*/30 * * * *"  # Every 30 minutes
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: etcd-backup
            image: busybox  # Lightweight image for simple commands
            command:
            - /bin/sh
            - -c
            - |
              # Take etcd snapshot
              ETCDCTL_API=3 etcdctl snapshot save /backup/etcd-snapshot-$(date +%Y-%m-%d-%H-%M-%S).db
              # Upload to S3
              aws s3 cp /backup/etcd-snapshot-*.db s3://my-k8s-backups/
          restartPolicy: OnFailure
```
Explanation:

schedule: "*/30 * * * *" means the backup will run every 30 minutes.
command: Inside the command section, we first run the etcdctl backup command and then upload it to S3.
aws s3 cp: The backup file is uploaded to an S3 bucket (e.g., my-k8s-backups).
Make Sure the AWS CLI is Configured: The container running this CronJob needs to have AWS CLI configured to access the S3 bucket. You can do this by either:

Mounting an IAM role for the pod with the necessary permissions.
Passing AWS credentials into the container.

# etcd on prem - 
```yml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: etcd-backup-pv
spec:
  capacity:
    storage: 100Gi  # Adjust as needed
  accessModes:
    - ReadWriteMany  # Allow multiple nodes to write
  nfs:
    path: /mnt/nfs-etcd-backups  # Change based on your setup
    server: <NFS_SERVER_IP>  # Replace with actual NFS server IP
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: etcd-backup-pvc
  namespace: kube-system
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 50Gi  # Adjust size as needed
  volumeName: etcd-backup-pv

# Then
apiVersion: batch/v1
kind: CronJob
metadata:
  name: etcd-backup
  namespace: kube-system  # Typically, etcd runs in kube-system
spec:
  schedule: "*/30 * * * *"  # Runs every 30 minutes
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: etcd-backup
            image: bitnami/etcd:latest  # Use a proper etcd image with etcdctl
            env:
            - name: ETCDCTL_API
              value: "3"
            - name: ETCDCTL_ENDPOINTS
              value: "https://127.0.0.1:2379"  # Change based on etcd setup
            - name: ETCDCTL_CACERT
              value: "/etc/etcd/cert/ca.crt"
            - name: ETCDCTL_CERT
              value: "/etc/etcd/cert/etcd-client.crt"
            - name: ETCDCTL_KEY
              value: "/etc/etcd/cert/etcd-client.key"
            command:
            - /bin/sh
            - -c
            - |
              # Delete backups older than 5 days
              find /backup -type f -name "etcd-snapshot-*.db" -mtime +5 -delete

              # Take etcd snapshot
              etcdctl snapshot save /backup/etcd-snapshot-$(date +%Y-%m-%d-%H-%M-%S).db

              # Optional: Upload to S3 (only if AWS CLI is installed and configured)
              # aws s3 cp /backup/etcd-snapshot-*.db s3://my-k8s-backups/
            volumeMounts:
            - name: etcd-backup-storage
              mountPath: /backup
          volumes:
          - name: etcd-backup-storage
            persistentVolumeClaim:
              claimName: etcd-backup-pvc  # Uses the NFS-backed PVC
          restartPolicy: OnFailure

```

# 4. PVC backup = 
```yml
# first create NFS based pvc for 2 nodes - 
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nfs-backup-pv
spec:
  capacity:
    storage: 200Gi
  accessModes:
    - ReadWriteMany
  nfs:
    path: /mnt/nfs-backups
    server: <NFS_SERVER_IP>
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: nfs-backup-pvc
  namespace: backup
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 100Gi
  volumeName: nfs-backup-pv


# Create a Kubernetes CronJob that will run our backup process on a schedule
apiVersion: batch/v1  # Specifies the API version for CronJob resources.
kind: CronJob         # Declares that we are creating a Kubernetes CronJob.
metadata:
  name: nfs-pv-backup  # Name of the CronJob, used for identification.
  namespace: backup    # Namespace where the CronJob will be deployed.
spec:
  schedule: "0 1 * * *"  # Specifies the schedule in cron format (runs daily at 1:00 AM).
  jobTemplate:  # Defines the job template that will be created for each execution.
    spec:
      template:  # Specifies the Pod template.
        spec:
          containers:  # Defines the containers that will run inside the Pod.
          - name: backup  # Name of the container.
            image: busybox  # Uses the lightweight BusyBox image for basic commands.
            command:  # Defines the command to run inside the container.
            - /bin/sh  # Uses the shell.
            - -c  # Executes the command as a string.
            - |
              # Delete backups older than 5 days
              find /backup -type f -name "data-*.tar.gz" -mtime +5 -delete

              # Creates a compressed tar archive of all data from the source PVC.
              # The filename includes the current date for versioning.
              tar -czf /backup/data-$(date +%Y%m%d).tar.gz -C /data .
            volumeMounts:  # Mounts volumes inside the container.
            - name: data-to-backup  # First volume (source PVC containing data to back up).
              mountPath: /data  # Mounts the PVC at /data inside the container.
              readOnly: true  # Ensures data is not modified (safe backup).
            - name: nfs-backup-storage  # Second volume (NFS PVC for storing backups).
              mountPath: /backup  # Mounts the backup storage at /backup inside the container.
          volumes:  # Defines the volumes available to the container.
          - name: data-to-backup  # Defines the first volume (source PVC).
            persistentVolumeClaim:
              claimName: app-data-pvc  # Uses the application's PVC as the source.
          - name: nfs-backup-storage  # Defines the second volume (NFS storage PVC).
            persistentVolumeClaim:
              claimName: nfs-backup-pvc  # Uses the NFS-backed PVC for backup storage.
          restartPolicy: OnFailure  # Restarts the job if it fails.

```
Process Explanation
1. Source PVC (app-data-pvc):
Named "data-to-backup" in the YAML configuration
Mounted at path /data inside the container
This contains your actual application data

2. Destination PVC (backup-storage-pvc):
Named "backup-storage" in the YAML configuration
Mounted at path /backup inside the container
This is where backups will be stored

Backup process:
The container reads all files from /data
Creates a compressed tar file at /backup/data-20250329.tar.gz
This tar file contains a complete copy of your source PVC

The CronJob runs at 1 AM every day
It spins up a pod with a busybox container
The pod mounts your application's PVC (app-data-pvc) as read-only
It also mounts a backup storage PVC (backup-storage-pvc) as read-write
The container creates a compressed tar archive of all data from your application's PVC
The backup is named with the current date (e.g., data-20250329.tar.gz)
When finished, the pod terminates until the next scheduled run


Zero downtime achieved by:

Mounting the source PVC as read-only, which doesn't interrupt applications using it
Using lightweight tools (tar) that have minimal impact on performance

Before implementation:

Create a dedicated backup-storage-pvc with sufficient space
Replace app-data-pvc with the actual name of the PVC you want to backup
Consider multiple CronJobs if you have multiple PVCs to backup
This approach directly backs up PVCs (which are what your applications are actually using) rather than the underlying PVs directly.

3. Create ServiceAccount with Permissions
The ServiceAccount and ClusterRole are necessary for granting permissions to create and manage snapshots. 
```yml
yamlCopyapiVersion: v1
kind: ServiceAccount
metadata:
  name: snapshot-sa
  namespace: default

---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: snapshot-creator
rules:
- apiGroups: ["snapshot.storage.k8s.io"]
  resources: ["volumesnapshots"]
  verbs: ["create", "delete", "list"]
- apiGroups: [""]
  resources: ["persistentvolumeclaims"]
  verbs: ["get", "list"]

---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: snapshot-sa-binding
subjects:
- kind: ServiceAccount
  name: snapshot-sa
  namespace: default
roleRef:
  kind: ClusterRole
  name: snapshot-creator
  apiGroup: rbac.authorization.k8s.io
```

3. Create CronJob for Snapshots

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: pv-snapshot-backup
spec:
  schedule: "0 2 * * *"  # Daily at 2 AM
  jobTemplate:
    spec:
      template:
        spec:
          serviceAccountName: snapshot-sa
          containers:
          - name: snapshot-creator
            image: bitnami/kubectl:latest
            command:
            - /bin/sh
            - -c
            - |
              kubectl create -f - <<EOF
              apiVersion: snapshot.storage.k8s.io/v1
              kind: VolumeSnapshot
              metadata:
                name: snapshot-pvc1-$(date +%Y%m%d-%H%M%S)
                namespace: default
              spec:
                volumeSnapshotClassName: snapshot-class
                source:
                  persistentVolumeClaimName: pvc1
              ---
              apiVersion: snapshot.storage.k8s.io/v1
              kind: VolumeSnapshot
              metadata:
                name: snapshot-pvc2-$(date +%Y%m%d-%H%M%S)
                namespace: default
              spec:
                volumeSnapshotClassName: snapshot-class
                source:
                  persistentVolumeClaimName: pvc2
              EOF
              
              # Cleanup: Keep only last 7 snapshots
              kubectl get volumesnapshots -o name | sort -r | tail -n +8 | xargs kubectl delete
          restartPolicy: OnFailure
Deployment Steps:

Save each YAML in a separate file
Apply in order:




# DR - 
Here's a step-by-step overview of how the traffic flow would work in your setup:​

User Access: A user accesses your website by entering your domain name in their browser.​

DNS Resolution: The browser queries your DNS server (e.g., GoDaddy's DNS service) to resolve your domain name to an IP address.​

Traffic Routing to HAProxy: The DNS server returns the IP address of your HAProxy load balancer. The user's browser then sends the HTTP request to this IP address.​

HAProxy Load Balancing:

Health Checks: HAProxy is configured to perform regular health checks on the nodes of both Cluster A and Cluster B.​

Traffic Distribution: Based on its configuration and the health status of the nodes, HAProxy forwards the incoming traffic to the appropriate cluster.​

Cluster Response: The selected cluster processes the request and sends the response back through HAProxy to the user's browser.​

By configuring HAProxy to monitor the health of both clusters, you ensure that traffic is only directed to healthy nodes. In the event that a node or even an entire cluster becomes unavailable, HAProxy can reroute traffic to the healthy cluster, maintaining the availability and reliability of your website.​

This setup effectively balances the load between your clusters and provides a failover mechanism to handle potential outages, ensuring a seamless user experience.