# 🧱 Nutanix 기반 Kubernetes Cluster 구축

본 문서는 **Nutanix AHV 가상화 환경**에서  
**Ansible(Kubespray)** 를 사용해 Kubernetes 클러스터를 구축하고,  
Ingress / GitOps / 확장성을 고려한 기본 아키텍처를 정리한 문서입니다.
향후 Private Cloud 환경에 쿠버네티스 도입 프로젝트 기회를 만들기 위함.

Devops Skill: CI/CD 파이프라인 구성(gitlab, gitlab-ci, ArgoCD, Repo)
SRE Skill: Grafana Observerbility Stack 구성(Prometheus, Loki, Tempo, Opentelemetry)

---

## 🎯 구축 목표

- Nutanix VM 환경에서 Kubernetes 클러스터 구축
- 개발(dev) / 스테이징(stg) / 운영(prod) 등 **다중 클러스터 운영 가능 구조**
- 물리 L4 장비 없이 **소프트웨어 기반 LoadBalancer 구성**
- GitOps(ArgoCD) 기반 애플리케이션 배포
- 보안 및 운영 확장을 고려한 네트워크 / IP 설계

---

## 🏗️ 전체 아키텍처 개요

- **Hypervisor**: Nutanix AHV
- **OS**: Ubuntu 22.04 LTS
- **Kubernetes 설치 방식**: Kubespray (Ansible 기반)
- **Container Runtime**: containerd
- **CNI**: Calico
- **Ingress**: ingress-nginx
- **LoadBalancer**: MetalLB (L2 Mode)
- **GitOps**: ArgoCD
- **Storage**: NFS (RWX 기반)

---

## 🌐 네트워크 구성 개요

### 1️⃣ VM(Node) 네트워크 (Nutanix Subnet)

- Subnet / VLAN: VLAN1214
- CIDR: 172.28.2.0/24
- 해당 대역은 Nutanix VM(Node)들이 실제로 사용하는 네트워크

#### 사용 정책
- 기존 VM 사용 IP와 충돌 방지
- Kubernetes 노드 및 LoadBalancer용 IP 구간 명확히 분리

---

### 2️⃣ Kubernetes 내부 네트워크

> ⚠️ 아래 CIDR은 Nutanix Subnet과 **완전히 다른 레이어**입니다.

| 구분 | CIDR | 설명 |
|---|---|---|
| Pod CIDR | 172.20.0.0/20 | Pod 전용 가상 네트워크 |
| Service CIDR | 172.21.0.0/24 | ClusterIP(Service) 전용 |

- 클러스터 간 CIDR 중복 금지
- dev / stg / prod / infra(jenkins, monitoring) 클러스터별로 분리 운영

---

## 🧮 IP 할당 정책 (172.28.2.0/24)

### ✅ Kubernetes 노드용 IP 구간
172.28.2.160 ~ 172.28.2.189

- Control Plane, Worker Node 공용
- 고정 IP 할당 (DHCP 사용하지 않음)

예시:
- cp-01 ~ cp-03 : 172.28.2.161 ~ 172.28.2.163
- worker nodes  : 172.28.2.170 ~

---

### ✅ MetalLB IP Pool (예약 구간)
172.28.2.210 ~ 172.28.2.229

- Ingress / ArgoCD / Monitoring 등 외부 노출 서비스용
- 절대 다른 VM에서 사용 금지
- 문서 및 운영 정책으로 명확히 예약

---

## 🚪 외부 접근 흐름

[Client]
↓
[Public IP or Internal Access]
↓ (DNAT / Routing)
[MetalLB LoadBalancer IP]
↓
[Ingress Controller]
↓
[Service (ClusterIP)]
↓
[Pod]

yaml
코드 복사

- VM(Node) IP를 직접 외부에 노출하지 않음
- 필요 시 방화벽/라우터에서 DNAT 구성

---

## 🗂️ 스토리지 구성 (NFS)

- Kubernetes 애드온 및 플랫폼 컴포넌트는 영속 스토리지 필수
- NFS 기반 RWX 스토리지 사용

### NFS 사용 대상
- ArgoCD (repo cache 등)
- GitLab (Repo / Artifact / Uploads 등)
- Monitoring Stack (Prometheus / Grafana)

> ⚠️ PVC가 준비되지 않으면 Helm 설치 시 Pod가 Pending 상태에 머무를 수 있음

---

## 🔐 Bastion / 운영 접근 방식

### 운영 환경 기준 접근 정책
- 로컬 PC → 직접 노드 접근 ❌ (차단 권장)
- JumpHost(Bastion) VM을 통해서만 내부 접근 ✅

### JumpHost 역할
- Ansible / Kubespray 실행
- kubectl / helm / terraform 관리
- (선택) Browser 기반 IDE 또는 Remote Development 환경

---

## ⚙️ Kubernetes 설치 순서 요약

1. Nutanix VM 생성 (Control Plane / Worker / JumpHost)
2. OS 기본 설정 (swap off, hostname, SSH)
3. Kubespray Inventory 및 변수 구성
4. Ansible로 Kubernetes 설치
5. NFS StorageClass 구성
6. MetalLB 설치 및 IP Pool 설정
7. ingress-nginx 설치
8. ArgoCD 설치 (GitOps 구성)
9. Cluster Autoscaler + 노드 증설 자동화(향후)

---

## 📌 운영 시 유의사항

- Pod / Service CIDR은 초기 설계가 매우 중요 (변경 어려움)
- MetalLB IP Pool은 DHCP 범위와 반드시 분리
- dev / prod 클러스터는 네트워크 분리 권장
- 클러스터 수 증가를 고려해 CIDR 할당 정책 문서화

---

## ✍️ 정리

> Kubernetes 클러스터 자체는 같은 서브넷에서도 동작하지만,  
> 운영·보안·확장성을 고려하면  
> IP / CIDR / 접근 경계는 반드시 초기부터 설계해야 한다.

---

📦 본 문서는 향후 다음을 고려한 확장 설계를 전제로 한다.

- Jenkins 전용 클러스터
- Monitoring 전용 클러스터
- 멀티 클러스터 연동

## Step1.
Nutanix 접근 설정 
- nutanix 내부에 vm host를 두고 control 할지
- 로컬 PC를 이용해 VPN 연결해서 control 할지 

## Step2.


## Step3.
