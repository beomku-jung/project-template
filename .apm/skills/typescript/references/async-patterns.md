# 비동기 예제

## HTTP 응답

- 선행 파일: [외부 입력 예제](type-patterns.md#외부-입력)의 `parse-user.ts`를 같은 디렉터리에 배치합니다.
- 실행 환경: `fetch`·`Response.json`·top-level await를 지원하는 독립된 Node.js 프로세스입니다. 사용 예제는 고정 응답으로 파싱·정규화 경로만 보여 주며 네트워크 연결 성공을 증명하지 않습니다.
- 실행 결과: 패키지의 TypeScript 실행 명령으로 사용 예제를 실행하면 `{ id: "u1", name: "민수" }`를 출력합니다. 전역 `fetch` 교체는 이 독립된 예제 안에서만 사용하고 `finally`에서 복구합니다.
- 실제 API 적용: `api.example.com`을 실제 API 주소로 바꿉니다. 고정 응답 교체 코드를 제품 코드의 실패 대체값으로 넣지 않습니다.

````ts
// load-user.ts
import { parseUser } from "./parse-user.ts";
import type { User } from "./parse-user.ts";

/**
 * API에서 사용자를 조회하고 {@link parseUser}로 응답을 검증합니다.
 *
 * @param id - URL 경로에 인코딩해 넣을 사용자 식별자
 *
 * @returns 검증·정규화된 {@link User}로 이행되는 Promise
 * @throws Error
 * 네트워크·HTTP·응답 파싱·검증 실패 시 Promise를 거부합니다.
 * 원래 오류는 cause에 보존합니다.
 *
 * @example 외부 요청 없이 사용자 응답 처리
 * ```ts
 * import { loadUser } from "./load-user.ts";
 *
 * const originalFetch = globalThis.fetch;
 *
 * try {
 *   globalThis.fetch = () => Promise.resolve(Response.json({ id: " u1 ", name: " 민수 " }));
 *
 *   const user = await loadUser("u1");
 *
 *   console.log(user); // { id: "u1", name: "민수" }
 * } finally {
 *   globalThis.fetch = originalFetch;
 * }
 * ```
 */
export async function loadUser(id: string): Promise<User> {
  try {
    const url = `https://api.example.com/users/${encodeURIComponent(id)}`;
    const response = await fetch(url);

    if (!response.ok) {
      throw new Error(`HTTP ${response.status}`);
    }

    const value: unknown = await response.json();

    return parseUser(value);
  } catch (cause) {
    throw new Error(`사용자 ${id} 조회 실패`, { cause });
  }
}
````

미존재를 정상 결과로 제공하는 API라면 그 HTTP 상태를 응답 본문 파싱 전에 구별하고, 반환 타입에도 그 결과를 표현합니다. 모든 비정상 응답을 `null`·빈 사용자 같은 성공값으로 바꾸지 않습니다.
이 예제는 취소 신호·시간 제한·재시도 계약을 제공하지 않습니다. 호출 계약이 취소를 요구하면 `AbortSignal`을 실제 요청에 전달하고, 요청·본문 처리 중 어느 단계에서 거부되는지 API에 드러냅니다.

참고: [Error.cause](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Error/cause)

## 파일 I/O

- 실행 환경: `mkdtempDisposable`·비동기 정리 심볼을 지원하는 Node.js와 `await using`·top-level await를 처리하는 런타임 또는 변환기가 필요합니다. 타입 선언이나 구문 변환만으로 Node.js API가 추가되지는 않습니다.
- 실행 효과: 사용 예제는 임시 디렉터리에 파일 세 개를 쓰고, 스코프 종료 시 디렉터리를 삭제합니다. 실행 명령과 자원 정리 지원은 패키지의 런타임·빌드 설정을 따릅니다.
- 성공 기준: `combineFiles`가 완료된 뒤 읽은 내용은 `"제목\n본문"`입니다. 출력 파일의 쓰기 완료는 원자적 교체나 저장 장치로의 영속화 보장을 뜻하지 않습니다.
- 참고: [Node.js 임시 디렉터리 정리](https://nodejs.org/api/fs.html#fspromisesmkdtempdisposableprefix-options)

````ts
// combine-files.ts
import { readFile, writeFile } from "node:fs/promises";

/**
 * 두 UTF-8 파일을 줄바꿈 하나로 이어 출력 파일에 씁니다.
 *
 * @remarks
 * - 두 파일을 모두 읽은 뒤 쓰기를 시작합니다. 읽기 실패 시에도 다른 읽기가 끝난 뒤 거부합니다.
 * - 기존 출력 파일은 덮어씁니다.
 * - 쓰기 실패 시 출력 파일이 일부만 기록될 수 있습니다.
 *
 * @param headerPath - 앞에 넣을 파일 경로
 * @param bodyPath - 뒤에 넣을 파일 경로
 * @param outputPath - 결과를 저장할 파일 경로
 *
 * @returns 출력 파일 쓰기가 완료되면 이행되는 Promise
 * @throws 파일 읽기·쓰기 실패 시 해당 오류로 Promise를 거부합니다.
 *
 * @example 임시 파일 결합 후 내용 확인
 * ```ts
 * import { mkdtempDisposable, readFile, writeFile } from "node:fs/promises";
 * import { tmpdir } from "node:os";
 * import { join } from "node:path";
 *
 * import { combineFiles } from "./combine-files.ts";
 *
 * await using directory = await mkdtempDisposable(join(tmpdir(), "combine-files-"));
 * const headerPath = join(directory.path, "header.txt");
 * const bodyPath = join(directory.path, "body.txt");
 * const outputPath = join(directory.path, "output.txt");
 *
 * await writeFile(headerPath, "제목", "utf8");
 * await writeFile(bodyPath, "본문", "utf8");
 *
 * await combineFiles(headerPath, bodyPath, outputPath);
 *
 * const content = await readFile(outputPath, "utf8");
 *
 * console.log(content); // "제목\n본문"
 * ```
 */
export async function combineFiles(
  headerPath: string,
  bodyPath: string,
  outputPath: string,
): Promise<void> {
  const reads = [readFile(headerPath, "utf8"), readFile(bodyPath, "utf8")] as const;

  try {
    const [header, body] = await Promise.all(reads);

    await writeFile(outputPath, `${header}\n${body}`, "utf8");
  } finally {
    // 읽기 하나가 실패해도 호출자가 정리하기 전에 나머지 읽기를 마칩니다.
    await Promise.allSettled(reads);
  }
}
````

`Promise.all`은 첫 거부를 전달하지만 이미 시작한 작업을 취소하지 않습니다. 이 예제의 `finally`는 두 읽기가 모두 끝날 때까지 기다리므로 호출자가 임시 디렉터리를 지우는 시점에 읽기 작업이 남지 않습니다.
`Promise.all` 자체에는 동시 실행 수 제한이 없습니다.

참고: [Promise.all](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise/all)
