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

오픈 포트
> Note:
> - 192.168.0.0/16 is the Pod CIDR used by Calico VXLAN overlay network.
> - Traffic is encapsulated over UDP 4789 and never exposed outside the cluster.
* Control Plane
To                         Action      From
--                         ------      ----
22/tcp                     ALLOW       Anywhere                  
6443/tcp                   ALLOW       172.31.0.0/16             
6443/tcp                   DENY        Anywhere                  
10250/tcp                  ALLOW       172.31.0.0/16             
4789/udp                   ALLOW       172.31.0.0/16             
Anywhere                   ALLOW       192.168.0.0/16            
22/tcp (v6)                ALLOW       Anywhere (v6)             
6443/tcp (v6)              DENY        Anywhere (v6) 

* Worker
To                         Action      From
--                         ------      ----
22/tcp                     ALLOW       Anywhere                  
10250/tcp                  ALLOW       172.31.0.0/16             
80/tcp                     ALLOW       Anywhere             http(진입 노드만)     
443/tcp                    ALLOW       Anywhere             https(진입 노드만)     
6443/tcp                   ALLOW       172.31.0.0/16             
4789/udp                   ALLOW       172.31.0.0/16             
Anywhere                   ALLOW       192.168.0.0/16            
22/tcp (v6)                ALLOW       Anywhere (v6)             
80/tcp (v6)                ALLOW       Anywhere (v6)             
443/tcp (v6)               ALLOW       Anywhere (v6) 

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
5) 모든 서버간 10250 포트 인바운드 오픈 / 모든 서버간 All Traffic 아웃바운드 오픈 필요.
  > 프라이빗 IP 대역에 한정하여 처리.

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
```

### 1-2) Worker 노드 Join ( Control Plane 과 Worker Node 연결 )
```text
1) 토큰 생성 : kubeadm token create --print-join-command
- 일회성/유효기간 있는 인증 토큰
- 기본 유효기간: 24시간
2) Control Plane 서버에서 아래 명령어를 실행하고, 출력된 join 명령을 Worker 서버에서 그대로 실행
- ※ 토큰 생성하면 자동으로 명령어 작성되서 나오므로 그대로 노드 워커에 명령어 실행(아래는 예시)
- ※ AWS 기준 보안 그룹 프라이빗 IP 대역(xxx.xx.0.0/16) 6443 포트 오픈 / 기타 방화벽 6443 포트 오픈 필요
- <IP> : AWS 기준 private ip 로 설정
- <TOKEN> : 3-0 에서 생성한 토큰 입력
sudo kubeadm join <IP>:6443 \
  --token <TOKEN> \
  --discovery-token-ca-cert-hash sha256:<HASH>
```
### 1-3) CNI( Container Network Interface ) 설치 - Calico
```text
kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml
※ Calico VXLAN-only 전환(4789 포트 오픈) : BGP 는 “네트워크 통제력이 필요한 환경”에서 쓰는 것. 
> BGP : “네트워크는 네가 책임져라”
   - 네트워크 설계
   - 라우팅
   - 방화벽
   - 장애 분석
> VXLAN : “네트워크는 플랫폼이 처리할게”
   - Kubernetes/CNI가 책임
   - 운영자는 정책만 관리

1) kubectl patch ippool default-ipv4-ippool \
  --type=merge \
  -p '{
    "spec": {
      "ipipMode": "Never",
      "vxlanMode": "Always"
    }
  }'

2) BGP 설정이 있으면 비활성화
kubectl get bgpconfiguration -A  결과 있으면,
kubectl patch bgpconfiguration default \
  --type=merge \
  -p '{"spec":{"logSeverityScreen":"Info","nodeToNodeMeshEnabled":false}}'

2-1) BGP 설정이 강제되고 있는 경우
kubectl -n kube-system patch cm calico-config --type=merge \
  -p '{"data":{"calico_backend":"vxlan"}}'

kubectl -n kube-system patch ds calico-node --type=merge -p '{
  "spec": {
    "template": {
      "spec": {
        "containers": [
          {
            "name": "calico-node",
            "env": [
              { "name": "CLUSTER_TYPE", "value": "k8s" },
              { "name": "CALICO_IPV4POOL_IPIP", "value": "Never" },
              { "name": "CALICO_IPV4POOL_VXLAN", "value": "Always" }
            ]
          }
        ]
      }
    }
  }
}'


kubectl -n kube-system patch ds calico-node --type='json' -p='[
  {"op":"replace","path":"/spec/template/spec/containers/0/livenessProbe/exec/command","value":["/bin/calico-node","-felix-live"]},
  {"op":"replace","path":"/spec/template/spec/containers/0/readinessProbe/exec/command","value":["/bin/calico-node","-felix-ready"]}
]'


3) Calico 컴포넌트 재시작 (필수)
kubectl -n kube-system rollout restart ds/calico-node
kubectl -n kube-system rollout restart deploy/calico-kube-controllers

*) UFW 켠 뒤 Forwarding(라우팅/포워딩) 완전 설정
- UFW 켜면 기본 정책 때문에 포워딩이 끊기는 케이스가 많음.
- 지금 tcpdump에서 VXLAN 캡슐은 들어오는데 내부 SYN만 반복되는 건 “포워딩/필터링”이 중간에서 잡아먹는 전형적인 패턴.
# 1) 커널 포워딩/브리지 netfilter 활성화 (README 0-2에 있는 핵심) :contentReference[oaicite:1]{index=1}
sudo modprobe br_netfilter
sudo sysctl -w net.ipv4.ip_forward=1
sudo sysctl -w net.bridge.bridge-nf-call-iptables=1

# 2) UFW 기본 포워딩 정책을 ACCEPT로 (이걸 안 하면 계속 꼬일 수 있음)
sudo sed -i 's/^DEFAULT_FORWARD_POLICY=.*/DEFAULT_FORWARD_POLICY="ACCEPT"/' /etc/default/ufw
sudo ufw reload

```



### 1-4) 노드 역할 분리 (보안 & 운영 핵심)
```text
0) 아래 명령어들은 Control Plane 에서 실행
  - kubectl get nodes 명령어 실행 후 NAME 정보를 명령어 중 <control-node>, <worker-node>에 각각 적용.

1) Control Plane에 Pod가 절대 올라가지 않도록 처리
kubectl taint nodes <control-node> node-role.kubernetes.io/control-plane=:NoSchedule
- 이미 적용된 경우 다음과 같은 에러 발생
error: node ip-172-31-92-23 already has node-role.kubernetes.io/control-plane taint(s) with same effect(s) and --overwrite is false

2) Worker 라벨
kubectl label node <worker-node> role=worker

3) edge-ingress: enable 설정
kubectl label node <노드이름> edge-ingress=enabled
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

## 4. k8s 구성
### 1) 설정 파일 구성
```text
k8s/
  edge/
    ingress-nginx/
      00-namespace.yaml
      01-rbac.yaml
      02-configmap.yaml
      03-daemonset.yaml
      04-service-internal.yaml
      05-ingressclass.yaml
      kustomization.yaml   # 설정 파일들 kustomize.yaml로 묶음
```

## 5. Ingress Controller 설정
### 1) edge, ingress - nginx 적용
```text
kubectl apply -k ./edge/ingress-nginx
```

## 6. Default 404 적용
### 1) default 404 페이지 적용
```text
kubectl apply -k ./edge/default-404
```


## 5. 와일드카드 SSL 적용
### 0️⃣ 사전 준비
```text
0-1. Cloudflare API Token 생성
Cloudflare 대시보드 → My Profile → API Tokens → Create Token
권한
Permissions
Zone → DNS → Edit
Zone Resources
Include → Specific zone → domain-nyd.uk
생성 후 API Token 값 복사 (한 번만 보여줌)

0-2. DNS 레코드 상태
A 레코드가 현재 워커 퍼블릭 IP를 가리키고 있어야 함
Proxy(오렌지 구름) ❌ DNS only
DNS 전파가 완벽하지 않아도 DNS-01은 Cloudflare API로 처리되므로 OK
```
### 1️⃣ cert-manager 설치
```text
1) cert-manager 네임스페이스 생성
kubectl create namespace cert-manager

2) CRD 설치
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.14.5/cert-manager.crds.yaml

3) cert-manager 본체 설치
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.14.5/cert-manager.yaml

4) 정상 기동 확인
kubectl -n cert-manager get pods
```

### 2️⃣ Cloudflare API Token을 Kubernetes Secret으로 저장
```text
1) export CF_API_TOKEN='여기에 토큰 문자열'

2) kubectl create secret generic cloudflare-api-token \
  -n cert-manager \
  --from-literal=api-token="${CF_API_TOKEN}" \
  --dry-run=client -o yaml | kubectl apply -f -

3) unset CF_API_TOKEN
```

### 3️⃣ ClusterIssuer 생성 (Let’s Encrypt + DNS-01)
```text
1) kubectl apply -f cloudflare-dns-cluster-issue.yaml
2) kubectl get clusterissuer letsencrypt-prod-dns
```

### 4️⃣ 와일드카드 Certificate 생성
```text
0) kubectl apply -f certificate-wildcard-domain.yaml
1) kubectl -n edge get certificate
2) kubectl -n edge describe certificate wildcard-domain-nyd-uk
3) kubectl -n edge get secret wildcard-domain-nyd-uk-tls
```

