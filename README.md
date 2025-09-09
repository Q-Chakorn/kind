# การทดสอบ Kubernetes Cluster ด้วย Kind

## 📋 ภาพรวม

เอกสารนี้รวบรวมการทดสอบและการตั้งค่าสำหรับ Kubernetes cluster แบบหลาย nodes โดยใช้ **Kind** (Kubernetes in Docker) ที่ประกอบด้วย 1 control plane และ 2 worker nodes

## 🏗️ สถาปัตยกรรม Cluster

```
┌─────────────────────────────────────┐
│            Kind Cluster             │
├─────────────────────────────────────┤
│  master-control-plane (ควบคุม)      │
│  ├── API Server                     │
│  ├── etcd                          │
│  ├── Controller Manager            │
│  └── Scheduler                     │
├─────────────────────────────────────┤
│  master-worker (Worker Node 1)      │
│  ├── kubelet                       │
│  ├── kube-proxy                    │
│  └── Container Runtime             │
├─────────────────────────────────────┤
│  master-worker2 (Worker Node 2)     │
│  ├── kubelet                       │
│  ├── kube-proxy                    │
│  └── Container Runtime             │
└─────────────────────────────────────┘
```

## 🚀 เริ่มต้นใช้งาน

### สิ่งที่ต้องมีก่อน
- Docker Desktop ที่กำลังทำงาน
- kubectl ติดตั้งแล้ว
- Kind ติดตั้งแล้ว

### 1. สร้าง Cluster แบบหลาย Nodes

```bash
# สร้างไฟล์ config สำหรับ cluster
cat > kind-config.yaml << EOF
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
- role: worker
- role: worker
EOF

# สร้าง cluster
kind create cluster --config kind-config.yaml --name master
```

### 2. ตรวจสอบ Cluster

```bash
# ดู nodes ทั้งหมด
kubectl get nodes -o wide

# ดู system pods
kubectl get pods -n kube-system

# ดูข้อมูล cluster
kubectl cluster-info
```

## 🧪 แอปพลิเคชันทดสอบ

### Nginx Deployment แบบหลาย Pods

ทดสอบการกระจาย load และการแจก pods ไปยัง nodes ต่างๆ:

```bash
kubectl apply -f test-nginx-deployment.yaml
```

**คุณสมบัติ:**
- 4 replica pods
- แจกจ่ายอัตโนมัติไปยัง worker nodes
- แสดงข้อมูล node และ pod
- มี service สำหรับกระจาย load

### DaemonSet สำหรับตรวจสอบ Nodes

deploy pods ตรวจสอบไปทุก nodes (รวม control plane):

```bash
kubectl apply -f test-daemonset.yaml
```

**คุณสมบัติ:**
- หนึ่ง pod ต่อหนึ่ง node
- ทำงานได้บน control plane (มี toleration)
- บันทึกข้อมูล node อย่างต่อเนื่อง

## 🔍 การตรวจสอบสุขภาพ (Health Checks)

### สุขภาพ Cluster พื้นฐาน

```bash
# ตรวจสอบ components
kubectl get componentstatuses

# สถานะ nodes
kubectl get nodes

# system pods
kubectl get pods -n kube-system

# events ล่าสุด
kubectl get events --sort-by='.lastTimestamp'
```

### การเชื่อมต่อเครือข่าย

```bash
# ทดสอบ DNS resolution
kubectl run test-pod --image=busybox --restart=Never --rm -i --tty -- \
  nslookup kubernetes.default.svc.cluster.local

# ทดสอบการเชื่อมต่อระหว่าง pods
kubectl run ping-test --image=busybox:1.35 --restart=Never --rm -i --tty -- sh -c "
echo 'Testing inter-pod connectivity:'
echo
for pod_ip in <pod_ip> <pod_ip> <pod_ip> <pod_ip>; do
  echo \"Pinging \$pod_ip:\"
  ping -c 2 \$pod_ip
  echo
done"
```

### ทดสอบ Load Balancing

```bash
# ทดสอบการกระจาย load ของ service
kubectl run test-client --image=curlimages/curl:8.4.0 --restart=Never --rm -i --tty -- sh -c "
for i in \$(seq 1 8); do
  echo '=== Request \$i ==='
  curl -s test-nginx-service | grep -E '(Node|Pod IP|Hostname)'
  echo
done"
```

## 📊 ผลการทดสอบ

### ✅ การทดสอบที่สำเร็จ

1. **การกระจาย Pods**
   - ✓ Nginx pods กระจายไปยัง worker nodes
   - ✓ DaemonSet pods ทำงานบนทุก nodes รวม control plane

2. **Load Balancing**
   - ✓ Service กระจาย traffic ไปยังหลาย pods
   - ✓ Round-robin load balancing ทำงานได้

3. **การสื่อสารเครือข่าย**
   - ✓ pods สื่อสารกันได้ข้าม nodes
   - ✓ DNS resolution ทำงานได้
   - ✓ Service discovery ใช้งานได้

4. **Components ของ Cluster**
   - ✓ System components ทั้งหมดปกติ
   - ✓ Control plane ตอบสนองดี
   - ✓ Worker nodes พร้อมใช้งาน

### สรุปการกระจาย Pods

| Node | System Pods | Test Pods | รวม |
|------|-------------|-----------|------|
| master-control-plane | 4 | 1 | 5 |
| master-worker | 3 | 3 | 6 |
| master-worker2 | 3 | 3 | 6 |

## 🛠️ ไฟล์การตั้งค่า

### kind-config.yaml
```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
- role: worker
- role: worker
```

### test-nginx-deployment.yaml
Nginx deployment แบบหลาย replica พร้อมแสดงข้อมูล node

### test-daemonset.yaml
DaemonSet สำหรับตรวจสอบ nodes ทั้งหมดใน cluster

## 🔧 คำสั่งที่มีประโยชน์

### การจัดการ Cluster

```bash
# สร้าง cluster
kind create cluster --config kind-config.yaml --name master

# ลบ cluster
kind delete cluster --name master

# ดู clusters ทั้งหมด
kind get clusters
```

### การ Debug

```bash
# ดูข้อมูลละเอียดของ nodes
kubectl describe nodes

# ดู logs ของ pod
kubectl logs <pod-name>

# ดู events ทั้งหมด
kubectl get events --all-namespaces

# การใช้ resource (ต้องมี metrics-server)
kubectl top nodes
kubectl top pods
```

### การทำความสะอาด

```bash
# ลบแอพทดสอบ
kubectl delete -f test-nginx-deployment.yaml
kubectl delete -f test-daemonset.yaml

# ลบ cluster
kind delete cluster --name master
```

## 🚨 ข้อจำกัดที่ทราบ

1. **Metrics Server**: ไม่ได้ติดตั้งมาตั้งแต่ต้น (คำสั่ง kubectl top ใช้ไม่ได้)
2. **Load Balancer**: ไม่รองรับ external LoadBalancer (ใช้ NodePort หรือ Ingress แทน)
3. **Persistent Storage**: จำกัดอยู่ที่ local storage เท่านั้น

## 📝 หมายเหตุ

- Kind ใช้ Docker containers เป็น Kubernetes nodes
- Control plane ทำงานที่ port 59296 (กำหนดแบบสุ่ม)
- Network range: 10.244.0.0/16 สำหรับ pod IPs
- Service network: 10.96.0.0/12

## 🎯 สิ่งที่ได้เรียนรู้

1. **การทำงานร่วมกันของ Nodes**: Control plane สามารถจัดการ worker nodes ได้อย่างมีประสิทธิภาพ
2. **การกระจาย Workload**: Kubernetes scheduler กระจาย pods ไปยัง nodes ต่างๆ อย่างเหมาะสม  
3. **Service Discovery**: DNS และ service mesh ทำงานได้ดีในสภาพแวดล้อมหลาย nodes
4. **Network Policy**: การสื่อสารระหว่าง pods ข้าม nodes ทำงานได้ราบรื่น

## 🔗 อ้างอิง

- [เอกสาร Kind](https://kind.sigs.k8s.io/)
- [เอกสาร Kubernetes](https://kubernetes.io/docs/)
- [kubectl Cheat Sheet](https://kubernetes.io/docs/reference/kubectl/cheatsheet/)

---

**ทดสอบบน:** macOS พร้อม Docker Desktop  
**Kubernetes Version:** v1.34.0
