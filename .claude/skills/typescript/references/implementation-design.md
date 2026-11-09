# 구현 선택

## 함수·클래스·팩토리

| 관리할 책임                                    | 기본 선택                          | 예                           |
| ---------------------------------------------- | ---------------------------------- | ---------------------------- |
| 입력을 받아 계산·변환                          | 함수                               | 금액 합계, 화면 문구 변환    |
| 여러 메서드가 공유하는 인스턴스 상태·자원 수명 | 클래스                             | 버퍼 비우기, 연결 시작·종료  |
| 같은 호출 계약을 제공하는 구현의 생성·선택     | 팩토리 함수                        | 설정에 따른 저장소 구현 선택 |
| 상태를 소유하는 인스턴스의 비동기 준비         | 비공개 생성자와 비동기 생성 메서드 | `static async open`          |
| 고정 값이나 무상태 함수 모음                   | 값·함수의 모듈 export              | 상태 코드, 시간 읽기         |

호출 하나와 상태 하나를 함께 감싸고 외부에 메서드 집합을 제공하지 않으면 클로저를 사용해도 됩니다. 무상태 함수 모음을 클래스의 정적 메서드로 묶지 않습니다.
자원 획득이 비동기이면 생성자에서 완료되지 않은 준비 작업을 시작하지 않습니다. [비동기 초기화와 소유권](async-lifecycle.md)처럼 준비가 끝난 인스턴스를 반환하고 정리 책임을 명시합니다.

참고: [Google TypeScript의 정적 컨테이너 클래스 대신 모듈 사용](https://google.github.io/styleguide/tsguide.html#container-classes)

### 인스턴스가 상태를 소유하는 예제

private 필드를 지원하는 TypeScript 실행 환경의 예제입니다.
`message-buffer.ts`를 저장한 뒤 TSDoc의 사용 예제를 같은 디렉터리의 별도 파일에 둡니다. 해당 패키지의 TypeScript 실행 명령으로 호출했을 때 각 `console.log`가 주석에 적힌 배열을 출력하면, 원본과 인스턴스의 격리·비우기·재사용 계약을 만족합니다.

````ts
// message-buffer.ts

/**
 * 메시지를 삽입 순서대로 보관하고 한 번에 꺼내는 버퍼입니다.
 *
 * @remarks
 * 각 인스턴스는 독립된 목록을 소유합니다.
 * {@link MessageBuffer.takeAll}로 비운 뒤에도 같은 인스턴스에 메시지를 추가할 수 있습니다.
 *
 * @example 추가한 메시지를 꺼내고 다시 사용
 * ```ts
 * import { MessageBuffer } from "./message-buffer.ts";
 *
 * const initial = ["first"];
 * const buffer = new MessageBuffer(initial);
 *
 * initial.push("outside");
 * buffer.append("second");
 *
 * console.log(buffer.takeAll()); // ["first", "second"]
 * console.log(buffer.takeAll()); // []
 *
 * buffer.append("third");
 *
 * console.log(buffer.takeAll()); // ["third"]
 * ```
 */
export class MessageBuffer {
  #messages: string[];

  /**
   * 초기 메시지로 버퍼를 만듭니다.
   *
   * @remarks 원본 배열을 복사하므로 생성 후 원본을 바꿔도 버퍼에는 반영하지 않습니다.
   *
   * @param initial - 순서대로 보관할 초기 메시지. 생략하면 빈 버퍼로 시작합니다.
   */
  constructor(initial: readonly string[] = []) {
    this.#messages = [...initial];
  }

  /**
   * 다음에 꺼낼 목록의 끝에 메시지를 추가합니다.
   *
   * @param message - 그대로 보관할 문구. 빈 문자열·중복 값도 유지합니다.
   */
  append(message: string): void {
    this.#messages.push(message);
  }

  /**
   * 보관한 메시지를 모두 꺼내 버퍼를 비웁니다.
   *
   * @remarks 반환한 배열을 수정해도 이후 버퍼 내용에는 영향을 주지 않습니다.
   *
   * @returns 삽입 순서의 메시지 배열. 빈 버퍼에서는 빈 배열을 반환합니다.
   */
  takeAll(): string[] {
    const messages = this.#messages;
    this.#messages = [];

    return messages;
  }
}
````

## 래퍼·도우미·중간 변수

입력·결과·오류·타입 추론·자원 수명 중 호출자에게 제공할 계약이 달라질 때만 래퍼를 둡니다.

| 형태                                            | 선택                                                                      |
| ----------------------------------------------- | ------------------------------------------------------------------------- |
| 인자를 그대로 전달하는 함수                     | 오류 변환·버전별 호출 차이·타입 추론 보존 등 추가 계약이 없으면 직접 호출 |
| 내장 API·라이브러리와 같은 동작의 재구현        | 입력·오류·부수 효과가 필요한 계약과 같으면 기존 API 사용                  |
| 반복한 속성 접근을 대체하는 지역 변수           | 같은 값을 재사용하거나, `undefined` 분기·타입 좁히기를 유지할 때 선언     |
| 짧은 속성 접근만 감싼 함수                      | 호출 계약을 추가하지 않으면 속성에 직접 접근                              |
| 반복되는 입력 정규화·오류 변환·자원 정리        | 동일한 입력·결과·실패 계약으로 사용할 때 도우미로 추출                    |
| 콜백 인자·요청 신호·오류 계약을 조정하는 어댑터 | 원본 API와 다른 인자 수·취소 방식·오류 표현을 맞추므로 유지               |

래퍼를 없애거나 표현을 합칠 때는 평가 횟수·순서·부수 효과, 동기 throw와 Promise 거부, 취소·정리 시점을 보존합니다.

### 콜백의 인자 전달

`map(Number.parseInt)`는 인덱스를 진법 인자로 전달합니다. 이렇게 호출 규약이 다르면 콜백으로 전달 인자를 맞춥니다.

```ts
// parse-decimals.ts
const texts = ["11", "5", "10"];

const numbers = texts.map((text) => Number.parseInt(text, 10));

console.log(numbers); // [11, 5, 10]
```

참고: [Google TypeScript의 콜백 인자 전달](https://google.github.io/styleguide/tsguide.html#prefer-passing-arrow-functions-as-callbacks)

### 호출부에서 구분해야 할 의미

단순 참·거짓 판정은 `boolean`으로, 호출자가 서로 다른 후속 처리를 해야 하는 결과는 리터럴·판별 유니온으로 표현합니다. 재시도 여부·한도는 이미 정해진 업무 규칙이라고 가정하며, 여기서는 그 입력과 결과의 구별만 다룹니다.

| 호출부의 문제                                                   | 선택                                                                   |
| --------------------------------------------------------------- | ---------------------------------------------------------------------- |
| `scheduleRetry(run, true)`처럼 인자의 의미를 선언에서 찾아야 함 | `retries`·`maxRetries`·`recovery`처럼 정해진 판단 입력의 이름을 드러냄 |
| `true`가 예약 성공과 한도 소진을 함께 뜻함                      | 호출자가 분기할 수 있게 서로 다른 판별자 반환                          |
| 결과만 계산하는 API가 저장·실행까지 암묵적으로 수행함           | 계산 API는 결과를 반환하고, 부수 효과를 수행하는 호출을 별도로 드러냄  |

````ts
// decide-retry.ts
export interface RetryRequest {
  /** 완료한 재시도 횟수입니다. 최초 시도는 포함하지 않습니다. */
  readonly retries: number;

  /** 최초 시도를 제외하고 허용할 최대 재시도 횟수입니다. */
  readonly maxRetries: number;
  readonly recovery: "transient" | "permanent";
}

export type RetryDecision =
  | { kind: "retry"; retryIndex: number }
  | { kind: "exhausted" }
  | { kind: "not-applicable" };

/**
 * 일시 오류의 다음 재시도를 계산합니다. 실제 예약·실행은 호출자가 맡습니다.
 *
 * @param request - 비음수 안전한 정수로 검증한 횟수와 오류 분류
 *
 * @returns 재시도할 순번, 한도 소진, 재시도 대상 아님을 구분한 결정
 *
 * @example
 * ```ts
 * import { decideRetry } from "./decide-retry.ts";
 *
 * const decision = decideRetry({ retries: 0, maxRetries: 2, recovery: "transient" });
 *
 * console.log(decision); // { kind: "retry", retryIndex: 1 }
 * ```
 */
export function decideRetry(request: RetryRequest): RetryDecision {
  if (request.recovery !== "transient") {
    return { kind: "not-applicable" };
  }

  if (request.retries >= request.maxRetries) {
    return { kind: "exhausted" };
  }

  return { kind: "retry", retryIndex: request.retries + 1 };
}
````

## 결과 확인

| 변경                     | 호출부에서 드러나야 할 결과                                                           |
| ------------------------ | ------------------------------------------------------------------------------------- |
| 함수에서 인스턴스로 전환 | 서로 다른 인스턴스의 상태가 섞이지 않고, 생성·준비·종료 시점이 호출 코드에 드러남     |
| 메서드를 콜백으로 전달   | `this`가 필요한 메서드는 인스턴스에 바인딩하거나 화살표 콜백으로 호출                 |
| 결과 타입 변경           | 모든 판별자에서 호출자가 필요한 데이터를 읽을 수 있고, 누락한 분기가 `never`로 진단됨 |
