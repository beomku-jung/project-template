# 유닛 테스트

외부 자원 없는 계산·분기 계약은 실제 함수를 호출해 검증합니다. 입력만 다르면 `it.each`·`it.for`, 준비나 결과의 의미가 다르면 별도 `it`로 나눕니다.

아래 계약은 주문 금액 30,000원 이상 무료, 미만 3,000원입니다. 테스트는 구현의 상수를 import하거나 같은 조건식을 다시 계산하지 않고 계약의 금액을 기대값으로 씁니다.

```ts
// get-shipping-fee.ts

const FREE_SHIPPING_THRESHOLD = 30_000;
const STANDARD_SHIPPING_FEE = 3_000;

/**
 * 주문 금액에 따른 배송비를 원 단위로 계산합니다.
 *
 * @param orderTotal - 0 이상의 유한한 주문 금액(원)
 *
 * @returns 30,000원 이상이면 0원, 미만이면 3,000원
 * @throws RangeError - 주문 금액이 음수이거나 유한한 수가 아닌 경우
 */
export function getShippingFee(orderTotal: number): number {
  if (!Number.isFinite(orderTotal) || orderTotal < 0) {
    throw new RangeError("주문 금액은 0 이상의 유한한 수여야 합니다.");
  }

  return orderTotal >= FREE_SHIPPING_THRESHOLD ? 0 : STANDARD_SHIPPING_FEE;
}
```

```ts
// get-shipping-fee.test.ts
import { describe, expect, it } from "vitest";

import { getShippingFee } from "./get-shipping-fee.ts";

describe("getShippingFee", () => {
  it.each([
    { total: 0, expected: 3_000 },
    { total: 29_999, expected: 3_000 },
    { total: 30_000, expected: 0 },
    { total: 30_001, expected: 0 },
  ])("주문 금액 $total원이면 배송비는 $expected원입니다", ({ total, expected }) => {
    const fee = getShippingFee(total);

    expect(fee).toBe(expected);
  });

  it.each([-1, NaN, Infinity])("유효하지 않은 주문 금액 %s를 거부합니다", (total) => {
    expect(() => getShippingFee(total)).toThrow(RangeError);
  });
});
```

경계 사례는 최솟값 0원, 기준 직전 29,999원, 기준 30,000원, 직후 30,001원을 직접 적어 포함 여부를 드러냅니다. 도우미로 이 선택을 숨기지 않습니다.
음수·`NaN`·`Infinity`는 `number` 범위 안의 실패 입력이므로 타입을 우회하지 않습니다. 공개 오류 계약인 `RangeError`를 단언하고 내부 상수·분기 호출 횟수는 단언하지 않습니다.

## 참고

- [it.each](https://vitest.dev/api/test#test-each)
- [toThrow](https://vitest.dev/api/expect#tothrow)
