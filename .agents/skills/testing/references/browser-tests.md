# 브라우저 테스트

사용자가 보는 DOM·이벤트·브라우저 API가 계약이면 브라우저 모드를 선택합니다. 단순 계산만 확인할 때는 브라우저를 띄우지 않습니다.
예제를 적용하려면 Vitest 브라우저 프로젝트와 브라우저 프로바이더(provider) 패키지가 있어야 합니다. 없으면 먼저 구성합니다. 프로바이더는 `page`·`expect.element` 타입도 제공합니다.
`include`·`exclude`로 브라우저 파일을 Node.js 프로젝트에서 제외하고 브라우저용 타입 검사에 포함합니다. Node.js 실행 성공으로 브라우저 검증을 대신하지 않습니다.

## 버튼 클릭으로 수량 증가

두 파일을 같은 디렉터리에 둡니다. JSX 없이 DOM을 사용하므로 확장자는 `.ts`입니다.

```ts
// create-cart-counter.ts

/**
 * 1에서 시작해 버튼을 누를 때마다 수량이 증가하는 UI를 만듭니다.
 *
 * @returns 문서에 연결되지 않은 요소. 호출자가 DOM에 추가하고 사용 후 제거합니다.
 */
export function createCartCounter(): HTMLElement {
  const root = document.createElement("div");

  let quantity = 1;
  const output = document.createElement("output");
  output.setAttribute("aria-label", "수량");
  output.textContent = String(quantity);

  const button = document.createElement("button");
  button.type = "button";
  button.textContent = "수량 추가";

  button.addEventListener("click", () => {
    quantity += 1;
    output.textContent = String(quantity);
  });

  root.append(output, button);

  return root;
}
```

```ts
// create-cart-counter.browser.test.ts
import { expect, it } from "vitest";
import { page } from "vitest/browser";

import { createCartCounter } from "./create-cart-counter.ts";

it("수량 추가 버튼을 누르면 화면의 수량이 1 증가합니다", async ({ onTestFinished }) => {
  const counter = createCartCounter();
  onTestFinished(() => counter.remove());

  document.body.append(counter);
  const quantity = page.getByLabelText("수량");

  await expect.element(quantity).toHaveTextContent("1");

  await page.getByRole("button", { name: "수량 추가", exact: true }).click();

  await expect.element(quantity).toHaveTextContent("2");
});
```

## 탐색·조작·단언

| 목적        | 기준                                                                                              |
| ----------- | ------------------------------------------------------------------------------------------------- |
| 요소 찾기   | `getByRole`·`getByLabelText` 우선; 역할·레이블로 안정적으로 구별할 수 없을 때 테스트 ID           |
| 사용자 조작 | `locator`의 `click`·`fill` 또는 `vitest/browser`의 `userEvent`; 반환 작업을 항상 `await`          |
| 화면 단언   | `await expect.element(locator)`로 사용자에게 보이는 상태를 기다림; 고정 시간 대기로 대체하지 않음 |
| 자원 정리   | 직접 만든 DOM은 제거; 렌더러가 자동 정리하지 않는 DOM·캐시·전역 상태는 정리 훅에 등록             |

`querySelector(...).click()`·수동 이벤트 호출로 브라우저의 조작 가능 상태 검사를 우회하지 않습니다.
컴포넌트 내부 메서드·상태 저장 구조·이벤트 핸들러 연결 여부보다 역할·레이블·화면 결과를 단언합니다. 접근성 이름 자체가 계약이면 그 이름도 직접 검증합니다.
화면이 늦게 바뀌면 요청 완료·상태 전이·locator의 관찰 조건을 확인합니다. 실패를 감추려고 대기 시간·재시도 횟수를 늘리지 않습니다.

JSX를 사용하면 프레임워크 렌더러와 `.tsx` 확장자로 바꿉니다.

| 검증할 동작                     | 실행 환경                                              |
| ------------------------------- | ------------------------------------------------------ |
| 페이지 전환·인증 UI             | 실제 앱 구성·메모리 라우터로 렌더링하고 HTTP 응답 통제 |
| 서버 연결·쿠키·CORS·서비스 워커 | 해당 앱의 E2E 환경                                     |

HTTP 응답을 통제한 UI 테스트는 실제 서버 연결·인증 성공을 입증하지 않습니다. 그 경계가 검증 대상이면 실제 연동 환경에서 확인합니다.

## 공통 렌더링과 정리

같은 브라우저 프로젝트가 소유하고 같은 수명으로 복구할 DOM·캐시·전역 상태만 `setupFiles`에 모읍니다. 한 파일의 준비는 그 파일에, 같은 영역에서 함께 쓰는 렌더링 도우미는 해당 영역의 `testing/`에 둡니다.
렌더링 도우미는 렌더링과 자신이 만든 자원의 정리를 함께 소유합니다. 사례별 HTTP 응답·옵션, 화면 전환·중간 언마운트, 화면 단언은 테스트에 남깁니다.

DOM 정리가 `afterEach`에 있고 호출별 캐시·모킹을 사용한다면, 그 캐시·모킹은 이후 `onTestFinished`에서 복구합니다. 다른 의존 관계는 [정리 순서](integration-tests.md#정리-순서와-완료)에 맞춥니다.
같은 `document`·전역 캐시를 변경하는 사례는 `concurrent`로 실행하지 않습니다.

## 참고

- [Vitest 컴포넌트 테스트](https://vitest.dev/guide/browser/component-testing)
- [Playwright 테스트 원칙](https://playwright.dev/docs/best-practices)
