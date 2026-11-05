# 패키지 진입점 설정

TypeScript 소스를 직접 처리하는 소비자용 구성 예제입니다.
이름·경로는 실제 패키지·모듈로 바꾸고 JSON은 `package.json`에 병합합니다.
빌드해 배포하면 산출물·타입 선언 경로를 지정합니다.

Node.js의 내장 타입 제거는 `node_modules` 아래의 TypeScript 파일에 적용되지 않습니다.
배포 패키지는 지원 로더 또는 JavaScript 산출물로 실행하며, 워크스페이스 실행과 별도로 검증합니다.

참고: [Node.js 의존성의 타입 제거 제한](https://nodejs.org/api/typescript.html#type-stripping-in-dependencies)

## 단일 배럴

공유 패키지 `@example/library`의 API를 선언한 파일들을 `src/index.ts` 하나에서 직접 재수출합니다.
[사용자 검증](type-patterns.md#외부-입력)과 [상태별 문구](type-patterns.md#상태별-데이터)를 배치하는 예제입니다.

```text
src/
├── index.ts
└── users/
    ├── parse-user.ts
    └── get-user-label.ts
```

```json
{
  "exports": "./src/index.ts"
}
```

```ts
// src/index.ts
export * from "./users/parse-user.ts";
export * from "./users/get-user-label.ts";
```

타입만 제공하는 모듈은 `export type *`로 연결합니다.
`export *`로 연결할 파일의 named export는 공개 계약으로 관리하고, 내부에서만 공유할 구현은 별도 파일에 둡니다.
제품 배럴은 내부 도우미·테스트 준비·실행부를 재수출하지 않습니다.
기존 공개 이름·경로와 전용 타입이 유지되는지 확인합니다.
CLI만 제공하면 배럴과 `exports`를 생략합니다.

`export *`는 default export를 포함하지 않습니다. 이름 충돌·기존 이름 보존·일부 API 공개에는 명시적 재수출을 사용합니다.

### 참고

- [재수출과 이름 충돌](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/export#re-exporting_aggregating)
- [타입 전용 재수출](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-5-0.html#support-for-export-type-)

## 분리가 필요한 배럴

공통 API에서 Node.js 전용 구현·외부 소비자용 검증 도구를 격리할 때의 구성입니다.
필요한 하위 경로만 추가하며, Node.js 전용 패키지라는 이유만으로 `/node`를 나누지는 않습니다.

```json
{
  "exports": {
    ".": "./src/index.ts",
    "./node": "./src/node.ts",
    "./testing": "./src/testing/index.ts"
  }
}
```

| import 경로                | 용도                    | 실행 환경            |
| -------------------------- | ----------------------- | -------------------- |
| `@example/library`         | 타입·스키마·계산        | 공통 JavaScript      |
| `@example/library/node`    | 파일·프로세스·서버 실행 | Node.js              |
| `@example/library/testing` | 소비자용 검증 도구      | 도구가 요구하는 환경 |

```ts
// src/node.ts
export * from "./files/combine-files.ts";
```

구현은 [파일 I/O 예제](async-patterns.md#파일-io)를 사용합니다.

제품 코드는 검증 도구에 의존하지 않고 기본 배럴도 검증 배럴을 재수출하지 않습니다.
공개 `testing` 배럴은 각 소유 영역의 외부 소비자용 API를 모읍니다.

| 소비 경계     | 확인할 내용                                    |
| ------------- | ---------------------------------------------- |
| 브라우저 실행 | 전이 의존성까지 Node.js 전용 구현 없이 빌드    |
| 타입 참조     | type-only 참조도 소비자의 타입 환경에서 해석   |
| 설치          | 하위 경로를 나눠도 패키지의 설치 의존성은 유지 |

## 환경별 구현 선택

| 바꿀 대상                        | 설정                        |
| -------------------------------- | --------------------------- |
| 같은 공개 API의 환경별 구현      | conditional exports         |
| 패키지 내부 의존성의 환경별 구현 | `imports`의 `#` 이름과 조건 |

조건은 객체의 키 순서대로 선택되므로 구체적인 조건을 먼저, 범용 대체 구현인 `default`를 마지막에 둡니다.
사용자 조건의 지원·활성화 여부를 실행 환경·번들러·타입 검사기에서 확인합니다.
소비 환경별로 선택된 구현과 공개 API·초기화 동작을 검증합니다.

### 참고

- [하위 경로](https://nodejs.org/api/packages.html#subpath-exports)
- [조건부 진입점](https://nodejs.org/api/packages.html#conditional-exports)
- [내부 imports](https://nodejs.org/api/packages.html#subpath-imports)

## 부수 효과를 보존하는 경로

`sideEffects`는 import 시 필요한 부수 효과를 번들러에 알립니다. 내보낸 함수를 호출할 때의 효과는 제외합니다.

| import 시 필요한 부수 효과               | 선언                |
| ---------------------------------------- | ------------------- |
| 없음                                     | `false`             |
| CSS·폴리필·전역 등록 등 일부 파일에 있음 | 해당 파일·glob 배열 |
| 확인하지 못함                            | 생략                |

`sideEffects` 배열에 없는 파일은 제거 가능한 것으로 취급됩니다. 초기화를 별도 경로로 제공하면 소비자가 그 경로를 import하도록 안내합니다.

```json
{
  "exports": {
    ".": "./src/index.ts",
    "./register": "./src/register.ts"
  },
  "sideEffects": ["**/*.css", "./src/register.ts"]
}
```

`import "@example/library/register"`로 초기화합니다.
루트 import로도 초기화하려면 `src/index.ts`에서 등록 파일을 import하고 루트 파일도 `sideEffects`에 포함합니다.
빌드 배포 시 두 필드의 경로를 산출물 기준으로 맞추고 프로덕션 번들에서 미사용 모듈 제거와 초기화·스타일 보존을 확인합니다.
`sideEffects`는 Node.js의 모듈 로드를 제어하지 않습니다.

참고: [webpack의 부수 효과 경로 처리](https://webpack.js.org/guides/tree-shaking/#mark-the-file-as-side-effect-free)

## 실행 명령

TypeScript 직접 실행을 지원하는 Node.js 환경의 CLI 예제입니다.

```json
{
  "scripts": {
    "start": "node src/main.ts"
  }
}
```

```text
packages/cli/src/
├── main.ts             # 인수·신호·출력·종료 코드 연결
└── run-cli.ts          # 명령 처리
```

배럴은 `main.ts`를 가져오거나 재수출하지 않습니다.
테스트용 자식 프로그램의 `fixtures/<프로그램>/main.ts`도 진입점 검색 규칙에 포함합니다.
미사용 코드 분석 도구에 공개 경로와 실행 진입점을 등록합니다.

부수 효과가 없는 도움말 명령부터 확인합니다.

```sh
pnpm --filter @example/cli run start --help
```

`pnpm --filter`의 스크립트는 패키지 디렉터리에서 실행되므로 파일 인수도 해당 작업 디렉터리를 기준으로 해석합니다.
