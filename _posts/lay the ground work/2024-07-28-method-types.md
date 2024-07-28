# 타입스크립트의 함수

웹 어플리케이션을 구현할 때 자주 사용하는 것으로 3가지 방법으로 타입을 만들 수 있습니다.

- 함수의 파라미터 타입
- 함수의 반환 타입
- 함수의 구조 타입

## 함수의 인자

타입스크립트에서는 함수의 인자를 모두 필수 값으로 판단합니다. 함수의 매개변수를 설정하게 되면 'undefined' 나 'null' 이라도 인자로 넘겨야 합니다. 컴파일러에서 정의된 매개변수 값이 넘어 왔는지 확인하기 때문입니다.
즉, 정의된 매개변수 값만 받을 수 있고 추가로 인자를 받을 수 없다는 것을 의미 합니다.

```ts
function sum(a: number, b: number): number {
  return a + b;
}
sum(10, 20); // 30
sum(10, 20, 30); // error, too many parameters
sum(10); // error, too few parameters
```

정의된 매개변수만큼 인자를 넘기지 않아도 자바스크립트의 특성과 반대됩니다.
이러한 특성을 살리고 싶다면, '?'를 이용하여 사용하면 됩니다.

```ts
function sum(a: number, b?: number): number {
  return a + b;
}
sum(10, 20); // 30
sum(10, 20, 30); // error, too many parameters
sum(10); // 타입 에러 없음
```

매개변수 초기화는 ES6 문법과 동일합니다.

```ts
function sum(a: number, b = "100"): number {
  return a + b;
}
sum(10, undefined); // 110
sum(10, 20, 30); // error, too many parameters
sum(10); // 110
```

## REST 문법이 적용된 매개변수

ES6 문법에서 지원하는 Rest 문법은 타입스크립트에서 아래와 같이 사용할 수 있습니다.

```ts
function sum(a: number, ...nums: number[]): number {
  const totalOfNums = 0;
  for (let key in nums) {
    totalOfNums += nums[key];
  }
  return a + totalOfNums;
}
```

## this

타입스크립트에서 자바스크립트의 'this'가 잘 못 사용되었을 때 감지할 수 있습니다.
'this'가 가리키는 것을 명시하려면 아래의 예제처럼 사용합니다.

```ts
function method(this: type) {
  //...
}
```

위 문법을 사용하여 실제 예제를 만들어 본다면,

```ts
interface Vue {
  el: string;
  count: number;
  init(this: Vue): () => {};
}

let vm: Vue = {
  el: "app",
  count: 30,
  init: function (this: Vue) {
    return () => {
      return this.count;
    };
  },
};

let getCount = vm.init();
let result = getCount();
console.log(result); // 30
```

위 코드를 타입스크립트로 컴파일 했을 때 만일 '--noImplicitThis' 옵션이 있더라도 에러가 발생하지 않습니다.

## 콜백에서의 this

앞에서 봤던 일반적인 상황에서의 'this'와는 다르게 콜백으로 함수가 전달되었을 때의 'this'를 구분해줘야 할 때가 있습니다. 아래와 같이 강제로 할 수 있습니다.

```ts
interface UIElement {
  //아래 함수의 `this: void`는 함수에 `this` 타입을 선언할 필요가 없다는 의미입니다.
  addClickListener(onClick: (this: void, e: Event) => void): void;
}

class Handler {
  info: string;
  onClick(this: Handler, e: Event) {
    //`UIElement` 인터페이스의 스펙에 `this`가 필요없다고 했지만 사용했기 때문에 에러가 발생합니다.
    this.info = e.message;
  }
}

let handler = new Handler();
uiElement.addClickListener(handler.onClick); //error
```

'UIElement' 인터페이스의 스펙에 맞춰 'Handler'를 구현한다면 아래와 같이 수정해야 합니다.

```ts
class Handler {
  info: string;
  onClick(this: void, e: Event) {
    //`this`의 타입이 void이기 때문에 여기서 `this`를 사용할 수 없습니다.
    console.log("clicked");
  }
}

let handler = new Handler();
uiElement.addClickListener(handler.onClick);
```
