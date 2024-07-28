# 인터페이스

인터페이스는 상호 간에 정의한 약속 혹은 규칙입니다. 타입스크립트에서의 인터페이스는 보통 아래와 같이 정의할 수 있습니다.

- 객체의 스펙
- 함수의 파라미터
- 함수의 스펙
- 배열과 객페를 접근하는 방식
- 클래스

```ts
let person = { name: "Capt", age: 28 };

function logAge(obj: { age: number }) {
  console.log(obj.age); // 28
}

logAge(person);
```

'logAge()' 함수에 받는 인자의 형태는 'age'를 속성으로 갖는 객체입니다. 이렇게 인자를 받을 때 단순한 타입 뿐만 아니라 객채의 속성 타입까지 정의할 수 있습니다.

여기서 인터페이스를 적용한다면 아래와 같을 것 입니다.

```ts
interface personAge {
  age: number;
}

function logAge(obj: personAge) {
  console.log(obj.age);
}

let person = { name: "Capt", age: 28 };
logAge(person);
```

이제는 'logAge()'의 인자가 좀 더 명시적으로 바뀌었습니다. 'logAge()'의 인자는 'personAge'라는 타입을 가져야 합니다.

따라서 위 예제를 보고 추론을 할 수 있습니다. 인터페이스를 인자로 받아 사용할 때 항상 인터페이스의 속성 갯수와 인자로 받는 객체의 속정 갯수를 일치시키지 않아도 된다.
즉, 인터페이스에 정의된 속성, 타입의 조건만 만족한다면 객채의 속성 갯수가 많아져도 상관이 없다는 것을 의미하게 됩니다. 그리고, 인터페이스에 선언된 속성 순서를 지키지 않아도 됩니다.

## 옵션 속성

인터페이스를 사용할 때 인터페이스에 정의되어 있는 속성을 모두 사용하지 않아도 됩니다. 이를 옵션 속성이라고 합니다.

```ts
interface 인터페이스_이름 {
  속성?: 타입;
}
```

위 문법처럼 속성 끝에 '?'를 붙입니다.

```ts
interface CraftBeer {
  name: string;
  hop?: number;
}

let myBeer = {
  name: "Saporo",
};
function brewBeer(beer: CraftBeer) {
  console.log(beer.name); // Saporo
}
brewBeer(myBeer);
```

'brewBeer()' 함수에서 'Beer' 인터페이스를 인자의 타입으로 선언했음에도, 인자로 넘긴 객체는 'hop' 속성이 없습니다, 왜냐하면 'hop'을 옵션 속성으로 정의했기 때문입니다.

## 옵션 속성의 장점

옵션 속성의 장점은 단순히 인터페이스를 사용할 때 속성을 선택적으로 적용할 수 있다는 것 뿐만아니라 인터페이스에 정의되어 있지 않은 속성에 대해서 인지시켜줄 수 있습니다.

```ts
interface CraftBeer {
  name: string;
  hop?: number;
}

let myBeer = {
  name: "Saporo",
};
function brewBeer(beer: CraftBeer) {
  console.log(beer.brewery); // Error: Property 'brewery' does not exist on type 'Beer'
}
brewBeer(myBeer);
```

위와 같이 인터페이스에 정의되어 있지 않는 속성에 대해 에러가 발생합니다. 만약 아래와 같이 오탈자가 났다면 에러가 발생 했을 것 입니다.

```ts
interface CraftBeer {
  name: string;
  hop?: number;
}

function brewBeer(beer: CraftBeer) {
  console.log(beer.nam); // Error: Property 'nam' does not exist on type 'Beer'
}
```

## 읽기 전용 속성

읽기 전용 속성은 인터페이스로 객체를 처음 생성할 때만 값을 할당하고 이후에 변경할 수 없는 속성을 말합니다. 아래와 같이 'readonly' 속성을 앞에 붙여 사용합니다.

```ts
interface Beer {
  readonly brand: string;
}
```

앞서 설명한 내용을 참고하여 인터페이스 객체에 선언하고, 수정하려할 때 에러가 납니다.

```ts
let myBeer: CraftBeer = {
  brand: "Belgian Monk",
};
myBeer.brand = "Korean Carpenter"; // error!
```

## 읽기 전용 배열

배열을 선언할 때 'ReadonlyArray<T>' 타입을 상요하면 읽기 전용 배열을 생성 할 수 있습니다.

```ts
let arr: ReadonlyArray<number> = [1, 2, 3];
arr.splice(0, 1); //error
arr.push(4); //error
arr[0] = 100; //error
```

위 처럼 배열을 'ReadonlyArray'로 선언하면 배열의 내용을 변경할 수 없습니다. 선언하는 시점에만 정의할 수 있습니다.

## 객체 선언과 관련된 타입 체킹

타입스크립트는 인터페이스를 이용하여 객체를 선언할 때 좀 더 엄밀한 속성 검사를 하게 됩니다.

```ts
interface CraftBeer {
  brand?: string;
}

function brewBeer(beer: CraftBeer) {
  // ...
}
brewBeer({ brandon: "what" }); // error: Object literal may only specify known properties, but 'brandon' does not exist in type 'CraftBeer'. Did you mean to write 'brand'?
```

'CreftBeer' 인터페이스에는 'brand'라고 선언되어 있지만 'brewBeer()' 함수에 인자로 넘기는 'myBeer' 객체에는 'brandon'이 선언되어 있어 오탈자 점검을 말하는 오류가 발생합니다.

이런 타입 추론을 무시하고 싶다면

```ts
let myBeer = { brandon: "what" };
brewBeer(myBeer as CraftBeer);
```

그럼에도 불구하고 인터페이스 정의하지 않은 속성들은 추가로 사용하고 싶을 때는

```ts
interface CraftBeer {
  brand?: string;
  [propName: string]: any;
}
```

## 함수 타입

인터페이스는 함수의 타입을 정의할 때에도 사용할 수 있습니다.

```ts
interface login {
  (username: string, password: string): boolean;
}
```

함수의 인자 타입과 반환 값 타입을 정의합니다.

```ts
let loginUser: login;
loginUser = function (id: string, pw: string) {
  console.log("login");
  return true;
};
```

## 클래스 타입

C#, Java 처럼 타입스크립트에서도 클래스가 일정 조건을 만족하도록 타입 규칙을 정의할 수 있습니다.

```ts
interface CraftBeer {
  beerName: string;
  nameBeer(beer: string): void;
}

class myBeer implements CraftBeer {
  beerName: string = "Guinness";
  nameBeer(b: string) {
    this.beerName = b;
  }
  constructor() {}
}
```

## 인터페이스 확장

클래스와 마찬가지로 인터페이스도 인터페이스 간 확장이 가능합니다.

```ts
interface Person {
  name: string;
}

interface Developer extends Person {
  skill: string;
}

let fe = {} as Developer;
fe.name = "Roy";
fe.skill = "TypeScript";
```

혹은 아래와 같이 여러 인터페이스를 상속 받아 사용할 수 있습니다.

```ts
interface Person {
  name: string;
}

interface Drinker {
  drink: string;
}

interface Developer extends Person, Drinker {
  skill: string;
}

let fe = {} as Developer;
fe.name = "Roy";
fe.skill = "TypeScript";
fe.drink = "Beer";
```

## 하이브리트 타입

자바스크립트의 유연하고 동적인 타입 특성에 따라 인터페이스 역시 여러가지 타입으르 조합하여 만들 수 있습니다. 아래 예제와 같이 함수 타입이면서 객체 타입을 정의할 수 있는 인터페이스가 있습니다.

```ts
interface CraftBeer {
  (beer: string): string;
  brand: string;
  brew(): void;
}

function myBeer(): CraftBeer {
  let my = function (beer: string) {} as CraftBeer;
  my.brand = "Beer Kitchen";
  my.brew = function () {};
  return my;
}

let brewedBeer = myBeer();
brewedBeer("My First Beer");
brewedBeer.brand = "Suwon Craft";
brewedBeer.brew();
```
