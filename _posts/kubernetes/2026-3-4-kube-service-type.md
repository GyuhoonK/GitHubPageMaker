---

layout: post
current: post
cover: assets/built/images/kubernetes/k8s-service.png
navigation: True
title: Kubernetes Service Type
date: 2026-03-04 22:30:00 +0900
tags: [kubernetes]
class: post-template
subclass: 'post tag-kubernetes'
author: GyuhoonK

---

# [K8s] 쿠버네티스 Service(서비스) 완벽 가이드: 개념부터 Ingress 연동까지

파드(Pod)를 띄우는 것만으로는 외부와 통신할 수 없으며, 내부 컴포넌트 간의 안정적인 연결도 보장할 수 없습니다. 외부와 통신을 위해서는 반드시 Serivce가 필요합니다.

이번 포스트에서는 쿠버네티스 Service가 왜 필요한지(배경지식)부터, 각 타입별 특징과 실제 사용 예시, 그리고 L7 라우팅을 위한 Ingress와 수동 Endpoints 연결 같은 응용 방법까지 한 번에 정리해 보겠습니다.

---

## 1. 배경지식: 왜 Service가 필요할까?

쿠버네티스의 기본 배포 단위인 **파드(Pod)는 '휘발성(Ephemeral)'을 가집니다.** 노드에 장애가 생기거나 스케일링이 발생하면 기존 파드는 죽고 새로운 파드가 생성됩니다. 이때 **파드의 IP 주소도 매번 새롭게 할당**됩니다. 

만약 프론트엔드 서버가 백엔드 파드의 IP를 직접 가리키고 있다면, 백엔드 파드가 재시작될 때마다 연결이 끊어지는 대참사가 발생합니다. 



이 문제를 해결하기 위해 등장한 것이 바로 Service입니다. Service는 파드들 앞단에 위치하여 **'변하지 않는 고정된 주소(IP 및 도메인)'**를 제공하는 로드밸런서 역할을 합니다.

---

## 2. 개념 정의: Service와 작동 원리

**Service**는 네트워크 상에서 동일한 역할을 하는 파드 집합을 묶어, 단일 진입점을 제공하는 쿠버네티스 리소스입니다.

* **고정 IP 제공:** 파드가 재생성되어도 Service의 IP(ClusterIP)는 변하지 않습니다.
* **로드 밸런싱:** 들어온 요청을 뒤에 연결된 여러 파드들에게 고르게 분산시킵니다.
* **서비스 디스커버리:** `my-service.default.svc.cluster.local`과 같은 내부 DNS 도메인을 제공하여, IP를 몰라도 이름만으로 통신할 수 있게 돕습니다.

### 어떻게 파드를 찾을까? (Label과 Selector)
Service는 **Label(레이블)**을 기준으로 트래픽을 전달할 파드를 찾습니다. Service YAML에 `selector: app: web`이라고 명시하면, 쿠버네티스는 `app: web` 레이블을 가진 파드들의 IP를 모아 **Endpoints**라는 객체로 관리하며 지속적으로 트래픽을 라우팅합니다.



---

## 3. 실제 사용: Service의 3가지 핵심 타입

Service는 접근 방식에 따라 크게 3가지 기본 타입으로 나뉩니다.



### ① ClusterIP (기본값)
* **특징:** 클러스터 **내부**에서만 통신 가능한 가상 IP를 할당합니다.
* **사용 사례:** 외부 인터넷에 노출되면 안 되는 내부 백엔드 API, 데이터베이스(DB), 캐시 서버 등.

```yaml
# clusterip-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: backend-db-svc
spec:
  type: ClusterIP  # 생략 시 기본값
  selector:
    app: backend-db
  ports:
    - port: 3306        # Service가 노출할 포트
      targetPort: 3306  # 실제 Pod가 바라보는 포트
```

### ② NodePort
* **특징:** 클러스터 내의 **모든 워커 노드(물리/가상 서버)의 특정 포트**(보통 30000~32767)를 열어 외부 접근을 허용합니다.
* **주의점:** 파드의 IP가 아닌, 노드(서버)의 Public/Private IP를 통해 접근합니다. (예: `http://<Node-IP>:31000`)
* **사용 사례:** 테스트 환경, 혹은 온프레미스 환경에서 자체 로드밸런서와 연결할 때.

```yaml
# nodeport-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: web-frontend-svc
spec:
  type: NodePort
  selector:
    app: web-frontend
  ports:
    - port: 80
      targetPort: 8080
      nodePort: 31000   # 외부에서 접근할 노드 포트 지정 (생략 시 자동 할당)
```

### ③ LoadBalancer
* **특징:** AWS, GCP, Azure 등 클라우드 공급자의 **실제 로드밸런서(L4)**를 프로비저닝하여 외부 트래픽을 클러스터로 가져옵니다.
* **사용 사례:** 외부 사용자에게 프로덕션 수준의 웹 서비스를 직접 노출할 때.

```yaml
# loadbalancer-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: public-web-svc
spec:
  type: LoadBalancer
  selector:
    app: public-web
  ports:
    - port: 80
      targetPort: 8080
```

---

## 4. 응용: ExternalName, 수동 Endpoints, 그리고 Ingress

Service를 더 강력하게 활용하는 응용 방법들입니다.

### ① ExternalName (외부 서비스를 내부처럼)
클러스터 외부에 있는 시스템(예: AWS RDS)을 클러스터 내부 파드들이 쉽게 호출하도록 DNS CNAME(별명)을 설정합니다. 파드들은 긴 외부 도메인 대신 `my-rds-svc`라는 짧은 이름으로 DB에 접근할 수 있습니다.

```yaml
# externalname-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: my-rds-svc
spec:
  type: ExternalName
  externalName: my-company-db.xyz.us-east-1.rds.amazonaws.com
```

### ② 수동 Endpoints 연결 (레거시 IP 연동)
클러스터 외부의 특정 IP(예: 사내 레거시 DB `192.168.100.50`)를 Service로 묶고 싶을 때 사용합니다. **Service에서 `selector`를 지우고, Service와 이름이 동일한 Endpoints를 직접 생성**합니다.

```yaml
# 1. Selector가 없는 Service
apiVersion: v1
kind: Service
metadata:
  name: legacy-db-svc  # 이름이 같아야 함
spec:
  ports:
    - port: 3306

---
# 2. 직접 타겟 IP를 지정하는 Endpoints
apiVersion: v1
kind: Endpoints
metadata:
  name: legacy-db-svc  # 이름이 같아야 함
subsets:
  - addresses:
      - ip: 192.168.100.50 # 레거시 DB IP
    ports:
      - port: 3306
```

### ③ Ingress (L7 도메인/경로 기반 라우팅)
LoadBalancer 타입은 서비스마다 하나의 클라우드 로드밸런서를 생성하므로 비용이 많이 듭니다. 또한, `example.com/api`와 `example.com/web`을 다른 서비스로 보내는 스마트한 라우팅이 불가능합니다. 

이를 해결하는 것이 **Ingress**입니다. Ingress는 클러스터 진입점에 위치하여 **HTTP/HTTPS(L7) 수준의 트래픽 라우팅과 SSL 인증서 처리**를 담당합니다. (단, 이를 실행할 `Nginx Ingress Controller` 같은 컨트롤러가 클러스터에 미리 설치되어 있어야 합니다.)



```yaml
# ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-shopping-ingress
spec:
  rules:
  - host: shop.example.com
    http:
      paths:
      - path: /pay
        pathType: Prefix
        backend:
          service:
            name: payment-svc # /pay 경로는 결제 서비스(ClusterIP)로
            port: 
              number: 80
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend-svc # 나머지 경로는 프론트엔드로
            port: 
              number: 80
```

---

## 5. 주의할 점 (Troubleshooting 포인트)

1.  **"파드는 정상인데 접속이 안 돼요!" (Selector 오타):** Service의 `selector` 레이블과 Pod의 `labels`가 정확히 일치하는지 확인하세요. 철자 하나만 틀려도 Endpoints 목록이 비워져 트래픽이 전달되지 않습니다. `kubectl get endpoints <서비스명>`으로 연결된 IP가 있는지 확인하는 것이 1순위입니다.
2.  **NodePort의 보안 취약점:** NodePort는 노드의 포트를 직접 외부에 개방하므로 보안에 취약할 수 있습니다. 프로덕션 환경에서는 클라이언트 트래픽을 직접 NodePort로 받기보다는, 로드밸런서나 Ingress를 거쳐 통신하도록 아키텍처를 구성하는 것이 좋습니다.
3.  **`.svc.cluster.local` 도메인의 한계:** 가끔 외부 애플리케이션에서 `http://my-service.default.svc.cluster.local`로 API를 찔러보고 안 된다고 하는 경우가 있습니다. 이 도메인은 CoreDNS가 관리하는 **클러스터 내부 전용 주소망**이므로 외부에서는 절대 해석(Resolve)할 수 없습니다. 외부 연동이 필요하다면 반드시 Ingress나 LoadBalancer 외부 IP를 사용하세요.