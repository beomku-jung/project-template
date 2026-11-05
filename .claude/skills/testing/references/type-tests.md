# 값과 타입 검증

공개 제네릭·유니온·타입 유틸리티처럼 추론 결과가 계약인 경우에 사용합니다.

일반 `.test.ts` 파일에서 함께 작성하되, 값은 Vitest 실행으로 검증하고 `expectTypeOf`는 테스트 파일을 프로젝트의 실제 타입 검사 범위에 포함해 검증합니다.

## 첫 원소의 값과 추론 결과

두 파일을 같은 디렉터리에 둡니다.

```ts
// first-item.ts

/**
 * 목록을 변경하지 않고 첫 원소를 반환합니다.
 *
 * @typeParam T - 목록 원소의 타입
 *
 * @returns 첫 원소. 빈 목록이면 undefined
 */
export function firstItem<T>(items: readonly T[]): T | undefined {
  return items[0];
}
```

```ts
// first-item.test.ts
import { expect, expectTypeOf, it } from "vitest";

import { firstItem } from "./first-item.ts";

it("읽기 전용 목록의 첫 상태를 반환합니다", () => {
  const statuses = ["pending", "paid"] as const;

  const status = firstItem(statuses);

  expect(status).toBe("pending");
  expectTypeOf(status).toEqualTypeOf<"pending" | "paid" | undefined>();
});

it("빈 목록이면 undefined를 반환합니다", () => {
  const values: number[] = [];

  const value = firstItem(values);

  expect(value).toBeUndefined();
  expectTypeOf(value).toEqualTypeOf<number | undefined>();
});
```

기대 타입은 직접 명시합니다. `ReturnType<typeof firstItem>`에서 가져오면 잘못된 반환 타입 변경도 따라가므로 회귀를 잡을 수 없습니다.

## 참고

- [Vitest 타입 테스트](https://vitest.dev/guide/testing-types)
- [expectTypeOf](https://vitest.dev/api/expect-typeof)
