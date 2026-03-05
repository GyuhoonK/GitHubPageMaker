---

layout: post
current: post
cover: assets/built/images/kubernetes/k8s-secret.jpg
navigation: True
title: ConfigMap과 Secret의 차이점
date: 2025-03-15 22:30:00 +0900
tags: [kubernetes]
class: post-template
subclass: 'post tag-kubernetes'
author: GyuhoonK

---

# Kubernetes: ConfigMap과 Secret의 차이점

Kubernetes에서 애플리케이션을 배포할 때, 환경 설정을 컨테이너 이미지와 분리하기 위해 사용하는 두 가지 주요 리소스가 바로 **ConfigMap**과 **Secret**입니다. 이 글에서는 두 리소스의 목적과 기술적 차이점을 명확히 비교해 봅니다.

---

## 💡 면접: ConfigMap과 Secret의 차이는 무엇인가요?

**"ConfigMap은 일반적인 설정 정보를, Secret은 민감한 보안 정보를 저장하는 데 사용됩니다."**

면접에서 이 질문을 받는다면 다음 세 가지 핵심 포인트를 대답하는 것이 좋습니다.

1. **목적의 차이**: ConfigMap은 애플리케이션의 포트, 로그 레벨 등 **일반적인 설정(Configuration)**을 저장하여 컨테이너 이미지를 재사용할 수 있게 돕고, Secret은 데이터베이스 비밀번호, API 토큰, TLS 인증서 등 노출되면 안 되는 **민감한 정보(Credential)**를 다룹니다.
2. **저장 및 조회 방식 (Base64)**: ConfigMap은 평문(Plain text)으로 데이터를 저장하지만, Secret은 데이터를 **Base64로 인코딩**하여 저장합니다. (단, Base64는 암호화가 아닌 단순 인코딩 방식임을 명시해야 합니다.)
3. **보안 매커니즘 (tmpfs 및 etcd 암호화)**: Secret을 파드(Pod)에 볼륨으로 마운트할 때는 디스크가 아닌 메모리 기반의 **tmpfs**를 사용하여 노드에 데이터가 남지 않도록 합니다. 또한, 클러스터의 데이터 저장소인 **etcd에 저장될 때 암호화(Encryption at Rest)**를 적용할 수 있는 것은 Secret뿐입니다.

---

## ⚙️ 비즈니스 로직과 기술적 관점에서의 본질적 차이

이 두 리소스는 쿠버네티스 아키텍처 내에서 완전히 다른 **비즈니스 요구사항**과 **기술적 특성**을 가집니다.

### 비즈니스 로직 (사용성) 관점
* **ConfigMap (유연성 및 이식성)**: 개발, 스테이징, 운영 환경에 따라 달라져야 하는 '비즈니스 환경 변수'를 코드로 분리하는 것이 주 목적입니다. 소스 코드를 수정하지 않고도 환경에 맞는 동작을 제어할 수 있게 해줍니다.
* **Secret (보안 및 규정 준수)**: 시스템이 외부 서비스와 통신하거나 인증을 수행할 때 필요한 '권한'을 안전하게 관리하는 것이 목적입니다. 보안 감사나 컴플라이언스(Compliance) 요구사항을 충족하기 위해 데이터 접근 통제 및 암호화 대상이 됩니다.

### 기술적 관점
가장 큰 기술적 차이는 **볼륨 마운트 방식**에 있습니다.
* 파드에 ConfigMap을 볼륨으로 마운트하면 노드의 로컬 디스크를 거칠 수 있습니다.
* 반면, Secret을 파드에 볼륨으로 마운트하면 쿠버네티스는 자동으로 `tmpfs`(RAM 기반 파일 시스템)를 사용합니다. 파드가 삭제되거나 노드가 재시작되면 데이터가 물리적 디스크에 전혀 남지 않고 즉시 휘발되므로 훨씬 안전합니다.

---

## 💻 `kubectl`에서의 차이점

운영자가 `kubectl` 명령어를 통해 리소스를 조회할 때 그 차이가 명확하게 드러납니다.

### ConfigMap 조회 (`kubectl get cm <이름> -o yaml`)
데이터가 평문(`Plain Text`)으로 그대로 노출됩니다.
```yaml
data:
  database_url: "mysql://localhost:3306/mydb"
  log_level: "INFO"
```

### Secret 조회 (`kubectl get secret <이름> -o yaml`)
데이터가 `Base64` 문자열로 인코딩되어 출력됩니다. 값을 확인하려면 디코딩 과정을 거쳐야 합니다.
```yaml
data:
  db_password: "c3VwZXJfc2VjcmV0X3Bhc3N3b3Jk" # "super_secret_password"의 base64 형태
```

> **주의:** Base64는 디코딩 툴만 있으면 누구나 평문으로 변환할 수 있습니다. 즉, `kubectl`에서 문자열이 당장 읽히지 않게 눈가림(Obfuscation)을 할 뿐, **보안적인 암호화(Encryption)가 아닙니다.**

---

## 🛡️ Secret은 민감 정보에 사용된다. 왜?

Base64 인코딩이 강력한 보안을 보장하지 않음에도 불구하고, 민감 정보를 ConfigMap이 아닌 Secret으로 관리해야 하는 진짜 이유는 쿠버네티스가 Secret에만 적용하는 **다중 보안 레이어** 때문입니다.

1. **메모리(tmpfs) 격리**: 앞서 언급했듯, 볼륨 마운트 시 노드의 물리 디스크에 기록되지 않아 디스크 탈취 공격으로부터 안전합니다.
2. **메모리 최적화 전달**: Secret을 참조하는 파드가 실행되는 워커 노드(Worker Node)의 Kubelet에게만 해당 Secret 데이터가 전송됩니다. 불필요한 노드에는 정보가 공유되지 않습니다.
3. **엄격한 RBAC 통제**: 쿠버네티스 관리자는 역할 기반 접근 제어(RBAC)를 통해 ConfigMap에 대한 읽기 권한은 개발자에게 열어두더라도, Secret에 대한 접근 권한은 철저하게 분리하고 제한할 수 있습니다.

---

## 🔒 etcd 암호화 (Encryption at Rest)

가장 강력한 기술적 차이점 중 하나는 쿠버네티스의 상태 데이터를 저장하는 **etcd에서의 암호화 지원 여부**입니다.

기본적으로 ConfigMap과 Secret 모두 etcd 내부에 평문으로 저장됩니다. 만약 해커가 etcd 데이터베이스 자체에 접근할 수 있다면 Secret의 Base64 데이터도 쉽게 탈취당합니다.

이를 방지하기 위해 쿠버네티스는 **저장 데이터 암호화(Encryption at Rest)** 기능을 제공합니다.
* 클러스터 관리자가 API 서버에 암호화 설정(`EncryptionConfiguration`)을 적용하면, **Secret 리소스에 한해서만** etcd에 쓰여질 때 AES-CBC, KMS 등의 알고리즘으로 암호화되어 저장됩니다.
* ConfigMap은 이 암호화 기능을 네이티브로 지원하지 않습니다. 따라서 민감 정보는 반드시 Secret에 담아야 etcd가 뚫리더라도 데이터를 안전하게 보호할 수 있습니다.

---

## 📝 요약

| 구분 | ConfigMap | Secret |
| :--- | :--- | :--- |
| **주 사용 목적** | 비민감성 환경 설정 (URL, 포트, 일반 텍스트) | 민감성 보안 정보 (비밀번호, 토큰, 인증서) |
| **kubectl 조회 시 포맷** | 평문 (Plain Text) | Base64 인코딩 |
| **볼륨 마운트 매커니즘** | 일반 디스크 마운트 사용 가능 | 메모리 기반 가상 파일 시스템(`tmpfs`) 사용 |
| **etcd 암호화 (At Rest)** | 지원하지 않음 (평문 저장) | API 서버 설정을 통해 **암호화 저장 지원** |
| **RBAC 통제** | 일반적으로 넓게 허용됨 | 엄격하게 제한하여 관리 (모범 사례) |

---

### [참고]
* [Kubernetes 공식 문서 - Secrets](https://kubernetes.io/docs/concepts/configuration/secret/)
* [Kubernetes 공식 문서 - ConfigMaps](https://kubernetes.io/docs/concepts/configuration/configmap/)
* [Kubernetes 공식 문서 - Encrypting Secret Data at Rest](https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data/)
