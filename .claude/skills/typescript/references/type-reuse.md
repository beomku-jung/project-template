# 타입 재사용

## 계약 파생

원본과 함께 바뀌는 계약에 `Pick`·`Omit`을 사용합니다. 독립적인 계약은 별도로 정의합니다.

````ts
// to-product-summary.ts

/**
 * 상품 목록과 생성 입력에서 공유하는 정보입니다.
 *
 * @see {@link ProductSummary} - 목록에 공개할 정보
 * @see {@link CreateProductInput} - 식별자를 할당하기 전 입력
 */
export interface Product {
  readonly id: string;
  name: string;
  price: number;
}

export type ProductSummary = Pick<Product, "id" | "name">;

export type CreateProductInput = Omit<Product, "id">;

/**
 * 상품에서 목록에 공개할 정보를 선택합니다.
 *
 * @param product - 변경하지 않고 읽을 원본 상품
 *
 * @returns id와 name만 담은 새 {@link ProductSummary} 객체
 *
 * @example
 * ```ts
 * import { toProductSummary } from "./to-product-summary.ts";
 *
 * const product = { id: "p1", name: "머그컵", price: 1_200 };
 *
 * const summary = toProductSummary(product);
 *
 * console.log(summary); // { id: "p1", name: "머그컵" }
 * console.log(product); // { id: "p1", name: "머그컵", price: 1200 }
 * ```
 */
export function toProductSummary(product: Product): ProductSummary {
  return { id: product.id, name: product.name };
}
````

| 재사용할 대상 | 예                                    |
| ------------- | ------------------------------------- |
| 함수 인자     | `Parameters<typeof toProductSummary>` |
| 함수 결과     | `ReturnType<typeof toProductSummary>` |

- 반환 객체의 필드도 제한하려면 예제처럼 새 객체를 만듭니다.
- 내장 유틸리티를 우선하고, 반복되거나 의미 있는 조합에 타입 이름을 붙입니다.

참고: [유틸리티 타입](https://www.typescriptlang.org/docs/handbook/utility-types.html)

## 제네릭

객체·키·반환 타입의 관계를 보존할 때 사용합니다.

````ts
// get-property.ts

/**
 * 선택한 속성의 타입을 유지하며 값을 읽습니다.
 *
 * @returns 선택한 속성의 값. 객체·배열도 복사하지 않습니다.
 *
 * @example
 * ```ts
 * import { getProperty } from "./get-property.ts";
 *
 * const product = { name: "머그컵", price: 1_200 };
 *
 * const price = getProperty(product, "price"); // number로 추론
 *
 * console.log(price.toFixed(2)); // "1200.00"
 * ```
 */
export function getProperty<T, K extends keyof T>(value: T, key: K): T[K] {
  return value[key];
}
````

| 필요한 관계                | 선택                                 |
| -------------------------- | ------------------------------------ |
| 구체 타입으로 충분         | `formatLabel(value: string): string` |
| 입력에 따라 반환 타입 결정 | 위의 `getProperty<T, K>`             |
| 단순한 타입으로 표현 불가  | 조건부·재귀 타입 검토                |

참고: [제네릭 함수 작성 기준](https://www.typescriptlang.org/docs/handbook/2/functions.html#guidelines-for-writing-good-generic-functions)

## 타입 오류 검증

[제네릭 예제](#제네릭)의 `get-property.ts`와 아래 코드를 타입 검사기의 입력 범위에 포함합니다. 코드를 실행하지 않고 허용할 호출과 거부할 호출을 타입 검사합니다.

```ts
import { getProperty } from "./get-property.ts";

const product = { name: "머그컵", price: 1_200 };

getProperty(product, "price");

// @ts-expect-error - 상품에 없는 stock 키는 허용하지 않습니다.
getProperty(product, "stock");
```

`@ts-expect-error`는 다음 줄에 오류가 없으면 검사에 실패합니다.

| 용도                          | 주석에 적을 내용   |
| ----------------------------- | ------------------ |
| 외부 선언 오류 임시 억제      | 이유·제거 조건     |
| 잘못된 입력을 거부하는 테스트 | 기대하는 오류 이유 |

참고: [타입 오류 억제](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-3-9.html#ts-expect-error-comments)
