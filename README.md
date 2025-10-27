# PostgreSQL Deployment for K3s

This repository contains Kubernetes YAML manifests for deploying PostgreSQL in a k3s cluster following best practices.

## Overview

The deployment includes:
- Dedicated namespace for isolation
- Persistent storage for data durability
- Secret management for credentials
- ConfigMap for PostgreSQL configuration
- StatefulSet for stable database deployment
- Service for cluster-internal access

## Files

The manifests are numbered for ordered deployment:

1. **[01-namespace.yaml](01-namespace.yaml)** - Creates the `postgres` namespace
2. **[02-persistentvolume.yaml](02-persistentvolume.yaml)** - Defines a 10Gi PersistentVolume using local-path storage
3. **[03-persistentvolumeclaim.yaml](03-persistentvolumeclaim.yaml)** - Claims persistent storage for PostgreSQL data
4. **[04-secret.yaml](04-secret.yaml)** - Stores database credentials (username, password, database name)
5. **[05-configmap.yaml](05-configmap.yaml)** - Contains PostgreSQL configuration and custom postgresql.conf
6. **[06-statefulset.yaml](06-statefulset.yaml)** - Deploys PostgreSQL 17 Alpine with health checks and resource limits
7. **[07-service.yaml](07-service.yaml)** - Exposes PostgreSQL via ClusterIP service

## Prerequisites

- k3s cluster up and running
- kubectl configured to access your cluster
- Sufficient storage space at `/mnt/data/postgres` (or modify the path in [02-persistentvolume.yaml](02-persistentvolume.yaml))
- Sealed Secrets controller installed in your cluster (see setup instructions below)
- kubeseal CLI tool installed locally

## Sealed Secrets Setup

This deployment uses [Sealed Secrets](https://github.com/bitnami-labs/sealed-secrets) for secure credential management. Sealed Secrets allow you to safely store encrypted secrets in Git.

### Install Sealed Secrets Controller

```bash
kubectl apply -f https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.27.2/controller.yaml
```

### Install kubeseal CLI Tool

```bash
# Download kubeseal
wget https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.27.2/kubeseal-0.27.2-linux-amd64.tar.gz

# Extract and install
tar -xvzf kubeseal-0.27.2-linux-amd64.tar.gz
sudo install -m 755 kubeseal /usr/local/bin/kubeseal

# Verify installation
kubeseal --version
```

### Create Your Sealed Secret

1. **Create a temporary secret file** (this file should NEVER be committed to Git):

```bash
cat > /tmp/postgres-secret.yaml <<EOF
apiVersion: v1
kind: Secret
metadata:
  name: postgres-secret
  namespace: postgres
  labels:
    app: postgresql
type: Opaque
stringData:
  POSTGRES_PASSWORD: "your_secure_password"
EOF
```

2. **Seal the secret** (this creates an encrypted version safe for Git):

```bash
kubeseal -f /tmp/postgres-secret.yaml -w 04-secret.yaml --format yaml
```

3. **Delete the temporary file**:

```bash
rm /tmp/postgres-secret.yaml
```

4. **Commit the sealed secret** to Git (it's now encrypted and safe!)

The SealedSecret will be automatically decrypted by the controller in your cluster and converted to a regular Secret that your PostgreSQL pod can use.

## Configuration

### Storage Configuration

The PersistentVolume uses `/mnt/data/postgres` on the host. To change this:

1. Edit [02-persistentvolume.yaml](02-persistentvolume.yaml)
2. Update the `hostPath.path` value
3. Ensure the directory exists on your k3s node

### Resource Limits

Default resource allocation in [06-statefulset.yaml](06-statefulset.yaml):
- Requests: 256Mi memory, 250m CPU
- Limits: 1Gi memory, 1000m CPU

Adjust these based on your workload requirements.

## Deployment

### Deploy All Resources

```bash
kubectl apply -f 01-namespace.yaml \
              -f 02-persistentvolume.yaml \
              -f 03-persistentvolumeclaim.yaml \
              -f 04-secret.yaml \
              -f 05-configmap.yaml \
              -f 06-statefulset.yaml \
              -f 07-service.yaml
```

Or deploy all at once:

```bash
kubectl apply -f .
```

### Verify Deployment

Check the status of your PostgreSQL deployment:

```bash
# Check all resources in the postgres namespace
kubectl get all -n postgres

# Check PersistentVolume and PersistentVolumeClaim
kubectl get pv,pvc -n postgres

# Check pod logs
kubectl logs -n postgres postgres-0

# Check pod status
kubectl describe pod -n postgres postgres-0
```

Wait for the pod to be in `Running` state and ready (1/1):

```bash
kubectl get pods -n postgres -w
```

## Accessing PostgreSQL

### From Within the Cluster

PostgreSQL is accessible at:
```
postgres-service.postgres.svc.cluster.local:5432
```

Connection string example (connects to the default `postgres` database with default `postgres` user):
```
postgresql://postgres:your_password@postgres-service.postgres.svc.cluster.local:5432/postgres
```

### Port Forward for Local Access

To access PostgreSQL from your local machine:

```bash
kubectl port-forward -n postgres svc/postgres-service 5432:5432
```

Then connect using:
```bash
psql -h localhost -p 5432 -U postgres -d postgres
```

### Using kubectl exec

Execute commands directly in the PostgreSQL pod:

```bash
# Access PostgreSQL CLI (using default postgres user and database)
kubectl exec -it -n postgres postgres-0 -- psql -U postgres -d postgres

# Run a single command
kubectl exec -it -n postgres postgres-0 -- psql -U postgres -d postgres -c "SELECT version();"
```

## Configuration Details

### PostgreSQL Settings

The deployment uses PostgreSQL 16 Alpine with optimized settings in [05-configmap.yaml](05-configmap.yaml):

- max_connections: 100
- shared_buffers: 256MB
- effective_cache_size: 1GB
- UTF-8 encoding
- Custom PGDATA location: `/var/lib/postgresql/data/pgdata`

### Health Checks

The StatefulSet includes:

**Liveness Probe**:
- Checks if PostgreSQL is responsive
- Initial delay: 30 seconds
- Period: 10 seconds

**Readiness Probe**:
- Checks if PostgreSQL is ready to accept connections
- Initial delay: 10 seconds
- Period: 5 seconds

### Persistent Storage

- Storage Class: `local-path` (k3s default)
- Capacity: 10Gi
- Access Mode: ReadWriteOnce
- Reclaim Policy: Retain (data persists after PVC deletion)

## Maintenance

### Backup Database

```bash
# Create a backup of the default postgres database
kubectl exec -n postgres postgres-0 -- pg_dump -U postgres postgres > backup.sql

# Or backup all databases
kubectl exec -n postgres postgres-0 -- pg_dumpall -U postgres > backup-all.sql
```

### Restore Database

```bash
# Restore from backup
cat backup.sql | kubectl exec -i -n postgres postgres-0 -- psql -U postgres -d postgres
```

### Scale (Not Recommended for Single Instance)

This deployment uses a single replica. For high availability, consider:
- PostgreSQL replication setup
- Patroni or similar HA solutions
- Cloud-managed PostgreSQL services

### Update PostgreSQL Version

1. Edit [06-statefulset.yaml](06-statefulset.yaml)
2. Change the image tag (e.g., `postgres:17-alpine`)
3. Apply the changes:
   ```bash
   kubectl apply -f 06-statefulset.yaml
   ```

## Troubleshooting

### Pod Not Starting

```bash
# Check pod events
kubectl describe pod -n postgres postgres-0

# Check logs
kubectl logs -n postgres postgres-0

# Check PVC status
kubectl get pvc -n postgres
```

### Connection Issues

```bash
# Test connectivity from another pod
kubectl run -it --rm --image=postgres:17-alpine --restart=Never postgres-client -n postgres -- \
  psql -h postgres-service -U postgres -d postgres
```

### Storage Issues

```bash
# Check PV and PVC binding
kubectl get pv,pvc -n postgres

# Check node storage
kubectl get nodes
kubectl describe node <node-name>
```

## Cleanup

To remove the PostgreSQL deployment:

```bash
# Delete all resources
kubectl delete -f 07-service.yaml \
               -f 06-statefulset.yaml \
               -f 05-configmap.yaml \
               -f 04-secret.yaml \
               -f 03-persistentvolumeclaim.yaml \
               -f 02-persistentvolume.yaml \
               -f 01-namespace.yaml

# Or delete the entire namespace (this will delete everything)
kubectl delete namespace postgres
```

**Note**: The PersistentVolume has a `Retain` policy, so data at `/mnt/data/postgres` will persist even after deletion. Manually remove it if needed.

## Security Considerations

- **Change default credentials** in [04-secret.yaml](04-secret.yaml)
- Consider using Kubernetes secrets encryption at rest
- Use network policies to restrict access to PostgreSQL
- Regularly update the PostgreSQL image for security patches
- For production, use stronger passwords and consider external secret management (e.g., Vault)
- The service uses ClusterIP, making it accessible only within the cluster

## Best Practices Implemented

- ✅ StatefulSet for stable network identity and ordered deployment
- ✅ Persistent storage with proper reclaim policy
- ✅ Resource requests and limits defined
- ✅ Health checks (liveness and readiness probes)
- ✅ Secrets for sensitive data
- ✅ ConfigMap for configuration management
- ✅ Namespace isolation
- ✅ Proper labeling for resource management
- ✅ Alpine-based image for smaller footprint

## License

This configuration is provided as-is for use in your k3s cluster.
