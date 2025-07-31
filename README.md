# OpenShift Application Data Protection (OADP) Guide

This guide demonstrates how to set up OADP for backing up and restoring applications in OpenShift Container Platform (OCP).

## Prerequisites

- OpenShift Container Platform cluster
- Object storage (MinIO) cluster
- NFS storage class configured
- Cluster-admin privileges

## Setup Steps

### 1. Install OADP Operator

Install the OADP operator from OperatorHub in your OpenShift cluster.

### 2. Configure Object Storage Access

Create a secret for object storage credentials:

```yaml
kubectl create secret generic cloud-credentials \
  --namespace openshift-adp \
  --from-literal=cloud="""[default]
aws_access_key_id=<YOUR_ACCESS_KEY>
aws_secret_access_key=<YOUR_SECRET_KEY>
``` 

### 3. Create Data Protection Application (DPA)

Save the following as `dpa.yaml` and apply with `oc apply -f dpa.yaml`:

```yaml
apiVersion: oadp.openshift.io/v1alpha1
kind: DataProtectionApplication
metadata:
  name: oadp-dpa
  namespace: openshift-adp
spec:
  configuration:
    nodeAgent:
      enable: true
      uploaderType: restic
    velero:
      defaultPlugins:
        - openshift
        - aws
  backupLocations:
    - velero:
        config:
          region: us-east-1
          s3ForcePathStyle: "true"
          s3Url: <YOUR_S3_ENDPOINT_URL>
        credential:
          key: cloud
          name: cloud-credentials
        default: true
        objectStorage:
          bucket: oadp-bucket-0
          prefix: gitlab-backups
        provider: aws
```

### 4. Create a Backup
Save the following as `backup.yaml` and apply with `oc apply -f backup.yaml`:
```yaml
apiVersion: velero.io/v1
kind: Backup
metadata:
  name: gitlab-system-backup-restic
  namespace: openshift-adp
spec:
  csiSnapshotTimeout: 10m0s
  defaultVolumesToFsBackup: true
  includeClusterResources: true
  includedNamespaces:
  - gitlab-system
  itemOperationTimeout: 4h0m0s
  metadata:
    labels:
      app: gitlab
      backup-type: namespace
  snapshotMoveData: false
  storageLocation: oadp-dpa-1
  ttl: 720h
```

### 5. Restore from Backup
Save the following as `restore.yaml` and apply with `oc apply -f restore.yaml`:

```yaml
apiVersion: velero.io/v1
kind: Restore
metadata:
  name: gitlab-system-restore-to-gitlab-system-ns-restic
  namespace: openshift-adp
spec:
  backupName: gitlab-system-backup-restic
  namespaceMapping:
    gitlab-system: gitlab-system
  excludedResources:
    - events
    - operatorgroups
    - subscriptions
    - installplans
    - catalogsources
    - clusterserviceversions
    - operatorconditions
  existingResourcePolicy: update
  restorePVs: true
```
