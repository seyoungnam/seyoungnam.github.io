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

#### hosts 필드

`hosts` 필드는 가상 서비스의 호스트들을 나열합니다. 즉, 사용자가 주소를 지정할 수 있는 대상 또는 라우팅 규칙이 적용되는 대상을 의미합니다. 이는 클라이언트가 서비스에 요청을 보낼 때 사용하는 주소입니다.

```yaml
hosts:
- reviews
```

가상 서비스의 호스트네임은 IP주소, DNS네임, 혹은 플랫폼에 따라 정규화된 도메인 이름(fully qualified domain name, FQDN)으로 확인되는 짧은 이름(예를 들자면 쿠버네티스 서비스 이름)이 될 수 있다. 와일드카드("*") 접두사도 사용 가능한데, 이를 활용하여 하나의 라우팅 규칙이 여러 서비스를 커버할 수 있다. 가상 서비스 호스트는 Istio의 서비스 레지스트리의 일부일 필요는 없으며 단순히 가상 대상일 뿐이다. 이를 통해 메쉬 내부에 가상 호스트에 대한 트래픽을 모델링할 수 있다(여기서 지정된 가상 호스트는 결국 실제 대상과 연결이 된다).  


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

> 이와 같이 짧은 이름을 사용하는 것은 대상 호스트와 가상 서비스가 실제로 동일한 쿠버네티스 네임스페이스에 존재하는 경우에만 작동합니다. 쿠버네티스 짧은 이름 사용은 자칫 오표기로 이어질 수 있기 때문에 프러덕션 환경에서는 정규화된 호스트 이름을 사용하는 것이 추천합니다.

