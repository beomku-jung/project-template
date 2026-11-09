# 모킹

검증 대상은 실제 구현으로 실행합니다. 범위 밖 의존성의 결과·실패·완료 순서를 재현해야 하거나 외부 상태를 격리해야 할 때만 모킹합니다. 함수 주입 지점이 이미 있으면 타입이 있는 `vi.fn`을 쓰고, import한 외부 모듈을 바꿔야 할 때는 `vi.mock`을 선택합니다. 테스트 편의만을 위해 제품 API에 새 주입 지점을 만들지는 않습니다.

## 타입이 있는 외부 의존성

주문 조회를 대체해 결과별 화면 문구와 오류 전달을 검증합니다. 주문 ID는 외부 조회 계약이므로 호출 인자도 단언합니다. 실제 DB 동작은 이 테스트의 검증 대상이 아닙니다.

```ts
// get-order-label.ts

export interface Order {
  id: string;

  /** 주문 합계(원). */
  total: number;
}

/** 주문이 없으면 undefined로 이행하고, 조회 실패 시 거부합니다. */
export type OrderLoader = (id: string) => Promise<Order | undefined>;

/**
 * {@link OrderLoader}로 조회한 주문을 식별자와 합계가 담긴 화면 문구로 바꿉니다.
 *
 * @param id - 조회할 주문 식별자
 * @param loadOrder - 주문 조회 함수
 *
 * @returns 주문이 있으면 표시 문구, 없으면 "주문 없음"으로 이행되는 Promise
 * @throws 조회 실패 시 원래 오류로 Promise를 거부합니다.
 */
export async function getOrderLabel(id: string, loadOrder: OrderLoader): Promise<string> {
  const order = await loadOrder(id);

  return order ? `${order.id}: ${order.total}원` : "주문 없음";
}
```

```ts
// get-order-label.test.ts
import { describe, expect, it, vi } from "vitest";

import { getOrderLabel } from "./get-order-label.ts";
import type { Order, OrderLoader } from "./get-order-label.ts";

describe("getOrderLabel", () => {
  it("주문이 있으면 해당 주문의 합계를 표시합니다", async () => {
    const order: Order = { id: "order-1", total: 3_600 };
    const loadOrder = vi.fn<OrderLoader>().mockResolvedValue(order);

    const label = await getOrderLabel("order-1", loadOrder);

    expect(label).toBe("order-1: 3600원");
    expect(loadOrder).toHaveBeenCalledWith("order-1");
  });

  it("주문이 없으면 미존재 문구를 표시합니다", async () => {
    const loadOrder = vi.fn<OrderLoader>().mockResolvedValue(undefined);

    await expect(getOrderLabel("missing", loadOrder)).resolves.toBe("주문 없음");
  });

  it("조회 실패는 호출부에 전달합니다", async () => {
    const failure = new Error("저장소 연결 실패");
    const loadOrder = vi.fn<OrderLoader>().mockRejectedValue(failure);

    await expect(getOrderLabel("order-1", loadOrder)).rejects.toBe(failure);
  });
});
```

계산·변환은 모킹 응답 자체가 아니라 공개 결과로 단언합니다. 원래 오류 객체를 전달하는 계약은 `rejects.toBe(failure)`로 검증합니다.

## 응답 완료 순서 제어

`Promise.withResolvers`를 지원하는 환경에서 위 `get-order-label.ts`를 사용합니다. 완료 순서를 테스트가 직접 소유해 재현하는 용도이며, 실제 대기 시간이나 임의 `sleep`으로 순서를 맞추지 않습니다. 제품 비동기 API의 취소·동시성 정책을 이 도우미에 맞춰 바꾸지는 않습니다.

```ts
// get-order-label-order.test.ts
import { expect, it, vi } from "vitest";

import { getOrderLabel } from "./get-order-label.ts";
import type { Order, OrderLoader } from "./get-order-label.ts";

it("두 번째 조회가 먼저 끝나도 각 주문의 문구를 반환합니다", async ({ onTestFinished }) => {
  const firstResponse = Promise.withResolvers<Order | undefined>();
  const loadOrder = vi
    .fn<OrderLoader>()
    .mockReturnValueOnce(firstResponse.promise)
    .mockResolvedValueOnce({ id: "order-2", total: 7_200 });

  const firstLabel = getOrderLabel("order-1", loadOrder);
  onTestFinished(async () => {
    // 단언 실패로 첫 응답을 전달하지 못해도 대기 중인 조회를 끝냅니다.
    firstResponse.resolve(undefined);
    await firstLabel;
  });

  const secondLabel = await getOrderLabel("order-2", loadOrder);

  expect(secondLabel).toBe("order-2: 7200원");

  firstResponse.resolve({ id: "order-1", total: 3_600 });

  await expect(firstLabel).resolves.toBe("order-1: 3600원");
});
```

### 참고

- [Promise의 완료 시점 제어](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise/withResolvers)
- [테스트 종료 시 정리](https://vitest.dev/api/hooks.html#ontestfinished)

## import한 외부 모듈 대체

Node.js의 파일 읽기를 모킹해 앞뒤 공백 제거와 오류 전달을 검증합니다. 경로·UTF-8 인코딩은 이 읽기 경계에 전달할 계약이므로 직접 단언합니다. 실제 파일 저장·조회 검증에는 이 모킹을 적용하지 않습니다.

```ts
// load-welcome-message.ts
import { readFile } from "node:fs/promises";

/**
 * UTF-8 파일에서 읽은 환영 문구의 앞뒤 공백을 제거합니다.
 *
 * @param filePath - 읽을 파일 경로. 상대 경로는 현재 작업 디렉터리 기준입니다.
 *
 * @returns 앞뒤 공백을 제거한 문구로 이행되는 Promise. 공백뿐이면 빈 문자열입니다.
 * @throws 파일 읽기 실패 시 원래 오류로 Promise를 거부합니다.
 */
export async function loadWelcomeMessage(filePath: string): Promise<string> {
  const content = await readFile(filePath, "utf8");

  return content.trim();
}
```

```ts
// load-welcome-message.test.ts
import { readFile } from "node:fs/promises";

import { beforeEach, describe, expect, it, vi } from "vitest";

import { loadWelcomeMessage } from "./load-welcome-message.ts";

vi.mock(import("node:fs/promises"));

const readFileMock = vi.mocked(readFile);

beforeEach(() => {
  readFileMock.mockReset();

  return () => {
    readFileMock.mockReset();
  };
});

describe("loadWelcomeMessage", () => {
  it("파일 내용의 앞뒤 공백을 제거합니다", async () => {
    readFileMock.mockResolvedValue("  환영합니다  \n");

    const message = await loadWelcomeMessage("welcome.txt");

    expect(message).toBe("환영합니다");
    expect(readFileMock).toHaveBeenCalledWith("welcome.txt", "utf8");
  });

  it("파일 읽기 실패는 호출부에 전달합니다", async () => {
    const failure = new Error("파일 읽기 실패");
    readFileMock.mockRejectedValue(failure);

    await expect(loadWelcomeMessage("missing.txt")).rejects.toBe(failure);
  });
});
```

- `vi.mock(import(...))`는 모듈 함수를 모킹하고, `vi.mocked`는 타입만 제공합니다.
- `vi.mock`은 import보다 먼저 처리됩니다. 팩토리가 공유할 값은 `vi.hoisted`에서 만들어 초기화 전 접근을 피하고, 응답은 각 사례에서 설정합니다.
- 같은 파일 내부의 직접 호출은 외부 모듈 모킹·`vi.spyOn`으로 바뀌지 않습니다. 모듈 경계나 기존 주입 지점에서 대체합니다.
- 브라우저의 네이티브 ESM export를 추적할 때는 직접 `spyOn`하는 대신 `vi.mock(import(...), { spy: true })`를 사용합니다.

## 공유 모킹의 등록과 초기화

모킹 등록과 사례별 초기화를 구분합니다. 공통화는 같은 검증 경계·상태 소유자·수명에 한정합니다.

| 처리                                   | 위치·시점                                                                                      |
| -------------------------------------- | ---------------------------------------------------------------------------------------------- |
| `vi.mock`·`vi.hoisted` 선언            | 테스트 파일; 부분 대체는 `importOriginal`로 나머지 export 보존                                 |
| 반환값 생성·호출 기록 초기화·출력 수집 | 공통 도우미; 자신이 소유한 모킹만 처리                                                         |
| 준비 훅 등록                           | 파일 또는 테스트 묶음 선언 시 한 번                                                            |
| 성공·실패 응답 설정                    | 각 테스트; 공통 초기화 뒤에 실행                                                               |
| 모킹 복구                              | [정리 순서](integration-tests.md#정리-순서와-완료)에 따라 해당 모킹을 쓰는 마지막 정리 작업 뒤 |

같은 모듈의 부분 모킹은 하나의 팩토리로 구성합니다. 별도 선언은 자동 병합되지 않습니다.
공유 모킹을 바꾸는 사례는 순차 실행하고 입력·출력 상태를 사례별로 만듭니다.

프로젝트의 모든 테스트에 같은 준비·복구를 적용할 때만 `setupFiles`에 둡니다. 대상 구현을 미리 import하면 모듈 캐시 때문에 테스트의 모킹 적용을 막을 수 있으므로 선행 import하지 않습니다.
등록 위치를 바꿨으면 구현을 먼저 import하는 사례에서도 모킹이 적용되는지 검증합니다. `vi.resetModules`로 모듈 캐시를 지워도 모킹 등록 자체는 해제되지 않습니다.

참고: [Vitest 모킹의 변환·캐시 제약](https://vitest.dev/api/vi.html#vi-mock)

## 시간 경계와 복구

현재 시각에 따른 분기를 검증할 때는 가짜 시계로 경계 직전·경계·직후를 지정하고, 사례가 끝나면 실제 시계로 복구합니다. 아래 계약의 시간 단위는 밀리초이며 만료 시각을 포함합니다.

```ts
// is-expired.ts

/**
 * 현재 시각이 만료 시각에 도달했는지 확인합니다.
 *
 * @param expiresAt - Unix epoch 기준 밀리초 단위의 만료 시각
 *
 * @returns 만료 시각과 같거나 지났으면 true
 */
export function isExpired(expiresAt: number): boolean {
  return Date.now() >= expiresAt;
}
```

```ts
// is-expired.test.ts
import { afterEach, expect, it, vi } from "vitest";

import { isExpired } from "./is-expired.ts";

afterEach(() => {
  vi.useRealTimers();
});

it.each([
  { name: "만료 직전", offsetMs: -1, expected: false },
  { name: "만료 시각", offsetMs: 0, expected: true },
  { name: "만료 직후", offsetMs: 1, expected: true },
])("$name이면 만료 여부는 $expected입니다", ({ offsetMs, expected }) => {
  const expiresAt = Date.UTC(2030, 0, 1);
  vi.useFakeTimers();
  vi.setSystemTime(expiresAt + offsetMs);

  const expired = isExpired(expiresAt);

  expect(expired).toBe(expected);
});
```

`vi.setSystemTime`은 시각만 바꿉니다. 타이머 콜백까지 실행하려면 [`await vi.advanceTimersByTimeAsync(ms)`](https://vitest.dev/api/vi.html#vi-advancetimersbytimeasync)로 진행합니다.
가짜 시계와 전역 모킹을 바꾸는 사례는 순차 실행합니다. 테스트별로 독립된 함수 모킹만 쓰는 사례는 공유 상태가 없을 때 `concurrent`를 선택하고, 그 콜백의 `{ expect }`로 단언합니다.
`expect.poll`은 상태만 조회할 수 있을 때 쓰고 콜백에는 조회만 둡니다. 임의 재시도·대기·`skip`으로 실패를 감추지 않습니다.

## 변경한 항목 복구

| 변경 항목                    | 정리 방법                                       |
| ---------------------------- | ----------------------------------------------- |
| 호출 기록만 초기화           | `mockClear()`·`vi.clearAllMocks()`; 구현은 유지 |
| 호출 기록·구현 초기화        | `mockReset()`·`vi.resetAllMocks()`              |
| `vi.spyOn`으로 교체한 메서드 | `mockRestore()`·`vi.restoreAllMocks()`          |
| 가짜 시계                    | `vi.useRealTimers()`                            |
| `vi.stubGlobal`·`vi.stubEnv` | `vi.unstubAllGlobals()`·`vi.unstubAllEnvs()`    |

자신이 만든 모킹의 메서드로 필요한 상태만 초기화합니다. 파일·프로젝트 전체가 같은 복구 수명을 소유할 때만 `vi.*AllMocks`를 공통 훅에 둡니다. `vi.restoreAllMocks`는 임의 `vi.fn`·모듈 모킹·가짜 시계·환경 변수를 복구하지 않습니다.
`mockReset`은 구현도 재설정하므로 응답은 초기화 뒤 각 테스트에서 지정합니다. 모킹은 그것을 사용하는 마지막 자원 정리 뒤에 복구합니다.

## 참고

- [Vitest 모듈 모킹](https://vitest.dev/guide/mocking/modules)
- [vi 유틸리티](https://vitest.dev/api/vi)
