# 타입 재사용

## 계약 파생

원본의 속성 이름·타입 변경을 그대로 따라야 하는 계약은 `Pick`·`Omit`으로 파생합니다. 별도로 버전 관리하거나 서로 다른 필드 의미를 가진 계약은 속성 모양이 같아도 별도로 선언합니다.

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

- `Pick`·`Omit`은 런타임 객체의 필드를 제거하지 않습니다. 반환 객체에서 비공개 필드도 빼야 하면 예제처럼 허용한 필드만 담은 새 객체를 만듭니다.
- 기존 계약을 표현할 수 있으면 내장 유틸리티 타입을 사용합니다. 같은 조합을 여러 선언이 쓰거나 공개 계약으로 참조해야 하면 타입 이름을 붙입니다.
- `Parameters`·`ReturnType`은 함수 선언이 바뀌면 함께 바뀝니다. 오버로드 함수에서는 마지막 시그니처를 기준으로 하므로 특정 호출의 결과 추론과 같다고 가정하지 않습니다.

참고: [유틸리티 타입](https://www.typescriptlang.org/docs/handbook/utility-types.html)

## 제네릭

입력 객체·키·반환 타입 사이의 관계를 보존해야 할 때 제네릭을 사용합니다. 타입 매개변수로 연결할 관계가 없으면 구체 타입이나 유니온을 사용합니다.

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

| 필요한 관계                       | 선택                                                                               |
| --------------------------------- | ---------------------------------------------------------------------------------- |
| 구체 타입으로 충분                | `formatLabel(value: string): string`처럼 선언                                      |
| 입력 키에 따라 반환 타입 결정     | 위의 `getProperty<T, K>`로 `T[K]` 보존                                             |
| 입력 타입에 따라 결과 구조가 다름 | 조건부 타입; 유한한 경우만 지원하면 구체 유니온·오버로드와 비교                    |
| 중첩 구조를 타입 수준에서 순회    | 재귀 타입; 지원할 깊이·입력 크기에서 컴파일러의 인스턴스화 한도 진단이 없는지 확인 |

참고: [제네릭 함수 작성 기준](https://www.typescriptlang.org/docs/handbook/2/functions.html#guidelines-for-writing-good-generic-functions)

## 타입 오류 검증

[제네릭 예제](#제네릭)의 `get-property.ts`와 아래 호출을 같은 타입 검사 입력에 포함합니다. 올바른 키는 진단 없이 통과하고, `stock` 호출은 다음 줄 오류를 요구하는 지시문을 소비해야 합니다. 아래 코드는 실행하지 않는 타입 검사 전용 예제입니다.
실제 테스트로 옮길 때는 특수한 타입 테스트 확장자를 만들지 않고 일반 `.test.ts` 파일의 테스트 본문 안에 둡니다.

```ts
import { getProperty } from "./get-property.ts";

const product = { name: "머그컵", price: 1_200 };

getProperty(product, "price");

// @ts-expect-error - 상품에 없는 stock 키는 허용하지 않습니다.
getProperty(product, "stock");
```

`@ts-expect-error`는 다음 줄에 컴파일러 오류가 없으면 사용하지 않은 지시문 오류를 냅니다. 원하는 진단 코드만 지정하는 기능은 아니므로, 전혀 다른 오류도 지시문을 소비할 수 있습니다. 이유를 적고 대상 표현을 좁혀 원래 거부하려던 계약을 분명히 합니다.
타입 검사 입력 범위는 [범위 진단 명령](package-configuration.md#타입-검사-범위)으로 확인합니다. Vitest가 `.test.ts`를 실행해 성공했다는 사실만으로 이 지시문이나 반환 타입 추론이 검사됐다고 판단하지 않습니다.

| 용도                                | 주석·선언 선택                                                                             |
| ----------------------------------- | ------------------------------------------------------------------------------------------ |
| 설치된 외부 선언의 오류를 임시 억제 | `@ts-expect-error`에 실제 런타임 계약의 근거·제거 조건을 적고, 영향받는 다음 표현에만 적용 |
| 잘못된 입력을 거부하는 타입 단언    | 거부할 키·인자 관계를 이유로 적고, 정상 호출도 같은 검사 범위에 포함                       |

검사를 끄는 `@ts-ignore`나 파일 전체의 `@ts-nocheck`로 바꾸어 지시문 오류를 숨기지 않습니다. 타입 진단이 사라졌다면 호출 계약이 바뀐 것인지 외부 선언이 고쳐진 것인지 판단해 지시문을 제거합니다.

참고: [타입 오류 억제](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-3-9.html#ts-expect-error-comments)
