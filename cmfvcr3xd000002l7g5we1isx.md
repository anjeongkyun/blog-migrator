---
title: "Gateway의 Connection Pool 고갈, 원인은 다운 스트림의 GC Pause였다"
seoTitle: "connection pool depletion was caused by downstream GC pauses"
datePublished: Mon Sep 22 2025 16:40:16 GMT+0000 (Coordinated Universal Time)
cuid: cmfvcr3xd000002l7g5we1isx
slug: gateway-connection-pool-gc-pause
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1758559287250/490c202b-dfbb-4f85-8f8d-f95fde5a34ae.png
ogImage: https://cdn.hashnode.com/res/hashnode/image/upload/v1758559312408/02265b2e-488b-4731-ac15-b564609d4c4e.png
tags: node-js, troubleshooting, webclient

---

## 문제 상황

Gateway 에러 그래프를 보니, 특정 시점에 수천 건의 에러가 짧은 시간 동안 집중적으로 발생했습니다.

> ##### **Gateway Errors Graph 이미지 첨부**

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1758555065318/4328d1a0-b7bb-47eb-b5d4-6ba8047274d1.png align="center")

로그를 확인해보니 아래 메시지가 반복적으로 찍히고 있었죠. 그것도 이틀 연속 간헐적으로 짧은 시간 동안 장애가 발생했어요.

```plaintext
org.springframework.web.reactive.function.client.WebClientRequestException: 
Pending acquire queue has reached its maximum size of 500
```

처음엔 단순한 네트워크 문제 같았지만, 이 로그는 사실 **WebClient의 Connection Pool 대기열이 꽉 찼다**는 뜻이었습니다.  
즉, Gateway에서 Account API로 요청을 보냈지만, **커넥션을 빌릴 수 없어서 요청 자체가 실패**한 것이죠.

### **원인 추적**

그럼 왜 커넥션 풀 고갈이 일어났을까요?  
에러 시점을 기준으로 Account API 지표를 확인해봤습니다.

> **Account API Metrics 이미지 첨부**

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1758556616621/4f58dbcb-79d5-4acc-8942-be692ca2261f.png align="center")

* **응답 지연**: 특정 시간대에 N초 이상 늘어남
    
* **Event Loop Delay**: 200ms 이상 급증
    

즉, Account API 자체가 제때 응답을 돌려주지 못했고, 이로 인해 Gateway에서 커넥션이 반환되지 못한 겁니다.

### 근본 원인

문제는 트래픽 패턴에 있었습니다.

서비스 특성상 예기치 못한 트래픽 피크는 드물었는데, 주간 트래픽 그래프를 보니 평소 대비 **6배 이상 급증**한 흔적이 있었습니다.

> **Gateway 주간 트래픽 그래프**

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1758555113006/b4d2a09f-4375-45ac-92d1-a9916f2dbf2c.png align="center")

참고로 제가 만들고있는 제품은 특정한 시간에 트래픽 피크를 치거나 하는 서비스는 아닙니다. (물론, 병원에서 가장 많이 이용하는 시간대는 있지만, 예상치못한 트래픽이 갑작스레 급증하거나 하진 않습니다.)

Gateway의 Weekly 트래픽 지표를 토대로 급증한 트래픽의 요청을 집계해보니 16일 늦은 저녁, 계정의 상태 변경을 최신화하기위해 클라이언트의 폴링 논리가 배포 영향이였던것이였죠.

### 대응

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1758556333547/114ebcf0-b6da-4078-8ffc-643348f77e5a.png align="center")

일단 상황을 빠르게 팀원분들께 공유했고, 세번째 프로덕션 장애를 겪지않기위해 Account API Scale-out과 HPA 검토를 진행했습니다.

또한, 기존 타임아웃이 5초로 설정돼 있어 커넥션이 오래 붙잡히던 문제를 해결하기 위해 타임아웃을 줄이는 응급조치도 병행했습니다.

## 이제부터 진짜 본론입니다

### 왜 Event Loop Delay가 그렇게 크게 발생했을까요?

이제부터 Node.js 런타임 내부 동작, GC Pause, 그리고 Event Loop Delay에 대한 이야기를 시작하려고 합니다.

Node.js 애플리케이션은 단일 스레드 기반 이벤트 루프 모델로 동작합니다.

즉, 모든 요청 처리는 결국 이벤트 루프라는 한 줄짜리 실행 트랙 위에서 순차적으로 흘러갑니다. 따라서, 이벤트 루프가 잠깐이라도 막히면, 동시에 들어오는 요청 자체가 영향을 받게됩니다. 이번 장애 상황의 핵심도 바로 여기에 있었습니다.

### Node.js 이벤트 루프의 내부

Node.js는 구글의 V8 자바스크립트 엔진 위에서 동작하며, 비동기 I/O를 담당하는 libuv 이벤트 루프를 가지고 있습니다. 이 이벤트 루프는 다음과 같은 단계(phase)를 매 tick마다 반복합니다.

1. Timers - `setTimeout`, `setInterval` 콜백 실행
    
2. I/O Callbacks - 파일/ 네트워크 요청 결과 처리
    
3. Idle, Prepare - 내부 준비 단계
    
4. Poll - 새로운 I/O 이벤트 대기 및 실행
    
5. Check - `setImmediate` 실행
    
6. Close Callbacks - 소켓 close 이벤트 처리
    

이 사이클이 빠르게 돌아가며 요청을 처리하는데, 보통 한 tick은 수 밀리초도 안걸리는게 일반적입니다. 하지만 문제는 어떤 이유로든 한 tick이 길어지면, 전체 응답성이 무너지게됩니다.

### Event Loop Delay란?

Event Loop Delay는 이벤트 루프가 제때 tick을 돌지 못했을 때 발생합니다.

예를 들어,

* 원래는 10ms 단위로 루프가 돌아야 하는데
    
* 어떤 작업이 200ms를 잡아먹었다면
    
* 그 순간 event loop delay는 190ms로 기록되게됩니다
    

즉, delay 지표는 `이벤트 루프가 얼마나 늦게 돌고 있는가`를 보여주는 신호(Signal)입니다.

이번 Account API의 장애 시점에서 이 값이 200ms 이상 치솟은 건, 루프가 그만큼 멈췄다는 뜻인거죠.

### 그래서 왜 멈춘건데? (GC Pause)

Node.js는 메모리를 자동으로 관리합니다. V8 엔진은 가비지 컬렉션(GC)을 통해 사용하지 않는 객체를 정리하는데, 이 과정에서 stop the world (일명 STW)가 발생하게됩니다.

#### **GC의 두 가지 단계**

1. Minor GC (Scavenge)
    
    * 새로 생성된 객체가 머무는 Young Generation 영역 청소함
        
    * 보통 빠르고 짧다 (몇 ms 정도)
        
2. Major GC (Mark-Sweep-Compact)
    
    * 오래 살아남은 객체들이 쌓이는 Old Generation 청소함
        
    * 객체 마킹 → 불필요 객체 제거 → 메모리 조각 압축
        
    * 이 과정에서 수십~수백 ms 동안 JS 실행이 멈추게 됨
        

여기서, GC Pause는 바로 이 멈춘 시간을 뜻합니다.

GC Pause는 곧바로 Event Loop Delay로 드러나게됩니다. 즉, 모든 GC Pause는 Event Loop Delay를 만듭니다.

### **이번 장애에서 벌어진 일?**

장애 시점의 Account API 지표를 다시 보면,

* Event Loop Delay 급증 (200ms+)
    
* GC Pause Count & Time 동반 증가 (40~50ms)
    

이 패턴은 전형적으로 Major GC가 발생하면서 이벤트 루프가 멈춘 상황을 보여줍니다.

즉, Account API는 트래픽이 몰리면서 메모리 할당/ 해제가 많아졌고, 그 결과 V8이 Major GC를 실행하고, 그 순간 이벤트 루프가 수백 ms 동안 멈추면서 들어온 요청들이 줄줄이 소세지로 대기 상태로 밀려버린것이죠.

### 그래서 Gateway가..

이걸 앞선 Gateway 문제와 연결해보면 다음과 같습니다.

1. Account API에서 GC Pause → Event Loop Delay 발생
    
2. 응답이 늦어짐 → Gateway WebClient가 커넥션을 붙잡고 대기
    
3. Pool에 여유 커넥션이 없음 → Pending Queue로 요청 밀림
    
4. Queue까지 한계치 도달 → `Pending acquire queue has reached its maximum size` 에러 발생
    
5. 클라이언트의 요청(Request)은 실패 응답(5xx)를 받게됨
    

즉, Gateway에서 본 문제는 사실 Account API 런타임 내부의 이벤트 루프 지연이 근본 원인이였던 겁니다.

### **근데 진짜 이것의 문제는..**

Account API는 모든 요청이 반드시 거쳐야 하는 계정/ 보안 서비스입니다.

여기서 이벤트 루프가 멈추면, 사실상 서비스 전체가 멈추는 것과 같습니다.

일반적인 도메인 집중적인 마이크로서비스라면 Circuit Breaker로 요청을 끊어버리는 선택을 할 수 있겠지만, Account API는 그럴 수조차 없습니다. (물론 회로 차단을 정말 잘 조절하면 효과는 볼 수 있겠지만요)

따라서, 근본 원인을 토대로 GC Pause와 Event loop Delay를 줄이는것이 서비스 안정성의 핵심 과제라고 생각됩니다.

### **대응 전략**

이번 장애에서 저희가 취한 응급 대응은 크게 두 가지였습니다.

1. 타임아웃 단축
    
    * 기존에는 Gateway WebClient의 타임아웃이 5초였습니다
        
    * 이 설정 때문에 Account API가 응답하지 못하면 커넥션을 최대 5초 동안 붙잡게 되었고, 그 결과 풀 고갈로 이어졌습니다.
        
    * 이를 단축해 Fail-Fast 전략으로 바꿔, 지연이 전체 시스템에 전파되는 걸 차단했습니다.
        
2. Account API Scale-out
    
    * Account API는 앞서 말했듯이, 모든 서비스 요청의 관문이기 때문에 단일 노드에서 발생하는 이벤트 루프는 곧바로 전체 장애로 이어집니다.
        
    * 따라서, Account API를 확장해 트래픽 분산을 수행하도록 했습니다. (기존 Pod Quantity \* 2)
        
    * Pod 수를 늘림으로써, 특정 인스턴스에서 GC Pause가 발생하더라도 이것이 전체 장애로 확산되는 것을 방지하고자 했습니다.
        

### 근본적인 개선 아이디어

위의 응급 대응으로 어느 정도 문제를 완화했지만, Event Loop Delay와 GC Pause는 언젠가/ 언제든지 다시 반복될 수 있습니다. 따라서 장기적으로 아래와 같은 접근이 필요하다고 생각했습니다.

1. 메모리 프로파일링 및 최적화
    
    * heap snapshot, heamdump 등을 활용해 어떤 객체가 주로 메모리를 점유하는지 확인합니다
        
    * json 직렬화/역직렬화 등에서 불필요한 객체 생성/ 복사가 있었다면 개선 대상으로 잡습니다.
        
2. GC 튜닝
    
    * Node.js 실행 시 `—max-old-space-size` 옵션으로 Heap 크기를 조정해 Major GC 빈도를 낮춥니다
        
3. 캐싱 전략
    
    * 동일한 계정 활성화, 또는 IP 검증 등이 반복된다면, 캐시 계층을 두어 불필요한 CPU/ Memory 소모를 줄일 수 있습니다.
        
    * 실제로 모든 API들이 Account API를 통할 때, 기본적으로 5번의 DB Query가 발생합니다. (Allowed-IP / MFA / Account 활성화 등)
        

(모아두기)

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1758555556736/ee1eab99-fcfb-41f9-8f8e-f4a38569c00a.png align="center")