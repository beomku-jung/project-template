# 타입 예제

## 외부 입력

외부 입력은 `unknown`으로 받고, 속성의 존재·타입·값의 제약을 검사한 뒤 공개 타입을 반환합니다. 기존 스키마 검증기가 같은 제약·정규화·추가 필드 처리 계약을 제공하면 재사용합니다.

````ts
// parse-user.ts

/**
 * 사용자 식별자와 화면에 표시할 이름입니다.
 *
 * @see {@link parseUser} - 외부 입력 검증과 정규화
 */
export interface User {
  readonly id: string;
  name: string;
}

/**
 * 외부 입력을 검증하고 {@link User}로 정규화합니다.
 *
 * @remarks
 * - id와 name의 앞뒤 공백을 제거합니다.
 * - 두 속성 외의 값은 결과에 포함하지 않으며, 원본은 변경하지 않습니다.
 *
 * @param value - id와 name을 검사할 외부 입력
 *
 * @returns 정규화된 새 사용자 객체
 * @throws TypeError
 * - 입력이 객체가 아닌 경우
 * - id·name이 문자열이 아니거나, 공백 제거 후 비어 있는 경우
 *
 * @example
 * ```ts
 * import { parseUser } from "./parse-user.ts";
 *
 * const user = parseUser({ id: " u1 ", name: " 민수 " });
 *
 * console.log(user); // { id: "u1", name: "민수" }
 * ```
 */
export function parseUser(value: unknown): User {
  if (typeof value !== "object" || value === null) {
    throw new TypeError("사용자 객체가 필요합니다.");
  }

  if (!("id" in value) || typeof value.id !== "string" || value.id.trim() === "") {
    throw new TypeError("id는 비어 있지 않은 문자열이어야 합니다.");
  }

  if (!("name" in value) || typeof value.name !== "string" || value.name.trim() === "") {
    throw new TypeError("name은 비어 있지 않은 문자열이어야 합니다.");
  }

  return { id: value.id.trim(), name: value.name.trim() };
}
````

| 입력 경계                        | 처리                                                             |
| -------------------------------- | ---------------------------------------------------------------- |
| `JSON.parse()`·`response.json()` | 반환값을 `unknown`에 받은 뒤 사용 전에 검증                      |
| `as User`·`!`·`any`              | 컴파일러가 다루는 타입만 바꾸므로 런타임 검증 대신 사용하지 않음 |

참고: [타입 좁히기](https://www.typescriptlang.org/docs/handbook/2/narrowing.html)

## 상태별 데이터

상태마다 사용할 수 있는 값이 다르면 판별 유니온으로 표현하고, 판별자를 좁힌 뒤 해당 값을 읽습니다. 아래 예제는 [외부 입력 예제](#외부-입력)의 `parse-user.ts`를 함께 사용합니다.

````ts
// get-user-label.ts
import type { User } from "./parse-user.ts";

/** 사용자 조회 단계별로 사용할 데이터를 묶습니다. */
export type UserState =
  | { status: "loading" }
  | {
      status: "ready";
      user: User;
    }
  | {
      status: "failed";
      error: Error;
    };

/**
 * {@link UserState}에 따라 대기 문구, 사용자 이름 또는 오류 메시지를 반환합니다.
 *
 * @example
 * ```ts
 * import { getUserLabel } from "./get-user-label.ts";
 *
 * console.log(getUserLabel({ status: "loading" })); // "불러오는 중"
 * console.log(getUserLabel({ status: "ready", user: { id: "u1", name: "민수" } })); // "민수"
 * ```
 */
export function getUserLabel(state: UserState): string {
  switch (state.status) {
    case "loading":
      return "불러오는 중";
    case "ready":
      return state.user.name;
    case "failed":
      return state.error.message;
    default: {
      const unexpected: never = state;

      throw new Error("알 수 없는 상태", { cause: unexpected });
    }
  }
}
````

`never` 대입은 새 상태를 추가했을 때 빠진 분기를 컴파일러 진단으로 드러냅니다. 외부 입력을 이 타입으로 단언했다고 런타임의 잘못된 판별자를 거부하는 것은 아니므로, 입력 경계의 검증은 별도로 둡니다.

## 읽기 전용 입력

함수가 원본 배열을 바꾸지 않으면 매개변수에 `readonly`를 사용합니다. 정렬 결과도 필요하면 원본을 수정하는 `sort()` 대신 대상 런타임이 지원하는 `toSorted()`를 사용합니다. 지원하지 않는 런타임에서는 복사 후 정렬하거나 해당 API를 제공하는 폴리필을 사용합니다.

````ts
// get-sorted-prices.ts

/**
 * 원본 가격 목록을 유지하며 낮은 가격부터 정렬합니다.
 *
 * @param prices - 정렬할 가격 목록
 *
 * @returns 오름차순으로 정렬한 새 배열. 빈 입력에는 빈 배열을 반환합니다.
 *
 * @example
 * ```ts
 * import { getSortedPrices } from "./get-sorted-prices.ts";
 *
 * const prices = [3_000, 1_200, 2_400];
 *
 * const sortedPrices = getSortedPrices(prices);
 *
 * console.log(sortedPrices); // [1200, 2400, 3000]
 * console.log(prices); // [3000, 1200, 2400]
 * ```
 */
export function getSortedPrices(prices: readonly number[]): number[] {
  return prices.toSorted((left, right) => left - right);
}
````

| 구분                     | 보장 범위                                                                       |
| ------------------------ | ------------------------------------------------------------------------------- |
| `readonly`·`Readonly<T>` | 해당 참조의 타입 검사; 런타임 동결·깊은 불변성은 제외                           |
| `toSorted()`             | 새 배열 반환; 객체 원소는 공유하므로 원본 객체도 보존해야 하면 해당 객체를 복사 |

참고: [TypeScript의 읽기 전용 속성](https://www.typescriptlang.org/docs/handbook/2/objects.html#readonly-properties)

## 선택 속성

속성 생략과 값 전달의 의미가 다르면 선택 속성으로 표현합니다. 아래 API에서는 생략은 유지, 빈 문자열은 비우기를 뜻합니다.

````ts
// create-profile-patch.ts

/** 프로필에서 변경할 값만 전달합니다. */
export interface ProfilePatch {
  /** 생략하면 유지하고, 빈 문자열이면 기존 닉네임을 비웁니다. */
  nickname?: string;
}

/**
 * 닉네임 입력을 프로필 변경 요청으로 변환합니다.
 *
 * @param nickname - 새 닉네임. undefined이면 변경 요청에서 생략합니다.
 *
 * @returns {@link ProfilePatch}의 규칙에 따라 변경할 값만 담은 새 객체
 *
 * @example
 * ```ts
 * import { createProfilePatch } from "./create-profile-patch.ts";
 *
 * console.log(createProfilePatch(undefined)); // {}
 * console.log(createProfilePatch("")); // { nickname: "" }
 * console.log(createProfilePatch("민수")); // { nickname: "민수" }
 * ```
 */
export function createProfilePatch(nickname: string | undefined): ProfilePatch {
  return nickname === undefined ? {} : { nickname };
}
````

- `exactOptionalPropertyTypes`를 켜면 `nickname?: string`에 `undefined`를 대입할 수 없습니다. 생략은 `{}`, 명시적 `undefined`도 받는 계약은 `nickname?: string | undefined`로 구분합니다.
- `0`·`false`·빈 문자열을 유지하고 `null`·`undefined`만 기본값으로 바꿀 때 `??`를 사용합니다. `null`이 독립된 의미를 가지면 그 분기를 먼저 처리합니다.

참고: [exactOptionalPropertyTypes](https://www.typescriptlang.org/tsconfig/exactOptionalPropertyTypes.html)

## 고정 값

런타임 고정 목록과 허용 타입이 함께 바뀌는 계약이면 값에서 타입을 파생합니다. 외부 계약이 목록과 별도로 바뀌면 목록을 그 계약의 유일한 원본으로 삼지 않습니다.

```ts
// access-levels.ts

export const ACCESS_LEVELS = ["reader", "editor"] as const;

export type AccessLevel = (typeof ACCESS_LEVELS)[number];
```

```ts
// access-labels.ts
import type { AccessLevel } from "./access-levels.ts";

export const ACCESS_LABELS = {
  reader: "읽기",
  editor: "편집",
} as const satisfies Record<AccessLevel, string>;
```

`as const`는 리터럴 타입과 읽기 전용 속성을 유지하고, [`satisfies`](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-4-9.html#the-satisfies-operator)는 추론 결과를 지정 타입으로 바꾸지 않으면서 대입 가능성을 검사합니다. 둘 다 런타임 객체를 동결하거나 외부 입력을 검증하지 않습니다.
