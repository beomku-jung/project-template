---
name: typescript
description: TypeScript의 API·타입 추론·모듈 계약·비동기 수명을 설계하고 구현합니다. TS/TSX 구현·리팩터링·타입 오류 수정·TSDoc 문법, 패키지의 exports·타입 검사 범위·빌드·실행 환경 구성에 사용합니다.
---

# TypeScript 설계·구현

함수·타입·파일·export 규칙은 사용자 기본값이며 TypeScript 자체의 필수 규칙은 아닙니다. 아래에 명시한 예외에서만 다른 형태를 선택합니다.
공개 API를 바꾸면 영향받는 입력·반환 타입·오류·초기화·종료 계약과 실제 호출부를 함께 갱신합니다.

## 작업 선택

| 요청                                 | 시작할 곳                                                         |
| ------------------------------------ | ----------------------------------------------------------------- |
| API 형태·타입·수명 설계              | [구현 선택](#구현-선택)·[타입](#타입)·[비동기·오류](#비동기오류)  |
| 구현·리팩터링·타입 오류 수정         | [기본 예제](#기본-예제)와 변경 대상의 [작업별 기준](#작업별-기준) |
| 패키지 생성·공개 경로·실행 환경 구성 | [패키지 구성](#패키지-구성)                                       |
| API 주석·사용 예제 작성              | [TSDoc](#tsdoc)                                                   |

## 기본 예제

위치만으로 의미를 구별하기 어려운 선택 인자가 여러 개면 이름 있는 옵션 객체로 묶습니다. 입력 하나나 의미가 분명한 필수 위치 인자는 그대로 둡니다.

````ts
// join-labels.ts

/** {@link joinLabels}의 구분자와 빈 목록 표시를 정합니다. */
export interface JoinLabelsOptions {
  /**
   * 문구 사이의 구분자입니다.
   *
   * @defaultValue `", "`
   */
  separator?: string;

  /**
   * 빈 목록에 표시할 문구입니다.
   *
   * @defaultValue `"없음"`
   */
  fallback?: string;
}

/**
 * 목록 순서대로 문구를 이어 붙입니다.
 *
 * @returns 합친 문구. 빈 목록이면 설정한 fallback을 반환합니다.
 *
 * @example
 * ```ts
 * import { joinLabels } from "./join-labels.ts";
 *
 * console.log(joinLabels(["읽기", "쓰기"], { separator: " · " })); // "읽기 · 쓰기"
 * console.log(joinLabels([])); // "없음"
 * ```
 *
 * @see {@link JoinLabelsOptions} - 구분자와 빈 목록 표시
 */
export function joinLabels(
  source: readonly string[],
  { separator = ", ", fallback = "없음" }: JoinLabelsOptions = {},
): string {
  if (source.length === 0) {
    return fallback;
  }

  return source.join(separator);
}
````

- 공개 함수는 함수 선언을, 콜백은 화살표 함수를 기본으로 합니다. 호출자가 제공하는 동적 `this`를 써야 하는 콜백은 일반 함수로 작성하고, 도구가 특정 선언 형태를 요구하면 그 규약을 따릅니다.
- 기본값 계산에서 외부 상태를 변경하지 않습니다.
- 준비·분기·반환·출력 사이에는 빈 줄을 둡니다. 같은 목적의 선언·단언은 묶고, 블록 첫머리는 띄우지 않습니다.

| 이름                | 예                                             |
| ------------------- | ---------------------------------------------- |
| 변수·함수·인스턴스  | `source`, `joinLabels`, `isReady`              |
| 타입·클래스         | `JoinLabelsOptions`, `User`                    |
| 정책 상수·고정 목록 | `ACCESS_LEVELS`; 일반 `const` 변수는 camelCase |

## 작업별 기준

### 구현 선택

호출 사이에 보존할 상태가 없으면 함수를 사용합니다. 인스턴스별 가변 상태·자원 수명이 있으면 클래스나 클로저로 묶고, 호출자가 구체 구현을 고르지 않게 하려면 팩토리 함수를 사용합니다.
API 형태를 바꿀 때는 [구현 선택 예제](references/implementation-design.md)의 호출·타입·수명 차이를 적용합니다.

표준 API·설치된 라이브러리가 필요한 입력·반환값·오류·취소·정리 계약을 이미 제공하면 직접 사용합니다.
계약이 다르면 차이를 처리하는 어댑터만 둡니다. 차이는 공식 문서와 설치 버전의 선언·구현에서 확인합니다.

### 모듈

공개 런타임 API는 named export 하나당 구현 파일 하나를 기본으로 하고, export 이름을 kebab-case 파일명으로 옮깁니다. 프레임워크·도구가 한 파일에 여러 export나 정해진 파일명을 요구하면 그 규약을 따릅니다.

```text
src/
├── index.ts                   # 소비자용 API 재수출
└── labels/
    ├── join-labels.ts         # joinLabels와 JoinLabelsOptions
    ├── join-labels.test.ts
    ├── split-labels.ts        # splitLabels
    └── testing/
        └── create-labels.ts   # 해당 영역의 테스트 도우미
```

| 선언의 관계                               | 배치                             |
| ----------------------------------------- | -------------------------------- |
| API 전용 옵션·결과 타입, 스키마 파생 타입 | 해당 API·스키마와 같은 파일      |
| API와 독립적으로 관리하는 타입 계약       | 소유 영역에서 계약별 파일        |
| 한 구현에서만 쓰는 도우미                 | 사용하는 구현 파일에 비공개 선언 |

공개 배럴은 패키지당 하나를 기본으로 하며, [단일 배럴 예제](references/entry-points.md#단일-배럴)처럼 API를 선언한 파일에서 직접 재수출합니다.
공통 진입점에서 Node.js·브라우저·worker·edge 등 특정 실행환경 전용 의존성을 격리하거나, 소비자용 테스트 도우미·이미 지원하는 공개 하위 경로를 제공할 때만 [추가 배럴](references/entry-points.md#분리가-필요한-배럴)을 둡니다.
다른 패키지는 공개 경로를, 내부 구현은 자신의 배럴을 거치지 않고 API를 선언한 파일을 참조합니다.

- named export를 기본으로 합니다. default export는 프레임워크·도구가 요구할 때만 사용합니다.
- 타입만 참조하는 의존성은 `import type`·`export type`으로 표시합니다. 클래스 생성·스키마 검증처럼 값도 쓰면 일반 import를 사용합니다.
- 설정·테스트 준비용 진입점에는 도구가 요구하는 선언·등록만 둡니다.
- 파일을 옮기면 TSDoc·테스트·호출부를 함께 갱신합니다.

### 패키지 구성

소비할 공개 경로·실행 환경과 대상 패키지의 README·루트 검사 구성을 확인합니다.
아래 참고 문서는 pnpm·TypeScript·Vitest를 사용하는 비공개 ESM 모노레포의 구성 예제를 다룹니다.

| 작업                                      | 참고 문서                                                            |
| ----------------------------------------- | -------------------------------------------------------------------- |
| 패키지 생성·의존성·검사 연결              | [패키지 구성](references/package-configuration.md)                   |
| `exports`·환경별 구현·부수 효과·실행 명령 | [진입점 설정](references/entry-points.md)                            |
| 빌드·릴리스 책임과 실행 검증              | [빌드와 릴리스](references/package-configuration.md#빌드와-릴리스)   |
| 타입 환경 분리·파일 탐색·경로 별칭        | [타입 검사 범위](references/package-configuration.md#타입-검사-범위) |

### 타입

| 작업                     | 참고 예제                                                                                                     |
| ------------------------ | ------------------------------------------------------------------------------------------------------------- |
| 외부 입력 검증·상태 표현 | [입력 검증](references/type-patterns.md#외부-입력)·[상태별 데이터](references/type-patterns.md#상태별-데이터) |
| 원본 보존·부분 갱신      | [읽기 전용](references/type-patterns.md#읽기-전용-입력)·[선택 속성](references/type-patterns.md#선택-속성)    |
| 값·타입의 관계 유지      | [고정 값](references/type-patterns.md#고정-값)·[제네릭](references/type-reuse.md#제네릭)                      |
| 기존 계약 재사용         | [계약 파생](references/type-reuse.md#계약-파생)                                                               |
| 타입 오류 억제·실패 사례 | [타입 오류 검증](references/type-reuse.md#타입-오류-검증)                                                     |

- 직접 선언하는 객체 계약은 `interface`를 기본으로 합니다. 유니온·튜플·교차·매핑·조건부 타입과 유틸리티 타입으로 파생한 계약은 `type`으로 표현합니다.
- 함수 선언의 매개변수 타입은 명시하고, 콜백은 문맥에서 추론합니다. 문맥 타입이 없거나 추론이 의도한 계약보다 넓으면 콜백에도 타입을 적습니다.
- 반환 타입은 공개 계약을 고정하거나, 재귀·조건 분기 등으로 추론 결과가 의도한 타입인지 읽기 어려울 때 명시합니다. 지역 계산의 자명한 타입은 추론을 유지합니다.
- 외부 입력은 `unknown`에서 검증합니다. 단언은 런타임 검사나 외부 선언의 오류 등 근거를 제시할 수 있는 경계에만 두고, 검증 대신 `as`·`!`·`any`를 쓰지 않습니다.
- 인덱스 조회의 `undefined`를 분기·기본값·명시적 실패 중 API 계약에 맞는 방식으로 처리합니다. 제네릭 호출에서 인자의 리터럴 타입을 보존해야 하면 `const` 타입 매개변수를 사용합니다.
- 타입 제거로 직접 실행할 때는 `enum`·런타임 `namespace`·생성자 매개변수 속성을 쓰지 않습니다. 변환 빌드를 소유한 패키지는 그 변환기의 지원 문법과 실제 산출물로 실행 가능 여부를 판단합니다.

### 비동기·오류

응답 검증·병렬 I/O는 [비동기 예제](references/async-patterns.md)를, 인스턴스의 초기화·종료는 [비동기 수명 예제](references/async-lifecycle.md)를 읽습니다.

| 상황                                                       | 처리                                                                                                |
| ---------------------------------------------------------- | --------------------------------------------------------------------------------------------------- |
| 완료를 기다릴 작업                                         | Promise 반환 또는 `await`                                                                           |
| 독립된 작업 / 앞 결과에 의존                               | `Promise.all` / 순차 `await`                                                                        |
| 동시 작업 수가 연결·메모리·서비스 한도를 넘을 수 있는 입력 | 해당 한도에 맞춰 동시 실행 수 제한                                                                  |
| 복구·오류 맥락 추가                                        | `catch`; 감싼 원인은 `{ cause }`로 보존                                                             |
| 실패를 값으로 반환하는 API                                 | 판별 유니온으로 설계하고 판별자로 분기; 오류 값의 참·거짓으로 판정하지 않음                         |
| 자원 정리                                                  | 런타임·변환기가 지원하면 `using`·`await using`, 아니면 `try`/`finally`; 사용 중인 작업 완료 후 정리 |

동기 throw와 Promise 거부의 구분이 유지되면 단순 전달의 `async`·`await`를 생략합니다.
`catch`·`finally`·자원 정리의 범위 안에서 완료를 기다려야 할 때는 `return await`를 유지합니다.

### TSDoc

공개 API·공유 도우미는 선언 바로 위에 TSDoc을 작성합니다. 도구가 생성하는 선언은 생성 원본의 주석을 수정합니다.
요약만 있으면 한 줄로 쓰고, 상세 설명·블록 태그·예제가 있으면 여러 줄로 나눕니다.

| 위치                                                                                                  | 담을 정보                                 | 예제                                                                           |
| ----------------------------------------------------------------------------------------------------- | ----------------------------------------- | ------------------------------------------------------------------------------ |
| 요약·[`@remarks`](https://tsdoc.org/pages/tags/remarks/)                                              | 목적, 상세 동작·부수 효과                 | [입력 검증](references/type-patterns.md#외부-입력)                             |
| 옵션 속성·[`@defaultValue`](https://tsdoc.org/pages/tags/defaultvalue/)                               | 허용 범위·생략 의미·기본값                | 위 `JoinLabelsOptions`                                                         |
| `@typeParam`·[`@param`](https://tsdoc.org/pages/tags/param/)                                          | 타입 관계·입력 제약; 함수 인자의 기본값   | [재시도 입력](references/implementation-design.md#호출부에서-구분해야-할-의미) |
| [`@returns`](https://tsdoc.org/pages/tags/returns/)·[`@throws`](https://tsdoc.org/pages/tags/throws/) | 반환값·실패 값·완료 시점 / 예외·거부 조건 | [비동기 예제](references/async-patterns.md)                                    |
| [`@example`](https://tsdoc.org/pages/tags/example/)                                                   | 준비·호출·결과                            | 위 `joinLabels`                                                                |
| [`{@link}`](https://tsdoc.org/pages/tags/link/)·[`@see`](https://tsdoc.org/pages/tags/see/)           | 관련 선언·외부 자료                       | 옵션·결과 타입, 공식 API 문서                                                  |

표 순서로 계약 설명에 필요한 그룹만 사용합니다. 타입·이름만 되풀이하는 `@param`·`@returns`는 생략합니다. 그룹 사이와 각 `@example` 앞에 빈 주석 줄을 두고, 블록 태그는 새 줄에서 시작합니다.
클래스의 전체 사용 흐름은 [클래스 주석](references/implementation-design.md#인스턴스가-상태를-소유하는-예제)에, 개별 계약은 생성자·메서드에 둡니다.

사용 중단은 `@deprecated`에 대체 API·이전 방법을, 구현 선택의 이유는 코드 옆 `//`에 적습니다.

#### 참조 문법

TSDoc에서는 선언·외부 URL을 `{@link 대상 | 표시 이름}`으로 연결하고, 별도 참고 항목은 `@see`에 둡니다.

```ts
/**
 * 목록 순서대로 문구를 이어 붙입니다. 구분자는 {@link JoinLabelsOptions | 옵션}으로 지정합니다.
 *
 * @see {@link https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/join | Array.join의 구분자 처리}
 */
```

## 결과 확인

| 변경                  | 소비자에서 확인할 계약·성공 기준                                                                   |
| --------------------- | -------------------------------------------------------------------------------------------------- |
| 배럴·공개 경로        | 지원하는 경로에서 값·타입을 import할 수 있고, 비공개 도우미·실행 진입점이 새로 노출되지 않음       |
| 파일 이동·클래스 전환 | 기존 호출의 초기화 순서·인스턴스 격리·종료 시점·`this` 바인딩이 유지되거나 명시한 새 계약과 일치   |
| 제네릭                | 허용한 호출의 반환·리터럴 타입이 의도대로 추론되고, 금지한 호출은 타입 진단을 발생시킴             |
| 패키지 구성           | 소비 패키지에 정의된 타입 검사·빌드·실행 명령에서 공개 경로가 해석되고 필요한 자산·초기화가 보존됨 |

코드·검사 설정·패키지 구성을 바꾸면 기존 프로젝트 검사 명령으로 타입 진단·런타임 테스트를 실행합니다. 기존 `check`가 포함하는 단계는 중복 실행하지 않고, 빠진 관련 검사는 패키지에 정의된 명령으로 실행합니다.
타입 검사 입력과 Oxlint 파일 탐색 범위는 별개입니다. [범위 진단 명령](references/package-configuration.md#타입-검사-범위)으로 변경한 소스·일반 `.test.ts`·설정 파일의 포함 여부를 판정합니다. 목록 출력은 실제 타입 진단·런타임 테스트를 대신하지 않습니다.
빌드·실행은 변경이 영향을 주는 공개 경로·자산·초기화·실행 환경을 패키지 소유 명령으로 확인합니다. 타입 검사 성공을 런타임 실행 성공으로, 워크스페이스 실행 성공을 설치한 배포 패키지의 실행 성공으로 대신하지 않습니다.
바뀐 실행 예제는 대상 환경에서 타입 검사하고 실행해 명시한 결과와 대조합니다. 타입 전용 예제는 타입 검사만 실행합니다. 파일명이 표시되어 있으면 그 이름을 사용하고, 예제 확인용으로 만든 임시 파일은 확인 후 제거합니다.
실행한 검사·결과와 미실행 범위·이유를 구분해 보고합니다.

## 참고

### 함수와 흐름

- [Google TypeScript: 이름 있는 함수의 선언](https://google.github.io/styleguide/tsguide.html#prefer-function-declarations-for-named-functions)
- [Google TypeScript: 선택 인자와 기본값 초기화](https://google.github.io/styleguide/tsguide.html#parameter-initializers)
- [Airbnb JavaScript: 반환 뒤 불필요한 else 제거](https://github.com/airbnb/javascript#blocks--no-else-return)

### 모듈과 타입

- [Google TypeScript: named export](https://google.github.io/styleguide/tsguide.html#exports)
- [Google TypeScript: 객체 계약에 interface 사용](https://google.github.io/styleguide/tsguide.html#prefer-interfaces-over-type-literal-aliases)
