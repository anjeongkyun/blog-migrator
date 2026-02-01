---
title: "현대 아키텍쳐에서 DTO Pattern이 중요한 이유"
seoTitle: "what is DTO pattern"
datePublished: Sun Feb 01 2026 06:21:39 GMT+0000 (Coordinated Universal Time)
cuid: cml3cs0dk000f02jy1eh12fvr
slug: dto-pattern
tags: dto, dto-pattern

---

## DTO Pattern이 뭔가요?

DTO Pattern은 `Sun Microsystems의 J2EE Core Patterns`에서 공식적으로 정리된 개념이다. (2001년 초판: Core J2EE Patterns) 최초 DTO는 분산 환경(EJB, RMI)에서 원격 호출 횟수를 줄이기 위한 목적으로 정의되었다.

이 개념을 2002년 마틴 파울러가 `Patterns of Enterprise Application Architecture` 책에서 DTO Pattern을 소개했다. (마틴 파울러는 창시자 보다는 전파자임)

<div data-node-type="callout">
<div data-node-type="callout-emoji">💡</div>
<div data-node-type="callout-text">In the field of programming a <strong>data transfer object</strong> (<strong>DTO</strong><a target="_self" rel="noopener noreferrer nofollow" href="https://en.wikipedia.org/wiki/Data_transfer_object?utm_source=chatgpt.com#cite_note-msdn-1" style="pointer-events: none"><sup>[1][2]</sup></a><a target="_self" rel="noopener noreferrer nofollow" href="https://en.wikipedia.org/wiki/Data_transfer_object?utm_source=chatgpt.com#cite_note-fowler-2" style="pointer-events: none">) i</a>s an object that carries data between processes. The motivation for its use is that communication between processes is usually done resorting to remote interfaces (e.g., web services), where each call is an expensive operation.<a target="_self" rel="noopener noreferrer nofollow" href="https://en.wikipedia.org/wiki/Data_transfer_object?utm_source=chatgpt.com#cite_note-fowler-2" style="pointer-events: none"><sup>[2]</sup></a> Because the majority of the cost of each call is related to the round-trip time between the client and the server, one way of reducing the number of calls is to use an object (the DTO) that aggregates the data that would have been transferred by the several calls, but that is served by one call only.</div>
</div>

[wikiedia에서 발췌한 글](https://en.wikipedia.org/wiki/Data_transfer_object?utm_source=chatgpt.com)이다. 핵심만 뽑아보자면 다음과 같다.

DTO의 사용 이유는 프로세스 간 통신이 보통 원격 인터페이스를 통해 통신이 이루어지기때문에, 각 호출 비용이 많이 든다. DTO는 이 왕복 시간을 줄이기 위해 여러 통신에서 전송될 데이터를 집계하는데 사용한다.

<div data-node-type="callout">
<div data-node-type="callout-emoji">💡</div>
<div data-node-type="callout-text">Fowler explained that the pattern’s main purpose is to reduce roundtrips to the server by batching up multiple parameters in a single call. This reduces the network overhead in such remote operations.</div>
</div>