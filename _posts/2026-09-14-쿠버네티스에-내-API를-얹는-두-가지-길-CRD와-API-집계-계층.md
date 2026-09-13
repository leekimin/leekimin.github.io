---
layout: post
title: "쿠버네티스에 내 API를 얹는 두 가지 길: CRD와 API 집계 계층"
description: "Pod나 Deployment처럼 내 도메인 객체도 kubectl로 다루고 싶다면? CRD로 리소스를 등록하고 컨트롤러로 살아 움직이게 만드는 법, 그리고 CRD로는 안 되는 순간에 꺼내는 API 집계 계층까지 정리했습니다."
categories: [k8s]
tags: [k8s, kubernetes, devops]
---

`kubectl get pods`는 매일 치는 명령어입니다. 그런데 어느 날 후배가 이렇게 묻습니다.

"선배, 우리 팀 결제 배치 작업도 `kubectl get paymentjob`으로 조회되면 좋겠는데요. 그거 되나요?"

됩니다. 그리고 그게 쿠버네티스가 10년 넘게 살아남은 이유이기도 합니다.

## API 서버는 왜 확장을 허락하는가

먼저 감부터 잡고 갑시다. 쿠버네티스에서 Pod, Service, Deployment는 특별한 존재가 아닙니다. 전부 그냥 **API 서버에 등록된 리소스 타입**일 뿐입니다. 사용자가 원하는 상태(`spec`)를 선언해서 etcd에 저장하면, 컨트롤러가 그걸 읽고 실제 상태(`status`)를 거기에 맞춰갑니다. 구조는 이게 전부입니다.

그러니 내 도메인 객체도 같은 규칙만 지키면 1급 시민이 될 수 있습니다. 인증, 인가(RBAC), 감사 로그, kubectl, 레이블 셀렉터, watch까지 전부 공짜로 따라옵니다. 이게 핵심 이득입니다. REST 서버를 따로 띄우면 저 목록을 전부 직접 만들어야 하니까요.

확장하는 길은 크게 두 갈래입니다. **CRD**와 **API 집계 계층(Aggregation Layer)**.

## 길 1: CRD — 99%는 여기서 끝납니다

CustomResourceDefinition은 "API 서버야, 앞으로 이런 모양의 리소스를 받아줘"라고 등록하는 선언입니다. 코드 한 줄 없이 YAML만으로 됩니다.

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  # 반드시 <plural>.<group> 형식이어야 합니다
  name: paymentjobs.batch.mycompany.io
spec:
  group: batch.mycompany.io
  scope: Namespaced
  names:
    plural: paymentjobs
    singular: paymentjob
    kind: PaymentJob
    shortNames: [pj]
  versions:
    - name: v1alpha1
      served: true      # 이 버전을 API로 제공할지
      storage: true     # etcd에 저장될 정본 버전 (딱 하나만 true)
      schema:
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              required: [merchantId, schedule]
              properties:
                merchantId:
                  type: string
                schedule:
                  type: string
                retryLimit:
                  type: integer
                  minimum: 0
                  maximum: 10
                  default: 3
            status:
              type: object
              properties:
                phase:
                  type: string
      subresources:
        status: {}      # status를 별도 서브리소스로 분리
      additionalPrinterColumns:
        - name: Merchant
          type: string
          jsonPath: .spec.merchantId
        - name: Phase
          type: string
          jsonPath: .status.phase
```

적용하고 확인해 봅시다.

```bash
kubectl apply -f paymentjob-crd.yaml

# 새 리소스가 API 목록에 등록됐는지 확인
kubectl api-resources --api-group=batch.mycompany.io

# 문서까지 자동 생성됩니다
kubectl explain paymentjob.spec.retryLimit
```

이제 진짜 객체를 만들어 봅니다.

```yaml
apiVersion: batch.mycompany.io/v1alpha1
kind: PaymentJob
metadata:
  name: daily-settlement
spec:
  merchantId: "store-1024"
  schedule: "0 3 * * *"
```

```bash
kubectl apply -f daily-settlement.yaml
kubectl get pj
# NAME               MERCHANT     PHASE
# daily-settlement   store-1024
```

후배가 원하던 화면이 나왔습니다. 그런데 PHASE 칸이 비어 있죠. 여기가 중요한 지점입니다.

## CRD만으로는 아무 일도 일어나지 않습니다

지금 우리가 만든 건 **타입이 있는 데이터베이스 레코드**입니다. 스키마 검증을 통과하고 etcd에 잘 저장됐지만, 그게 끝입니다. 결제 배치는 단 한 건도 돌지 않았습니다.

리소스를 살아 움직이게 하는 건 컨트롤러입니다. 컨트롤러는 단순한 무한 루프예요. "선언된 상태를 읽고, 현실과 비교하고, 차이를 메운다." 이걸 **조정 루프(reconcile loop)**라고 부릅니다.

```go
func (r *PaymentJobReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    var job batchv1alpha1.PaymentJob
    if err := r.Get(ctx, req.NamespacedName, &job); err != nil {
        // 이미 삭제된 객체면 조용히 종료
        return ctrl.Result{}, client.IgnoreNotFound(err)
    }

    // 원하는 상태: 이 PaymentJob에 대응하는 CronJob이 하나 떠 있어야 한다
    desired := buildCronJob(&job)

    // 소유권을 걸어두면 PaymentJob 삭제 시 CronJob도 같이 정리됩니다
    if err := ctrl.SetControllerReference(&job, desired, r.Scheme); err != nil {
        return ctrl.Result{}, err
    }

    // 생성/수정을 구분하지 말고 서버사이드 어플라이로 수렴시킵니다
    if err := r.Patch(ctx, desired, client.Apply,
        client.ForceOwnership, client.FieldOwner("paymentjob-controller")); err != nil {
        return ctrl.Result{}, err
    }

    job.Status.Phase = "Scheduled"
    return ctrl.Result{}, r.Status().Update(ctx, &job)
}
```

여기서 반드시 기억할 것 하나. **Reconcile은 언제든 몇 번이든 다시 호출됩니다.** 네트워크가 끊겨도, 컨트롤러가 재시작해도, 아무 변화가 없어도 호출될 수 있습니다. 그래서 "생성한다"가 아니라 "있어야 할 상태로 맞춘다"로 써야 합니다. 멱등성이 안 지켜지면 CronJob이 수십 개 복제되는 사고가 납니다.

CRD와 이 컨트롤러를 묶어서 배포한 것, 그게 흔히 말하는 **오퍼레이터(Operator)**입니다. 대단한 개념이 아니라 조합의 이름일 뿐입니다.

## 길 2: API 집계 계층 — CRD가 막히는 순간

CRD는 etcd에 저장되는 선언적 객체를 전제로 합니다. 그런데 이런 요구는 CRD로 못 풉니다.

- 데이터를 etcd에 저장하고 싶지 않다 (예: 실시간 CPU 사용량)
- 응답을 매 요청마다 계산해서 내려줘야 한다
- 스토리지 백엔드를 직접 제어해야 한다

대표 사례가 `kubectl top`입니다. 노드 메트릭을 etcd에 넣을 이유가 없죠. 이럴 때는 직접 만든 API 서버를 쿠버네티스 API 서버 뒤에 끼워 넣습니다.

```yaml
apiVersion: apiregistration.k8s.io/v1
kind: APIService
metadata:
  name: v1beta1.metrics.k8s.io
spec:
  group: metrics.k8s.io
  version: v1beta1
  service:
    name: metrics-server
    namespace: kube-system
    port: 443
  groupPriorityMinimum: 100
  versionPriority: 100
```

이렇게 등록하면 `/apis/metrics.k8s.io/v1beta1` 로 들어온 요청을 API 서버가 metrics-server로 **프록시**합니다. 사용자 입장에서는 여전히 하나의 쿠버네티스 API입니다.

대신 대가가 큽니다. 인증서 관리, TLS 갱신, 고가용성, 버전 협상을 전부 직접 책임져야 하고, 그 서버가 죽으면 해당 API 그룹 전체가 죽습니다. 그래서 원칙은 간단합니다. **CRD로 되면 CRD로 하세요.** 집계 계층은 위 세 가지 이유 중 하나에 해당할 때만 꺼내는 카드입니다.

## 실무에서 부딪히는 것들

**버전은 처음부터 v1alpha1로 시작하세요.** 언젠가 필드 구조를 바꾸게 됩니다. `v1alpha1`과 `v1beta1`을 동시에 served로 두고 conversion webhook으로 변환하면 사용자 매니페스트를 깨지 않고 넘어갈 수 있습니다. 처음부터 `v1`로 시작하면 도망갈 길이 없습니다.

**스키마 검증에 인색하지 마세요.** `required`, `minimum`, `enum`을 촘촘히 걸면 잘못된 값이 etcd에 들어가기 전에 API 서버가 막아줍니다. 컨트롤러 안에서 방어 코드를 짜는 것보다 훨씬 싸게 먹힙니다.

**`status`는 사용자가 쓰는 필드가 아닙니다.** `subresources.status`를 켜두면 `spec` 업데이트와 `status` 업데이트 권한이 분리되어, 컨트롤러와 사용자가 서로의 쓰기를 덮어쓰는 사고를 막을 수 있습니다.

**RBAC을 잊지 마세요.** CRD를 등록해도 권한을 주지 않으면 아무도 접근할 수 없습니다.

```bash
kubectl auth can-i create paymentjobs --as=dev-user
```

## 정리

쿠버네티스 API 확장은 결국 한 문장입니다. **내 도메인 개념을 선언적 리소스로 정의하고(CRD), 그 선언을 현실로 만드는 루프를 붙인다(컨트롤러).** 저장이 필요 없거나 응답을 계산해야 하는 예외적인 경우에만 API 집계 계층을 씁니다.

처음 CRD를 만들면 "이게 되네?" 싶은 순간이 옵니다. 그 순간부터 쿠버네티스는 컨테이너 오케스트레이터가 아니라, 여러분 팀의 인프라를 기술하는 공용 언어가 됩니다.


---

> **이 글의 본문은 자동 생성되었습니다.**
> 학습 기록을 남기기 위해 매일 정해진 시각에 스케줄러가 Claude Code를 호출해 작성하고, 사람의 검수 없이 그대로 게시합니다.
> 따라서 사실과 다른 내용이나 오래된 정보가 포함될 수 있습니다. 명령어와 설정은 실제 환경에서 검증한 뒤 사용하세요.
