---
title: Subject vs EventEmitter
categories: [framework, angular]
tags: [angular, observable, subject, eventemitter]     # TAG names should always be lowercase
---

지난 포스트에서 언급했다시피 `얼마전 서로 다른 컴포넌트에서 상태를 전달하기 위해` 코드 작성 중 의문이 생겼다.
Angular에서 컴포넌트간 데이터를 전달할 때 3가지 방식이 있다.

1. EventEmitter
2. service 패턴
3. RxJS

이번 포스트는 위 세가지 방법의 차이점에 대해 소개해볼 생각이다.

# EventEmitter

EventEmitter는 아래와 같이 선언하고,
> @Output() postData = new EventEmitter()

아래와 같이 listen한다.
> @Input() listPost = [];

![사용 예시](https://stackblitz.com/edit/angular-post-app-event-emitter-5z4t15)

위 방식은 부모-자식 관계의 component들이 서로간의 데이터를 전달할 때
Angular에서 소개하는 방식이다.
타입 지정이 자유롭고 간편하게 사용할 수 있다.
다만, Output/Input에 대한 명시적인 처리는 없어서(내가 모르는건지...)
둘 중 하나만 선언해도 빌드 에러가 발생하진 않는다.

# service 패턴

service 패턴은 두 component가 공유하는 한 service를 생성하는 것이다.

![사용 예시](https://stackblitz.com/edit/angular-service-to-service-communication-mxbp4i)

이 경우, 두 component들은 해당 service에 의존성을 가지게 되기 때문에 높은 결합력을 가진다.

# RxJS

subject를 이용해 이벤트를 발생시키고, subscribe를 통해 구독하여 데이터를 전달받는다.

![사용 예시](https://stackblitz.com/edit/angular-service-to-service-communication-xlnw6r)

RxJS에 대해 능숙하고 구독/발행 패턴에 대한 이해도가 깊다면 강력한 기능들을 만들 수 있다.

# 결론

service 패턴은 그렇다치고...
RxJS와 EventEmitter의 동작이 어떤 차이가 있는지 몰라서 찾아보았더니
둘은 완벽하게 동일하며(!) 내부적으로 EventEmitter는 RxJS의 객체를 상속받아 super로 호출하여 사용중이었다.
차이가 있다면 EventEmitter는 _isAsync라는 프로퍼티가 추가되어 있었는데
아마도 Angular에서 비동기 방식을 다룰 때 사용할 것으로 예상된다. (필자는 한번도 사용해보진 않았다.)

그래도 Angular가 부모-자식 관계의 Component에 대해 EventEmitter를 소개한 이유가 있을거라 생각되어
부모-자식 관계의 데이터 전달은 EventEmitter, 의존성 주입이 필요할 땐 Service,
그 외엔 RxJS를 사용하기로 정하였다.
