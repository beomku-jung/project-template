# 비동기 초기화와 소유권

비동기 준비를 끝낸 인스턴스를 반환하고, 직접 획득한 자원만 실패·종료 시 정리합니다.

## 준비가 끝난 인스턴스 반환

Node.js의 `FileHandle`·`mkdtempDisposable`·비동기 정리 심볼을 지원하는 런타임과 `await using`을 처리하는 런타임 또는 변환기가 필요합니다. `AsyncDisposable` 타입 선언이나 구문 변환만으로 Node.js API가 추가되지는 않습니다.
`text-log.ts`를 저장하고 TSDoc의 사용 예제를 같은 디렉터리의 별도 파일에 둡니다. 패키지가 정의한 실행 명령으로 호출했을 때 쓰기·종료를 마친 파일 내용이 `"ready\n"`이면 예제의 정상 흐름을 만족합니다. 종료는 파일 삭제나 디스크 영속화를 뜻하지 않으며, 임시 디렉터리 삭제는 바깥 스코프가 맡습니다.

````ts
// text-log.ts
import { open } from "node:fs/promises";
import type { FileHandle } from "node:fs/promises";

export interface TextLogOptions {
  /** 추가 모드로 연 일반 파일입니다. 생략하면 path를 열며, 제공하면 path는 사용하지 않습니다. */
  file?: FileHandle;
}

/**
 * UTF-8 문구를 파일 끝에 기록하고 직접 연 파일의 수명을 관리합니다.
 *
 * @remarks 호출자가 각 쓰기를 순서대로 기다린 뒤 종료해야 합니다.
 *
 * @example 임시 파일에 기록한 뒤 닫기
 * ```ts
 * import { mkdtempDisposable, readFile } from "node:fs/promises";
 * import { tmpdir } from "node:os";
 * import { join } from "node:path";
 *
 * import { TextLog } from "./text-log.ts";
 *
 * await using directory = await mkdtempDisposable(join(tmpdir(), "text-log-"));
 * const path = join(directory.path, "events.log");
 *
 * {
 *   await using log = await TextLog.open(path);
 *
 *   await log.append("ready\n");
 * }
 *
 * const content = await readFile(path, "utf8");
 *
 * console.log(content); // "ready\n"
 * ```
 */
export class TextLog implements AsyncDisposable {
  readonly #file: FileHandle;
  readonly #ownsFile: boolean;
  #closed = false;

  private constructor(file: FileHandle, ownsFile: boolean) {
    this.#file = file;
    this.#ownsFile = ownsFile;
  }

  /**
   * 파일을 준비하고 일반 파일인지 확인한 뒤 반환합니다.
   *
   * @remarks 초기화 실패 시에도 직접 연 파일만 닫습니다. 실패 전에 새로 만든 파일은 제거하지 않습니다.
   *
   * @param path - 파일을 주입하지 않았을 때 추가 모드로 열 경로. 파일이 없으면 생성합니다.
   * @param options - 파일 주입은 {@link TextLogOptions}를 따릅니다.
   *
   * @returns 준비가 끝나 쓰기를 시작할 수 있는 로그 인스턴스
   * @throws 파일 열기·확인 실패 시 거부합니다. 일반 파일이 아니면 TypeError로 거부합니다.
   * 정리까지 실패하면 원래 오류와 정리 오류를 AggregateError에 보존합니다.
   */
  static async open(path: string, { file }: TextLogOptions = {}): Promise<TextLog> {
    const ownsFile = file === undefined;
    const handle = file ?? (await open(path, "a"));

    try {
      const stats = await handle.stat();

      if (!stats.isFile()) {
        throw new TypeError("일반 파일이 필요합니다.");
      }

      return new TextLog(handle, ownsFile);
    } catch (error) {
      if (ownsFile) {
        try {
          await handle.close();
        } catch (closeError) {
          throw new AggregateError([error, closeError], "파일 준비와 정리에 실패했습니다.");
        }
      }

      throw error;
    }
  }

  /**
   * 파일 끝에 문구를 기록합니다. 줄바꿈은 호출자가 제공합니다.
   *
   * @returns 파일 쓰기가 완료되면 이행됩니다.
   * @throws 종료 후 호출 또는 파일 쓰기 실패 시 거부합니다.
   */
  async append(text: string): Promise<void> {
    if (this.#closed) {
      throw new Error("이미 종료한 로그입니다.");
    }

    await this.#file.writeFile(text, "utf8");
  }

  /** 추가 쓰기를 막고 직접 연 파일을 닫습니다. 주입받은 파일의 종료 책임은 호출자에게 남습니다. */
  async [Symbol.asyncDispose](): Promise<void> {
    if (this.#closed) {
      return;
    }

    this.#closed = true;

    if (this.#ownsFile) {
      await this.#file.close();
    }
  }
}
````

## 확인할 경계

| 경계                        | 구현·호출자가 지킬 결과                                                                         |
| --------------------------- | ----------------------------------------------------------------------------------------------- |
| 생성                        | 생성자는 핸들만 받으며, 비동기 파일 열기·종류 확인은 `TextLog.open`에서 완료한 뒤 인스턴스 반환 |
| 직접 파일 획득 후 준비 실패 | 열린 핸들을 닫고 원래 오류를 다시 던짐; 닫기도 실패하면 두 오류를 `AggregateError`에 보존       |
| 파일 주입                   | 준비 실패·정상 종료 모두 주입된 핸들을 닫지 않음; 핸들의 계속 사용·종료는 호출자가 소유         |
| 정상 쓰기·종료              | 호출자가 각 `append`의 완료를 순서대로 기다린 뒤 정리; 직접 연 파일만 닫음                      |
| 정리 시작 후 호출           | `append`를 거부하고, 반복 정리는 아무 작업도 하지 않음                                          |

`#closed`는 파일 닫기를 시도하기 전에 설정됩니다. 닫기 실패는 호출자에게 거부로 전달하지만 반복 정리로 다시 닫기를 시도하지는 않습니다.
