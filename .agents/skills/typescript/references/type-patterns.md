# 타입 예제

## 외부 입력

기존 스키마 검증기가 있으면 재사용합니다.

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

| 입력 경계                        | 처리                         |
| -------------------------------- | ---------------------------- |
| `JSON.parse()`·`response.json()` | `unknown`에 받은 뒤 검증     |
| `as User`·`!`·`any`              | 런타임 검증을 대신할 수 없음 |

참고: [타입 좁히기](https://www.typescriptlang.org/docs/handbook/2/narrowing.html)

## 상태별 데이터

상태마다 필요한 값이 다를 때 사용합니다. [외부 입력 예제](#외부-입력)의 `parse-user.ts`를 함께 사용합니다.

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

`never`는 새 상태의 분기 누락을 검사합니다.

## 읽기 전용 입력

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

| 구분                     | 보장 범위                                              |
| ------------------------ | ------------------------------------------------------ |
| `readonly`·`Readonly<T>` | 해당 참조의 타입 검사; 런타임 동결·깊은 불변성은 제외  |
| `toSorted()`             | 새 배열 반환; 객체 원소는 공유하므로 변경 시 별도 복사 |

참고: [TypeScript의 읽기 전용 속성](https://www.typescriptlang.org/docs/handbook/2/objects.html#readonly-properties)

## 선택 속성

일부 필드만 갱신하는 요청에 사용합니다.

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

- `exactOptionalPropertyTypes`를 켜면 `nickname?: string`에 `undefined`를 대입할 수 없습니다. 값을 보내지 않을 때는 속성을 생략합니다.
- `0`·`false`·빈 문자열을 유지할 기본값 처리는 `??`를 사용합니다.

참고: [exactOptionalPropertyTypes](https://www.typescriptlang.org/tsconfig/exactOptionalPropertyTypes.html)

## 고정 값

목록과 타입이 함께 바뀌어야 하면 값에서 타입을 파생합니다.

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

`as const`는 리터럴 타입을 유지하고, [`satisfies`](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-4-9.html#the-satisfies-operator)는 구조를 검사합니다. 런타임 입력은 별도로 검증합니다.
