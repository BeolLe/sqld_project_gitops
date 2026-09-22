# SolSQLD GitOps

[SolSQLD](https://github.com/BeolLe/sqld_project)의 Kubernetes 배포 설정과 분석 파이프라인 실행 환경을 관리합니다. 애플리케이션 소스와 분리하여 ArgoCD Application, 이미지 갱신 정책, Airflow·dbt 실행 이미지, Redash·모니터링 구성을 보관합니다.

서비스 기능과 dbt 데이터 흐름은 [앱 README](https://github.com/BeolLe/sqld_project#분석-데이터-흐름)에 정리되어 있습니다. 이 저장소에서는 **그 코드가 어떤 이미지와 배포 설정으로 실행되는지** 확인할 수 있습니다.

## 담당 범위와 코드

**서현석 담당:** Kubernetes·GitOps 배포 구성, Airflow·dbt 실행 환경, Redash 및 모니터링 운영 설정.

| 담당 영역 | 구현 내용 | 관련 코드 |
| --- | --- | --- |
| 배포 구성 관리 | App-of-Apps로 개별 Application을 연결하고 앱·배치·분석 환경을 구분 | [루트 Application](infra/argocd/root-apps.yaml) · [관리 대상 목록](infra/argocd/kustomization.yaml) |
| 앱 배포·이미지 갱신 | 프론트엔드·백엔드 이미지 태그를 Git에 기록하고 ArgoCD로 반영 | [프론트엔드](infra/frontend) · [백엔드](infra/backend) · [프론트 이미지 갱신](infra/argocd/front-image-updater.yaml) · [백엔드 이미지 갱신](infra/argocd/back-k8s-updater.yaml) |
| 공개 서비스 구성 | `app-public` 네임스페이스용 워크로드와 Gateway 경로 정의 | [공통 워크로드](infra/apps/sqld/base) · [공개 서비스 오버레이](infra/apps/sqld/overlays/public) |
| Airflow 실행 환경 | Helm·KubernetesExecutor·git-sync 설정과 커스텀 이미지 빌드 | [Application](infra/argocd/argocd-airflow.yaml) · [Helm values](infra/airflow/values.yaml) · [Dockerfile](images/airflow/Dockerfile) · [빌드 워크플로우](.github/workflows/airflow-ghcr.yaml) |
| dbt 실행 이미지 | 모델 코드와 분리된 `dbt-postgres` 실행 이미지를 GHCR에 발행 | [Dockerfile](images/dbt-runner/Dockerfile) · [빌드 워크플로우](.github/workflows/dbt-runner-ghcr.yaml) |
| 분석 결과 조회 | Redash 서버·스케줄러·워커·Redis 배포 구성 | [Redash 리소스](infra/analytics/redash) · [Application](infra/argocd/argocd-analytics.yaml) |
| 모니터링·데이터 보존 | Grafana PVC 연결, 서버 사용량 대시보드의 코드 관리, Prometheus 알림 규칙 | [Helm values](infra/cluster/monitoring/monitoring-values.yaml) · [PV·PVC](infra/cluster/monitoring/storage/grafana-storage.yaml) · [대시보드](infra/cluster/monitoring/dashboards) · [알림 규칙](infra/cluster/monitoring/alerts/prometheus-rule.yaml) |

## 앱 배포 흐름

`앱 코드 변경 → GitHub Actions 이미지 빌드 → GHCR → Image Updater의 Git 갱신 → ArgoCD 동기화 → Kubernetes`

- 앱 이미지는 [앱 저장소의 워크플로우](https://github.com/BeolLe/sqld_project/tree/main/.github/workflows)에서 빌드합니다.
- `front-k8s`·`back-k8s`는 `sha-` 태그 이미지를 대상으로 갱신하며, 각각 [프론트 이미지 정보](infra/frontend/.argocd-source-front-k8s.yaml)와 [백엔드 이미지 정보](infra/backend/.argocd-source-back-k8s.yaml)에 기록합니다. 두 Application의 배포 대상은 `app` 네임스페이스입니다.
- `app-public`용 매니페스트는 별도로 관리합니다. 해당 경로를 동기화하는 Application은 현재 `infra/argocd`의 관리 대상 목록에 포함되어 있지 않으므로, 위 자동 갱신과 공개 서비스 배포를 동일한 것으로 보지 않습니다.
- Airflow 이미지는 [별도 Image Updater](infra/argocd/airflow-image-updater.yaml)가 `latest`의 digest를 감지해 Helm values에 기록합니다.

## 분석 파이프라인 실행 환경

모델·DAG 코드와 실행 이미지는 역할을 나누어 관리합니다.

1. **Airflow DAG 공급:** [Helm values](infra/airflow/values.yaml)의 git-sync가 `airflow-practice` 저장소에서 DAG를 가져옵니다.
2. **dbt 실행:** dbt DAG가 별도 Kubernetes Pod를 만들고, 초기화 컨테이너에서 모델 코드를 가져온 뒤 `ghcr.io/beolle/sqld-dbt-runner` 이미지로 `dbt build`를 실행합니다. 이 저장소의 Dockerfile에는 실행 의존성이 있고, 모델 SQL은 포함하지 않습니다.
3. **결과 활용:** PostgreSQL Gold 지표를 Redash에서 조회하고, 별도 Airflow DAG가 일일 Slack 리포트에 활용합니다. 이 저장소는 Redash 배포를 담당하며, 지표 집계·리포트 로직은 모델·DAG 저장소에 있습니다.

모델·DAG 코드 위치: [dbt 모델](https://github.com/BeolLe/airflow-practice/tree/main/solsqld/analytics/dbt), [dbt 실행 DAG](https://github.com/BeolLe/airflow-practice/blob/main/solsqld/analytics_dbt.py), [일일 리포트](https://github.com/BeolLe/airflow-practice/blob/main/solsqld/slack_daily_report.py). 해당 저장소는 현재 비로그인 상태에서 열리지 않습니다. 공개 코드로는 위 표의 실행 이미지·빌드·배포 설정을 확인할 수 있습니다.

## 운영 범위와 주의점

- **자동 동기화 범위:** ArgoCD Application이 참조하는 경로만 동기화합니다. 저장소에 있는 모든 인프라 파일이 자동 적용되는 것은 아닙니다.
- **모니터링 배포:** `kube-prometheus-stack`의 Helm values와 별도 ArgoCD 관리 리소스를 구분합니다. Grafana 스토리지·대시보드·알림 리소스는 Application으로 관리하지만, 모니터링 Helm 릴리스 설정 변경은 별도 Helm 적용이 필요합니다.
- **Grafana 데이터 보존:** 기존 PVC 연결과 `Retain` 정책의 로컬 PV를 사용합니다. Pod 재생성에 대비한 영속화이며, 노드·디스크 장애에 대비한 백업을 대신하지 않습니다.
- **인증 정보:** Airflow·dbt·Grafana 등의 인증 정보는 배포 환경의 Kubernetes Secret을 참조합니다. README에는 비밀값을 기재하지 않습니다.

그 밖의 클러스터 자산은 [DB](infra/cluster/db), [네트워크](infra/cluster/network), [Gateway](infra/gateway), [Cloudflare](infra/cluster/cloudflare) 디렉터리에 있습니다. 이 저장소는 현재 운영 구성을 관리하며, 새 클러스터를 한 번에 설치하는 도구는 아닙니다.
