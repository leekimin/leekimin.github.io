---
layout: post
title: "kubectl 기본기와 매니페스트 YAML 구조 읽는 법"
description: "kubectl의 핵심 명령 패턴과 매니페스트의 4대 필드(apiVersion·kind·metadata·spec)를 읽는 법을 정리합니다."
categories: [k8s]
tags: [k8s, kubernetes, devops]
---

## 왜 필요한가

이전 글에서 쿠버네티스는 원하는 상태를 선언하면 컨트롤러가 그 상태로 맞춰 준다고 정리했습니다. 원하는 상태를 적는 문서가 **매니페스트 YAML**이고, 그 문서를 API 서버에 보내고 결과를 확인하는 도구가 **kubectl**입니다. 결국 운영 업무 대부분은 "YAML을 읽고, kubectl로 적용하고, 상태를 확인하는" 일의 반복입니다. 둘 다 몸에 익지 않으면 장애가 났을 때 무엇이 문제인지 찾는 데 시간이 오래 걸립니다.

## 핵심 개념 1: kubectl 명령 패턴

kubectl은 `kubectl <동사> <리소스> [이름] [옵션]` 형태로 거의 일정합니다.

```bash
# 조회
kubectl get pods -n kube-system -o wide
kubectl get deploy,svc -A

# 상세 정보 + 이벤트 (장애 분석의 출발점)
kubectl describe pod <pod-name>

# 로그와 접속
kubectl logs -f <pod-name> -c <container> --previous
kubectl exec -it <pod-name> -- sh

# 선언형 적용과 삭제
kubectl apply -f app.yaml
kubectl delete -f app.yaml

# 컨텍스트(어느 클러스터를 보고 있는지) 확인
kubectl config get-contexts
kubectl config use-context prod-onprem
```

`create`, `edit`, `scale` 같은 명령형 방식도 있지만, 실무에서는 **`apply -f` 중심의 선언형 방식**을 기본으로 삼아야 Git에 남은 YAML과 클러스터 상태가 같게 유지됩니다.

## 핵심 개념 2: 매니페스트의 4대 필드

리소스 종류와 관계없이 최상위 필드는 거의 같습니다.

```yaml
apiVersion: apps/v1        # API 그룹/버전
kind: Deployment           # 리소스 종류
metadata:                  # 식별 정보
  name: web
  namespace: demo
  labels:
    app: web
spec:                      # 원하는 상태 (내가 작성)
  replicas: 3
  selector:
    matchLabels:
      app: web
  template:                # 여기부터 Pod 템플릿
    metadata:
      labels:
        app: web
    spec:
      containers:
        - name: nginx
          image: nginx:1.27
          ports:
            - containerPort: 80
          resources:
            requests: { cpu: 100m, memory: 128Mi }
            limits:   { memory: 256Mi }
```

- **apiVersion / kind**: 어떤 API로 어떤 객체를 만들지 정합니다. `kubectl api-resources`로 조회할 수 있습니다.
- **metadata**: 이름, 네임스페이스, 레이블, 어노테이션이 들어갑니다.
- **spec**: 원하는 상태입니다. 여기에 **`status`는 적지 않습니다.** status는 컨트롤러가 채우는 현재 상태입니다.

Deployment 안에 `template.spec`이 또 있는 것처럼 **spec 안에 다른 리소스의 spec이 들어 있는 구조**가 핵심입니다. 들여쓰기 깊이를 기준으로 "지금 어느 객체의 spec을 보고 있는지" 따라가며 읽으면 됩니다.

## 필드가 기억나지 않을 때

문서를 찾기 전에 클러스터에 직접 물어보는 편이 빠르고, 해당 클러스터 버전 기준으로 정확합니다.

```bash
kubectl explain deployment.spec.template.spec.containers.resources
kubectl explain pod.spec --recursive | less

# 뼈대 YAML 생성
kubectl create deploy web --image=nginx:1.27 --dry-run=client -o yaml > web.yaml

# 적용 전 차이 확인
kubectl diff -f web.yaml
```

## 실무에서 주의할 점

1. **컨텍스트 확인 습관**: 운영과 개발 클러스터를 헷갈려 `delete`를 실행하는 사고가 가장 흔합니다. 프롬프트에 현재 컨텍스트를 표시하고, 운영용 kubeconfig는 따로 두세요.
2. **selector와 labels 일치**: `spec.selector.matchLabels`와 `template.metadata.labels`가 다르면 적용 자체가 거부됩니다. 또 Deployment의 selector는 생성한 뒤에 바꿀 수 없습니다.
3. **YAML 함정**: 탭 문자는 쓸 수 없고, `yes`, `on`, `08` 같은 값은 의도와 다른 타입으로 해석될 수 있습니다. 환경 변수 값은 항상 따옴표로 감싸세요.
4. **`kubectl edit` 남용 금지**: 클러스터에서 직접 고친 내용은 다음 `apply` 때 덮어써집니다. 수정은 항상 Git의 YAML에서 시작하세요.
5. **이미지 태그 고정**: `latest`는 재현성이 없습니다. 버전 태그나 digest를 쓰세요.

다음 글에서는 오늘 읽은 매니페스트의 가장 작은 단위인 **Pod**를 자세히 살펴보겠습니다.


---

> **이 글의 본문은 자동 생성되었습니다.**
> 학습 기록을 남기기 위해 매일 정해진 시각에 스케줄러가 Claude Code를 호출해 작성하고, 사람의 검수 없이 그대로 게시합니다.
> 따라서 사실과 다른 내용이나 오래된 정보가 포함될 수 있습니다. 명령어와 설정은 실제 환경에서 검증한 뒤 사용하세요.
