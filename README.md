# Kubernetes 클러스터 애플리케이션 배포 및 모니터링 시스템 구축

<!-- Stack badges -->
<p align="left">
  <a href="https://kubernetes.io/">
    <img src="https://img.shields.io/badge/Kubernetes-v1.30-blue.svg?logo=kubernetes&logoColor=white" />
  </a>
  <a href="https://ubuntu.com/">
    <img src="https://img.shields.io/badge/Ubuntu-24.04%20LTS-E95420.svg?logo=ubuntu&logoColor=white" />
  </a>
  <a href="https://projectcalico.org/">
    <img src="https://img.shields.io/badge/CNI-Calico-00A3E0.svg?logo=linux&logoColor=white" />
  </a>
  <a href="https://containerd.io/">
    <img src="https://img.shields.io/badge/Runtime-containerd-575757.svg?logo=containerd&logoColor=white" />
  </a>
  <a href="https://prometheus.io/">
    <img src="https://img.shields.io/badge/Monitoring-Prometheus-E6522C.svg?logo=prometheus&logoColor=white" />
  </a>
  <a href="https://grafana.com/">
    <img src="https://img.shields.io/badge/Dashboard-Grafana-F46800.svg?logo=grafana&logoColor=white" />
  </a>
  <img src="https://img.shields.io/badge/Exporter-Node%20Exporter-00C853.svg" />
  <img src="https://img.shields.io/badge/Exporter-MySQL%20Exporter-4479A1.svg?logo=mysql&logoColor=white" />
  <a href="https://spring.io/projects/spring-boot">
    <img src="https://img.shields.io/badge/Spring%20Boot-3.5.6-6DB33F.svg?logo=springboot&logoColor=white" />
  </a>
  <img src="https://img.shields.io/badge/Metrics-Micrometer-2E5E82.svg" />
  <a href="https://www.java.com/">
    <img src="https://img.shields.io/badge/Java-17-007396.svg?logo=openjdk&logoColor=white" />
  </a>
  <a href="https://helm.sh/">
    <img src="https://img.shields.io/badge/Helm-3-277A9F.svg?logo=helm&logoColor=white" />
  </a>
  <a href="https://www.docker.com/">
    <img src="https://img.shields.io/badge/Docker-28.4.0-2496ED.svg?logo=docker&logoColor=white" />
  </a>
</p>


본 가이드는 기존에 구축된 Kubernetes 클러스터를 기반으로 Spring Boot 애플리케이션의 컨테이너화부터 Prometheus/Grafana를 활용한 종합적인 모니터링 시스템 구축까지의 전체 DevOps 파이프라인을 다룹니다. 실무에서 활용 가능한 Docker Hub 연동, 헬름 차트 관리, 그리고 운영 모니터링 구현을 포함합니다.

## 📋 목차

- [프로젝트 개요](#프로젝트-개요)
- [기술 스택](#기술-스택)
- [애플리케이션 컨테이너화](#애플리케이션-컨테이너화)
- [Helm 패키지 매니저 구성](#helm-패키지-매니저-구성)
- [Spring Boot 애플리케이션 배포](#spring-boot-애플리케이션-배포)
- [모니터링 시스템 구축](#모니터링-시스템-구축)
- [서비스 접근 및 네트워킹](#서비스-접근-및-네트워킹)
- [트러블슈팅 가이드](#트러블슈팅-가이드)
- [운영 모범 사례](#운영-모범-사례)

## 🎯 프로젝트 개요

### 아키텍처 구성도

```
┌─────────────────────────────────────────────────────────────────┐
│                    Kubernetes 클러스터                          │
├─────────────────────────────────────────────────────────────────┤
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐ │
│  │   myserver01    │  │   myserver02    │  │   myserver03    │ │
│  │  (Master Node)  │  │ (Worker Node 1) │  │ (Worker Node 2) │ │
│  │   10.0.2.15     │  │   10.0.2.20     │  │   10.0.2.25     │ │
│  └─────────────────┘  └─────────────────┘  └─────────────────┘ │
├─────────────────────────────────────────────────────────────────┤
│                        애플리케이션 레이어                      │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐ │
│  │  Spring Boot    │  │   Prometheus    │  │    Grafana      │ │
│  │   Flash Game    │  │   Monitoring    │  │   Dashboard     │ │
│  │    :8080        │  │     :9090       │  │     :30030      │ │
│  └─────────────────┘  └─────────────────┘  └─────────────────┘ │
├─────────────────────────────────────────────────────────────────┤
│                        서비스 타입                              │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐ │
│  │   ClusterIP     │  │    NodePort     │  │    Ingress      │ │
│  │ (내부 통신 전용) │  │ (외부 접근용)   │  │ (도메인 라우팅) │ │
│  └─────────────────┘  └─────────────────┘  └─────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
```

### 구현 목표

1. **컨테이너화**: Spring Boot 애플리케이션을 Docker 이미지로 빌드하고 Docker Hub에 배포
2. **쿠버네티스 배포**: Deployment와 Service를 통한 애플리케이션 배포 및 노출
3. **패키지 관리**: Helm을 활용한 쿠버네티스 리소스 템플릿화
4. **모니터링 시스템**: Prometheus와 Grafana를 통한 실시간 메트릭 수집 및 시각화
5. **네트워크 구성**: 다양한 서비스 타입을 활용한 트래픽 라우팅

## 🛠 기술 스택

### 컨테이너 및 오케스트레이션
- **Docker**: 애플리케이션 컨테이너화
- **Kubernetes v1.30**: 컨테이너 오케스트레이션
- **containerd**: 컨테이너 런타임

### 애플리케이션 스택
- **Spring Boot**: Java 기반 웹 애플리케이션 프레임워크
- **Spring Actuator**: 애플리케이션 메트릭 및 헬스체크
- **Micrometer Prometheus**: 메트릭 수집 및 노출

### 모니터링 및 시각화
- **Prometheus**: 메트릭 수집 및 저장
- **Grafana**: 시각화 및 대시보드
- **kube-prometheus-stack**: 쿠버네티스 통합 모니터링 솔루션

### 패키지 관리 및 배포
- **Helm v3**: 쿠버네티스 패키지 매니저
- **Docker Hub**: 컨테이너 이미지 레지스트리

## 📦 애플리케이션 컨테이너화

### Spring Boot 애플리케이션 준비

먼저 Spring Boot 애플리케이션에 모니터링을 위한 의존성을 추가합니다.

```xml
<!-- pom.xml에 추가 -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-registry-prometheus</artifactId>
</dependency>
```

### application.yml 설정

```yaml
spring:
  application:
    name: SpringApp

server:
  port: 8080

management:
  endpoints:
    web:
      exposure:
        include: "prometheus,health,metrics"
  prometheus:
    metrics:
      export:
        enabled: true
```

> **Spring Actuator란?** Spring Boot에서 제공하는 프로덕션 준비 기능으로, 애플리케이션의 상태 모니터링, 메트릭 수집, HTTP 추적 등을 제공합니다. `/actuator/health`, `/actuator/prometheus` 등의 엔드포인트를 통해 애플리케이션 정보에 접근할 수 있습니다.

### Dockerfile 작성

보안을 강화한 멀티 스테이지 Dockerfile을 작성합니다.

```dockerfile
# Java 17 JRE 기반(Alpine)
FROM eclipse-temurin:17-jre-alpine

# 비루트 사용자 생성(Alpine busybox 방식)
RUN addgroup -S app && adduser -S app -G app
USER app

WORKDIR /app
COPY SpringAppSampleGradle1-0.0.1-SNAPSHOT.jar app.jar

# 헬스체크는 Actuator health 기준으로 예시
HEALTHCHECK --interval=30s --timeout=5s --retries=3 CMD wget -qO- http://localhost:8080/actuator/health || exit 1

EXPOSE 8080
ENTRYPOINT ["java","-jar","/app/app.jar"]
```

### Docker 이미지 빌드 및 배포

```bash
# 이미지 빌드
docker build -t flashgame:latest .

# Docker Hub에 태그 및 푸시
docker tag flashgame:latest byeonggill/flashgame:latest
docker login
docker push byeonggill/flashgame:latest

# 로컬 테스트
docker run -d -p 8080:8080 --name flashgame-container flashgame:latest

# 헬스체크 확인
curl http://localhost:8080/actuator/health
curl http://localhost:8080/actuator/prometheus
```

![Docker 이미지 빌드 과정 스크린샷]

## ⚙️ Helm 패키지 매니저 구성

### Helm 설치 및 초기 설정

```bash
# Helm 설치
curl -fsSL https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# 버전 확인
helm version

# 저장소 추가
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
```

> **Helm이란?** Kubernetes의 패키지 매니저로, 복잡한 애플리케이션을 차트(Chart)라는 패키지 형태로 관리할 수 있습니다. 템플릿을 통해 쿠버네티스 리소스를 재사용 가능하고 버전 관리가 가능한 형태로 배포할 수 있습니다.

### 커스텀 Helm 차트 생성

NGINX 애플리케이션을 위한 Helm 차트를 생성하여 템플릿 구조를 이해해보겠습니다.

```bash
# 차트 생성
helm create mychart

# 차트 구조 확인
tree mychart/
```

### values.yaml 커스터마이징

애플리케이션별 설정을 values.yaml에 정의합니다:

```yaml
# 레플리카 수 설정
replicaCount: 1

# 컨테이너 이미지 설정
image:
  repository: byeonggill/nginx-image
  pullPolicy: IfNotPresent
  tag: v2.2

# 서비스 설정
service:
  type: ClusterIP
  port: 80

# 인그레스 설정
ingress:
  enabled: true
  className: "nginx"
  hosts:
    - host: myapp.fisa.com
      paths:
        - path: /
          pathType: Prefix

# 리소스 제한
resources:
  limits:
    cpu: 500m
    memory: 512Mi
  requests:
    cpu: 100m
    memory: 256Mi

# 헬스체크 설정
livenessProbe:
  httpGet:
    path: /
    port: http
  initialDelaySeconds: 20
  periodSeconds: 10

readinessProbe:
  httpGet:
    path: /
    port: http
  initialDelaySeconds: 20
  periodSeconds: 10
```

## 🚀 Spring Boot 애플리케이션 배포

### Deployment 및 Service 매니페스트

```yaml
# app-deployment-service.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: spring-boot-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: spring-boot
  template:
    metadata:
      labels:
        app: spring-boot
    spec:
      containers:
        - name: flashgame
          image: byeonggill/flashgame:latest
          ports:
            - containerPort: 8080 
          livenessProbe:
            httpGet:
              path: /actuator/health
              port: 8080
            initialDelaySeconds: 30
            periodSeconds: 10
      imagePullSecrets:
        - name: dockerhub-secret
---
apiVersion: v1
kind: Service
metadata:
  name: spring-boot-service
spec:
  selector:
    app: spring-boot
  ports:
    - protocol: TCP
      port: 8080
      targetPort: 8080
  type: ClusterIP
```

> **Deployment vs Service**: Deployment는 파드의 생성과 업데이트를 관리하며, Service는 파드에 대한 네트워크 접근을 제공합니다. Service는 파드가 재시작되어 IP가 변경되어도 일관된 엔드포인트를 제공합니다.

### Docker Hub 인증 설정

프라이빗 레지스트리 접근을 위한 시크릿을 생성합니다.

```bash
# Docker Hub 인증 시크릿 생성
kubectl create secret docker-registry dockerhub-secret \
  --docker-server=https://index.docker.io/v1/ \
  --docker-username=byeonggill \
  --docker-password=<your-password> \
  --docker-email=qudrlf0104@naver.com

# 애플리케이션 배포
kubectl apply -f app-deployment-service.yaml

# 배포 상태 확인
kubectl get all
kubectl get pods -o wide
```

![애플리케이션 배포 상태 확인 스크린샷]

### ImagePullBackOff 문제 해결

Docker Hub Rate Limit으로 인한 이미지 풀 실패 시 해결 방법:

```bash
# 파드 상세 정보 확인
kubectl describe pod <pod-name>

# 이벤트 확인
kubectl get events --sort-by=.metadata.creationTimestamp

# 시크릿이 올바르게 적용되었는지 확인
kubectl get secrets
kubectl describe secret dockerhub-secret
```

## 📊 모니터링 시스템 구축

### kube-prometheus-stack 설치

Prometheus와 Grafana를 포함한 통합 모니터링 스택을 설치합니다.

```bash
# 모니터링 네임스페이스 생성
kubectl create namespace monitoring

# Prometheus Helm 저장소 추가
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
```

### 모니터링 설정 (values.yaml)

Spring Boot 애플리케이션을 스크레이핑하도록 Prometheus를 설정합니다.

```yaml
# values.yaml
grafana:
  adminUser: admin
  adminPassword: adminpw
  service:
    type: NodePort
    nodePort: 30030
  ingress:
    enabled: false
  resources:
    requests:
      cpu: 100m
      memory: 256Mi
    limits:
      cpu: 500m
      memory: 512Mi

prometheus:
  prometheusSpec:
    additionalScrapeConfigs:
      - job_name: 'spring-app'
        metrics_path: '/actuator/prometheus'
        static_configs:
          - targets: ['spring-boot-service.default.svc.cluster.local:8080']
  resources:
    requests:
      cpu: 100m
      memory: 256Mi
    limits:
      cpu: 500m
      memory: 512Mi
```

> **Prometheus 스크레이프 설정**: `additionalScrapeConfigs`를 통해 커스텀 애플리케이션의 메트릭을 수집할 수 있습니다. Spring Boot Actuator의 `/actuator/prometheus` 엔드포인트에서 Micrometer가 수집한 메트릭을 Prometheus 형식으로 노출합니다.

### 모니터링 스택 배포

```bash
# kube-prometheus-stack 설치
helm install prometheus prometheus-community/kube-prometheus-stack \
  -n monitoring \
  -f values.yaml

# 설치 확인
helm list -n monitoring
kubectl get all -n monitoring
```

### 모니터링 서비스 확인

```bash
# 모니터링 서비스 목록 확인
kubectl get svc -n monitoring

# Prometheus 대상 확인
kubectl -n monitoring port-forward svc/prometheus-kube-prometheus-prometheus 9090:9090

# Grafana 접속 (NodePort 사용)
# http://<node-ip>:30030 (admin/adminpw)
```

![모니터링 서비스 배포 상태 스크린샷]

## 🌐 서비스 접근 및 네트워킹

### 서비스 타입별 특징

| 서비스 타입 | 용도 | 접근 방법 | 사용 사례 |
|-------------|------|-----------|-----------|
| ClusterIP | 내부 통신 | 클러스터 내부에서만 접근 | 마이크로서비스 간 통신 |
| NodePort | 외부 노출 | `<NodeIP>:<NodePort>` | 개발/테스트 환경 |
| LoadBalancer | 외부 노출 | 클라우드 LB 통해 접근 | 프로덕션 환경 |
| Ingress | HTTP 라우팅 | 도메인/경로 기반 라우팅 | 웹 애플리케이션 |

### 포트 포워딩을 통한 로컬 접근

개발 및 테스트 목적으로 포트 포워딩을 사용합니다.

```bash
# Spring Boot 애플리케이션 접근
kubectl port-forward svc/spring-boot-service 8080:8080

# 애플리케이션 확인
curl http://localhost:8080/actuator/health
curl http://localhost:8080/actuator/prometheus

# Prometheus 접근
kubectl -n monitoring port-forward svc/prometheus-kube-prometheus-prometheus 9090:9090

# Grafana 접근 (NodePort가 아닌 경우)
kubectl -n monitoring port-forward svc/prometheus-grafana 3000:80
```

### NodePort를 통한 외부 노출

```yaml
# nginx3-service.yaml 예시
apiVersion: v1
kind: Service
metadata:
  name: nginx3
  namespace: default
spec:
  type: NodePort
  selector:
    app: nginx3
  ports:
    - protocol: TCP
      port: 8083
      targetPort: 80
      nodePort: 30083   # 30000-32767 범위
```

> **NodePort 주의사항**: NodePort는 모든 노드의 동일한 포트를 사용하므로, 포트 충돌을 방지하기 위해 30000-32767 범위에서만 할당이 가능합니다.

### Ingress를 통한 도메인 기반 라우팅

```yaml
# nginx3-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nginx3-ingress
  namespace: default
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  - host: myapp.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: nginx3
            port:
              number: 8083
```

## 🔍 트러블슈팅 가이드

### 일반적인 배포 문제

#### 1. 파드가 ContainerCreating 상태에서 멈춤

**원인 분석:**
```bash
kubectl describe pod <pod-name>
kubectl get events --sort-by=.metadata.creationTimestamp
```

**일반적인 원인:**
- 이미지 풀 실패 (ImagePullBackOff)
- 볼륨 마운트 실패
- 리소스 부족
- 노드 선택자 불일치

#### 2. 서비스 접근 불가

**네트워크 연결성 테스트:**
```bash
# 파드 간 네트워크 테스트
kubectl run test-pod --image=busybox --rm -it -- ping <service-name>

# DNS 해결 테스트
kubectl run test-pod --image=busybox --rm -it -- nslookup <service-name>

# 서비스 엔드포인트 확인
kubectl get endpoints <service-name>
```

#### 3. Prometheus 타깃 Down 상태

**진단 단계:**
```bash
# Prometheus 설정 확인
kubectl -n monitoring get prometheus -o yaml

# 서비스 디스커버리 확인
kubectl get svc <target-service>

# 네트워크 정책 확인
kubectl get networkpolicies --all-namespaces
```

### 모니터링 관련 문제

#### Grafana 데이터 소스 연결 실패

**해결 방법:**
1. Grafana에서 데이터 소스 URL 확인: `http://prometheus-kube-prometheus-prometheus.monitoring.svc.cluster.local:9090`
2. 네트워크 연결성 테스트
3. Prometheus 서비스 상태 확인

#### 메트릭 수집 실패

**확인 사항:**
1. Actuator 엔드포인트 접근성: `curl http://<pod-ip>:8080/actuator/prometheus`
2. 프로메테우스 스크레이프 설정
3. 서비스 레이블 셀렉터 일치성

## 💡 운영 모범 사례

### 보안 강화

#### 1. 컨테이너 보안
```dockerfile
# 비루트 사용자 사용
RUN addgroup -S app && adduser -S app -G app
USER app

# 읽기 전용 파일시스템
securityContext:
  readOnlyRootFilesystem: true
  runAsNonRoot: true
  runAsUser: 1000
```

#### 2. 시크릿 관리
```bash
# 시크릿 암호화 확인
kubectl get secrets -o yaml | grep -v "data:"

# RBAC 설정
kubectl auth can-i get pods --as=system:serviceaccount:default:my-serviceaccount
```

### 리소스 관리

#### 1. 리소스 제한 설정
```yaml
resources:
  requests:
    cpu: 100m
    memory: 256Mi
  limits:
    cpu: 500m
    memory: 512Mi
```

#### 2. 수평적 파드 자동 스케일링 (HPA)
```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: spring-boot-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: spring-boot-app
  minReplicas: 1
  maxReplicas: 5
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
```

### 모니터링 최적화

#### 1. 알림 규칙 설정
```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: spring-boot-alerts
spec:
  groups:
  - name: spring-boot
    rules:
    - alert: SpringBootDown
      expr: up{job="spring-app"} == 0
      for: 1m
      labels:
        severity: critical
      annotations:
        summary: "Spring Boot application is down"
```

#### 2. 대시보드 템플릿 활용
- **Node Exporter Full (ID: 1860)**: 시스템 리소스 모니터링
- **Spring Boot Micrometer (ID: 10231)**: 애플리케이션 메트릭
- **Kubernetes Cluster (ID: 7249)**: 클러스터 상태

## 📈 성능 최적화

### 애플리케이션 최적화

#### JVM 튜닝
```yaml
env:
- name: JAVA_OPTS
  value: "-Xms512m -Xmx1024m -XX:+UseG1GC"
```

#### Spring Boot 설정 최적화
```yaml
spring:
  datasource:
    hikari:
      maximum-pool-size: 20
      minimum-idle: 5
  jpa:
    hibernate:
      ddl-auto: validate
    show-sql: false
```

### 클러스터 최적화

#### 노드 어피니티 설정
```yaml
affinity:
  nodeAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      nodeSelectorTerms:
      - matchExpressions:
        - key: node-type
          operator: In
          values:
          - compute-optimized
```

#### Pod Disruption Budget 설정
```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: spring-boot-pdb
spec:
  minAvailable: 1
  selector:
    matchLabels:
      app: spring-boot
```

## 🎉 결론

본 프로젝트를 통해 다음과 같은 성과를 달성했습니다:

### 주요 성과
1. **완전한 CI/CD 파이프라인**: 소스코드부터 프로덕션 배포까지
2. **컨테이너 오케스트레이션**: Kubernetes를 활용한 애플리케이션 관리
3. **통합 모니터링**: Prometheus/Grafana 기반 실시간 메트릭 수집
4. **인프라스트럭처 as Code**: Helm을 통한 선언적 배포 관리
5. **운영 자동화**: 헬스체크, 자동 재시작, 스케일링

### 학습 포인트
- **Docker 멀티스테이지 빌드**를 통한 최적화된 이미지 생성
- **Kubernetes 리소스 관리** (Deployment, Service, Ingress, Secret)
- **Helm 차트**를 통한 애플리케이션 패키징
- **Prometheus 메트릭 수집** 및 **Grafana 시각화**
- **다양한 서비스 타입**을 활용한 네트워크 구성
- **트러블슈팅** 및 **성능 최적화** 실무 경험

### 향후 개선 방안
1. **GitOps 도입**: ArgoCD를 통한 지속적 배포
2. **보안 강화**: Pod Security Standards, Network Policies 적용
3. **고가용성**: 멀티 마스터 노드 구성
4. **백업/복구**: etcd 백업 및 재해복구 계획
5. **로깅**: ELK Stack을 통한 중앙화된 로그 관리

## 📚 참고 자료

- [Kubernetes 공식 문서](https://kubernetes.io/docs/)
- [Prometheus 모니터링 가이드](https://prometheus.io/docs/)
- [Helm 차트 개발 가이드](https://helm.sh/docs/)
- [Spring Boot Actuator 문서](https://docs.spring.io/spring-boot/docs/current/reference/html/actuator.html)
- [Docker 베스트 프랙티스](https://docs.docker.com/develop/dev-best-practices/)

---

**이전 Repository**: [k8s-cluster-setting](https://github.com/Gill010147/k8s-cluster-setting)

**Date**: 2025-09-26  
