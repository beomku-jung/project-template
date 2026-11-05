# 통합 테스트

| 검증할 내용         | 예제·기준                                                    |
| ------------------- | ------------------------------------------------------------ |
| 파일 저장·조회      | [메모 저장 후 다시 읽기](#실제-파일에-저장한-메모-다시-읽기) |
| 공통 준비·자원 정리 | [수명과 훅 선택](#수명과-훅-선택)                            |
| DB·HTTP 검증        | [다른 자원에 적용](#다른-자원에-적용)                        |
| 배포물·원격 호환성  | [실제 환경과 배포물](#실제-환경과-배포물)                    |

## 실제 파일에 저장한 메모 다시 읽기

`mkdtempDisposable`·`await using`을 지원하는 Node.js 환경에서 실행합니다.

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

| 사용 조건                                                            | 기본 선택                                                             |
| -------------------------------------------------------------------- | --------------------------------------------------------------------- |
| 테스트 본문에서 사용을 마치는 `Disposable`·`AsyncDisposable`         | `using`·`await using` 그대로 사용                                     |
| 준비 훅에서 만든 자원을 테스트까지 유지하거나 `afterEach`에서도 사용 | 획득 직후 현재 컨텍스트의 `onTestFinished`에 정리 등록                |
| DOM 제거·보류 Promise 해제처럼 별도의 정리 함수가 필요               | `onTestFinished`에 정리 함수 등록                                     |
| 모든 사례에 공통으로 필요한 상태 복구                                | `afterEach`                                                           |
| 테스트 묶음 전체에서 같은 인스턴스를 사용                            | 격리를 보장할 수 있을 때 `beforeAll` + 반환 정리 함수 또는 `afterAll` |

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

| 정리할 대상                | 완료 확인                                                                  |
| -------------------------- | -------------------------------------------------------------------------- |
| 직접 연 파일·서버          | 종료 Promise를 기다린 뒤 자원 회수 확인                                    |
| 진행 중인 요청·작업        | 취소·중단을 요청하고 완료를 기다린 뒤 사용 자원 정리                       |
| 테스트가 만든 보류 Promise | [응답 순서 예제](mocking.md#응답-완료-순서-제어)처럼 단언 실패 시에도 해제 |

테스트 타임아웃은 작업을 중단하지 않습니다. 정리 스코프·훅에서 실제 종료를 기다립니다.

### 참고

- [TypeScript의 using 스코프](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-5-2.html#using-declarations-and-explicit-resource-management)
- [Vitest 정리 훅](https://vitest.dev/api/hooks.html#ontestfinished)

## 다른 자원에 적용

시작 비용이 큰 자원을 공유하더라도 컨텍스트·데이터는 사례별로 격리합니다.

### DB

- 실제 엔진의 전용 DB·스키마·트랜잭션에서 검증할 마이그레이션·쿼리를 실행합니다.
- 공유 테이블에서는 해당 사례의 데이터만 정리합니다.
- 롤백은 같은 트랜잭션의 롤백 가능한 변경에만 적용됩니다. 파일·외부 요청은 별도로 정리합니다.

참고: [LobeHub DB 테스트 예제](https://github.com/lobehub/lobehub/blob/canary/.agents/skills/testing/references/db-model-test.md)

### HTTP 서버

임의 포트로 시작하고 종료 훅을 등록합니다. 계약에 따른 상태 코드·응답을 검증하고, 저장 동작을 다루면 저장 결과도 확인합니다.

## 실제 환경과 배포물

로컬 대체로 검증할 수 없는 계약에는 실제 환경을 사용합니다. 아래 조건은 검증할 계약에 해당할 때 적용합니다.

| 대상                    | 확인할 조건                                                                           |
| ----------------------- | ------------------------------------------------------------------------------------- |
| 배포물                  | 저장소·개발 실행 환경이 없는 컴퓨터·컨테이너에서 배포 파일과 선언한 의존성만으로 실행 |
| 네이티브 코드·정적 자산 | 도움말 실행에 그치지 않고 해당 연산·요청 수행                                         |
| 상태·세션이 있는 앱     | 재시작 시 보존·폐기 계약과 종료 뒤 자원 회수                                          |
| 원격 서비스             | 인증·호환성 등 로컬에서 검증할 수 없는 계약, 호출·시간·비용 한도와 정리 조건          |

검증한 리비전·운영체제·아키텍처·외부 조건과 미확인 범위를 보고합니다.
