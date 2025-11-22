# CKA Exam Preparation Cheat Sheet (v1.34)

## Exam Information

- **Duration**: 2 hours
- **Questions**: 15-20 performance-based tasks
- **Passing Score**: 66%
- **Kubernetes Version**: v1.34
- **Cost**: $445 USD (includes one free retake)
- **Format**: Online, proctored, command-line based
- **Allowed Resources**: kubernetes.io/docs and kubernetes.io/blog

## Exam Domain Breakdown (Total: 100%)

### 1. Troubleshooting (30%)
- Evaluate cluster and node logging
- Understand how to monitor applications
- Manage container stdout & stderr logs
- Troubleshoot application failure
- Troubleshoot cluster component failure
- Troubleshoot networking

### 2. Cluster Architecture, Installation & Configuration (25%)
- Manage role-based access control (RBAC)
- Use Kubeadm to install a basic cluster
- Manage a highly-available Kubernetes cluster
- Provision underlying infrastructure to deploy a Kubernetes cluster
- Perform a version upgrade on a Kubernetes cluster using Kubeadm
- Implement etcd backup and restore

### 3. Services & Networking (20%)
- Understand host networking configuration on the cluster nodes
- Understand connectivity between Pods
- Understand ClusterIP, NodePort, LoadBalancer service types and endpoints
- Know how to use Ingress controllers and Ingress resources
- Know how to configure and use CoreDNS
- Choose an appropriate container network interface plugin

### 4. Workloads & Scheduling (15%)
- Understand deployments and how to perform rolling update and rollbacks
- Use ConfigMaps and Secrets to configure applications
- Know how to scale applications
- Understand the primitives used to create robust, self-healing application deployments
- Understand how resource limits can affect Pod scheduling
- Awareness of manifest management and common templating tools

### 5. Storage (10%)
- Understand storage classes, persistent volumes
- Understand volume mode, access modes and reclaim policies for volumes
- Understand persistent volume claims primitive
- Know how to configure applications with persistent storage

---

## Essential kubectl Commands & Aliases

### Setup (Do this first!)
```bash
# Set alias for kubectl
alias k=kubectl
export do="--dry-run=client -o yaml"
export now="--force --grace-period=0"

# Enable autocomplete
source <(kubectl completion bash)
complete -F __start_kubectl k

# Set default namespace for session
kubectl config set-context --current --namespace=<namespace>

# Verify context
kubectl config current-context

# Set vim as default editor
export KUBE_EDITOR="vim"
```

### Context & Configuration
```bash
kubectl cluster-info
kubectl cluster-info dump  # Detailed cluster state
kubectl config view
kubectl config get-contexts
kubectl config use-context <context-name>
kubectl config current-context
kubectl api-resources  # List all resource types
kubectl api-resources --namespaced=true  # Namespaced resources
kubectl api-resources --namespaced=false  # Cluster-scoped resources
kubectl api-versions   # List API versions
kubectl version --short  # Kubernetes version
```

---

## 1. TROUBLESHOOTING (30% - Highest Weight!)

### Cluster & Node Troubleshooting
```bash
# Check cluster health
kubectl get nodes
kubectl get nodes -o wide
kubectl get cs  # Component status (deprecated in v1.34, check directly)
kubectl cluster-info
kubectl cluster-info dump  # Detailed diagnostics

# Check node details and events
kubectl describe node <node-name>
kubectl get events --sort-by='.lastTimestamp'
kubectl get events -n <namespace> --sort-by='.lastTimestamp'
kubectl get events --field-selector involvedObject.kind=Pod
kubectl get events --field-selector involvedObject.name=<pod-name>

# Check node resources
kubectl top nodes
kubectl top nodes --sort-by=cpu
kubectl top nodes --sort-by=memory
kubectl describe node <node-name> | grep -A 5 "Allocated resources"

# Check node conditions
kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.conditions[?(@.type=="Ready")].status}{"\n"}{end}'

# SSH to nodes (if needed)
ssh node01
systemctl status kubelet
systemctl status containerd
systemctl status docker  # If using Docker
journalctl -u kubelet -f
journalctl -u kubelet --no-pager | tail -50
journalctl -u kubelet --since "10 minutes ago"

# Check kubelet config
cat /var/lib/kubelet/config.yaml
ps aux | grep kubelet

# Check certificate expiration
kubeadm certs check-expiration

# Check static pod manifests
ls -la /etc/kubernetes/manifests/
cat /etc/kubernetes/manifests/kube-apiserver.yaml
```

### Pod & Application Troubleshooting
```bash
# Check pod status
kubectl get pods -A
kubectl get pods -o wide
kubectl get pods --field-selector status.phase=Running
kubectl get pods --field-selector status.phase=Pending
kubectl get pods --field-selector status.phase=Failed
kubectl describe pod <pod-name>
kubectl get pod <pod-name> -o yaml
kubectl get pod <pod-name> -o json | jq '.status.conditions'

# Check logs
kubectl logs <pod-name>
kubectl logs <pod-name> -c <container-name>
kubectl logs <pod-name> --all-containers=true
kubectl logs <pod-name> --previous  # Previous crashed container
kubectl logs <pod-name> --tail=50
kubectl logs <pod-name> --since=1h
kubectl logs <pod-name> --since-time=2024-01-01T00:00:00Z
kubectl logs -f <pod-name>  # Follow logs
kubectl logs -l app=nginx  # Logs from all pods with label

# Check events
kubectl get events -n <namespace>
kubectl get events --sort-by='.lastTimestamp' -A
kubectl describe pod <pod-name> | grep -A 10 Events

# Execute commands in pod
kubectl exec -it <pod-name> -- /bin/sh
kubectl exec -it <pod-name> -- /bin/bash
kubectl exec <pod-name> -- cat /var/log/app.log
kubectl exec <pod-name> -- env
kubectl exec <pod-name> -- ps aux
kubectl exec <pod-name> -c <container-name> -- sh  # Multi-container

# Debug with ephemeral containers (v1.25+)
kubectl debug <pod-name> -it --image=busybox --target=<container-name>
kubectl debug <pod-name> -it --image=busybox --copy-to=<new-pod-name>
kubectl debug node/<node-name> -it --image=busybox

# Check resource usage
kubectl top pod <pod-name>
kubectl top pod <pod-name> --containers

# Check pod startup
kubectl get pod <pod-name> -w  # Watch pod status changes
```

### Network Troubleshooting
```bash
# Check services
kubectl get svc -A
kubectl get svc -o wide
kubectl describe svc <service-name>
kubectl get endpoints
kubectl get endpoints <service-name>

# Test DNS
kubectl run test-dns --image=busybox:1.28 --rm -it -- nslookup kubernetes.default
kubectl run test-dns --image=busybox:1.28 --rm -it -- nslookup <service-name>
kubectl run test-dns --image=busybox:1.28 --rm -it -- nslookup <service-name>.<namespace>.svc.cluster.local

# Test connectivity
kubectl run test-curl --image=curlimages/curl --rm -it -- curl <service-name>:<port>
kubectl run test-wget --image=busybox --rm -it -- wget -O- <service-name>
kubectl run test-netcat --image=busybox --rm -it -- nc -zv <service-name> <port>

# Check network policies
kubectl get networkpolicies -A
kubectl describe networkpolicy <policy-name>
kubectl get networkpolicy <policy-name> -o yaml

# CoreDNS troubleshooting
kubectl get pods -n kube-system | grep coredns
kubectl logs -n kube-system <coredns-pod>
kubectl logs -n kube-system -l k8s-app=kube-dns
kubectl describe cm coredns -n kube-system

# Check CNI plugin
ls -la /etc/cni/net.d/
cat /etc/cni/net.d/*

# Check iptables rules (on node)
iptables -t nat -L -n | grep <service-name>
iptables-save | grep <service-name>

# Check kube-proxy
kubectl get pods -n kube-system | grep kube-proxy
kubectl logs -n kube-system <kube-proxy-pod>
kubectl get configmap kube-proxy -n kube-system -o yaml

# Port connectivity test
kubectl run test-busybox --image=busybox --rm -it -- telnet <host> <port>
```

### Component Troubleshooting
```bash
# Check control plane components
kubectl get pods -n kube-system
kubectl get pods -n kube-system -o wide
kubectl logs -n kube-system kube-apiserver-<node>
kubectl logs -n kube-system kube-controller-manager-<node>
kubectl logs -n kube-system kube-scheduler-<node>
kubectl logs -n kube-system etcd-<node>

# Check if static pods are running
ls -la /etc/kubernetes/manifests/
cat /etc/kubernetes/manifests/kube-apiserver.yaml
cat /etc/kubernetes/manifests/kube-controller-manager.yaml
cat /etc/kubernetes/manifests/kube-scheduler.yaml
cat /etc/kubernetes/manifests/etcd.yaml

# Kubelet issues
systemctl status kubelet
systemctl restart kubelet
journalctl -u kubelet -f
journalctl -u kubelet --since "10 minutes ago"
journalctl -u kubelet -n 100
cat /var/lib/kubelet/config.yaml
ps aux | grep kubelet

# Container runtime issues
systemctl status containerd
systemctl status docker
crictl ps  # List containers
crictl pods  # List pods
crictl logs <container-id>
crictl inspect <container-id>

# Check API server
kubectl get --raw /healthz
kubectl get --raw /readyz
kubectl get --raw /livez
curl -k https://localhost:6443/healthz

# Check scheduler
kubectl get events -n kube-system | grep scheduler
kubectl logs -n kube-system kube-scheduler-<node>

# Check controller manager
kubectl logs -n kube-system kube-controller-manager-<node>
kubectl get events -A | grep controller

# Check certificates
openssl x509 -in /etc/kubernetes/pki/apiserver.crt -text -noout
kubeadm certs check-expiration
```

---

## 2. CLUSTER ARCHITECTURE, INSTALLATION & CONFIGURATION (25%)

### RBAC (Role-Based Access Control)

#### Create Role & RoleBinding
```bash
# Create role imperatively
kubectl create role pod-reader --verb=get,list,watch --resource=pods

# Create rolebinding
kubectl create rolebinding read-pods --role=pod-reader --user=jane

# Check permissions
kubectl auth can-i get pods --as jane
kubectl auth can-i create deployments --as jane
```

#### Role & RoleBinding YAML
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: default
rules:
- apiGroups: [""]
  resources: ["pods", "pods/log"]
  verbs: ["get", "list", "watch"]

---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods
  namespace: default
subjects:
- kind: User
  name: jane
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

#### ClusterRole & ClusterRoleBinding
```bash
# Create cluster role
kubectl create clusterrole node-reader --verb=get,list --resource=nodes

# Create cluster role binding
kubectl create clusterrolebinding read-nodes --clusterrole=node-reader --user=jane
```

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: secret-reader
rules:
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["get", "watch", "list"]

---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: read-secrets-global
subjects:
- kind: Group
  name: manager
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: secret-reader
  apiGroup: rbac.authorization.k8s.io
```

#### ServiceAccount
```bash
# Create service account
kubectl create serviceaccount app-sa

# Create token for SA (v1.24+)
kubectl create token app-sa

# Bind role to service account
kubectl create rolebinding app-sa-binding --role=pod-reader --serviceaccount=default:app-sa

# Use in pod
kubectl run nginx --image=nginx --serviceaccount=app-sa
```

### Kubeadm Cluster Operations

#### Initialize Cluster (Master Node)
```bash
# Initialize control plane
kubeadm init --pod-network-cidr=10.244.0.0/16 --apiserver-advertise-address=<master-ip>

# Setup kubeconfig
mkdir -p $HOME/.kube
cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
chown $(id -u):$(id -g) $HOME/.kube/config

# Install CNI plugin (e.g., Calico)
kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml
```

#### Join Worker Nodes
```bash
# Get join command from master
kubeadm token create --print-join-command

# On worker node
kubeadm join <master-ip>:6443 --token <token> --discovery-token-ca-cert-hash sha256:<hash>
```

#### Cluster Upgrade with Kubeadm
```bash
# On master node
# 1. Check upgrade plan
kubeadm upgrade plan

# 2. Upgrade kubeadm
apt-mark unhold kubeadm
apt-get update
apt-get install -y kubeadm=1.34.0-00
apt-mark hold kubeadm

# 3. Verify version
kubeadm version

# 4. Apply upgrade
kubeadm upgrade apply v1.34.0

# 5. Drain master node
kubectl drain <master-node> --ignore-daemonsets

# 6. Upgrade kubelet and kubectl
apt-mark unhold kubelet kubectl
apt-get install -y kubelet=1.34.0-00 kubectl=1.34.0-00
apt-mark hold kubelet kubectl

# 7. Restart kubelet
systemctl daemon-reload
systemctl restart kubelet

# 8. Uncordon node
kubectl uncordon <master-node>

# On worker nodes
# 1. Drain node from master
kubectl drain <worker-node> --ignore-daemonsets --delete-emptydir-data

# 2. On worker node - upgrade kubeadm
apt-mark unhold kubeadm
apt-get update
apt-get install -y kubeadm=1.34.0-00
apt-mark hold kubeadm

# 3. Upgrade node
kubeadm upgrade node

# 4. Upgrade kubelet and kubectl
apt-mark unhold kubelet kubectl
apt-get install -y kubelet=1.34.0-00 kubectl=1.34.0-00
apt-mark hold kubelet kubectl
systemctl daemon-reload
systemctl restart kubelet

# 5. Uncordon from master
kubectl uncordon <worker-node>
```

### ETCD Backup & Restore

#### Backup ETCD
```bash
# Find etcd pod and get certificate paths
kubectl describe pod etcd-master -n kube-system

# Backup command
ETCDCTL_API=3 etcdctl snapshot save /backup/etcd-snapshot.db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# Verify backup
ETCDCTL_API=3 etcdctl snapshot status /backup/etcd-snapshot.db --write-out=table
```

#### Restore ETCD
```bash
# Restore etcd
ETCDCTL_API=3 etcdctl snapshot restore /backup/etcd-snapshot.db \
  --data-dir=/var/lib/etcd-restore \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# Update etcd manifest to use new data directory
vi /etc/kubernetes/manifests/etcd.yaml
# Change: --data-dir=/var/lib/etcd-restore

# Restart kubelet
systemctl restart kubelet
```

---

## 3. SERVICES & NETWORKING (20%)

### Service Types

#### ClusterIP (Internal access only)
```bash
# Create service imperatively
kubectl expose pod nginx --port=80 --target-port=80 --name=nginx-service

# Generate YAML
kubectl create service clusterip nginx-service --tcp=80:80 --dry-run=client -o yaml
```

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  selector:
    app: nginx
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
  type: ClusterIP
```

#### NodePort (External access via node IP)
```bash
kubectl expose deployment nginx --type=NodePort --port=80 --target-port=80 --name=nginx-nodeport
```

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-nodeport
spec:
  selector:
    app: nginx
  type: NodePort
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
    nodePort: 30080  # Optional, will auto-assign if not specified
```

#### LoadBalancer (Cloud load balancer)
```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-lb
spec:
  selector:
    app: nginx
  type: LoadBalancer
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
```

### Ingress

#### Ingress Controller Setup (Nginx)
```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.8.1/deploy/static/provider/cloud/deploy.yaml
```

#### Ingress Resource
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
  - host: app.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app-service
            port:
              number: 80
```

### Network Policies

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend
  namespace: prod
spec:
  podSelector:
    matchLabels:
      role: backend
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          role: frontend
    ports:
    - protocol: TCP
      port: 8080
  egress:
  - to:
    - podSelector:
        matchLabels:
          role: database
    ports:
    - protocol: TCP
      port: 3306
```

### CoreDNS

```bash
# Check CoreDNS pods
kubectl get pods -n kube-system -l k8s-app=kube-dns

# View CoreDNS config
kubectl get configmap coredns -n kube-system -o yaml

# Test DNS resolution
kubectl run test-dns --image=busybox:1.28 --rm -it -- nslookup kubernetes.default
```

---

## 4. WORKLOADS & SCHEDULING (15%)

### Deployments

#### Create Deployment
```bash
# Imperative
kubectl create deployment nginx --image=nginx --replicas=3

# Generate YAML
kubectl create deployment nginx --image=nginx --replicas=3 --dry-run=client -o yaml > deploy.yaml
```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.21
        ports:
        - containerPort: 80
        resources:
          requests:
            memory: "64Mi"
            cpu: "250m"
          limits:
            memory: "128Mi"
            cpu: "500m"
```

#### Rolling Updates & Rollbacks
```bash
# Update image
kubectl set image deployment/nginx nginx=nginx:1.22
kubectl set image deployment/nginx nginx=nginx:1.22 --record

# Check rollout status
kubectl rollout status deployment/nginx

# View rollout history
kubectl rollout history deployment/nginx

# Rollback to previous version
kubectl rollout undo deployment/nginx

# Rollback to specific revision
kubectl rollout undo deployment/nginx --to-revision=2

# Pause/Resume rollout
kubectl rollout pause deployment/nginx
kubectl rollout resume deployment/nginx
```

#### Scale Deployment
```bash
kubectl scale deployment nginx --replicas=5

# Autoscale
kubectl autoscale deployment nginx --min=3 --max=10 --cpu-percent=80
```

### ConfigMaps & Secrets

#### ConfigMaps
```bash
# Create from literal
kubectl create configmap app-config --from-literal=key1=value1 --from-literal=key2=value2

# Create from file
kubectl create configmap app-config --from-file=config.properties

# Create from env file
kubectl create configmap app-config --from-env-file=config.env
```

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  database_url: "mysql://db:3306"
  log_level: "info"
  config.properties: |
    key1=value1
    key2=value2
```

#### Using ConfigMap in Pod
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-pod
spec:
  containers:
  - name: app
    image: myapp
    # Method 1: All keys as env vars
    envFrom:
    - configMapRef:
        name: app-config
    # Method 2: Specific keys
    env:
    - name: DB_URL
      valueFrom:
        configMapKeyRef:
          name: app-config
          key: database_url
    # Method 3: Mount as volume
    volumeMounts:
    - name: config-volume
      mountPath: /etc/config
  volumes:
  - name: config-volume
    configMap:
      name: app-config
```

#### Secrets
```bash
# Create secret
kubectl create secret generic db-secret \
  --from-literal=username=admin \
  --from-literal=password=pass123

# Create TLS secret
kubectl create secret tls tls-secret --cert=path/to/cert --key=path/to/key

# Decode secret
kubectl get secret db-secret -o jsonpath='{.data.password}' | base64 --decode
```

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
type: Opaque
data:
  username: YWRtaW4=  # base64 encoded
  password: cGFzczEyMw==

---
# Using secret in pod
apiVersion: v1
kind: Pod
metadata:
  name: app-pod
spec:
  containers:
  - name: app
    image: myapp
    env:
    - name: DB_USER
      valueFrom:
        secretKeyRef:
          name: db-secret
          key: username
    - name: DB_PASS
      valueFrom:
        secretKeyRef:
          name: db-secret
          key: password
```

### Resource Limits & Scheduling

#### Pod with Resource Limits
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: resource-pod
spec:
  containers:
  - name: app
    image: nginx
    resources:
      requests:
        memory: "64Mi"
        cpu: "250m"
      limits:
        memory: "128Mi"
        cpu: "500m"
```

#### Node Affinity
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: disktype
            operator: In
            values:
            - ssd
  containers:
  - name: nginx
    image: nginx
```

#### Taints & Tolerations
```bash
# Add taint to node
kubectl taint nodes node01 key=value:NoSchedule

# Remove taint
kubectl taint nodes node01 key=value:NoSchedule-
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  tolerations:
  - key: "key"
    operator: "Equal"
    value: "value"
    effect: "NoSchedule"
  containers:
  - name: nginx
    image: nginx
```

#### Node Selector
```bash
# Label node
kubectl label nodes node01 disktype=ssd
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  nodeSelector:
    disktype: ssd
  containers:
  - name: nginx
    image: nginx
```

### DaemonSets
```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluentd
  namespace: kube-system
spec:
  selector:
    matchLabels:
      name: fluentd
  template:
    metadata:
      labels:
        name: fluentd
    spec:
      containers:
      - name: fluentd
        image: fluentd:latest
```

### StatefulSets
```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
spec:
  serviceName: "nginx"
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80
        volumeMounts:
        - name: www
          mountPath: /usr/share/nginx/html
  volumeClaimTemplates:
  - metadata:
      name: www
    spec:
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: 1Gi
```

---

## 5. STORAGE (10%)

### Persistent Volumes (PV)

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-data
spec:
  capacity:
    storage: 5Gi
  volumeMode: Filesystem
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: manual
  hostPath:
    path: /mnt/data
```

### Persistent Volume Claims (PVC)

```bash
# Create PVC imperatively
kubectl create pvc my-pvc --storage=2Gi --access-modes=ReadWriteOnce
```

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-data
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 2Gi
  storageClassName: manual
```

### Using PVC in Pod
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-with-storage
spec:
  containers:
  - name: app
    image: nginx
    volumeMounts:
    - mountPath: /data
      name: storage
  volumes:
  - name: storage
    persistentVolumeClaim:
      claimName: pvc-data
```

### Storage Classes

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast
provisioner: kubernetes.io/aws-ebs
parameters:
  type: gp3
  iopsPerGB: "10"
  fsType: ext4
volumeBindingMode: WaitForFirstConsumer
reclaimPolicy: Delete
```

### Volume Types

#### emptyDir
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-pd
spec:
  containers:
  - image: nginx
    name: test-container
    volumeMounts:
    - mountPath: /cache
      name: cache-volume
  volumes:
  - name: cache-volume
    emptyDir: {}
```

#### hostPath
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-pd
spec:
  containers:
  - image: nginx
    name: test-container
    volumeMounts:
    - mountPath: /test-pd
      name: test-volume
  volumes:
  - name: test-volume
    hostPath:
      path: /data
      type: Directory
```

---

## Quick Reference Commands

### Pod Operations
```bash
# Create pod
kubectl run nginx --image=nginx --restart=Never
kubectl run nginx --image=nginx --port=80 --expose  # Creates pod + service

# Generate YAML
kubectl run nginx --image=nginx --restart=Never --dry-run=client -o yaml > pod.yaml

# Get pods
kubectl get pods
kubectl get pods -o wide
kubectl get pods --all-namespaces
kubectl get pods -w  # Watch

# Delete pod
kubectl delete pod nginx
kubectl delete pod nginx --force --grace-period=0

# Edit pod
kubectl edit pod nginx

# Replace pod (after editing YAML)
kubectl replace --force -f pod.yaml
```

### Deployment Operations
```bash
# Create
kubectl create deployment nginx --image=nginx --replicas=3
kubectl create deployment nginx --image=nginx --port=80 --replicas=3

# Generate YAML
kubectl create deployment nginx --image=nginx --replicas=3 --dry-run=client -o yaml > deploy.yaml

# Expose
kubectl expose deployment nginx --port=80 --type=NodePort
kubectl expose deployment nginx --port=80 --target-port=8080 --type=ClusterIP

# Scale
kubectl scale deployment nginx --replicas=5
kubectl scale deployment nginx --replicas=0  # Scale down to 0

# Update
kubectl set image deployment/nginx nginx=nginx:1.22
kubectl set image deployment/nginx nginx=nginx:1.22 --record
kubectl set resources deployment nginx -c=nginx --limits=cpu=200m,memory=512Mi

# Rollout
kubectl rollout status deployment/nginx
kubectl rollout history deployment/nginx
kubectl rollout history deployment/nginx --revision=2
kubectl rollout undo deployment/nginx
kubectl rollout undo deployment/nginx --to-revision=2
kubectl rollout pause deployment/nginx
kubectl rollout resume deployment/nginx
kubectl rollout restart deployment/nginx  # Restart all pods

# Edit
kubectl edit deployment nginx

# Get
kubectl get deployments
kubectl get deployment nginx -o yaml
kubectl describe deployment nginx

# Delete
kubectl delete deployment nginx

# Autoscale (HPA)
kubectl autoscale deployment nginx --min=3 --max=10 --cpu-percent=80
kubectl get hpa
```

### Namespace Operations
```bash
# Create namespace
kubectl create namespace dev
kubectl create ns dev  # Short form

# Set default namespace
kubectl config set-context --current --namespace=dev

# Get current namespace
kubectl config view --minify | grep namespace

# Get resources in namespace
kubectl get all -n dev
kubectl get pods -n dev
kubectl get svc -n dev

# Get resources in all namespaces
kubectl get pods -A
kubectl get pods --all-namespaces

# Delete namespace
kubectl delete namespace dev

# Describe namespace
kubectl describe namespace dev

# Get namespace with resource quotas
kubectl get resourcequota -n dev
kubectl describe resourcequota -n dev
```

### Labels & Selectors
```bash
# Add label
kubectl label pod nginx env=prod
kubectl label pod nginx tier=frontend
kubectl label node node01 disktype=ssd

# Update label
kubectl label pod nginx env=staging --overwrite

# Remove label
kubectl label pod nginx env-
kubectl label node node01 disktype-

# Show labels
kubectl get pods --show-labels
kubectl get nodes --show-labels
kubectl get all --show-labels

# Filter by label (equality-based)
kubectl get pods -l env=prod
kubectl get pods -l env!=prod
kubectl get pods -l 'env in (prod,dev)'
kubectl get pods -l 'env notin (prod,dev)'
kubectl get pods -l env  # Has label env
kubectl get pods -l '!env'  # Doesn't have label env

# Filter by multiple labels
kubectl get pods -l env=prod,tier=frontend
kubectl get pods -l 'env=prod,tier in (frontend,backend)'

# Label nodes
kubectl label node node01 disktype=ssd
kubectl label node node01 region=us-east

# Get specific columns with labels
kubectl get pods -L env,tier
kubectl get nodes -L disktype,region

# Count pods by label
kubectl get pods -l app=nginx --no-headers | wc -l
```

### Output Formatting
```bash
# Wide output
kubectl get pods -o wide
kubectl get nodes -o wide

# YAML output
kubectl get pods -o yaml
kubectl get pod nginx -o yaml

# JSON output
kubectl get pods -o json
kubectl get pod nginx -o json | jq '.status.phase'

# Name only
kubectl get pods -o name

# Custom columns
kubectl get pods -o custom-columns=NAME:.metadata.name,STATUS:.status.phase
kubectl get pods -o custom-columns=NAME:.metadata.name,IMAGE:.spec.containers[*].image

# JSONPath
kubectl get pods -o jsonpath='{.items[*].metadata.name}'
kubectl get pods -o jsonpath='{.items[*].spec.containers[*].image}'
kubectl get pods -o jsonpath='{.items[*].status.podIP}'
kubectl get nodes -o jsonpath='{.items[*].status.addresses[?(@.type=="InternalIP")].address}'
kubectl get pods -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.phase}{"\n"}{end}'

# Sort by
kubectl get pods --sort-by=.metadata.creationTimestamp
kubectl get pods --sort-by=.status.startTime
kubectl get pods --sort-by=.metadata.name

# Show labels in columns
kubectl get pods -L app,env
kubectl get nodes -L disktype,region

# Combined filters and output
kubectl get pods -l app=nginx -o wide --sort-by=.metadata.creationTimestamp

# Go template
kubectl get pods -o go-template='{{range .items}}{{.metadata.name}}{{"\n"}}{{end}}'

# Watch resources
kubectl get pods -w
kubectl get pods -w -o wide

# No headers
kubectl get pods --no-headers
kubectl get pods --no-headers | wc -l  # Count pods
```

### Other Useful Commands
```bash
# Explain resource
kubectl explain pod
kubectl explain pod.spec
kubectl explain pod.spec.containers
kubectl explain pod.spec.containers.resources
kubectl explain deployment.spec.strategy

# API resources
kubectl api-resources
kubectl api-resources --namespaced=true
kubectl api-resources --namespaced=false
kubectl api-resources -o wide
kubectl api-resources --verbs=list,get
kubectl api-versions

# Copy files
kubectl cp <pod-name>:/path/to/file /local/path
kubectl cp /local/path <pod-name>:/path/to/file
kubectl cp <namespace>/<pod-name>:/path/to/file /local/path -c <container-name>

# Port forwarding
kubectl port-forward pod/nginx 8080:80
kubectl port-forward deployment/nginx 8080:80
kubectl port-forward service/nginx 8080:80
kubectl port-forward pod/nginx 8080:80 --address=0.0.0.0

# Proxy
kubectl proxy --port=8080
kubectl proxy --port=8080 --address=0.0.0.0

# Top (requires metrics-server)
kubectl top nodes
kubectl top nodes --sort-by=cpu
kubectl top nodes --sort-by=memory
kubectl top pods
kubectl top pods --sort-by=cpu
kubectl top pods --sort-by=memory
kubectl top pods --containers
kubectl top pod <pod-name> --containers

# Diff (compare changes before applying)
kubectl diff -f pod.yaml

# Apply with record
kubectl apply -f deployment.yaml --record
kubectl apply -f deployment.yaml --record --force

# Dry run and validation
kubectl apply -f pod.yaml --dry-run=client
kubectl apply -f pod.yaml --dry-run=server
kubectl apply -f pod.yaml --validate=true

# Multiple resources
kubectl get pods,svc,deployments
kubectl delete pods,svc --all

# Wait for condition
kubectl wait --for=condition=Ready pod/nginx --timeout=60s
kubectl wait --for=condition=Available deployment/nginx --timeout=300s

# Annotations
kubectl annotate pod nginx description="My nginx pod"
kubectl annotate pod nginx description-  # Remove annotation

# Patch resources
kubectl patch pod nginx -p '{"spec":{"containers":[{"name":"nginx","image":"nginx:1.22"}]}}'
kubectl patch deployment nginx -p '{"spec":{"replicas":5}}'

# Service account token
kubectl create token <service-account-name>
kubectl create token <service-account-name> --duration=1h

# Drain and cordon nodes
kubectl drain <node-name> --ignore-daemonsets
kubectl drain <node-name> --ignore-daemonsets --delete-emptydir-data
kubectl drain <node-name> --ignore-daemonsets --force
kubectl cordon <node-name>
kubectl uncordon <node-name>

# Taint nodes
kubectl taint nodes node01 key=value:NoSchedule
kubectl taint nodes node01 key=value:NoExecute
kubectl taint nodes node01 key=value:PreferNoSchedule
kubectl taint nodes node01 key:NoSchedule-  # Remove taint

# Get cluster info
kubectl cluster-info
kubectl cluster-info dump
kubectl cluster-info dump --output-directory=/path/to/dir

# Check API access
kubectl auth can-i create deployments
kubectl auth can-i create deployments --as john
kubectl auth can-i create deployments --as system:serviceaccount:default:my-sa
kubectl auth can-i '*' '*'  # Check all permissions
```

---

## Exam Tips & Strategies

### Before the Exam
1. **Set up aliases and autocomplete** at the start
2. **Practice with killer.sh** - You get 2 free attempts with exam registration
3. **Know where to find information** in kubernetes.io/docs quickly
4. **Practice typing speed** - Every second counts
5. **Master vim/nano** - You'll need to edit YAML files

### During the Exam
1. **Read each question carefully** - Note the context/namespace
2. **Always run the context switch command** provided in each question
3. **Start with high-weight questions** you're confident about
4. **Use imperative commands** whenever possible for speed
5. **Flag difficult questions** and move on - come back later
6. **Verify your work** before moving to next question
7. **Keep track of time** - 2 hours for 15-20 questions
8. **Use kubectl --help** and kubernetes.io extensively

### Time Management
- Spend ~5-7 minutes per question average
- Don't get stuck on one question for more than 10-15 minutes
- Leave buffer time for review (15-20 minutes)
- Troubleshooting questions may take longer

### Common Mistakes to Avoid
1. Not switching context/namespace
2. Forgetting to uncordon nodes after maintenance
3. Not verifying changes took effect
4. Making typos in YAML indentation
5. Not checking if resources are in the correct namespace
6. Forgetting to save files after editing

### Speed Tips
1. Use `--dry-run=client -o yaml` to generate manifests
2. Use `kubectl replace --force` for quick pod updates
3. Copy-paste from kubernetes.io docs when needed
4. Use `kubectl explain` for quick field lookups
5. Master `grep`, `awk`, and `sed` for filtering output

---

## Essential Vim Commands

```bash
# Navigation
i          # Insert mode
Esc        # Normal mode
:wq        # Save and quit
:q!        # Quit without saving
dd         # Delete line
yy         # Copy line
p          # Paste
u          # Undo
Ctrl+r     # Redo
/pattern   # Search
n          # Next search result
:%s/old/new/g  # Replace all
:set paste     # Enable paste mode
:set number    # Show line numbers
```

---

## Important Documentation Links

During the exam, you can access:
- https://kubernetes.io/docs/
- https://kubernetes.io/blog/

### Frequently Needed Pages
- RBAC: kubernetes.io/docs/reference/access-authn-authz/rbac/
- Network Policies: kubernetes.io/docs/concepts/services-networking/network-policies/
- Storage: kubernetes.io/docs/concepts/storage/persistent-volumes/
- kubeadm: kubernetes.io/docs/setup/production-environment/tools/kubeadm/
- etcd backup: kubernetes.io/docs/tasks/administer-cluster/configure-upgrade-etcd/

---

## Resource Requests vs Limits

| Metric | Requests | Limits |
|--------|----------|--------|
| Purpose | Minimum guaranteed | Maximum allowed |
| Scheduling | Used by scheduler | Not used by scheduler |
| Enforcement | Not enforced | Strictly enforced |
| CPU | Throttled at limit | - |
| Memory | - | Pod killed if exceeded (OOMKilled) |

---

## Service Types Comparison

| Type | Access | Use Case |
|------|--------|----------|
| ClusterIP | Internal only | Internal communication |
| NodePort | External via Node:Port | Development/testing |
| LoadBalancer | External via Cloud LB | Production external access |
| ExternalName | CNAME redirect | External service mapping |

---

## AccessModes for Volumes

| Mode | Abbreviation | Description |
|------|--------------|-------------|
| ReadWriteOnce | RWO | Single node read-write |
| Rea
