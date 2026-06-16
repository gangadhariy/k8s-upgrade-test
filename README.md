# 🚀 Kubernetes Safe Upgrade Lab (k8s-upgrade-test)

This lab demonstrates a **zero-downtime Kubernetes upgrade using kubeadm** from v1.30 → v1.31 while running live traffic through Ingress.

---

# 🧠 Cluster Setup

- 1 Control Plane Node
- 2 Worker Nodes
- Kubernetes: v1.30 → v1.31
- Container runtime: containerd
- CNI: Calico (or default)
- Ingress: nginx (NodePort)

---

# 🎯 Goal

Perform Kubernetes upgrade without downtime:

✔ No request loss  
✔ No app downtime  
✔ Safe rolling upgrade  
✔ Live traffic during upgrade  

---

# 📦 App Deployment

Namespace:
```bash
kubectl create namespace upgrade-test
```

---

Deployment:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-app
  namespace: upgrade-test
spec:
  replicas: 2
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
      maxSurge: 1
  selector:
    matchLabels:
      app: hello-app
  template:
    metadata:
      labels:
        app: hello-app
    spec:
      containers:
      - name: app
        image: hashicorp/http-echo
        args:
          - "-text=Hello the server is running fine"
        ports:
        - containerPort: 5678
```

---

Service:
```yaml
apiVersion: v1
kind: Service
metadata:
  name: hello-svc
  namespace: upgrade-test
spec:
  selector:
    app: hello-app
  ports:
  - port: 80
    targetPort: 5678
  type: ClusterIP
```

---

Ingress:
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: hello-ingress
  namespace: upgrade-test
spec:
  ingressClassName: nginx
  rules:
  - host: hello.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: hello-svc
            port:
              number: 80
```

---

Pod Disruption Budget:
```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: hello-pdb
  namespace: upgrade-test
spec:
  minAvailable: 1
  selector:
    matchLabels:
      app: hello-app
```

---

# 🌐 Continuous Traffic Test

```bash
while true; do curl -H "Host: hello.local" http://<NODE-IP>:<NODEPORT>; echo; sleep 0.2; done
```

Expected:
```
Hello the server is running fine
```

---

# 🧩 STEP 1: REPO SETUP (ALL NODES)

##Run on ALL nodes:
```
sudo rm -f /etc/apt/sources.list.d/kubernetes.list
sudo mkdir -p /etc/apt/keyrings
```
```
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.31/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
```
```
echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.31/deb/ /" | sudo tee /etc/apt/sources.list.d/kubernetes.list
```
```
sudo apt-get update
```
```
apt-cache madison kubeadm
```

# 🟦 CONTROL PLANE UPGRADE
```
sudo apt-get install -y kubeadm=1.31.1-1.1
```
```
sudo kubeadm upgrade plan
```
```
sudo kubeadm upgrade apply v1.31.1
```
```
sudo apt-get install -y kubelet=1.31.1-1.1 kubectl=1.31.1-1.1
```
```
sudo systemctl restart kubelet
```
```
kubectl uncordon <control-plane-node>
```

# 🟩 WORKER NODE UPGRADE (ONE BY ONE)
```
kubectl drain <worker-node> --ignore-daemonsets --delete-emptydir-data
```
```
ssh ubuntu@<worker-node>
```
```
sudo apt-get install -y kubeadm=1.31.1-1.1 kubelet=1.31.1-1.1
```
```
sudo kubeadm upgrade node
```
```
sudo systemctl restart kubelet
```
exit
```
kubectl uncordon <worker-node>
```

# ⚠️ DRAIN NOTE

If you see:
Cannot evict pod as it would violate PodDisruptionBudget

This is expected and safe.

---

# 🔍 VERIFY

kubectl get nodes
kubectl get pods -A
curl -H "Host: hello.local" http://<NODE-IP>:<NODEPORT>

---

# 🎯 RESULT

✔ Kubernetes upgraded safely  
✔ No downtime  
✔ Pods rescheduled automatically  
✔ Ingress remains stable  
✔ Live traffic continues  

---

# 🚀 END OF LAB
