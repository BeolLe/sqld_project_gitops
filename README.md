# sqld_project_gitops

sqld_project_gitops는 SolSQLD 프로젝트의 GitOps 및 플랫폼 운영 자산을 관리하는 저장소입니다.
애플리케이션 코드 저장소와 분리하여, Kubernetes 리소스의 desired state, ArgoCD Application, Image Updater 설정, Airflow 배포 자산을 이 저장소에서 관리합니다.

## 저장소 역할

이 저장소는 다음을 담당합니다.

- ArgoCD App-of-Apps 및 개별 Application 정의
- 프론트엔드 / 백엔드 / Gateway 배포 매니페스트
- 클러스터 공통 인프라 리소스
- Airflow values 및 Airflow 커스텀 이미지 빌드 자산
- ArgoCD Image Updater write-back 대상

즉, 애플리케이션 소스 레포가 아니라 **배포 기준 상태를 관리하는 GitOps 저장소**입니다.

## 디렉터리 구조

```text
.
├─ infra/
│  ├─ argocd/        # root-apps, Application, ImageUpdater 리소스
│  ├─ frontend/      # 프론트엔드 배포 리소스
│  ├─ backend/       # 백엔드 배포 리소스
│  ├─ gateway/       # Gateway / Route / cloudflared 관련 리소스
│  ├─ airflow/       # Airflow values 및 운영 설정
│  └─ cluster/       # 공통 클러스터 자산
│     ├─ cloudflare/
│     ├─ db/
│     ├─ gateway/
│     ├─ monitoring/
│     └─ network/
├─ images/
│  └─ airflow/       # Airflow 커스텀 이미지 Dockerfile 및 빌드 자산
└─ .github/workflows/ # GitOps/Airflow 관련 CI 워크플로우
```

## 운영 원칙

### 1. 앱 코드와 배포 기준 상태 분리
- 애플리케이션 코드 변경은 앱 레포에서 관리
- 배포 리소스와 자동 커밋은 이 GitOps 레포에서 관리

### 2. Git이 단일 진실 소스
- ArgoCD는 이 저장소의 manifest와 values를 기준으로 클러스터를 동기화
- 수동 변경보다 Git 기준 desired state를 우선

### 3. Secret 원문은 Git에 저장하지 않음
- 민감한 값은 Kubernetes Secret으로 별도 주입
- 이 저장소에는 Secret 참조 구조만 유지

### 4. Image Updater는 Git write-back 기준
- 새 이미지 감지 시 values 또는 `.argocd-source-*.yaml`에 digest를 기록
- 실제 워크로드 반영 시점은 ArgoCD sync 정책에 따름

## 현재 관리 범위

### 애플리케이션 배포
- `front-k8s`
- `back-k8s`
- `gateway`
- `airflow`

### 플랫폼 / 클러스터 자산
- PostgreSQL
- MetalLB
- Envoy Gateway 관련 리소스
- Prometheus / Grafana
- Cloudflare tunnel 및 내부 접근 구성

### Airflow
- Helm values
- GHCR 이미지 빌드 자산
- Image Updater 연동

## 배포 흐름

```text
앱 레포 코드 변경
→ GitHub Actions로 이미지 빌드 및 GHCR push
→ Image Updater가 새 digest 감지
→ 이 GitOps 레포에 write-back
→ ArgoCD가 변경 사항 sync
→ Kubernetes 배포 반영
```

## 참고

- 앱 레포: `sqld_project_git`
- 이 레포는 앱 코드가 아니라 배포 기준 상태를 관리합니다.
- 세부 아키텍처 배경과 의사결정은 ADR 및 별도 문서에서 관리합니다.
