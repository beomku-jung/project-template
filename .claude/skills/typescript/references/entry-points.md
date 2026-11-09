# 패키지 진입점 설정

런타임·번들러가 워크스페이스의 `.ts` 소스를 직접 처리하는 소비자용 구성 예제입니다.
이름·경로는 실제 패키지·모듈로 바꾸고 JSON은 `package.json`에 병합합니다.
빌드해 배포하면 `exports`의 실행 경로와 타입 선언 경로를 실제 JavaScript·선언 산출물로 바꿉니다.

Node.js의 내장 타입 제거는 `node_modules` 아래의 TypeScript 파일에 적용되지 않습니다.
배포 패키지는 지원 로더 또는 JavaScript 산출물로 실행합니다. 설치한 소비자의 실행 명령이 공개 경로를 해석해 API를 호출해야 하며, 워크스페이스 소스를 직접 실행한 결과로 대신하지 않습니다.

참고: [Node.js 의존성의 타입 제거 제한](https://nodejs.org/api/typescript.html#type-stripping-in-dependencies)

## 단일 배럴

공개 배럴은 패키지당 하나를 기본으로 합니다. 공유 패키지 `@example/library`의 API를 선언한 파일들을 `src/index.ts` 하나에서 직접 재수출하며, 영역별 중간 배럴은 만들지 않습니다.
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
소비자의 기존 import에서 공개 값·전용 타입을 계속 참조할 수 있어야 합니다. 이름 충돌·기존 이름 보존·비공개 선언 제외 등 일부 API만 재수출해야 하면 export를 명시적으로 나열합니다.
다른 코드가 import할 API 없이 CLI만 제공하면 배럴과 `exports`를 생략합니다.

`export *`는 default export를 포함하지 않습니다.

### 참고

- [재수출과 이름 충돌](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/export#re-exporting_aggregating)
- [타입 전용 재수출](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-5-0.html#support-for-export-type-)

## 분리가 필요한 배럴

공통 API에서 Node.js·브라우저·worker·edge 등 특정 실행환경 전용 의존성을 분리하거나, 소비자용 테스트 도우미를 별도 공개 경로로 제공할 때의 구성입니다. 기존에 지원하는 공개 하위 경로를 유지할 때도 별도 배럴을 둘 수 있습니다.
이 조건에 해당하는 하위 경로만 추가하며, 패키지 전체가 한 실행환경 전용이면 그 이유만으로 하위 경로를 나누지 않습니다. 아래는 Node.js 전용 구현을 `/node`로 분리하는 예제입니다.

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

| 소비 경계     | 성공 기준                                                                                             |
| ------------- | ----------------------------------------------------------------------------------------------------- |
| 브라우저 실행 | 소비자의 브라우저 빌드 명령이 전이 의존성의 Node.js 내장 모듈·전용 로더를 요구하지 않고 산출물을 만듦 |
| 타입 참조     | 소비자의 타입 검사 명령이 공개 선언과 type-only 전이 참조를 소비 환경의 전역 타입만으로 해석          |
| 설치          | 하위 경로를 사용하지 않아도 필수 의존성은 설치되므로, 선택 설치가 필요하면 패키지를 분리              |

## 환경별 구현 선택

| 바꿀 대상                        | 설정                        |
| -------------------------------- | --------------------------- |
| 같은 공개 API의 환경별 구현      | conditional exports         |
| 패키지 내부 의존성의 환경별 구현 | `imports`의 `#` 이름과 조건 |

조건은 객체의 키 순서대로 선택되므로 구체적인 조건을 먼저, 범용 대체 구현인 `default`를 마지막에 둡니다.
사용자 조건을 쓰면 소비자의 실행 명령·번들러 설정·타입 검사 설정에서 같은 조건을 활성화합니다. 도구가 지원하지 않는 조건은 활성화됐다고 가정하지 않습니다.
각 지원 환경의 소비 명령에서 의도한 구현이 선택되고, 같은 공개 이름·입력·반환 타입과 명시한 초기화 효과를 제공해야 합니다. 타입 검사기가 선택한 선언과 런타임이 선택한 구현의 계약도 같아야 합니다.

### 참고

- [하위 경로](https://nodejs.org/api/packages.html#subpath-exports)
- [조건부 진입점](https://nodejs.org/api/packages.html#conditional-exports)
- [내부 imports](https://nodejs.org/api/packages.html#subpath-imports)

## 부수 효과를 보존하는 경로

`sideEffects`는 import 시 필요한 부수 효과를 번들러에 알립니다. 내보낸 함수를 호출할 때만 발생하는 효과는 이 판정에서 제외합니다.

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
빌드 배포 시 두 필드의 경로를 산출물 기준으로 맞춥니다. 소비 패키지의 프로덕션 빌드 명령으로 만든 번들에서 미사용 export는 제거되고, 명시적으로 import한 등록 코드·스타일은 남아야 합니다. 그 번들의 실행에서 등록 결과·스타일 적용이 나타나야 합니다.
`sideEffects`는 Node.js의 모듈 로드나 import 순서를 제어하지 않습니다.

참고: [webpack의 부수 효과 경로 처리](https://webpack.js.org/guides/tree-shaking/#mark-the-file-as-side-effect-free)

## 실행 명령

TypeScript 직접 실행을 지원하는 Node.js 환경의 CLI 예제입니다. Node.js가 변환하지 않는 문법과 `tsconfig`의 경로 별칭을 사용하지 않는다는 전제입니다. 빌드가 필요한 패키지는 아래 명령 대신 그 패키지의 JavaScript 산출물을 실행합니다.

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
테스트용 자식 프로그램을 두면 `fixtures/<프로그램>/main.ts`도 진입점 검색 규칙에 포함합니다.
미사용 코드 분석 도구에 공개 경로와 실행 진입점을 등록합니다. 동적 로드·조건별 경로도 빠뜨리지 않으며, 분석 도구의 진입점 등록이 런타임의 경로 해석을 대신하지는 않습니다.

아래 명령은 `@example/cli`의 `start`가 `--help`를 받아 부수 효과 없이 도움말을 출력하는 경우에 사용합니다. 도움말과 종료 코드 0은 명령 연결의 성공 기준일 뿐, 실제 입력 처리·배포 자산의 동작까지 증명하지는 않습니다.

```sh
pnpm --filter @example/cli run start --help
```

`pnpm --filter`의 스크립트는 패키지 디렉터리에서 실행되므로 파일 인수도 해당 작업 디렉터리를 기준으로 해석합니다.
