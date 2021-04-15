---
title: 스코프, 블럭 레벨/함수 레벨 스코프, lexical/동적 스코프, 스코프체인, 클로저
categories: [languages, javascript]
tags: [javascript, scope]     # TAG names should always be lowercase
---

# 스코프

`식별자(변수)의 유효범위`

* 자바스크립트 변수에 대한 접근 권한을 정의하는 것이다.
* `global` 영역과 `local` 영역으로 나뉜다.
  * global: 코드 어디에서든지 참조 가능
  * local: 함수 코드 블록이 만든 스코프, 함수 자신과 하위 함수에서만 참조 가능
* var로 변수를 선언할 경우, `hoisting`이 발생하며, 이는 식별자에 영향을 줄 수 있다.

# 블럭 레벨 스코프 vs 함수 레벨 스코프

```javascript
function foo (a) {
  if(a) {
    // let b = 'let';
    var c = 'var';
  }
  // console.log(b);
  console.log(c);
}

foo(true);
```

C언어는 `블록 레벨 스코프`를 따른다. 하지만 자바스크립트는 `함수 레벨 스코프`를 따른다.

* |-|var|let|const|
  |scope|함수 레벨|블록 레벨|블록 레벨|
  |재선언|O|X|X|
  |재할당|O|O|X|
* **블록 레벨 스코프**: 모든 코드 블록(함수, if 문, for 문, while 문, try/catch 문 등) 내에서 선언된 변수는 `코드 블록 내에서만 유효`하며 코드 블록 외부에서는 참조할 수 없다. 
즉, 코드 블록 내부에서 선언한 변수는 지역 변수이다.
foo => let 출력 시 에러 발생
* **함수 레벨 스코프**: 함수 내에서 선언된 변수는 `함수 내에서만 유효`하며 함수 외부에서는 참조할 수 없다. 
즉, 함수 내부에서 선언한 변수는 지역 변수이며 함수 외부에서 선언한 변수는 모두 전역 변수이다.
foo => var 출력


# lexical/동적 스코프

```javascript
var x = 1;

function foo() {
  var x = 10;
  bar();
}

function bar() {
  console.log(x);
}

foo(); // ?
bar(); // ?
```

* **동적 스코프(Dynamic scope)**: 함수를 어디서 호출하였는지에 따라 상위 스코프를 결정
  * foo => 10
  * bar => 1
* **렉시컬 스코프(Lexical scope)**: 함수를 어디서 선언하였는지에 따라 상위 스코프를 결정
  * foo => 1
  * bar => 1
* 자바스크립트는 렉시컬 스코프를 따른다.

# 스코프 체인

* **Identifiers(식별자)를 찾는 일련의 과정**
* `[[Scopes]]`로 참조 

![스코프 체인](/images/scope_chain.png)
![스코프 체인](/images/scope_chain2.png)
![스코프 체인](https://baeharam.github.io/images/javascript/sc/sc1.png)

# 클로저

* **독립적인 변수를 가리키는 함수**

> 클로저 = 함수 + 함수를 둘러싼 환경(Lexical enviornment)

```javascript
for (let i = 0; i < 10; i++) {
  (function (j) {
    setTimeout(function () {
      console.log(j);
    }, 100);
  })(i);
}
```
