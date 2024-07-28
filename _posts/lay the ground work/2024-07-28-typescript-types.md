# 타입스크립트 기본 타입

타입스크립트로 변수나 함수와 같은 자바스크립트 코드에 타입을 정의할 수 있습니다.

- Boolean
- Number
- String
- Object
- Array
- Tuple
- Enum
- any
- void
- null
- undefined
- never

자바를 시작하면서 배웠던 익숙한 타입들이 보이네요.
타입스크립트를 배우면서 처음 본 타입 또는 한번 더 봐야할 타입들만 만들어 보겠습니다.

String, Number, Boolean, Object, Array는 Skip 하겠습니다.

## Tuple

듀플은 배열의 길이가 고정되고 각 요소의 타입이 지정되어 있는 배열 형식을 말합니다.

```ts
let arr: [string, number] = ["hi", 10];
```

만약 정의하지 않은 타입, 인덱스로 접근할 경우 오류가 발생합니다.

```ts
arr[1].concat("!"); //Error, 'number' does not have 'concat'
arr[5] = "hello"; //Error, Property '5' does not exist on type '[string, number]'.
```

## Enum

이넘은 C, Java에서 흔하게 상수의 집합들을 정의할 때 사용합니다.

```ts
enum Avengers {
  Capt,
  IronMan,
  Thor,
}
let hero: Avengers = Avengers.Capt;
```

인덱스 번호로도 접근이 가능합니다.

```ts
enum Avengers {
  Capt,
  IronMan,
  Thor,
}
let hero: Avengers = Avengers[0];
```

원한다면 이넘의 인덱스를 사용자 편의로 변경하여 사용이 가능합니다.

```ts
enum Avengers {
  Capt = 2,
  IronMan,
  Thor,
}
let hero: Avengers = Avengers[2]; // Capt
let hero: Avengers = Avengers[4]; // Thor
```

## any

기존 자바스크립트로 구현되어 있는 웹 서비스 코드에 타입스크립트를 점진적으로 적용할 때 활용하면 좋은 타입니다. 단어 의미 그대로 모든 타입에 대해서 허용한다는 의미입니다.

```ts
let str: any = "hi";
let num: any = 10;
let arr: any = ["a", 2, true];
```

## Never

함수의 끝에 절대 도달하지 않는다는 의미 갖는 타입니다.

```ts
// 이 함수는 절대 함수의 끝까지 실행되지 않는다는 의미
function neverEnd(): never {
  while (true) {}
}
```
