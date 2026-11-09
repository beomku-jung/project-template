# 통합 테스트

파일·DB·HTTP 경계의 실제 저장·조회가 계약일 때 통합 테스트를 선택합니다. 해당 경계는 실제 구현을 사용하고, 그 밖의 원격 의존성만 실패 재현·격리에 필요한 만큼 대체합니다. 저장 함수를 모킹한 뒤 같은 응답을 단언하는 것으로 저장 계약을 검증하지 않습니다.

## 실제 파일에 저장한 메모 다시 읽기

`mkdtempDisposable`·`await using`을 지원하는 Node.js 환경에서 실행합니다.
아래 예제는 입력·기대값을 테스트 표에 두고 실제 파일을 다시 읽습니다. 임시 디렉터리와 파일 경로는 사례별로 소유하므로 `concurrent`로 실행할 수 있습니다.

```ts
// save-note.ts
import { writeFile } from "node:fs/promises";

/**
 * 앞뒤 공백을 제거한 메모로 UTF-8 파일을 덮어씁니다.
 *
 * @remarks
 * 파일이 없으면 생성하지만 상위 디렉터리는 만들지 않습니다.
 * 쓰기 실패 시 기존 내용 복구를 보장하지 않습니다.
 *
 * @param filePath - 저장할 파일 경로. 상대 경로는 현재 작업 디렉터리 기준입니다.
 * @param content - 저장할 메모. 공백뿐이면 빈 파일로 저장합니다.
 *
 * @returns 파일 쓰기가 완료되면 이행되는 Promise
 * @throws 파일 쓰기 실패 시 해당 오류로 Promise를 거부합니다.
 */
export function saveNote(filePath: string, content: string): Promise<void> {
  return writeFile(filePath, content.trim(), "utf8");
}
```

```ts
// save-note.integration.test.ts
import { mkdtempDisposable, readFile } from "node:fs/promises";
import { tmpdir } from "node:os";
import { join } from "node:path";

import { it } from "vitest";

import { saveNote } from "./save-note.ts";

it.concurrent.for([
  {
    name: "메모의 앞뒤 공백을 제거해 다시 읽습니다",
    input: "  다음 회의 안건  \n",
    expected: "다음 회의 안건",
  },
  {
    name: "공백뿐인 메모를 빈 문자열로 저장합니다",
    input: "  \n",
    expected: "",
  },
])("$name", async ({ input, expected }, { expect }) => {
  await using directory = await mkdtempDisposable(join(tmpdir(), "note-store-"));
  const filePath = join(directory.path, "note.txt");

  await saveNote(filePath, input);
  const content = await readFile(filePath, "utf8");

  expect(content).toBe(expected);
});
```

`await using`은 정상 완료·예외 발생 시 모두 임시 디렉터리와 그 내용의 삭제를 기다립니다.

### 참고

- [Vitest의 입력별 테스트와 컨텍스트](https://vitest.dev/api/test#test-for)
- [Node.js 임시 디렉터리 정리](https://nodejs.org/api/fs.html#fspromisesmkdtempdisposableprefix-options)

## 수명과 훅 선택

자원을 쓰는 테스트 본문이 수명을 소유합니다. 준비를 공통화할 때는 아래 수명에 따라 훅을 고르고, 사례별 데이터·단언은 테스트에 남깁니다.

| 사용 조건                                                            | 기본 선택                                                                           |
| -------------------------------------------------------------------- | ----------------------------------------------------------------------------------- |
| 테스트 본문에서 사용을 마치는 `Disposable`·`AsyncDisposable`         | `using`·`await using` 그대로 사용                                                   |
| 준비 훅에서 만든 자원을 테스트까지 유지하거나 `afterEach`에서도 사용 | 획득 직후 현재 컨텍스트의 `onTestFinished`에 정리 등록                              |
| DOM 제거·보류 Promise 해제처럼 별도의 정리 함수가 필요               | `onTestFinished`에 정리 함수 등록                                                   |
| 모든 사례에 공통으로 필요한 상태 복구                                | `afterEach`                                                                         |
| 모든 사례가 같은 인스턴스를 써야 하고 사례별 상태를 분리할 수 있음   | `beforeAll` + 반환 정리 함수 또는 `afterAll`; 각 사례가 변경할 데이터는 별도로 생성 |

`beforeEach` 안의 `await using`은 준비 훅이 끝날 때 정리되므로 테스트 본문까지 자원을 유지하지 못합니다.

### 준비 훅에서 만든 자원 유지

위 `save-note.ts`를 사용하는 공통 준비 예제입니다.
테스트 묶음에서 경로 변수를 공유하므로 `concurrent`로 실행하지 않습니다.

```ts
// note-files.integration.test.ts
import { mkdtempDisposable, readFile } from "node:fs/promises";
import { tmpdir } from "node:os";
import { join } from "node:path";

import { beforeEach, expect, it } from "vitest";

import { saveNote } from "./save-note.ts";

let directoryPath: string;

beforeEach(async ({ onTestFinished }) => {
  const directory = await mkdtempDisposable(join(tmpdir(), "note-files-"));

  // 이후 준비가 실패해도 정리되도록 자원을 얻은 직후 등록합니다.
  onTestFinished(() => directory[Symbol.asyncDispose]());

  directoryPath = directory.path;
});

it.each(["첫 메모", "다음 메모"])("파일에 저장한 %s를 다시 읽습니다", async (content) => {
  const filePath = join(directoryPath, "note.txt");

  await saveNote(filePath, content);
  const savedContent = await readFile(filePath, "utf8");

  expect(savedContent).toBe(content);
});
```

### 정리 순서와 완료

1. 테스트 본문의 `using`·`await using`
2. `afterEach`
3. `beforeEach`가 반환한 정리 함수
4. `onTestFinished` — 등록의 역순

모킹은 이를 사용하는 마지막 정리 작업 뒤에 복구합니다.

직접 연 파일·서버는 종료 Promise를 기다린 뒤 자원 회수를 확인합니다.

테스트 타임아웃은 작업을 중단하지 않습니다. 진행 중인 작업이 있으면 정리 스코프·훅에서 다음 순서를 지킵니다.

1. 취소·중단 계약이 있는 작업에는 종료를 요청합니다. 테스트가 만든 보류 Promise가 있으면 [응답 순서 예제](mocking.md#응답-완료-순서-제어)처럼 단언 실패 시에도 해제합니다. 두 조건이 모두 해당하면 둘 다 수행합니다.
2. 원래 작업을 모두 추적해 `Promise.allSettled`로 이행·거부가 끝날 때까지 기다립니다. `Promise.all`의 첫 거부로 나머지 작업을 남기지 않습니다.
3. 모든 작업이 끝나면 사용 자원을 정리하고 각 거부 결과를 검사합니다. 본문에서 계약에 따른 기대 거부로 단언이 성공했다면 다시 실패시키지 않습니다. 테스트가 직접 요청한 취소 사유와 해당 작업의 취소 계약에 일치하는 거부도 예상 종료로 처리합니다. 오류 이름이 `AbortError`라는 이유만으로 무시하지 않습니다.
4. 예상되지 않은 거부는 자원 정리가 끝난 뒤 실패로 보고하거나 전파합니다.

정리 예제에도 전체 작업의 완료 대기·자원 정리·거부 보고를 처음부터 포함합니다.

### 참고

- [TypeScript의 using 스코프](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-5-2.html#using-declarations-and-explicit-resource-management)
- [Vitest 정리 훅](https://vitest.dev/api/hooks.html#ontestfinished)

## 다른 자원에 적용

시작 비용이 큰 서버·DB 프로세스는 공유할 수 있지만, 검증 경계·상태 소유자·수명이 같고 사례별 컨텍스트·데이터를 분리할 수 있을 때만 공유합니다. 서로의 변경을 읽거나 지울 수 있으면 인스턴스를 분리하거나 순차 실행합니다.

### DB

- 실제 엔진의 전용 DB·스키마·트랜잭션에서 검증할 마이그레이션·쿼리를 실행합니다. 쿼리 호출 여부 대신 조회 결과·제약 위반·저장 상태를 계약과 대조합니다.
- 공유 테이블에서는 해당 사례의 데이터만 정리합니다. 다른 사례가 사용할 데이터를 전체 삭제하는 공통 정리는 만들지 않습니다.
- 롤백은 같은 트랜잭션의 롤백 가능한 변경에만 적용됩니다. 파일·외부 요청은 별도로 정리합니다.

참고: [LobeHub DB 테스트 예제](https://github.com/lobehub/lobehub/blob/canary/.agents/skills/testing/references/db-model-test.md)

### HTTP 서버

임의 포트로 시작하고 종료 훅을 등록합니다. 계약에 따른 상태 코드·응답을 검증하고, 저장 동작을 다루면 저장 결과도 확인합니다.
서버 준비·요청 완료 Promise를 직접 기다립니다. 별도 작업의 상태만 조회할 수 있으면 조회만 하는 `expect.poll`로 완료 조건을 기다리고, 그 안에서 생성 요청을 반복하지 않습니다.

## 실제 환경과 배포물

로컬 대체로 검증할 수 없는 계약에만 실제 환경을 사용합니다. 원격 호출은 허용된 격리 환경·호출 및 비용 한도·정리 조건을 갖춘 경우에만 실행합니다. 조건이 없으면 미검증으로 보고하고, 빈 테스트나 무조건 `skip`으로 성공 처리하지 않습니다.

| 대상                    | 확인할 조건                                                                           |
| ----------------------- | ------------------------------------------------------------------------------------- |
| 배포물                  | 저장소·개발 실행 환경이 없는 컴퓨터·컨테이너에서 배포 파일과 선언한 의존성만으로 실행 |
| 네이티브 코드·정적 자산 | 도움말 실행에 그치지 않고 해당 연산·요청 수행                                         |
| 상태·세션이 있는 앱     | 재시작 시 보존·폐기 계약과 종료 뒤 자원 회수                                          |
| 원격 서비스             | 인증·호환성 등 로컬에서 검증할 수 없는 계약, 호출·시간·비용 한도와 정리 조건          |

검증한 리비전·운영체제·아키텍처·외부 조건과 미확인 범위를 보고합니다.
