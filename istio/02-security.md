---
layout: post
title: Concepts - Security
date: 2024-02-04 23:45:00
giscus_comments: true
description: 
tags: 
# categories: sample-posts
---

모놀리식(monolithic) 어플리케이션을 여러개의 서비스로 분해하면 더 나은 민첩성과 확장성, 그리고 서비스 재사용성 등 다양한 이점을 얻을 수 있습니다. 그러나 마이크로서비스는 다음과 같은 특별한 보안 요구사항이 발생합니다:

- 중간자 공격(man-in-the-middle attacks)을 방어하기 위해 트래픽 암호화가 필요
- 유연한 서비스 접근 제어를 위해 상호 TLS와 세분화된 접근 정책 설정이 필요
- 누가 언제 무엇을 했는지 확인을 위해 감사 도구가 필요

Istio는 이러한 문제를 해결하기 위한 포괄적인 보안 솔루션을 제공합니다. 이 페이지에서는 Istio 보안 기능을 사용하여 어디서든 서비스를 보호할 수 있는 방법에 대해 간략하게 살펴 볼 예정입니다. 특히 Istio 보안 기능을 통해 데이터, 엔드포인트, 통신 및 플랫폼에 대한 내외부 위협을 모두 완화할 수 있습니다.

![Security overview](https://istio.io/latest/docs/concepts/security/overview.svg)

Istio 보안 기능은 강한 신분 확인, 강력한 정책, 투명한 TLS 암호화, 인증, 승인 및 감사(authentication, authorization and audit, AAA) 도구를 제공하여 서비스와 데이터를 보호합니다. Istio의 보안 목표는 다음과 같습니다:

- 기본값으로서의 보안: 어플리케이션 코드 및 인프라 변경 불필요
- 심층 방어: 기존 보안 시스템과 통합하여 여러 계층의 방어 시스템 제공
- 제로 트러스트 네트워크: 신뢰할 수 없는 네트워크에 보안 솔루션을 구축

배포된 서비스에서 Istio 보안 기능을 사용하려면 [상호 TLS 마이그레이션 문서](https://istio.io/latest/docs/tasks/security/authentication/mtls-migration/)를 참고하십시오. 보안 기능 사용에 대한 자세한 지침을 보려면 [보안 작업](https://istio.io/latest/docs/tasks/security/)을 참고하십시오.

<br>

## 아키텍처 훑어보기

Istio 내 보안은 다수의 구성요소를 지니고 있습니다:

- 키 및 인증서 관리를 위한 인증 기관(Certificate Authority, CA)
- 설정 API 서버가 프록시에 배포하는 요소들:
  - 인증 정책
  - 승인 정책
  - 보안 명명 정보
- 사이드카 및 바깥 프록시는 정책 적용 지점(Policy Enforcement Points, PEP)로 작동하여 클라이언트와 서버 간의 통신을 보호
- 원격 측정(telemetry) 및 감사 관리를 위한 Envoy 프록시 확장 세트

제어 영역(Control plane)은 API 서버의 설정을 처리하고 데이터 영역(data plane)내 PEP(정책 적용 지점)를 설정합니다. PEP는 Envoy를 사용하여 구현됩니다. 다음 다이어그램은 관련 아키텍쳐를 보여줍니다.

![Security Architecture](https://istio.io/latest/docs/concepts/security/arch-sec.svg)

앞으로 이어질 섹션에서는 Istio의 보안 기능에 대해 더 자세하게 다룰 예정입니다.


<br>

## Istio 신원 정보

신원 정보는 모든 보안 인프라에 적용되는 기본 개념입니다. 워크로드 간 통신이 시작될 때 두 당사자는 상호 인증을 위해 신분 정보가 포함하고 있는 자격 정보를 교환해야 합니다. 클라이언트 측에서는 서버의 신분을 [보안 이름(secure naming)](https://istio.io/latest/docs/concepts/security/#secure-naming) 정보와 비교하여 워크로드 실행자가 기승인 되었는지 확인 합니다. 서버 측에서는 입력된 [승인 정책](https://istio.io/latest/docs/concepts/security/#authorization-policies)에 의거, 클라이언트가 접근할 수 있는 정보를 결정하고, 누가 언제 무엇을 접근했는지 감사하고, 사용한 작업량에 따라 요금을 부과하고, 요금을 지불하지 못한 클라이언트의 접근을 거부합니다. 

Istio의 신원 정보 모델은 최고 수준의 "서비스 신원(`service identity`)"을 사용하여 요청자의 신원을 결정합니다. 이 모델은 서비스 신원이 인간 사용자, 개별 워크로드 혹은 워크로드 그룹 등 다양한 대상을 대표할 수 있어, 뛰어난 유연성과 세분성을 제공합니다. 서비스 신원이 부여되지 않은 플랫폼에서는 Istio가 서비스 이름과 같이 일부 워크로드 인스턴스를 대표하는 다른 신원 정보를 사용할 수 있습니다.

다음 목록은 다양한 플랫폼에서 사용할 수 있는 서비스 신원의 예를 보여줍니다.

- 쿠버네티스: 쿠버네티스의 서비스 계정(service account)
- GCE: GCP 서비스 계정(service account)
- 온프레미스(비 쿠버네티스): user account, custom service account, Istio service account, 혹은 GCP service account. 커스텀 서비스 계정은 고객의 Identity Directory에서 관리하는 신원 정보와 같은 기존 서비스 계정을 의미합니다.


<br>

## 신원 및 인증서 관리

Istio는 X.509 인증서를 사용하여 모든 워크로드에 강력한 신원 정보를 안전하게 제공합니다. 각 Envoy 프록시와 함께 실행되는 Istio 에이전트는 istiod와 협력하여 대규모로 키 및 인증서를 주기적으로 재생성하고 이를 자동화 합니다. 아래 그림은 신원 정보 생성 흐름을 보여줍니다.

![Identity Provisioning Workflow](https://istio.io/latest/docs/concepts/security/id-prov.svg)

Istio는 아래의 흐름에 따라 키와 인증서를 생성합니다:

1. `istiod`는 인증서 서명 요청([certificate signing requests](https://en.wikipedia.org/wiki/Certificate_signing_request), CSR)을 받아들이기 위해 gRPC 서비스를 제공합니다.
2. 프로세스가 시작되면 Istio 에이전트는 개인 키와 CSR을 생성한 다음 서명을 위해 해당 자격 정보과 함께 CSR을 `istiod`로 보냅니다.
3. `istiod`의 CA는 CSR에 포함된 자격 정보가 유효한지 확인합니다. 검증이 성공하면 CSR에 서명하여 인증서를 생성합니다.
4. 워크로드가 시작되면 Envoy는 Envoy SDS(Secret Discovery Service) API를 통해 같은 컨테이너에 있는 Istio 에이전트의 인증서와 키를 요청합니다.
5. Istio 에이전트는 `istiod`로부터 받은 인증서와 개인 키를 Envoy SDS API를 통해 Envoy로 보냅니다.
6. Istio 에이전트는 워크로드별 인증서가 만료되었는지 모니터링하고, 만료되었을 경우 위 프로세스를 반복합니다.

<br>

## 인증(Authentication)

Istio는 두가지 형태의 인증방법을 제공합니다:

- 대등 인증: 연결을 시도하는 클라이언트를 확인하기 위해 서비스 간 인증에 사용됩니다. Istio는 전송 인증을 위한 풀스택 솔루션으로 [mutual TLS](https://en.wikipedia.org/wiki/Mutual_authentication)를 제공하며, 이는 서비스 코드의 변경 없이 이루어 집니다. 이 솔루션은 다음과 같습니다:
  - 클러스터와 클라우드 간 상호 운용성을 지원하고자 각 서비스에 강력한 신원 정보를 제공
  - 서비스간 통신을 보호
  - 키 및 인증서 생성, 배포, 순환을 자동화하는 키 관리 시스템 제공
- 요청 인증: 요청에 첨부된 자격 증명 정보를 확인하고자 최종 사용자 인증에 사용됩니다. Istio는 JSON Web Token(JWT) 검증 방식의 요청별 인증을 지원하며, 맞춤형 인증 공급자 또는 OpenID Connect 공급자를 사용하여 간소화된 개발자 경험을 지원합니다. 인증 공급자의 예는 다음과 같습니다:
  - [ORY Hydra](https://www.ory.sh/)
  - [Keycloak](https://www.keycloak.org/)
  - [AuthO](https://auth0.com/)
  - [Firebase Auth](https://firebase.google.com/docs/auth/)
  - [Google Auth](https://developers.google.com/identity/protocols/OpenIDConnect)

어느 인증 공급자를 쓰건 상관없이 Istio는 맞춤형 쿠버네티스 API를 통해 Istio 설정 저장소(`Istio config store`)에 인증 정책을 저장합니다. Istio는 각 프록시에 대한 키와 인증 정책을 지속적으로 업데이트 합니다. 또한 Istio는 정책 변경이 시행되기 전 새로 변경된 정책이 보안 상태에 어떤 영향을 미칠 수 있는지 이해하는데 도움이 되도록 허용 모드(permissive mode)에서의 인증을 지원합니다.

<br>

<!-- ### Mutual TLS 인증 -->

