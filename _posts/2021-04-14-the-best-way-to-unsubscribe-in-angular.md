---
title: Angular에서 unsubscribe를 사용하는 방법
categories: [framework, angular]
tags: [design-pattern, angular, subscribe, unsubscribe, memory-leak]     # TAG names should always be lowercase
---

# 개요

※ 이 포스트는 ![6 Ways to Unsubscribe from Observables in Angular](https://blog.bitsrc.io/6-ways-to-unsubscribe-from-observables-in-angular-ab912819a78f) 를 번역 및 가공하여 작성하였습니다.

얼마전 서로 다른 컴포넌트에서 상태를 전달하기 위해 `subscribe`를 사용할 일이 있었다.
일반적으로 `subscribe`는 `component의 생명주기`가 다하는 순간 `unsubscribe`되면서 파괴되지만 종종 그렇지 않을 경우도 있다. 이땐 수동으로 unsubscribe를 해줘야 한다. 그렇지 않으면 불필요한 리소스를 소모하여 **앱 성능을 저하**시키고, **메모리 누수가 발생**할 수 있다.

* 메모리 누수 예제
```javascript
@Component({...})
export class AppComponent implements OnInit {
    subscription: Subscription
    ngOnInit () {
        var observable = Rx.Observable.interval(1000);
        this.subscription = observable.subscribe(x => console.log(x));
    }
}
```

1초 간격으로 콘솔을 찍어주는 옵저버를 구독한다고 했을 때 이 녀석은 `AppComponent`가 파괴된 이후에도 동작한다.

그럼 이 구독자를 어떻게 해제시키는게 좋을까?

# 1. unsubscribe 메소드를 사용하자

> Observable에는 기본적으로 리소스를 해제하거나 Observable 실행을 취소 하는 unsubscribe()기능이 있다.

Observable에서 unsubscibe() 메소드를 호출하면 해당 구독은 취소된다.
Angular에선 일반적으로 `OnDestroy` 타이밍에 취소되지만 그렇지 않을 경우, 명시적으로 취소해야 한다.


```javascript
@Component({...})
export class AppComponent implements OnInit, OnDestroy {
    subscription: Subscription
    ngOnInit () {
        var observable = Rx.Observable.interval(1000);
        this.subscription = observable.subscribe(x => console.log(x));
    }
    ngOnDestroy() {
        this.subscription.unsubscribe()
    }
}
```

구독 갯수가 여러개일 경우, 배열로 묶거나 아래와 같이 사용할 수 있다.

```javascript
@Component({...})
export class AppComponent implements OnInit, OnDestroy {

    subscription: Subscription
    ngOnInit () {
        var observable1$ = Rx.Observable.interval(1000);
        var observable2$ = Rx.Observable.interval(400);
        var subscription1$ = observable.subscribe(x => console.log("From interval 1000" x));
        var subscription2$ = observable.subscribe(x => console.log("From interval 400" x));
        this.subscription.add(subscription1$)
        this.subscription.add(subscription2$)
    }
    ngOnDestroy() {
        this.subscription.unsubscribe()
    }
}
```

# 2. async | pipe 를 사용하자

`async` 파이프는 `Observable`이나 `Promise`를 구독하고 발행된 마지막 값을 리턴한다.
새로운 값이 발행되면, `async`파이프는 변경사항을 체크할 component를 표시한다.
component가 파괴될 때 구독은 자동으로 취소된다.

async 파이프 사용법:

```javascript
@Component({
    ...,
    template: `
        <div>
         Interval: {{observable$ | async}}
        </div>
    `
})
export class AppComponent implements OnInit {
    observable$
    ngOnInit () {
        this.observable$ = Rx.Observable.interval(1000);
    }
}
```

이 방식의 장점은 구독 취소에 대해 신경쓰지 않아도 component가 파괴될 때 자동으로 파괴된단 점이다.

# 3. RxJS take* 연산자를 사용하자

RxJS엔 Angular 프로젝트에서 `구독/취소`를 선언적으로 관리할 수 있는 유용한 연산자들이 있다.
그중 하나가 take* 연산자들이다.

* take(n)
* takeUntil(notifier)
* takeWhile(predicate)

## take(n)

이 연산자는 파라미터(n)에 주어진 값 만큼만 구독을 수행하고 완료된다.
한정된 횟수만 구독하고 싶을 경우 효과적!

```javascript
@Component({
    ...
})
export class AppComponent implements OnInit, OnDestroy {
    subscription$
    ngOnInit () {
        var observable$ = Rx.Observable.interval(1000);
        this.subscription$ = observable$.pipe(take(1)).
        subscribe(x => console.log(x))
    }
    ngOnDestroy() {
        this.subscription$.unsubscribe()
    }
}
```

값을 발생하지 않을 경우, 구독이 취소되지 않으므로 component가 파괴될 때 명시적으로 취소해준다.

## takeUntil(notifier)

```javascript
@Component({...})
export class AppComponent implements OnInit, OnDestroy {
    notifier = new Subject()
    ngOnInit () {
        var observable$ = Rx.Observable.interval(1000);
        observable$.pipe(takeUntil(this.notifier))
        .subscribe(x => console.log(x));
    }
    ngOnDestroy() {
        this.notifier.next()
        this.notifier.complete()
    }
}
```

이 연산자는 notifier에서 알람이 발생할 때 까지 구독한다.
역시 component가 파괴될 때 구독을 끝내는 코드를 넣어주는 것이 좋다.

## takeWhile(predicate)

```javascript
@Component({...})
export class AppComponent implements OnInit, OnDestroy {
    subscription$
    ngOnInit () {
        var observable$ = Rx.Observable.interval(1000);
        this.subscription$ = observable$.pipe(takeWhile(value => value < 10))
        .subscribe(x => console.log(x));
    }
    ngOnDestroy() {
        this.subscription$.unsubscribe()
    }
}
```

조건을 충족할 때 까지 구독한다.
마찬가지로 component가 파괴될 때 구독을 취소해준다.

# 4. RxJS first 연산자를 사용하자

`first()` 연산자는 take(1) 또는 takeWhile의 역할을 한다.
- 인자 값을 주지 않고 실행하면 take(1) 처럼 동작
- 인자 값에 조건을 주면 takeWhile 처럼 동작

```javascript
@Component({...})
export class AppComponent implements OnInit {
    observable$
    ngOnInit () {
        this.observable$ = Rx.Observable.interval(1000);
        this.observable$.pipe(first(val => val === 10))
        .subscribe(x => console.log(x));
    }
}
```

마찬가지로 컴포넌트가 파괴될 때 구독을 취소해준다.

# 5. 데코레이터를 이용한 구독 취소

위에서 서술한 것과 같이 구독을 따로 취소하지 않을 경우 지속적인 리소스 낭비 또는 메모리 누수가 발생할 수 있다.
데코레이터를 이용해 간단히 기능을 해당 컴포넌트에 주입할 수 있다.

```javascript
function AutoUnsub() {
    return function(constructor) {
        const orig = constructor.prototype.ngOnDestroy
        constructor.prototype.ngOnDestroy = function() {
            for(const prop in this) {
                const property = this[prop]
                if(typeof property.subscribe === "function") {
                    property.unsubscribe()
                }
            }
            orig.apply()
        }
    }
}
```

컴포넌트 내에서 전체 프로퍼티 중 구독 기능을 가진 프로퍼티를 찾아내어 취소해준다.

```javascript
@Component({
    ...
})
@AutoUnsub
export class AppComponent implements OnInit {
    observable$
    ngOnInit () {
        this.observable$ = Rx.Observable.interval(1000);
        this.observable$.subscribe(x => console.log(x))
    }
}
```

더 이상 OnDestroy에서 구독에 대해 신경쓰지 않아도 된다.
subscribe가 없는 Observable일 경우 문제가 발생할 수 있다는데...(세상에 그런건 없을거라 적혀있긴 하지만)
그것도 예외처리만 잘 해주면 문제 없을 듯 하다.

이 포스트에서 가장 마음에 드는 정보이며 이것을 위해 정리했다고 해도 과언이 아닐 정도...

# 6. tsline를 사용하자

lint를 커스터마이징하여 OnDestroy가 없을 경우 에러나 경고를 보낼 수 있다고 한다.

```javascript
// ngOnDestroyRule.ts
import * as Lint from "tslint"
import * as ts from "typescript"
import * as tsutils from "tsutils"
export class Rule extends Lint.Rules.AbstractRule {
    public static metadata: Lint.IRuleMetadata = {
        ruleName: "ng-on-destroy",
        description: "Enforces ngOnDestory hook on component/directive/pipe classes",
        optionsDescription: "Not configurable.",
        options: null,
        type: "style",
        typescriptOnly: false
    }
    public static FAILURE_STRING = "Class name must have the ngOnDestroy hook";
    public apply(sourceFile: ts.SourceFile): Lint.RuleFailure[] {
        return this.applyWithWalker(new NgOnDestroyWalker(sourceFile, Rule.metadata.ruleName, void this.getOptions()))
    }
}
class NgOnDestroyWalker extends Lint.AbstractWalker {
    visitClassDeclaration(node: ts.ClassDeclaration) {
        this.validateMethods(node)
    }
    validateMethods(node: ts.ClassDeclaration) {
        const methodNames = node.members.filter(ts.isMethodDeclaration).map(m => m.name!.getText());
        const ngOnDestroyArr = methodNames.filter( methodName => methodName === "ngOnDestroy")
        if( ngOnDestroyArr.length === 0)
            this.addFailureAtNode(node.name, Rule.FAILURE_STRING);
    }
}
```

이런 식의 lint 정의도 가능하단 것을 오늘 배웠다. 잘만 응용한다면 강력한 lint를 구현할 수 있을지도...

# 결론

subscribe는 생각보다 조심히 다루어야 될 녀석인 것 같다.
예제 코드를 보면 너무도 쉽게 간과될 수 있는 것 처럼 보인다.
5번 방법이 거진 엑스칼리버다. 걍 5번만 기억해도 될듯.

더불어 메모리 누수나 리소스 낭비에 너무 무관심했단 생각이 든다.
다음엔 메모리 누수에 대해 좀더 자세히 조사해볼 생각이다.

