---
layout: post
title: Concepts - Traffic Management
date: 2024-01-30 22:45:00
giscus_comments: true
description: 
tags: 
# categories: sample-posts
---

Istio의 트래픽 라우팅 규칙은 서비스 간 트래픽과 API 요청에 대한 컨트롤을 매우 쉽게 만들어 줍니다. 또한 Istio는 서킷 브레이커, 타임아웃, 재시도 등과 같은 서비스 단의 기능 설정이 매우 간편하며, A/B 테스팅, 카나리아 배포, 단계적 배포 등과 같은 중요한 작업 세팅도 용이합니다. 더하여 어플리케이션이 외부 서비스 혹은 네트워크 소실에 탄력적 대응을 손쉽게 할 수 있는 다양한 기능을 제공합니다.

Istio의 트래픽 관리 모델은 당신의 서비스 바로 옆에 배치/구동되는 Envoy proxy를 통해 구현됩니다. 당신의 mesh services가 보내고 받는 모든 트래픽(data plane 트래픽)은 목적지 도달 전 Envoy를 거치게 되며, 이는 서비스 단의 변화 없이도 손쉽게 트래픽을 조정하고 관리하도록 도와 줍니다.

만약 이 페이지에 제시된 기능들이 어떻게 작동하는지에 대한 세부사항에 관심이 있다면, [architecture overview](https://istio.io/latest/docs/ops/deployment/architecture/) 파트에서 Istio의 트래픽 관리 시행에 관한 더 많은 정보를 얻으실 수 있습니다. 이 페이지 내 나머지 부분은 Istio의 트래픽 관리 기능 소개를 위해 할애 하겠습니다.

<br>

## Istio 트래픽 관리 소개

메쉬 내에서 트래픽을 보내기 위해서는, Istio가 모든 엔드포인트 및 관련 서비스들을 모두 파악하고 있어야 합니다. service registry를 등록하기 위해서는, Istio가 서비스 발견 시스템(service discovery system)에 연결되어 있어야 합니다. 즉, Istio가 쿠버네티스 클러스터에 설치되어 있다면 Istio는 클러스터 내 서비스와 관련 엔드포인트들을 자동적으로 발견할 것입니다.

이 service registry를 활용하여 Envoy proxy들은 트래픽을 관련 서비스에 보낼 수 있게 됩니다. 마이크로서비스 기반 서비스 대부분은 동일 서비스에 대한 복수의 인스턴스가 작동하고 있으며, 이는 최소 요청 모델(least requests pool)에서 로드 밸런싱 풀(load balancing pool)을 형성 합니다. 최소 요청 모델은 가장 부하가 적게 걸린 인스턴스에 먼저 트래픽을 배치하는 방식으로 풀 내 트래픽 분산을 유도합니다.

Istio의 기본적인 서비스 발견 시스템과 로드 밸런싱이 서비스 메쉬의 핵심이긴 하나, 이는 Istio 기능의 일부에 불과합니다. 대부분의 사용자들은 A/B 테스트의 일환으로 새 버전의 서비스에 특정 비율의 트래픽만 보내거나 특정 하위 집합에만 다른 로드 밸런싱 정책을 적용하는 등 메쉬 내 트래픽에 대한 좀 더 정밀한 컨트롤을 원합니다. 메쉬 안팎을 드나드는 트래픽에 특수한 규칙을 적용한다던지 혹은 service registry에 외부 서비스를 등록하는 것 또한 사용자들이 원하는 것 중 하나일 것입니다. 이 모든 것들은 Istio의 트래픽 관리 API를 통해 트래픽 환경설정(configuration)을 Istio에 추가하기만 하면 실현 됩니다.

다른 Istio 환경설정과 마찬가지로, API는 YAML로 표현되는 쿠버네티스의 커스텀 리소스 정의(custom resource definitions, CRDs)를 통해 명시되며, 이어지는 예시를 통해 볼 수 있을 것입니다.

이 가이드북은 트래픽 관리 API 리소스 예시와 이를 활용하여 무엇을 할 수 있는지 보여 줄 것입니다. 관련 리소스는 아래와 같습니다.

- [Virtual services](https://istio.io/latest/docs/concepts/traffic-management/#virtual-services)
- [Destination rules](https://istio.io/latest/docs/concepts/traffic-management/#destination-rules)
- [Gateways](https://istio.io/latest/docs/concepts/traffic-management/#gateways)
- [Service entries](https://istio.io/latest/docs/concepts/traffic-management/#service-entries)
- [Sidecars](https://istio.io/latest/docs/concepts/traffic-management/#sidecars)

또한, 이 가이드북은 API 리소스를 통해 구현되는 [네트워크 탄력성과 테스트 기능](https://istio.io/latest/docs/concepts/traffic-management/#network-resilience-and-testing)에 대한 간략한 설명도 제공합니다.

<br>

## 가상 서비스(Virtual services)

가상 서비스는 destination rules와 함께 Istio의 트래픽 라우팅 기능의 핵심 구성 요소입니다. 사용자는 가상 서비스를 통해 Istio 서비스 메쉬 내에서 요청(requests)이 어떻게 서비스까지 도달하는지 설정할 수 있습니다. 각 가상 서비스는 라우팅 규칙들을 포함하고 있으며 규칙들은 순서대로 평가됩니다. 적시된 라우팅 규칙 중 하나가 주어진 요청과 일치될 경우 Istio는 해당 요청을 적시된 대상으로 보내게 됩니다. 메쉬 내에서는 사용 사례에 따라 여러 가상 서비스가 존재할 수도 혹은 하나도 존재하지 않을 수도 있습니다.

<br>

### 왜 가상 서비스를 쓰는가?

가상 서비스는 Istio의 트래픽 관리를 더욱 유연하고 강력하게 만드는 핵심요소 입니다. 이는 가상 서비스가 클라이언트의 요청을 구현하는 대상 워크로드와 요청을 보내는 위치의 개념을 분리하였기 때문입니다. 또한 가상 서비스는 각 워크로드 마다 다른 라우팅 규칙을 다양한 방식으로 명시할 수 있습니다.

가상 서비스는 왜 유용한 것일까요? 가상 서비스가 없다면 Envoy는 위에 언급한 것처럼 모든 서비스에 대해 최소 요청 로드 밸런싱 방식으로만 트래픽을 분배하게 될 것입니다. 하지만 각 워크로드별 특성을 부여하여 이 단점을 보완할 수 있습니다. 예를 들자면, 각 워크로드에 다른 버전을 부여하여, 서비스 버전별 트래픽 비중을 달리하거나 혹은 내부 사용자으로부터의 트래픽을 특정 버전의 서비스에만 배정하는 방식으로 A/B 테스트를 진행할 수 있을 것입니다.

가상 서비스를 통해 하나 혹은 그 이상의 호스트 이름에 대해 트래픽 동작을 지정할 수 있습니다. 가상 서비스 내 라우팅 규칙을 이용하여 트래픽을 적절한 대상으로 어떻게 보낼지 Envoy에 알려 줍니다. 경로 대상은 동일한 서비스의 다른 버전일 수도 혹은 완전히 다른 서비스일 수도 있습니다.

일반적인 사용 사례는 서비스 하위 집합 내에 있는 다양한 버전의 서비스로 트래픽을 보내는 것입니다. 클라이언트는 가상 서비스 호스트에 요청을 보내고, Envoy는 해당 요청을 가상 서비스 규칙에 따라 다른 서비스 버전으로 보냅니다. 예를 들자면, "전체 요청의 20%는 신규 버전 서비스로" 혹은 "특정 사용자들로부터 오는 요청은 버전 2로"와 같은 규칙을 지정할 수 있습니다. 이를 통해 신규버전 서비스로의 트래픽 비중을 서서히 늘리는 카나리아 배포를 할 수 있습니다. 트래핑 라우팅은 인스턴스 배포와 완전히 분리될 수 있습니다. 즉, 신규 서비스 버전이 구동되는 인스턴스의 개수는 트래핑 라우팅의 참조 없이 트래픽 부하에 따라 늘리거나 줄일 수 있습니다. 반면에 쿠버네티스와 같은 컨테이너 오케스트레이션 플랫폼들은 인스턴스 개수 기반의 트래픽 배분만 지원하기에 복잡해 질 가능성이 있습니다. 가상 서비스가 어떻게 카나리아 배포를 도울 수 있는지 [Canary Deployment using Istio](https://istio.io/latest/blog/2017/0.1-canary/)에서 확인하실 수 있습니다.

가상 서비스는:

- 하나의 가상 서비스로 여러 어플리케이션 서비스를 처리할 수 있습니다. 예를 들어, 단일 가상 서비스로 여러 개의 "실제" 서비스를 묶는 것은 사용자들로 하여금 서비스 이용방식의 변화 없이 모놀리식(monolithic) 어플리케이션을 마이크로 서비스 기반의 종합 서비스로 바꾸는데 매우 유용합니다. "monolith.com 내 몇몇 URI에 대한 요청은 마이크로서비스 A로 보냄"와 같은 규칙은 라우팅 규칙에 기재하기만 하면 됩니다. [아래 예제](https://istio.io/latest/docs/concepts/traffic-management/#more-about-routing-rules)를 통해 더 알아 보겠습니다.
- 게이트웨이와 함께 트래픽 규칙을 구성하여 송수신 트래픽을 제어합니다.

서비스 하위집합 별로 트래픽 규칙을 설정은 가상 서비스 내 대상 규칙(destination rules)을 통해 가능합니다. 서비스 하위집합 및 기타 대상별 정책을 개별 개체에 명시해 두는 것은 가상 서비스가 바뀌더라도 쉽게 재사용할 수 있게 만듭니다. 다음 섹션에서 대상 규칙에 대해 자세히 알아 보겠습니다.

<br>

### 가상 서비스 예시

아래 가상 서비스는 수신된 요청을 요청자에 따라 다른 버전의 서비스로 보냅니다.

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: reviews
spec:
  hosts:
  - reviews
  http:
  - match:
    - headers:
        end-user:
          exact: jason
    route:
    - destination:
        host: reviews
        subset: v2
  - route:
    - destination:
        host: reviews
        subset: v3
```
<br>

#### hosts 필드

`hosts` 필드는 가상 서비스의 호스트들을 나열합니다. 즉, 사용자가 주소를 지정할 수 있는 대상 또는 라우팅 규칙이 적용되는 대상을 의미합니다. 이는 클라이언트가 서비스에 요청을 보낼 때 사용하는 주소입니다.

```yaml
hosts:
- reviews
```

가상 서비스의 호스트네임은 IP주소, DNS네임, 혹은 플랫폼에 따라 정규화된 도메인 이름(fully qualified domain name, FQDN)으로 확인되는 짧은 이름(예를 들자면 쿠버네티스 서비스 이름)이 될 수 있다. 와일드카드("*") 접두사도 사용 가능한데, 이를 활용하여 하나의 라우팅 규칙이 여러 서비스를 커버할 수 있다. 가상 서비스 호스트는 Istio의 서비스 레지스트리의 일부일 필요는 없으며 단순히 가상 대상일 뿐이다. 이를 통해 메쉬 내부에 가상 호스트에 대한 트래픽을 모델링할 수 있다(여기서 지정된 가상 호스트는 결국 실제 대상과 연결이 된다).  

<br>

#### 전송 규칙(Routing rules)

`http` 섹션은 가상 서비스의 전송 규칙을 명시하는 곳으로, HTTP/1.1, HTTP2, 그리고 gRPC 트래픽을 `hosts` 필드에 명시된 대상에 보내기 위한 일치 조건과 작업이 기술됩니다(같은 논리로 `tcp`와 `tls` 섹션에 TCP를 위한 전송 규칙과 종료되지 않은 TLS 트래픽에 대한 전송 규칙을 기술 합니다). 전송 규칙은 트래픽을 보내고 싶은 대상(destination)과 0개 이상의 일치 조건(matching conditions)으로 구성됩니다.

##### 일치 조건(Match condition)

위 예시의 첫번째 전송 규칙은 조건(condition)에 맞춰 적용하고 싶기에 `match` 필드로 시작한다. 그 조건이란 "jason"이란 사용자가 전송하는 모든 요청이며, 따라서 `headers`, `end-user`, 그리고 `exact`란 필드를 통해 적절한 요청을 선택하게 된다.

```yaml
- match:
   - headers:
       end-user:
         exact: jason
```

##### 대상(Destination)

`route` 섹션의 `destination` 필드는 이 조건에 부합하는 트래픽에 대한 실제 대상을 명시하는 곳이다. 가상 서비스의 `host(s)` 섹션과는 달리, `destination` 필드의 `host` 는 반드시 Istio 서비스 레지스트리에 존재하는 실제 대상어야 하며, 그렇지 않으면 Envoy는 어디에 트래픽을 전송할지 알 수 없게 된다. 또한 `host`는 프록시가 포함된 메쉬 서비스거나 서비스 항목(service entry)을 통해 추가된 비 메쉬 서비스일 수 있다. 이 경우 쿠버네티스에서 실행 중인 서비스 이름을 명시할 수 있다:

```yaml
route:
- destination:
    host: reviews
    subset: v2
```

이 페이지에 나오는 이번 그리고 다른 예제에서는 단순화를 위해 대상 호스트(destination hosts)에 쿠버네티스에서 쓰는 짧은 이름(short name)을 사용합니다. 전송 규칙들이 하나씩 평가될 때 Istio는 명시된 짧은 이름에 해당 가상 서비스의 네임스페이스를 더하여 호스트의 정확한 이름을 얻는다. 이 예시에 짧은 이름을 사용한다는 것은 원하는 네임스페이스에 복사하여 사용해 볼 수 있다는 의미이기도 합니다.

> 이와 같이 짧은 이름을 사용하는 것은 대상 호스트와 가상 서비스가 실제로 동일한 쿠버네티스 네임스페이스에 존재하는 경우에만 작동합니다. 쿠버네티스 짧은 이름 사용은 자칫  잘못된 설정으로 이어질 수 있기 때문에 프러덕션 환경에서는 정규화된 호스트 이름을 사용하는 것을 추천합니다.

`destination` 섹션은 또한 규칙의 조건과 일치하는 요청이 이동할 쿠버네티스 서비스의 하위집합(위 예에서는 v2로 하위집합 설정)을 지정할 수 있습니다. 아래 [대상 규칙(Destination rules)](https://istio.io/latest/docs/concepts/traffic-management/#destination-rules) 섹션에서 어떻게 서비스 하위집합을 설정할 수 있는지 보게 될 것입니다.


<br>

### 전송 규칙에 관하여

위에서 본 것처럼, 전송 규칙은 특정 하위집합 트래픽을 특정 대상으로 보내기 위한 강력한 툴 입니다. 전송 규칙 부분을 통해 트래픽 포트, 헤더 필드, URI 등과 같은 일치 조건을 설정할 수 있습니다. 예를 들어, 아래 가상 서비스는 평점(ratings)과 리뷰(reviews)라는 두개의 다른 서비스로 트래픽을 보내는데, 이 각각의 서비스는 마치 http://bookinfo.com/라 불리우는 (실제로 존재하지 않는) "가상" 서비스 하부 서비스처럼 여겨집니다. 가상 서비스 규칙은 요청 URI에 따라 트래픽을 매칭시킨 후 요청을 적절한 서비스에 전송합니다.

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: bookinfo
spec:
  hosts:
    - bookinfo.com
  http:
  - match:
    - uri:
        prefix: /reviews
    route:
    - destination:
        host: reviews
  - match:
    - uri:
        prefix: /ratings
    route:
    - destination:
        host: ratings
```

매칭 조건에는 정확한 값, 접두사, 혹은 정규표현식 등을 선택할 수 있습니다.

AND 조건을 걸기 위해서는 다수의 일치 조건을 동일한 `match` 블록에 나열하면 되고, OR 조건을 위해서는 다수의 `match` 블록을 동일한 규칙에 나열하면 됩니다. 또한 한 가상 서비스는 다수의 전송 규칙을 가질 수 있습니다. 이는 하나의 가상 서비스로 다양한 형태의 전송 조건을 만들 수 있게 해줍니다. 매칭 조건 필드 및 관련 값에 대한 전체 리스트는 `HTTPMatchRequest` [참조](https://istio.io/latest/docs/reference/config/networking/virtual-service/#HTTPMatchRequest)에서 볼 수 있습니다.

전송 규칙 파트에서는 매칭 조건은 물론, 서비스별 트래픽 분산 비중도 부여할 수 있습니다. 이는 A/B 테스트와 카나리아 배포 시 유용하게 쓰입니다.

```yaml
spec:
  hosts:
  - reviews
  http:
  - route:
    - destination:
        host: reviews
        subset: v1
      weight: 75
    - destination:
        host: reviews
        subset: v2
      weight: 25
```

전송 규칙을 활용하면 트래픽에 대해 몇몇 작업 수행도 가능합니다. 예를 들자면:

- 헤더를 추가하거나 지우기
- URL 재작성
- 특정 요청에 대해 [재시도 정책](https://istio.io/latest/docs/concepts/traffic-management/#retries) 설정

가능한 작업에 대해 더 알고 싶다면, `HTTPRoute` [참조](https://istio.io/latest/docs/reference/config/networking/virtual-service/#HTTPRoute)를 보시길 권장합니다.

<br>

## 대상 규칙(Destination rules)

[대상 규칙](https://istio.io/latest/docs/reference/config/networking/destination-rule/#DestinationRule)은 [가상 서비스](https://istio.io/latest/docs/concepts/traffic-management/#virtual-services)와 함께 Istio의 트래픽 전송 기능에서 가장 핵심적인 부분 입니다. 가상 서비스는 트래픽을 주어진 대상에 어떻게 보내는지를 정의한다면, 대상 규칙은 해당 대상에 도착한 트래픽을 어떻게 처리할 것인지에 대해 설정 합니다. 대상 규칙은 가상 서비스의 전송 규칙이 평가되고 난 후 적용되기에, 트래픽의 "실제" 대상에 적용 된다고 볼 수 있습니다.

특히, 대상 규칙을 통해 서비스 하부집합을 설정할 수 있는데, 가령 버전 이름으로 대상 서비스의 모든 인스턴스들을 묶을 수 있습니다. 가상 서비스의 전송 규칙에 명시된 이들 서비스 하부집합을 사용하여 서비스의 각기 다른 인스턴스에 대한 트래픽을 조절할 수 있습니다.

대상 정책은 전체 대상 서비스 혹은 특정 서비스 하부집합에 요청을 보낼 때, Envoy의 트래픽 정책을 조정을 가능하게 합니다. 즉, 선호하는 로드밸런싱 모델, TLS 보안 모드, 혹은 서킷 브레이커 세팅 등을 선택할 수 있습니다. 대상 규칙 옵션의 전체 리스트는 [대상 규칙 참조](https://istio.io/latest/docs/reference/config/networking/destination-rule/)에서 볼 수 있습니다. 

<br>

### 로드밸런싱 옵션

기본적으로 Istio는 신규 요청이 가장 적은 수의 요청을 처리한 인스턴스에 우선 배분되는 최소 요청 로드밸런싱 정책을 사용합니다. 또한 Istio는 특정 서비스 혹은 서비스 하위집합향 요청에 대해 아래와 같은 모델들을 지원하며, 대상 규칙에 적시하기만 하면 됩니다.

- 무작위(Random): 여러 인스턴스 중 무작위로 선발하여 요청을 전달
- 비중(Weighted): 인스턴스 별 명시된 비중에 따라 요청을 배분
- Round robin: 인스턴스 순서대로 요청을 배분 

각 옵션에 대한 더 자세한 사항은 [Envoy 로드밸런싱 문서](https://www.envoyproxy.io/docs/envoy/v1.5.0/intro/arch_overview/load_balancing)를 참조하세요.

<br>

### 대상 규칙 예제

아래 대상 규칙은 `my-svc` 대상 서비스에 대한 세가지 하부집합에 대해 각기 다른 로드밸런싱 정책을 적용한 예시입니다.

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: my-destination-rule
spec:
  host: my-svc
  trafficPolicy:
    loadBalancer:
      simple: RANDOM
  subsets:
  - name: v1
    labels:
      version: v1
  - name: v2
    labels:
      version: v2
    trafficPolicy:
      loadBalancer:
        simple: ROUND_ROBIN
  - name: v3
    labels:
      version: v3
```

각 하부집합은 하나 혹은 다수의 `labels` 로 정의되어 있는데, 이 `labels`은 쿠버네티스 pod과 같은 객체를 묘사하는 키/값 쌍 입니다. 이들은 쿠버네티스 서비스의 deployment 객체에 `metadata` 로 적용되어 있으며 각기 다른 버전을 구별하는데 쓰입니다.

대상 규칙은 모든 하부집합에 대한 기본 트래픽 정책과 특정 하부집합에만 적용되는 정책 모두를 포함하고 있습니다. 기본 정책은, 위 `subsets` 필드에 정의된 것처럼, `v1`과 `v3` 하부집합에 대해 simple random load balancer를 설정했으며, `v2`에는 round-robin load balancer가 적용되어 있습니다.

<br>

## 게이트웨이(Gateways)

게이트웨이는 메쉬의 인바운드 및 아웃바운드 트래픽을 관리하는 객체로, 메쉬에 들어오거나 나갈 트래픽을 지정할 수 있습니다. 게이트웨이의 설정은 서비스 워크로드 옆에서 실행되고 있는 사이드카 Envoy 프록시가 아니라 메쉬 가장자리에서 실행되는 독립형 Envoy 프록시에 적용됩니다.

쿠버네티스 수신 API와 같이 시스템에 들어오는 트래픽을 제어하는 다른 메커니즘과 달리, Istio 게이트웨이를 사용하면 Istio 트래픽 전송의 모든 기능들을 유연하게 활용할 수 있습니다. Istio의 게이트웨이 자원을 통해 노출할 포트, TLS 세팅 등등 네트워크 레이어 4~6에 대한 로드 밸런싱 속성을 설정할 수 있습니다. 그 이후 어플리케이션 레이어 트래픽 전송(L7) 규칙을 같은 API 자원에 추가하는 대신 일반적인 Istio 가상 서비스를 게이트웨이에 연결하면 됩니다. 이를 통해 Istio 메쉬 내 다른 data plane 트래픽과 마찬가지로 게이트웨이 트래픽을 관리할 수 있습니다.

게이트웨이는 주로 수신 트래픽을 관리하는데 사용되지만 송신 게이트웨이를 설정할 수도 있습니다. 송신 게이트웨이는 메쉬에서 나가는 트래픽에 대한 전용 출구 노드를 설정하여, 외부 네트워크에 접근 가능한 서비스를 제한하거나 메쉬에 보안 강화를 목적으로 [송신 트래픽의 보안 제어](https://istio.io/latest/blog/2019/egress-traffic-control-in-istio-part-1/)를 활성화 할 수 있습니다. 또한 게이트웨이를 사용하여 순수 내부 프록시를 구성할 수도 있습니다.

Istio는 기본 설정이 되어 있는 게이트웨이 프록시 배포(`istio-ingressgateway`와 `istio-egressgateway`)를 제공합니다. Istio 데모 설치를 선택할 경우 둘다 제공되며, 기본 프로필 Istio를 선택할 경우엔 `istio-ingressgateway`만 배포 됩니다. 배포 버전에 상관없이 개별 설정을 통해 각기 다른 게이트웨이와 게이트웨이 프록시를 배포하고 설정할 수 있습니다.

<br>

### 게이트웨이 예시

아래 예시는 외부 HTTPS 수신 트래픽에 대한 게이트웨이 설정을 보여주고 있습니다.

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: ext-host-gwy
spec:
  selector:
    app: my-gateway-controller
  servers:
  - port:
      number: 443
      name: https
      protocol: HTTPS
    hosts:
    - ext-host.example.com
    tls:
      mode: SIMPLE
      credentialName: ext-host-cert
```

이 게이트웨이 설정은 `ext-host.example.com`향 HTTPS 트래픽을 443 포트를 통해 메쉬 내부로 받아들이는 역할만 할 뿐 그 이후의 전송 방식에 대한 내용은 없습니다.

그 이후의 전송 방식을 기재하고 위 게이트웨이가 정상 작동하기 위해서는, 위 게이트웨이와 한 가상 서비스를 연결시켜야 합니다. 이는 아래와 같이 가상 서비스의 `gateways` 필드를 통해 구현됩니다.

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: virtual-svc
spec:
  hosts:
  - ext-host.example.com
  gateways:
  - ext-host-gwy
```

그런 다음, 위 가상 서비스에 외부 트래픽에 대한 전송 규칙을 제시하면 되겠습니다.

<br>

## 서비스 항목(Service entries)

서비스 항목을 사용하여 Istio가 내부적으로 유지 관리하는 서비스 레지스트리에 항목을 추가할 수 있습니다. 추가된 서비스 항목은 마치 메쉬 내 서비스인 것처럼 취급되어 Envoy 프록시가 해당 서비스에 트래픽을 전송합니다. 서비스 항목 설정은 메쉬 밖에서 구동되고 있는 서비스에 대한 트래픽을 관리할 수 있게 해주며, 다음과 같은 업무를 포함합니다:

- 웹에 노출되어 있는 API와 같은 외부 대상에 대한 트래픽 또는 레거시 인프라의 서비스에 대한 트래픽 방향을 바꾸거나 전달합니다.
- 외부 대상에 대한 [재시도](https://istio.io/latest/docs/concepts/traffic-management/#retries), [시간초과](https://istio.io/latest/docs/concepts/traffic-management/#timeouts), 그리고 [오류 주입](https://istio.io/latest/docs/concepts/traffic-management/#fault-injection) 정책을 설정합니다.
- [가상머신을 메쉬에 추가](https://istio.io/latest/docs/examples/virtual-machines/)하여 가상머신 내에서 메쉬 서비스를 실행합니다.

메쉬 서비스들이 의존하는 모든 외부 서비스들을 서비스 항목을 통해 추가할 필요는 없습니다. Istio는 기본적으로 알 수 없는 서비스에 대한 요청을 통과하도록 Envoy 프록시를 설정합니다. 그러나 Istio 기능을 통해 메쉬에 등록되지 않은 대상으로의 트래픽을 제어할 수는 없습니다.

<br>

### 서비스 항목 예시

아래 서비스 항목 예시는 `ext-svc.example.com`란 외부 서비스를 Istio의 서비스 레지스트리에 등록하는 방법을 보여줍니다.

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: ServiceEntry
metadata:
  name: svc-entry
spec:
  hosts:
  - ext-svc.example.com
  ports:
  - number: 443
    name: https
    protocol: HTTPS
  location: MESH_EXTERNAL
  resolution: DNS
```

외부 리소스는 `hosts`필드를 사용하여 지정할 수 있으며, 완전한 주소값 혹은 와일드카드("*")로 시작하는 도메인네임을 사용하면 됩니다.

메쉬 내 다른 서비스에 대한 트래픽을 설정하는 것과 동일한 방식으로 가상 서비스와 대상 규칙 설정을 통해, 서비스 항목에 대한 트래픽을 보다 세세하게 제어할 수 있습니다. 예를 들어, 아래 대상 규칙은 `ext-svc.example.com` 외부 서비스로의 요청에 대한 TCP 연결 초과시간을 조절할 수 있습니다:

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: ext-res-dr
spec:
  host: ext-svc.example.com
  trafficPolicy:
    connectionPool:
      tcp:
        connectTimeout: 1s
```

더 많은 설정 옵션이 궁금하다면 [서비스 항목 참조](https://istio.io/latest/docs/reference/config/networking/service-entry)를 보시길 바랍니다.

<br>

## 사이드카(Sidecars)

Istio는 기본적으로 연결된 워크로드의 모든 포트에서 트래픽을 허용하고, 트래픽을 전달할 때 메쉬 내 모든 워크로드에 도달하도록 Envoy 프록시를 설정합니다. 사이드카 설정을 통하여 다음을 수행할 수 있습니다:

- Envoy 프록시가 받아들이는 포트와 프로토콜을 정밀 설정
- Envoy 프록시가 접근할 수 있는 서비스 집합을 제한

메쉬 내 모든 프록시가 모든 서비스에 접근할 수 있게 설정하면 높은 메모리 사용량으로 인해 잠재적으로 메쉬 성능에 부정적 영향을 미칠 수 있습니다. 따라서 대규모 어플리케이션에서는 사이드카의 접근성을 제한하는 것이 좋습니다.

특정 네임스페이스 내 모든 워크로드에 동일 사이드카 설정을 적용하거나 `workloadSelector`를 사용하여 특정 워크로드를 선택할 수 있습니다. 예를 들어 아래 사이드카 설정은 `bookinfo` 네임스페이스 내 서비스는 동일 네임스페이스 혹은 Istio control plane(Istio의 송신 및 telemetry 기능 때문에 필요)내의 서비스로의 접근만 허용함을 보여줍니다. 

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: Sidecar
metadata:
  name: default
  namespace: bookinfo
spec:
  egress:
  - hosts:
    - "./*"
    - "istio-system/*"
```

더 자세한 사항은 [사이드카 참조](https://istio.io/latest/docs/reference/config/networking/sidecar/)를 보시기 바랍니다.

<br>

## 네트워크 탄력성 및 테스트

Istio는 메쉬 주변 트래픽 전송을 도울 뿐만 아니라, 런타임 시 동적으로 설정되는 사전 동의 오류 복구 및 오류 주입 기능을 제공합니다. 이러한 기능 사용은 어플리케이션의 안정적 운영에 도움이 되며, 서비스 메쉬가 비작동 노드에 대응하고 국지적 오류가 다른 노드로 전파되는 것을 방지할 수 있습니다.

<br>

### 시간초과(Timeouts)

시간초과는 Envoy 프록시가 특정 서비스의 응답을 기다려야 하는 시간으로, 서비스가 응답을 무기한 기다리지 않고 예측 가능한 시간 내에 호출이 성공하거나 실패하도록 보장합니다. HTTP 요청에 대한 Envoy 시간초과 기능은 Istio에서 기본적으로 비활성화 되어 있습니다.

일부 어플리케이션 및 서비스의 경우 Istio의 기본 제한시간이 적절하지 않을 수 있습니다. 예를 들어 시간초과가 너무 길면 실패한 서비스의 응답 대기시간이 지나치게 길어질 수 있고, 시간초과가 너무 짧으면 여러 서비스의 호출에 대한 응답이 오기도 전에 호출 실패로 불필요한 단정을 지을 가능성이 생깁니다. Istio를 통해 서비스 코드를 건드릴 필요없이 가상 서비스를 사용하여 서비스 별로 동적으로 시간초과를 쉽게 조정할 수 있어, 최적의 시간초과 설정 및 사용이 가능합니다. 다음은 등급(`ratings`) 서비스의 v1 하위집합에 대한 호출에 대해 10초의 시간제한을 지정하는 가상서비스의 예시입니다.

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: ratings
spec:
  hosts:
  - ratings
  http:
  - route:
    - destination:
        host: ratings
        subset: v1
    timeout: 10s
```


<br>

### 재시도(Retries)

재시도 설정은 초기 호출이 실패한 경우 Envoy 프록시가 서비스 연결을 시도하는 최대 횟수를 지정합니다. 재시도는 서비스나 네트워크의 일시적 과부하 등 일시적 문제로 인한 영구적 호출 실패를 방지하며, 따라서 서비스 가용성과 어플리케이션 성능을 높여줍니다. 재시도 간격(25ms 이상)은 가변적이며 Istio에 의해 자동적으로 결정되는데, 서비스가 과도한 요청으로 휩싸이지 않도록 방지합니다. HTTP 요청에 대한 기본 재시도 설정은 오류로 반응하기 전에 두번 재시도하는 것입니다. 

시간초과와 마찬가지로 Istio의 기본 재시도 설정은 지연시간이나 가용성 측면에서 어플리케이션 요구사항에 적합하지 않을 수 있습니다(비작동 서비스에 대한 너무 많은 재시도는 속도를 느리게 만듬). 또한 서비스 코드의 변화 없이도 가상 서비스에서 서비스 별 재시도 설정을 조정할 수 있습니다. 더하여 재시도별 제한시간을 추가하는 등 더욱 세부적인 재시도 동작을 설정할 수 있습니다. 아래 예시는 초기 호출 실패 후 이 서비스 하위집합에 대한 연결 시도를 최대 3회 그리고 각각 2초의 시간초과를 적용한 모습입니다.

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: ratings
spec:
  hosts:
  - ratings
  http:
  - route:
    - destination:
        host: ratings
        subset: v1
    retries:
      attempts: 3
      perTryTimeout: 2s
```

<br>

### 회로 차단기(Circuit breakers)

회로 차단기는 탄력적인 마이크로서비스 기반 어플리케이션을 생성하기 위해 Istio가 제공하는 또다른 유용한 메커니즘 입니다. 회로 차단기를 통해 동시 연결수 또는 동일 호스트에 대한 호출 실패 횟수 등 서비스 내 개별 호스트에 대한 호출 제한을 설정합니다. 명시된 제한 횟수에 도달하면 회로 차단기가 발동되며 해당 호스트에 대한 추가 연결이 중지됩니다. 회로 차단기 패턴을 사용하면 빠르게 오류를 선언하여 클라이언트가 과부하되거나 오류가 발생한 호스트에 연결을 시도하는 것을 막습니다.

회로 차단은 로드 밸런싱 풀의 실제 메쉬 대상에 적용되며, 회로 차단 임계값은 대상 규칙을 통해 각 개별 호스트에 적용할 수 있습니다. 다음 예시는 v1 하위집합의 `reviews` 서비스 워크로드에 대한 동시 연결 수를 100으로 제한하는 방법을 보여줍니다.


```yaml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: reviews
spec:
  host: reviews
  subsets:
  - name: v1
    labels:
      version: v1
    trafficPolicy:
      connectionPool:
        tcp:
          maxConnections: 100
```

회로 차단기 설정에 대해 더 자세한 내용은 [회로 차단기](https://istio.io/latest/docs/tasks/traffic-management/circuit-breaking/)를 참고 하십시오.

<br>

### 오류 주입

네트워크에 오류 복구 정책들을 설정한 후, Istio의 오류 주입 메커니즘을 사용하여 어플리케이션의 오류 복구 능력을 전체적으로 테스트할 수 있습니다. 오류 주입은 오류를 얼마나 잘 견디고 오류로부터 복구할 수 있는지 확인하기 위해 시스템에 고의적으로 오류를 발생시키는 테스트 방법입니다. 오류 주입의 사용은 오류 복구 정책이 호환되지 않거나 혹은 정책이 너무 제한적이어서 중요한 서비스를 사용할 수 없게 되는 상황이 발생하지 않도록 확인하는 데 특히 유용할 수 있습니다.

> 현재 오류 주입은 동일 가상 서비스에서 재시도 혹은 시간초과와 함께 동시에 설정될 수 없습니다. [트래픽 관리 문제](https://istio.io/latest/docs/ops/common-problems/network-issues/#virtual-service-with-fault-injection-and-retrytimeout-policies-not-working-as-expected)를 참조하시길 바랍니다.

네트워크 계층에서 패킷 지연 또는 pod 죽이기와 같은 오류 유발 메커니즘과 달리, Istio는 어플리케이션 계층에서 오류를 주입할 수 있습니다. 이를 통해 HTTP 오류 코드와 같은 보다 관련성이 높은 오류를 삽입하여 더 유의미한 결과를 얻을 수 있습니다.

가상 서비스를 활용하여 두가지 종류의 오류를 주입할 수 있습니다:

- 지연(Delays): 지연은 타이밍 오류이며, 증가된 네트워크 대기 시간 또는 업스트림 서비스 과부화를 모방합니다.
- 중단(Aborts): 중단은 충돌 실패이며, 업스트림 서비스의 오류를 모방합니다. 중단은 일반적으로 HTTP 오류 코드 또는 TCP 연결 실패의 형태로 나타납니다.

예를 들어, 아래 가상 서비스는 `ratings` 서비스에 대한 1,000번의 요청마다 5초의 지연을 도입한 예를 보여 줍니다.

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: ratings
spec:
  hosts:
  - ratings
  http:
  - fault:
      delay:
        percentage:
          value: 0.1
        fixedDelay: 5s
    route:
    - destination:
        host: ratings
        subset: v1
```

지연과 중단 설정에 대한 더 자세한 사항은 [오류 주입](https://istio.io/latest/docs/tasks/traffic-management/fault-injection/)을 참조 바랍니다.

<br>

### 어플리케이션에 적용하기

Istio 오류 복구 기능은 애플리케이션의 동작에 영향을 주지 않습니다. 어플리케이션은 응답 반환 전에 Envoy 사이드카 프록시가 호출된 서비스에 대한 실패를 처리하고 있는지 알 수 없습니다. 즉, 오류 복구 정책이 어플리케이션 코드를 통해 설정되어 있더라도 Istio 오류 복구 기능은 독립적으로 작동하므로 충돌이 발생할 수 있다는 뜻입니다. 예를 들어, 가상 서비스와 어플리케이션 모두에 시간제한이 설정되어 있다고 가정해 보겠습니다. 어플리케이션은 서비스에 대한 API 호출에 대해 2초의 제한 시간을 설정합니다. 그러나 가상 서비스에서는 1회 재시도에 3초 제한 시간을 설정했습니다. 이 경우 어플리케이션의 시간 초과가 먼저 시작되므로 Envoy 시간 초과 및 재시도는 아무런 영향을 미치지 않습니다.

Istio 오류 복구 기능은 메시 내 서비스의 안정성과 가용성을 향상시키는 반면, 어플리케이션은 오류나 오류를 처리하고 적절한 대체 조치를 취해야 합니다. 예를 들어 부하 분산 풀의 모든 인스턴스가 실패하면 Envoy는 `HTTP 503` 코드를 반환합니다. 어플리케이션은 `HTTP 503` 오류 코드를 처리하는 데 필요한 대체 논리(fallback logic)를 구현해야 합니다.