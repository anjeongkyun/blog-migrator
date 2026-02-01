---
title: "현대 아키텍쳐에서 DTO Pattern이 중요한 이유"
seoTitle: "Why DTO Pattern Still Matters in Modern Architecture"
seoDescription: "DTO Pattern의 본질은 무엇일까? 도메인 모델과 DTO를 혼동하며 생기는 설계 고민부터, 느린 네트워크 환경에서 DTO가 왜 여전히 중요한지 실무 경험을 바탕으로 정리한다."
datePublished: Sun Feb 01 2026 06:21:39 GMT+0000 (Coordinated Universal Time)
cuid: cml3cs0dk000f02jy1eh12fvr
slug: dto-pattern
tags: dto, dto-pattern

---

## DTO Pattern은 어디서 시작됐을까?

DTO Pattern은 `Sun Microsystems의 J2EE Core Patterns`에서 공식적으로 정리된 개념이다. (2001년 초판: Core J2EE Patterns) 최초 DTO는 분산 환경(EJB, RMI)에서 원격 호출 횟수를 줄이기 위한 목적으로 정의되었다.

이 개념을 2002년 마틴 파울러가 `Patterns of Enterprise Application Architecture` 책에서 DTO Pattern을 소개했다. (마틴 파울러는 창시자 보다는 전파자임)

<div data-node-type="callout">
<div data-node-type="callout-emoji">💡</div>
<div data-node-type="callout-text">In the field of programming a <strong>data transfer object</strong> (<strong>DTO</strong><a target="_self" rel="noopener noreferrer nofollow" href="https://en.wikipedia.org/wiki/Data_transfer_object?utm_source=chatgpt.com#cite_note-msdn-1" style="pointer-events: none"><sup>[1][2]</sup></a><a target="_self" rel="noopener noreferrer nofollow" href="https://en.wikipedia.org/wiki/Data_transfer_object?utm_source=chatgpt.com#cite_note-fowler-2" style="pointer-events: none">) i</a>s an object that carries data between processes. The motivation for its use is that communication between processes is usually done resorting to remote interfaces (e.g., web services), where each call is an expensive operation.<a target="_self" rel="noopener noreferrer nofollow" href="https://en.wikipedia.org/wiki/Data_transfer_object?utm_source=chatgpt.com#cite_note-fowler-2" style="pointer-events: none"><sup>[2]</sup></a> Because the majority of the cost of each call is related to the round-trip time between the client and the server, one way of reducing the number of calls is to use an object (the DTO) that aggregates the data that would have been transferred by the several calls, but that is served by one call only.</div>
</div>

[wikiedia에서 발췌한 글](https://en.wikipedia.org/wiki/Data_transfer_object?utm_source=chatgpt.com)이다. 핵심만 뽑아보자면 다음과 같다.

> DTO는 프로세스 간 데이터를 전달하기 위한 객체이며,  
> 원격 인터페이스 기반 통신에서 발생하는 비싼 호출 비용을 줄이기 위해 사용된다.

핵심만 정리하면 이렇다.

* 프로세스 간 통신은 보통 원격 인터페이스(Web Service 등)를 사용한다
    

* 호출 1회마다 비용이 크다
    
    * 네트워크 왕복 시간
        
    * 직렬화 / 역직렬화
        
    * 인증, 로깅, 레이트 리밋 같은 부가 오버헤드
        
* 그래서 여러 호출로 나뉠 데이터를 DTO 하나로 묶어 한 번에 전달한다
    

즉, 아래로 요약해볼 수 있겠다

* DTO → 데이터를 전달하기 위한 그릇
    
* DTO Pattern → 여러 번의 호출을 한 번의 호출로 줄이기 위해 데이터를 묶어서 전달하는 패턴
    

## DTO 본질에 집중하기

DTO는 클라이언트가 필요로 하는 데이터를, 가장 효율적인 형태로 전달하기 위해 존재한다.

그런데 우리는 DTO를 설계할 때 종종 이런 고민을 한다.

* 이 필드는 어느 Aggregate에 속하지?
    
* 이 DTO가 여러 도메인을 참조해도 괜찮을까?
    
* 이 구조가 도메인 경계를 침범하는 건 아닐까?
    

물론 이런 고민 자체는 나쁘지 않다고 생각한다. (장 단점이 꽤나 명확한거같음)  
다만, 이 고민이 DTO 설계를 멈추게 만들거나, 불필요하게 복잡한 구조를 낳고 있다면 필자는 잠깐 멈추고 생각해볼 필요가 있다고 생각한다.

* DTO는 도메인 경계를 설명하기 위한 객체가 아니다.
    
* DTO는 도메인 모델을 보호하기 위한 객체도 아니다.
    
* ***DTO는 계약(contract)이다.***
    

그리고 필자는 그 계약은 상대방이 우선되어 설계되어야 한다고 생각한다

## 현실의 네트워크는 이상적이지 않다

우리 시스템의 고객(병원)은 네트워크 환경이 상상 이상으로 느리다.  
(가끔 느린 수준이 아니라, 병원 열 곳 중 아홉곳은 버벅임 때문에 정상 사용이 어려울 정도임)

이런 환경에서 도메인 모델을 닮은 DTO를 여러 개 쪼개서 내려보내는 것은, 아키텍쳐적으로는 아름다울 수 있어도 매우 비효율적이다.

## 그래서 우리는 이런 선택을 한다

* 여러 도메인의 데이터를 하나의 DTO를 묶는다
    
* 화면 하나에 필요한 데이터를 한 번의 호출로 내려준다
    
* 도메인 간 의존성은 DTO 레벨에서 허용한다
    

도메인 간 의존성, 트랜잭션 경계, 비지니스 규칙은 여전히 서버 내부에서 관리된다. 그래서 DTO는 그 결과를 외부에 전달하는 표현일 뿐, 도메인 모델처럼 순수해야 한다고 생각하기 시작하면 DTO는 본래 목적을 잃고, API는 호출 횟수가 늘어나며, 성능 문제를 유발할 수 있다.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1769930858313/fc856904-1ddf-4cce-b9b4-d68663cccfb4.png align="center")

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1769931002587/68aa0910-48be-4000-9862-a86541461651.png align="center")

참고로, 위와 같이 현업 상황 상 백엔드 리소스가 없다는것을 알고, (센스있게) 백엔드 엔지니어 의존없이 진행하기위해 FE엔지니어 판단하에 여러 API를 조합해서 매꾸는 선택도 자주 발생할 수 있는데 이것은 여러가지 비효율적인 문제가 있긴하지만 (최고보다는) 최선의 결정이라고 생각한다.

다만, 이 선택에 대한 공유만 잘 이루어지고 백로그 테스크만 잘 잡아둔다면 될거같다. 이 백로그 작업을 하는것은 더 낮은 레이턴시로 고객 경험을 올리고, 시스템의 불필요한 부하도 줄일 수 있기때문에 챕터 미션으로만 지속적으로 관리되면 될거같다는 의견이다.

이 내용을 토대로 팀 내 FE 개발자분들과 이야기를 나눠보려고한다. 아마 모두가 동일한 생각과 기준을 갖고있겠지만 현실은 너무 바쁘고, 정신없기때문에 우리가 놓친게 있는지 되새김질을 하기 위해서말이다.