# 구현 선택

호출 예제에서 필요한 입력·결과·상태를 정하고 구현 형태를 선택합니다.

## 함수·클래스·팩토리

| 관리할 책임                                | 기본 선택               | 예                          |
| ------------------------------------------ | ----------------------- | --------------------------- |
| 입력을 받아 계산·변환                      | 함수                    | 금액 합계, 화면 문구 변환   |
| 여러 동작이 공유하는 상태·불변식·자원 수명 | 클래스 검토             | 버퍼 비우기, 연결 시작·종료 |
| 구현 선택·생성 정책                        | 팩토리 함수             | 설정에 따른 저장소 선택     |
| 상태를 소유하는 인스턴스의 비동기 준비     | 비동기 생성 메서드 검토 | `static async open`         |
| 고정 값이나 무상태 함수 모음               | 값·함수의 모듈 export   | 상태 코드, 시간 읽기        |

하나의 동작과 상태를 간결하게 묶는 클로저도 유지할 수 있습니다.
인스턴스가 준비부터 종료까지 자원을 관리하면 [비동기 초기화와 소유권](async-lifecycle.md)을 읽습니다.

참고: [Google TypeScript의 정적 컨테이너 클래스 대신 모듈 사용](https://google.github.io/styleguide/tsguide.html#container-classes)

### 인스턴스가 상태를 소유하는 예제

private 필드를 지원하는 TypeScript 실행 환경의 예제입니다.
`message-buffer.ts`를 저장한 뒤 TSDoc의 사용 예제를 같은 디렉터리의 별도 파일에서 실행합니다.

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

정책에 이름을 붙이거나 자원 정리 경계를 드러내려면 한 번 쓰는 코드도 분리할 수 있습니다.

| 형태                                                | 판단                                                         |
| --------------------------------------------------- | ------------------------------------------------------------ |
| 인자를 그대로 전달하는 함수                         | 변환·정책·호환성·타입 추론 등 별도 계약이 없다면 직접 호출   |
| 내장 API·라이브러리와 같은 동작의 재구현            | 입력·오류·부수 효과가 계약에 맞으면 기존 API·라이브러리 사용 |
| 제품 코드의 미구현 연동을 대신하는 성공값·빈 메서드 | 실제 구현으로 연결하고, 미구현 범위는 별도로 표시            |
| 의미 있는 조건 이름                                 | 사용 지점 가까이에 유지                                      |
| 짧은 속성 접근만 감싼 함수·한 번 쓰는 별칭          | 인라인으로 읽는 편이 명확한지 비교                           |
| 반복되는 정책·검증·자원 정리 절차                   | 같은 변경 이유를 공유할 때 추출                              |
| 콜백 인자·요청 신호·오류 계약을 조정하는 어댑터     | 실제 계약 차이를 처리하므로 유지                             |

표현을 합칠 때는 평가 횟수·순서와 부수 효과를 보존합니다.

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

단순 참·거짓 판정은 `boolean`으로, 서로 다른 후속 처리는 구분할 수 있는 결과로 표현합니다.

| 호출부의 문제                                                   | 정제 방향                                             |
| --------------------------------------------------------------- | ----------------------------------------------------- |
| `scheduleRetry(run, true)`처럼 인자의 의미를 선언에서 찾아야 함 | 재시도 횟수·한도·오류 분류처럼 실제 판단 입력을 명명  |
| `true`가 예약 성공과 한도 소진을 함께 뜻함                      | 호출자가 구분할 결과를 리터럴·판별 유니온으로 반환    |
| 계산 결과를 얻는 호출이 저장·실행까지 암묵적으로 수행함         | 계산과 부수 효과의 소유자를 정하고 호출 순서를 드러냄 |

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

설계한 API로 소비자의 정상·실패 흐름을 표현할 수 있는지 확인합니다.
기존 구현을 바꾸면 변경 전후의 공개 계약을 비교하고 호출부·타입·문서를 갱신합니다.
인스턴스나 자원을 다루면 격리·정리 동작도 비교합니다.
클래스로 전환하면 생성 시점·비동기 준비·`this` 바인딩도 확인합니다.
