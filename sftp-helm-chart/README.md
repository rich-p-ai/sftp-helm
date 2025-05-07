```markdown
# SFTP Server Helm Chart for OpenShift

A production-ready Helm chart for deploying secure SFTP servers on OpenShift using atmoz/sftp image with NodePort exposure.

## Table of Contents
1. [Prerequisites](#prerequisites)
2. [Installation](#installation)
3. [Configuration](#configuration)
4. [Accessing SFTP](#accessing-sftp)
5. [Persistent Storage](#persistent-storage)
6. [Troubleshooting](#troubleshooting)
7. [Uninstallation](#uninstallation)
8. [Security Considerations](#security-considerations)

## Prerequisites 

- OpenShift 4.6+ cluster with admin access
- Helm 3.8+ installed locally
- `oc` CLI configured with cluster access
- OpenShift project/namespace created
- Generated SSH host keys (instructions below)

## Installation 

### 1. Clone the Helm Chart Repository
```
git clone https://github.com/your-org/sftp-helm-chart.git
cd sftp-helm-chart
```

### 2. Generate SSH Host Keys
```
mkdir -p ssh-keys && cd ssh-keys
ssh-keygen -t ed25519 -f ssh_host_ed25519_key -N ""
ssh-keygen -t rsa -b 4096 -f ssh_host_rsa_key -N ""
cd ..
```

### 3. Create OpenShift Service Account
```
oc create serviceaccount sftp-sa
oc adm policy add-scc-to-user anyuid -z sftp-sa
```

### 4. Update Configuration Values
Edit `values.yaml`:
```
sshHostKeys:
  ed25519:
    private: |-
      $(cat ssh-keys/ssh_host_ed25519_key | sed 's/^/      /')
    public: |-
      $(cat ssh-keys/ssh_host_ed25519_key.pub | sed 's/^/      /')
  rsa:
    private: |-
      $(cat ssh-keys/ssh_host_rsa_key | sed 's/^/      /')
    public: |-
      $(cat ssh-keys/ssh_host_rsa_key.pub | sed 's/^/      /')
```

### 5. Deploy the Helm Chart
```
helm install sftp-server . \
  --namespace sftp-project \
  --create-namespace \
  --set persistence.storageClass=openshift-storage
```

## Configuration 

### Core Settings (values.yaml)
```
# User configuration (username:password[:e][:uid[:gid[:dir1[,dir2]...]]]
users:
  - "sftp-user:secure-password:1001:100:uploads"
  - "backup-user:another-password:1002:100:backups"

# Network configuration
service:
  nodePort: 31022  # Valid range: 30000-32767
  
# Resource allocation
resources:
  requests:
    memory: "256Mi"
    cpu: "100m"
  limits:
    memory: "512Mi"
    cpu: "500m"
```

### Advanced Options
- **Custom Ports**: Modify `service.nodePort` (ensure port is allowed in network policies)
- **Multiple Users**: Add entries to `users` array
- **Storage Class**: Set `persistence.storageClass` to match your cluster's storage provisioner
- **Resource Limits**: Adjust CPU/memory in `resources` section

## Accessing SFTP 

### Get Connection Details
```
export NODE_IP=$(oc get nodes -o jsonpath='{.items.status.addresses[?(@.type=="InternalIP")].address}')
echo "SFTP Server: sftp://$NODE_IP:31022"
```

### Connect Using SFTP Client
```
sftp -P 31022 sftp-user@$NODE_IP
```

### Verify Server Fingerprint
```
ssh-keyscan -p 31022 $NODE_IP
```

## Persistent Storage 

### Storage Configuration
```
persistence:
  size: 5Gi
  accessMode: ReadWriteMany
  mountPath: /home
```

### Backup Strategy
1. Create snapshot of Persistent Volume
2. Use SFTP mirroring with `rsync`
3. Enable PVC expansion in StorageClass

## Troubleshooting 

### Common Issues
**Connection Refused**
```
oc get svc sftp-server -o jsonpath='{.spec.ports.nodePort}'
oc get nodes -o wide
```

**Authentication Failures**
```
oc logs deployment/sftp-server --tail=50
```

**Permission Issues**
```
oc rsh deployment/sftp-server ls -la /home
```

### Audit Logs
```
oc logs deployment/sftp-server | grep 'sshd'
```

## Uninstallation 

### Remove Deployment
```
helm uninstall sftp-server
oc delete pvc sftp-server-data
```

### Cleanup Resources
```
oc delete serviceaccount sftp-sa
oc adm policy remove-scc-from-user anyuid -z sftp-sa
```

## Security Considerations 

1. **Key Rotation**: Rotate SSH host keys quarterly
2. **Network Policies**:
   ```
   apiVersion: networking.k8s.io/v1
   kind: NetworkPolicy
   metadata:
     name: sftp-firewall
   spec:
     podSelector:
       matchLabels:
         app: sftp
     ingress:
     - ports:
       - protocol: TCP
         port: 22
   ```
3. **Audit Logging**: Enable syslog forwarding
4. **Image Security**: Use signed images from verified registry

## Upgrades
```
helm upgrade sftp-server . \
  --set image.tag=alpine-3.18 \
  --reuse-values
```

## Monitoring
```
# Sample Prometheus annotations
prometheus:
  scrape: true
  port: 9115
  path: /metrics
```
