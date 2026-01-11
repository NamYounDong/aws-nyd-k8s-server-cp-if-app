# 쿠버네티스 + 인프라 + 앱 서버 3대 구조
## 0. 사전 조건
### 1) 스토리지 : local-path-provisioner 또는 AWS EBS CSI Driver (단순 구성)
### 2) 예상 비용 : 총합: 9~10만 원대(t3 → t3a 전환 시 추가 절감 가능)
### 3) SSL 인증서
```text
- ingress + cert-manager로 SSL 자동화
- 없는 host/path → Ingress 기본 404 안내 페이지
- Jenkins 리소스 과점유 방지 (requests/limits)
- metrics-server, kubectl top 사용 (모니터링)
- control-plane 자원 고갈 방지 (taint 유지)
```
### 0-1. 서버 공통
1) Swap 비활성화 (필수) : kubelet은 swap 켜져 있으면 정상 동작 ❌
```text
sudo swapoff -a
sudo sed -i '/ swap / s/^/#/' /etc/fstab
```
### 0-2. 커널 모듈 & 네트워크 설정
```text
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

sudo sysctl --system
```
### 0-3. containerd 설치
```text
sudo apt update
sudo apt install -y containerd

sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml

🔐 보안/안정 설정 (중요)
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
sudo systemctl restart containerd
sudo systemctl enable containerd
```

### 4) 0-4. kubeadm / kubelet / kubectl 설치
```text
sudo apt install -y apt-transport-https ca-certificates curl gpg

curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.29/deb/Release.key \
 | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.29/deb/ /" \
| sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt update
sudo apt install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```


## 1. 쿠버네티스 (controll plane) 서버
```text
1) 인스턴스: t3.small (또는 t3a.small) - ST : 추후 20GB
2) 역할:
   - kube-apiserver
   - controller-manager
   - scheduler
   - etcd (단일)
   - ingress / cert-manager / core 애드온
3) 특징:
   - NoSchedule taint 기본 적용
   - 필요 시 일부 시스템 파드만 허용

```
### 1-1) Kubernetes 클러스터 생성 (Control Plane)
```text
1) CIDR(Pod IP 대역) 설정
sudo kubeadm init \
  --pod-network-cidr=192.168.0.0/16

2) kubeconfig 설정
mkdir -p $HOME/.kube
sudo cp /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

3) Worker 노드 Join ( Control Plane 과 Worker Node 연결 )
- Control Plane에서 출력된 join 명령을 Worker 서버에서 그대로 실행

```

## 2. 인프라 서버 ( DB / Redis / Kafka / RabbitMQ ) - ST : 추후 30GB
```text
1) 인스턴스: t3.small
2) 역할:
   - DB (MariaDB / PostgreSQL)
   - Redis
   - Kafka / RabbitMQ (경량)
   - 모니터링 일부
3) 특징:
   - Jenkins는 별도 노드 분리 안 하고 앱과 공존
   - 리소스 requests/limits로 통제
```



## 3. Application + Jenkins 서버 - ST : 추후 50GB
```text
1) 인스턴스: t3.medium (또는 t3a.medium)
2) 역할:
- 실제 서비스 앱 파드
   - Jenkins (CI/CD)
   - 백엔드 / 프론트엔드
3) 특징:
   - nodeSelector 또는 taint/toleration으로 인프라 전용화
```