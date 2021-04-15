---
title: ES6+ 비동기 프로그래밍
categories: [languages, javascript]
tags: [javascript, es6, programming, exception]     # TAG names should always be lowercase
---

# 개요

자바스크립트의 버전 ES6 이상에서의 비동기 처리와 에러 핸들링에 대해 코드를 예시로 가이드를 제공한다.

* 자바스크립트에 대한 기본적인 지식 필요
* 제너레이터, 이터레이터, 이터러블, 함수형 프로그래밍, 특히 순수함수에 대해 선행학습을 권장한다.
* 순수 함수 등의 함수형 프로그래밍 지식에 대해 서술하는 것은 주제를 벗어나므로 해당 일감에선 배제한다.

# 가이드

## 문제

만약 외부 이미지의 높이를 합산해야되는 업무가 있다고 가정하자.
자바스크립트로 작성하게 될 경우, 아래와 같은 코드가 기본적으로 연상될 것이다.

[코드 1]
```javascript
const {log, clear} = console;

// 올바른 객체
const imgObjs = [
  { name: "HEART", url: "https://s3.marpple.co/files/m2/t3/colored_images/45_1115570_1162087_150x0.png" },
  { name: "6", url: "https://s3.marpple.co/f1/2018/1/1054966_1516076919028_64501_150x0.png"},
  { name: "하트", url: "https://s3.marpple.co/f1/2019/1/1235206_1548918825999_78819_150x0.png" },
  { name: "도넛", url:"https://s3.marpple.co/f1/2019/1/1235206_1548918758054_55883_150x0.png"},
];

// 잘못된 요소가 포함된 객체
const imgObjs2 = [
  { name: "HEART", url: "https://s3.marpple.co/files/m2/t3/colored_images/45_1115570_1162087_150x0.png" },
  { name: "6", url: "https://s3.marpple.co/f1/2018/1/1054966_1516076919028_64501_150x0.jpg"},  // <-- png를 jpg로 수정
  { name: "하트", url: "https://s3.marpple.co/f1/2019/1/1235206_1548918825999_78819_150x0.png" },
  { name: "도넛", url:"https://s3.marpple.co/f1/2019/1/1235206_1548918758054_55883_150x0.png"},
];

// 모든 이미지의 높이를 더하는 함수
const getImageHeight = function() {
  return imgObjs.map(obj => {
	let img = new Image();
  	img.src = obj.url;
  	return img;
  })
  .map(img => img.height)
  .reduce((cur,acc) => cur+acc, 0)
};

log(getImageHeight());
```

하지만 위 코드를 실행하면 결과값은 0이 나온다. 왜일까?
new Image()를 할당한 객체는 페이지 로드 이후에 image를 불러오게 된다.
이 과정에서 io 이벤트가 발생하여 비동기로 동작하게 된다.
아래 코드를 추가하면 onload 이후에 img에서 height이 정상 출력됨을 확인할 수 있다.

[코드 2]
```javascript
img.onload = function() {
  	log(img.height);
}
```

그렇다면 이 비동기 동작을 어떻게 동기적으로 처리해야 할까?

## 나쁜 코드 예시

Promise 객체를 리턴해주면 resolve 트리거가 올때까지 wait 상태가 된다.
이를 이용해 비동기 처리를 할 수 있다.

[코드 3]
```javascript
    // 이미지 로드 함수
    const loadImage = url =>
      new Promise((resolve, reject) => {
        let img = new Image();
        img.src = url;
        img.onload = function() {
          resolve(img);
        };
        img.onerror = function(e) {
          reject('error!!!');
        };
      });

    // 이미지 높이 더하는 함수
    const getImageHeight = function() {
      return imgObjs
        .map(async obj => (await loadImage(obj.url)).height)
        .reduce(async (cur, acc) => (await cur) + (await acc), 0);
    };

    getImageHeight().then(log);
```

먼저 load 동작을 함수로 선언한 다음 promise type으로 래핑했다.
그 후, 함수 호출부는 async/await으로 처리하였다.
585가 출력되었다면 정상적으로 출력된 것이다.

하지만 이는 그리 좋은 코드가 아니다.
예시로 작성해놓은 imgObjs2를 보면, 파일 하나가 jpg로 되어 있는데 이는 원래 png 파일을 오동작하도록 임의로 변경한 것이다.
imgObjs2를 getImageHeight의 리턴 객체에 넣어보면 에러가 발생한다.

잘못된 이미지 파일을 걸러내기 위해 try / catch 로 처리해보자

[코드 4]
```javascript
const getImageHeight = function() {
      try {
        return imgObjs2  // <-- 잘못된 객체로 변경
          .map(async obj => {
            try {
              return (await loadImage(obj.url)).height;
            } catch (e) {
              log(e);
              throw e;
            }
          })
          .reduce(async (cur, acc) => (await  cur) + (await acc), 0);
      } catch (e) {
        if (e === 'error!!!') {
          log(0);
        }
      }
    };
```

try / catch 로 문제가 발생하는 부분을 묶어 throw로 던져주고, 바깥에서 이를 처리하였다.
모양새가 별로지만 그럭저럭 에러 핸들링이 된 느낌이다.
하지만 이 코드에는 에러 핸들링 이전보다 더욱 치명적인 결함이 있다.
img.onload 함수 위에 로그를 찍어보자.

[코드 5]
```javascript
        ...
        img.src = url;
        log('함수 호출!!!', url)
        img.onload = function() {
          resolve(img);
        };
        ...
```

출력되는 로그를 살펴보면 에러가 발생함에도 이후 반복문이 계속 동작함을 알 수 있다.
에러 핸들링으로 에러를 처리했기 때문에 우리는 예외 처리가 완료되었다고 생각하지만, 이는 더 큰 결함을 발생시킬 수 있는 가능성을 내제하고 있다.
생각보다 이런 경우는 우리가 개발할 때 흔히 발생할 수 있는 케이스이며, 대부분의 자바스크립트 개발자는 한번쯤 이렇게 코드를 생산해본 경험이 있을 것이다.
그렇다면 과연 이 비동기 동작과 에러를 어떻게 처리하는게 깔끔한 방법일까?

## 좋은 코드 예시

먼저 방금 전의 코드를 살펴보자.

[코드 6]
```javascript
    const getImageHeight = function() {
      return imgObjs
        .map(async obj => (await loadImage(obj.url)).height)
        .reduce(async (cur, acc) => (await cur) + (await acc), 0);
    };
}

getImageHeight2();
```

이곳에서 map 함수를 직접 구현해볼 것이다.
map 함수는 제너레이터를 이용해서 구현해보았으며, reduce 함수는 잠시 변수 형태로 풀어서 작성하였다.
내부 map(오른쪽)은 imgObjs 배열을 받아서 img 객체를 로드하여 배열 형태로 반환한다.
외부 map(왼쪽)은 받아온 img 객체에서 height를 뽑아내어 배열 형태로 반환한다.

여기서 중요한 것은 map 함수인데
* 이터레이터를 추상화하고, 
* 바깥에서 일으키고자 하는 부수효과를 내부에서 발생시키고,
* 어떤 값을 꺼내갈 것이냐를 직접 기입하는 것이 아니라 함수에게 완전히 위임한다.
이것이 오늘의 핵심이다.

[코드 7]
```javascript

// 제너레이터를 이용한 map 함수 구현
function* map(f, iter) {
  for (const val of iter) {
    yield f(val);
  }
}

// reduce를 풀어서 작성하고 try/catch 추가
async function getImageHeight2() {
  try{
    let acc = 0;
    for await (const val of map(img => img.height, map(({url})=> loadImage(url), imgObjs))) {
      acc = acc + val;
    }
    log(acc);
  } catch(e) {
    log(0);
  }
  
}
getImageHeight2();
```

위 코드를 실행하면 결과가 NaN으로 출력될 것이다. 왜일까?
구현한 map 함수 반복문 내부에서 로그를 찍어보면 다음과 같이 출력되는 것을 확인할 수 있다.

![로그 출력 결과](/images/picture232-1.png)

JSON 객체 타입으로 출력된 것은 내부 map(오른쪽)이 실행됨에 따라 그 이터레이터의 value 값이 출력된 것이고,
Promise 타입으로 출력된 것은 외부 map(왼쪽)이 실행되면서 내부 map에서 리턴된 타입이 Promise 타입이기에 저런 결과가 나온 것이다.
이것은 구현한 map에서 이터레이터 값의 타입에 따라 그에 맞는 동작을 수행하도록 처리하여 해결한다.

[코드 8]
```javascript
// 인스턴스 타입에 따라 분기 처리
function* _map(f, iter) {
  for (const val of iter) {
    yield val instanceof Promise ? val.then(f): f(val);
  }
}
```

실행해보면 정상 출력됨을 확인할 수 있다.

이제 reduce도 직접 정의해보자
비동기를 다룰 수 있는 reduce를 제작하기 위해 reduceAsync 함수를 정의한다.

map과 마찬가지로 
* 이터레이터를 추상화하고, 
* 바깥에서 일으키고자 하는 부수효과를 내부에서 발생시키고,
* 어떤 값을 꺼내갈 것이냐를 직접 기입하는 것이 아니라 함수에게 완전히 위임한다.

[코드 9]
```javascript
async function reduceAsync(f, acc, iter) {
	for await (const val of iter) {
		acc = f(acc,val);
	}
	return acc;
}
```

이렇게 map과 reduce를 구현하면 getImageHeight함수를 깔끔하게 정리할 수 있다.
`내부에서의 try / catch문을 제거하고 함수 호출부문에서 예외처리를 하도록 유도한다.`

[코드 10]
```javascript
const getImageHeight2 = imgs => 
    reduceAsync((cur,acc) => cur + acc, 0,
  	      map(img => img.height,
  	        map(({url}) => loadImage(url), imgs))
  	)

getImageHeight2(imgObjs).catch(e => log('err')).then(log);
getImageHeight2(imgObjs2).catch(e => log('err')).then(log);
```

왜 이렇게 기본 메소드를 힘들게 구현했을까?
이전 img.onload 이전 찍었던 로그가 남아있다면 이유를 알 수 있다.

![로그 출력 결과](/images/picture232-2.png)

함수 호출이 진행되다가 에러를 만나자 에러를 발산하고 반복이 중단된다.

# 정리

지금까지 비동기 처리와 에러 핸들링에 대해 다루어보았다.
이 예시의 핵심은 `에러를 분출시킨다` 는 것이다.
함수 내부에서 try / catch나 if 문으로 에러 핸들링을 하는 것은 함수의 용도를 제약시키고 코드를 사용하는 유저에게 확장성 있는 기능을 제공하기 어렵다.
* 어떤 유저는 글로벌 필터에서 에러 핸들링을 하길 원할 수 있다.
* 어떤 유저는 함수 호출부에서 에러 핸들링을 하길 원할 수 있다.
만약 함수 내부에서 나쁜 예시와 같은 부수효과를 작성하게 될 경우 위 케이스들과 같은 유저들은 절대 원하는 값을 얻을 수 없다.
심지어 내부의 try/catch에서 throw가 실수로 빠지는 등 어설프게 작성되었다면 치명적인 결과를 초래할 수 있다.
`순수함수` 로 작성했다면 그 함수는 올바른 input이 들어왔다면 반드시 보장된 output이 존재하기 때문에 예외처리는 불필요하며 위와 같은 이유로 해서도 안된다.

지금까지의 내용을 정리해보면 다음과 같다.

1. Promise, async/await, try/catch를 정확히 다루는 것이 중요하다.
2. 제너레이터/이터레이터/이터러블을 잘 응용하면 코드의 표현력을 더할 뿐 아니라 에러 핸들링도 더 잘할 수 있다.
3. 순수 함수에서는 에러가 발생되도록 그냥 두는 것이 더 좋다.
4. 에러 핸들링 코드는 부수효과를 일으킬 코드 주변에 작성하는 것이 좋다.
5. 불필요하게 에러 핸들링을 미리 해두는 것은 에러를 숨길 뿐이다.
6. 발생되는 모든 에러를 확인하는 것이 고객 입장에선 좋을 수 있다.(sentry.io)

# ○ 참고문서

* 함수형 자바스크립트 프로그래밍. 유인동 저 | 인사이트(insight)
* 
[함수형 프로그래밍과 ES6+](https://www.youtube.com/watch?v=4sO0aWTd3yc&ab_channel=naverd2&fbclid=IwAR0b7hHb8AwexfL1PUKs2r6wKAg6gI1il5HI5oJ5HA81hdEoFeovQ3Q7jN4)
