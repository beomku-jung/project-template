# 값과 타입 검증

공개 제네릭·유니온·타입 유틸리티의 추론 결과가 계약이면 값의 테스트에 타입 단언을 함께 둡니다. 구현 가까이의 일반 `.test.ts`에 작성하고 JSX는 `.test.tsx`, 기존 `.spec`은 그대로 사용합니다. 타입 테스트 전용 접미사·디렉터리는 만들지 않습니다.
값은 Vitest 실행으로, `expectTypeOf`는 해당 파일을 포함한 실제 타입 진단으로 검증합니다. 일반 Vitest 실행은 TypeScript를 변환할 뿐 타입 오류를 검사하지 않습니다.

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

기대 타입은 공개 계약에 따라 직접 명시합니다. `ReturnType<typeof firstItem>`에서 가져오면 잘못된 반환 타입 변경도 따라가므로 회귀를 잡을 수 없습니다. 값의 단언과 타입의 단언은 같은 사례에 남겨 어떤 입력에서 어떤 추론을 보장하는지 드러냅니다.

## 파일 포함과 검사 명령

아래 명령은 `libs/example/src/`의 위 두 파일을 패키지 `tsconfig.json`이 함께 포함하는 경우입니다. 실행 전에 실제 루트·패키지의 `package.json`, Vitest 설정, tsconfig에서 다음 전제를 대조합니다.

- 루트 `test`는 `vitest run`, `check`는 포맷 → Oxlint 린트 및 `--type-check` → 철자 → 미사용 검사 → 런타임 테스트를 `&&`로 연결합니다.
- 루트 Vitest의 `projects`는 `apps/*`·`libs/*`·`tools/*`를 수집하며, 예제 패키지에도 테스트 설정이 있습니다.
- 루트 tsconfig는 `*.config.ts`만 포함합니다. 선택적 루트 `typecheck`는 `tsc && pnpm --recursive --if-present run typecheck`로 루트와 스크립트가 있는 패키지를 검사합니다.

저장소 루트에서 실행하고 각 블록의 `package_path`를 실제 경로로 바꿉니다. 전제가 다르면 실제 스크립트·설정에 맞춥니다. `libs/example`은 설명용 경로이며, 이 예제만을 근거로 없는 스크립트·패키지·도구를 추가하지 않습니다.

```sh
package_path=libs/example
pnpm exec tsc --project "$package_path/tsconfig.json" --showConfig
pnpm exec tsc --project "$package_path/tsconfig.json" --listFilesOnly
pnpm exec oxlint --debug files
```

- `--showConfig`: 상속을 반영한 `files`·`include`·`exclude`와 실행 환경의 `types`·`lib`를 확인합니다.
- `--listFilesOnly`: 결과에 구현 `src/first-item.ts`와 일반 테스트 `src/first-item.test.ts`의 실제 경로가 모두 있어야 합니다. 이 목록에는 import로 포함된 파일·타입 선언도 나옵니다.
- `oxlint --debug files`: Oxlint가 선택한 진입 파일에도 두 파일이 있는지 별도로 대조합니다. tsconfig의 `include`·`exclude`만으로 Oxlint의 탐색 범위를 보장하지 않습니다.

위 명령은 설정·파일 목록의 증거이지 타입 진단·런타임 실행의 증거가 아닙니다. 테스트가 빠졌으면 포함 범위를 고칩니다. Node.js·브라우저 전역 타입 충돌로 설정을 나눴으면 테스트용 설정으로 `--project`를 바꿔 확인하고 기존 타입 검사 단계에도 연결합니다.

특정 사례의 런타임 재현은 파일 경로로, 전체 검사는 `check`로 실행합니다.

```sh
package_path=libs/example

# 해당 파일의 런타임만 빠르게 재현할 때
pnpm test "$package_path/src/first-item.test.ts"

# 전체 검사: 앞 단계가 실패하면 뒤 단계는 실행되지 않음
pnpm check
```

패키지의 경로가 루트 `projects` 검색 범위에 있어도 패키지 테스트 설정이 실제 파일을 수집해야 합니다. 미수집 성공이나 `passWithNoTests`는 이 사례의 통과가 아닙니다. `--type-check`는 루트 `check`에만 두고 전역 `typeCheck` 옵션·`fix`·훅에 추가하지 않습니다.

Oxlint와 `tsc`의 진단 차이를 확인하거나 `tsc` 검사를 별도로 요청받으면 아래 선택적 명령을 사용합니다. `pnpm typecheck`는 실제 선언된 기존 스크립트가 있을 때만 사용합니다.

```sh
package_path=libs/example

# 위 파일을 포함하는 설정에 대한 직접 타입 검사
pnpm exec tsc --project "$package_path/tsconfig.json" --noEmit

# 루트와 각 패키지에 선언된 typecheck 스크립트 실행
pnpm typecheck
```

직접 `tsc`와 기존 `typecheck`가 같은 설정을 검사하면 하나만 실행합니다. 루트 tsconfig가 설정 파일만 포함하고 패키지 `typecheck`가 선택적인 구성에서는 루트 성공만으로 패키지 테스트의 타입 검증을 보고하지 않습니다. 실제 패키지 스크립트와 포함 목록에서 대상 테스트를 확인합니다.
누락을 감추려고 타입 단언 파일을 런타임 수집에서 제외하거나 특수 접미사로 바꾸지 않습니다. 실행한 명령·대상 설정·실제 포함 파일·진단 결과를 보고합니다.

## 참고

- [Vitest 타입 테스트](https://vitest.dev/guide/testing-types)
- [expectTypeOf](https://vitest.dev/api/expect-typeof)
- [TypeScript CLI의 설정·파일 목록 옵션](https://www.typescriptlang.org/docs/handbook/compiler-options.html#cli-commands)
- [TypeScript include](https://www.typescriptlang.org/tsconfig/include.html)
- [TypeScript exclude와 import 포함](https://www.typescriptlang.org/tsconfig/exclude.html)
- [Oxlint CLI의 파일 탐색 진단](https://oxc.rs/docs/guide/usage/linter/cli.md)
