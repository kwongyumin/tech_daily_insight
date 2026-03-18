# Helm Chart로 Kubernetes 애플리케이션 패키징

## 개요

Kubernetes 환경에서 애플리케이션을 배포하다 보면, 수십 개의 YAML 파일을 관리하는 일이 금세 복잡해진다. Deployment, Service, ConfigMap, Secret, Ingress, HPA… 이 모든 리소스를 환경별로(dev/staging/prod) 따로 관리하려면 파일 복사와 수동 수정이 반복되고, 실수가 생기기 마련이다.

**Helm**은 이러한 문제를 해결하기 위한 Kubernetes 패키지 매니저다. Chart라는 단위로 애플리케이션을 패키징하고, 값(Values)을 주입해 환경별 설정을 분리하며, 릴리스 이력 관리와 롤백까지 지원한다. 이 글에서는 실무에서 바로 활용할 수 있는 수준으로 Helm Chart 구조부터 멀티 환경 배포, 그리고 주의사항까지 다룬다.

---

## 핵심 개념

### Chart 구조

Helm Chart는 정해진 디렉터리 구조를 따른다.

```
my-app/
├── Chart.yaml          # 차트 메타데이터
├── values.yaml         # 기본 값 정의
├── values-dev.yaml     # 개발 환경 오버라이드 값
├── values-prod.yaml    # 운영 환경 오버라이드 값
├── templates/
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── ingress.yaml
│   ├── configmap.yaml
│   ├── hpa.yaml
│   ├── _helpers.tpl    # 재사용 가능한 템플릿 함수
│   └── NOTES.txt       # 배포 후 출력 메시지
└── charts/             # 의존 차트 (서브차트)
```

### 핵심 컴포넌트

| 컴포넌트 | 역할 |
|---|---|
| `Chart.yaml` | 차트 이름, 버전, 앱 버전 등 메타데이터 |
| `values.yaml` | 템플릿에 주입될 기본 변수 정의 |
| `templates/` | Go 템플릿 문법으로 작성된 Kubernetes 매니페스트 |
| `_helpers.tpl` | 공통 레이블, 이름 등 재사용 함수 모음 |
| Release | `helm install/upgrade`로 생성되는 배포 단위 |

### Helm 3의 특징

Helm 2에서는 클러스터 내부에 Tiller 서버가 필요했지만, **Helm 3부터는 Tiller가 제거**되어 클라이언트만으로 동작한다. 릴리스 상태는 각 네임스페이스의 Secret에 저장된다.

---

## 실전 예제

Spring Boot 기반의 REST API 서버를 Helm Chart로 패키징하는 전체 흐름을 살펴보자.

### 1. Chart.yaml 작성

```yaml
apiVersion: v2
name: my-spring-app
description: Spring Boot REST API Application
type: application
version: 1.2.0        # 차트 버전 (SemVer)
appVersion: "3.1.5"   # 애플리케이션 버전
maintainers:
  - name: devteam
    email: devteam@example.com
```

### 2. values.yaml 기본값 정의

```yaml
replicaCount: 2

image:
  repository: registry.example.com/my-spring-app
  tag: "latest"
  pullPolicy: IfNotPresent

imagePullSecrets:
  - name: registry-secret

service:
  type: ClusterIP
  port: 80
  targetPort: 8080

ingress:
  enabled: true
  className: nginx
  host: api.example.com
  tls:
    enabled: true
    secretName: api-tls-secret

resources:
  requests:
    cpu: "250m"
    memory: "512Mi"
  limits:
    cpu: "1000m"
    memory: "1Gi"

autoscaling:
  enabled: true
  minReplicas: 2
  maxReplicas: 10
  targetCPUUtilizationPercentage: 70

env:
  SPRING_PROFILES_ACTIVE: "prod"
  SERVER_PORT: "8080"

config:
  datasource:
    url: "jdbc:postgresql://db:5432/mydb"
    username: "appuser"

# Secret 값은 별도 관리 (Vault, SealedSecret 등)
secrets:
  datasource:
    password: ""
```

### 3. _helpers.tpl 작성

```
{{/*
공통 이름 생성
*/}}
{{- define "my-spring-app.name" -}}
{{- default .Chart.Name .Values.nameOverride | trunc 63 | trimSuffix "-" }}
{{- end }}

{{- define "my-spring-app.fullname" -}}
{{- if .Values.fullnameOverride }}
{{- .Values.fullnameOverride | trunc 63 | trimSuffix "-" }}
{{- else }}
{{- $name := default .Chart.Name .Values.nameOverride }}
{{- printf "%s-%s" .Release.Name $name | trunc 63 | trimSuffix "-" }}
{{- end }}
{{- end }}

{{/*
공통 레이블
*/}}
{{- define "my-spring-app.labels" -}}
helm.sh/chart: {{ .Chart.Name }}-{{ .Chart.Version }}
app.kubernetes.io/name: {{ include "my-spring-app.name" . }}
app.kubernetes.io/instance: {{ .Release.Name }}
app.kubernetes.io/version: {{ .Chart.AppVersion | quote }}
app.kubernetes.io/managed-by: {{ .Release.Service }}
{{- end }}

{{/*
Selector 레이블
*/}}
{{- define "my-spring-app.selectorLabels" -}}
app.kubernetes.io/name: {{ include "my-spring-app.name" . }}
app.kubernetes.io/instance: {{ .Release.Name }}
{{- end }}
```

### 4. Deployment 템플릿

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "my-spring-app.fullname" . }}
  labels:
    {{- include "my-spring-app.labels" . | nindent 4 }}
spec:
  {{- if not .Values.autoscaling.enabled }}
  replicas: {{ .Values.replicaCount }}
  {{- end }}
  selector:
    matchLabels:
      {{- include "my-spring-app.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      labels:
        {{- include "my-spring-app.selectorLabels" . | nindent 8 }}
    spec:
      {{- with .Values.imagePullSecrets }}
      imagePullSecrets:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          ports:
            - name: http
              containerPort: {{ .Values.service.targetPort }}
              protocol: TCP
          envFrom:
            - configMapRef:
                name: {{ include "my-spring-app.fullname" . }}-config
            - secretRef:
                name: {{ include "my-spring-app.fullname" . }}-secret
          livenessProbe:
            httpGet:
              path: /actuator/health/liveness
              port: http
            initialDelaySeconds: 30
            periodSeconds: 10
          readinessProbe:
            httpGet:
              path: /actuator/health/readiness
              port: http
            initialDelaySeconds: 20
            periodSeconds: 5
          resources:
            {{- toYaml .Values.resources | nindent 12 }}
```

### 5. ConfigMap 및 Secret 템플릿

```yaml
# templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ include "my-spring-app.fullname" . }}-config
  labels:
    {{- include "my-spring-app.labels" . | nindent 4 }}
data:
  {{- range $key, $value := .Values.env }}
  {{ $key }}: {{ $value | quote }}
  {{- end }}
  SPRING_DATASOURCE_URL: {{ .Values.config.datasource.url | quote }}
  SPRING_DATASOURCE_USERNAME: {{ .Values.config.datasource.username | quote }}
```

```yaml
# templates/secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: {{ include "my-spring-app.fullname" . }}-secret
  labels:
    {{- include "my-spring-app.labels" . | nindent 4 }}
type: Opaque
data:
  SPRING_DATASOURCE_PASSWORD: {{ .Values.secrets.datasource.password | b64enc | quote }}
```

### 6. 환경별 values 파일

```yaml
# values-dev.yaml
replicaCount: 1

image:
  tag: "develop-latest"

ingress:
  host: api-dev.example.com
  tls:
    enabled: false

resources:
  requests:
    cpu: "100m"
    memory: "256Mi"
  limits:
    cpu: "500m"
    memory: "512Mi"

autoscaling:
  enabled: false

env:
  SPRING_PROFILES_ACTIVE: "dev"

config:
  datasource:
    url: "jdbc:postgresql://dev-db:5432/mydb_dev"
```

### 7. 배포 명령어

```bash
# 차트 문법 검사
helm lint ./my-spring-app

# 렌더링 결과 미리보기 (클러스터 연결 불필요)
helm template my-app ./my-spring-app -f values-prod.yaml

# 개발 환경 배포
helm upgrade --install my-app-dev ./my-spring-app \
  --namespace dev \
  --create-namespace \
  -f values.yaml \
  -f values-dev.yaml \
  --set secrets.datasource.password="$(kubectl get secret db-secret -o jsonpath='{.data.password}' | base64 -d)"

# 운영 환경 배포
helm upgrade --install my-app-prod ./my-spring-app \
  --namespace production \
  -f values.yaml \
  -f values-prod.yaml \
  --atomic \        # 실패 시 자동 롤백
  --timeout 5m

# 릴리스 이력 확인
helm history my-app-prod -n production

# 롤백
helm rollback my-app-prod 2 -n production
```

### 8. Chart 의존성 관리

외부 차트(예: Redis, PostgreSQL)를 의존성으로 추가할 수 있다.

```yaml
# Chart.yaml에 추가
dependencies:
  - name: redis
    version: "18.x.x"
    repository: "https://charts.bitnami.com/bitnami"
    condition: redis.enabled
```

```bash
# 의존성 다운로드
helm dependency update ./my-spring-app
```

---

## 주의사항 및 트레이드오프

### Secret 관리는 Helm 밖에서

`values.yaml`에 시크릿을 평문으로 넣거나, `--set`으로 전달하는 방식은 히스토리에 노출될 위험이 있다. 실무에서는 다음 중 하나를 선택하는 것이 좋다.

- **Sealed Secrets**: 퍼블릭 키로 암호화 후 Git 커밋 가능
- **External Secrets Operator**: AWS Secrets Manager, Vault 등 외부 저장소와 연동
- **Helm Secrets 플러그인**: `helm-secrets` + SOPS를 이용한 암호화 값 관리

### Chart 버전과 App 버전 혼동 주의

`Chart.yaml`의 `version`은 차트 자체의 버전이고, `appVersion`은 컨테이너 이미지 버전이다. 이 둘을 혼동하면 릴리스 추적이 어려워진다. CI/CD 파이프라인에서 이미지 빌드 시 `appVersion`을 자동으로 갱신하도록 스크립트를 구성하는 것이 좋다.

```bash
# CI에서 appVersion 자동 업데이트
sed -i "s/^appVersion:.*/appVersion: \"${GIT_TAG}\"/" Chart.yaml
```

### `helm upgrade`의 3-way Merge 동작 이해

Helm 3는 이전 릴리스 매니페스트, 현재 클러스터 상태, 새 매니페스트 세 가지를 비교하는 3-way merge 방식을 사용한다. 클러스터에서 수동으로 리소스를 수정했다면 예상치 못한 동작이 발생할 수 있으므로, **Helm으로 배포한 리소스는 반드시 Helm을 통해서만 수정**해야 한다.

### 너무 많은 if/else는 독이 된다

Go 템플릿의 조건문을 과도하게 사용하면 차트가 읽기 어려워진다. 복잡성이 높아진다면 **서브차트 분리** 또는 **Kustomize 병행 사용**을 고려해야 한다. Helm과 Kustomize는 상호 배타적이지 않으며, `helm template` 출력을 Kustomize로 후처리하는 패턴도 실무에서 자주 사용된다.

### OCI 레지스트리 활용

Helm 3.8부터 Chart를 OCI(Docker) 레지스트리에 저장할 수 있다. 기존 ChartMuseum 대신 AWS ECR, GCP Artifact Registry 등을 그대로 활용할 수 있어 인프라 단순화에 유리하다.

```bash
# OCI 레지스트리에 push
helm push my-spring-app-1.2.0.tgz oci://registry.example.com/helm-charts

# OCI 레지스트리에서 설치
helm install my-app oci://registry.example.com/helm-charts/my-spring-app --version 1.2.0
```

---

## 정리

Helm Chart는 단순한 YAML 템플릿 도구를 넘어, Kubernetes 애플리케이션의 **패키징, 배포, 버전 관리** 전반을 다루는 실질적인 인프라 표준이 되었다. 핵심을 정리하면 다음과 같다.

1. **구조화된 디렉터리와 `_helpers.tpl`**로 재사용성과 일관성을 확보한다.
2. **환경별 values 파일**로 단일 차트에서 멀티 환경 배포를 지원한다.
3. **Secret은 반드시 Helm 외부(Sealed Secrets, External Secrets 등)로** 관리한다.
4. `helm template`과 `helm lint`를 CI 파이프라인에 포함해 **배포 전 검증**을 자동화한다.
5. 차트 버전 관리는 **SemVer를 엄격히 준수**하고, appVersion과 명확히 구분한다.

처음에는 `helm create` 명령으로 생성된 보일러플레이트에서 시작해 점진적으로 커스터마이징하는 방식을 권장한다. 복잡한 멀티 서비스 환경이라면 **Helmfile**을 함께 도입해 여러 Chart의 배포 순서와 의존성을 선언적으로 관리하는 것도 좋은 다음 단계다.