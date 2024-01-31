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