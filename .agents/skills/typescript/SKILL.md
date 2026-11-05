---
name: typescript
description: TypeScript의 API·타입·모듈 계약·비동기 수명을 설계하고 구현·검토합니다. API 형태·상태 표현·함수와 클래스 선택, TS/TSX 구현·리팩터링·타입 오류 수정·TSDoc 작성에 사용합니다. 패키지 생성과 exports·타입 검사·빌드·실행 환경 구성도 다룹니다.
---

# TypeScript 설계·구현

요구사항·호출 방식·실행 환경을 확인합니다. 기존 API를 바꾸면 공개 계약과 호출부를 대조합니다.

## 작업 선택

| 요청                                 | 시작할 곳                                                         |
| ------------------------------------ | ----------------------------------------------------------------- |
| API 형태·타입·수명 설계              | [구현 선택](#구현-선택)·[타입](#타입)·[비동기·오류](#비동기오류)  |
| 구현·리팩터링·타입 오류 수정         | [기본 예제](#기본-예제)와 변경 대상의 [작업별 기준](#작업별-기준) |
| 패키지 생성·공개 경로·실행 환경 구성 | [패키지 구성](#패키지-구성)                                       |
| API 주석·사용 예제 작성              | [TSDoc](#tsdoc)                                                   |

## 기본 예제

선택 인자의 순서가 불분명하면 옵션 객체로 묶습니다.

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

- 공개 함수는 함수 선언을, 콜백은 화살표 함수를 기본으로 합니다.
- 기본값 계산에서 외부 상태를 변경하지 않습니다.
- 준비·분기·반환·출력 사이에는 빈 줄을 둡니다. 같은 목적의 선언·단언은 묶고, 블록 첫머리는 띄우지 않습니다.

| 이름                | 예                                             |
| ------------------- | ---------------------------------------------- |
| 변수·함수·인스턴스  | `source`, `joinLabels`, `isReady`              |
| 타입·클래스         | `JoinLabelsOptions`, `User`                    |
| 정책 상수·고정 목록 | `ACCESS_LEVELS`; 일반 `const` 변수는 camelCase |

## 작업별 기준

### 구현 선택

함수·클래스·팩토리는 상태·수명·생성 책임으로 선택합니다.
API 형태를 정하거나 기존 구조를 바꿀 때 [구현 선택 예제](references/implementation-design.md)를 읽습니다.
설계안은 호출 예제로 입력·결과·실패를 구체화하고, 상태·자원이 있으면 소유자와 수명을 정합니다.

같은 계약을 제공하는 표준 API·설치된 라이브러리를 직접 사용합니다.
사용할 API의 공식 문서와 설치 버전의 타입·구현을 대조하고, 입력·오류·취소·자원 정리 동작이 달라지면 그 차이를 처리합니다.
공식 `llms.txt`가 있으면 문서 탐색에 활용합니다.

### 모듈

공개 런타임 API는 named export 하나당 구현 파일 하나로 나누고, export 이름을 kebab-case 파일명으로 옮깁니다.

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
실행 환경·소비 목적의 격리나 기존 공개 경로 유지가 필요하면 [추가 배럴](references/entry-points.md#분리가-필요한-배럴)을 둡니다.
다른 패키지는 공개 경로를, 내부 구현은 자신의 배럴을 거치지 않고 API를 선언한 파일을 참조합니다.
프레임워크·도구가 한 진입점에서 여러 export를 요구하면 해당 파일 규약을 따릅니다.

- 타입 의존성은 `import type`·`export type`으로 표시합니다. default export는 도구가 요구할 때 사용합니다.
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

- 객체 계약은 `interface`, 유니온·튜플·조합은 `type`을 기본으로 합니다.
- 매개변수 타입은 명시하되 콜백은 문맥에서 추론합니다. 반환 타입은 계약을 고정하거나 추론 결과가 복잡할 때 명시합니다.
- 외부 입력은 `unknown`에서 검증합니다. 불가피한 단언은 근거가 확인되는 좁은 경계에 둡니다.
- 인덱스 조회의 `undefined`를 처리합니다. 리터럴 추론이 필요하면 `const` 타입 매개변수를 사용합니다.
- 타입 제거로 실행할 때는 `enum`·런타임 `namespace`·생성자 매개변수 속성을 피합니다.

### 비동기·오류

응답 검증·병렬 I/O는 [비동기 예제](references/async-patterns.md)를, 인스턴스의 초기화·종료는 [비동기 수명 예제](references/async-lifecycle.md)를 읽습니다.

| 상황                         | 처리                                                                        |
| ---------------------------- | --------------------------------------------------------------------------- |
| 완료를 기다릴 작업           | Promise 반환 또는 `await`                                                   |
| 독립된 작업 / 앞 결과에 의존 | `Promise.all` / 순차 `await`                                                |
| 대량 입력                    | 동시 실행 수 제한                                                           |
| 복구·오류 맥락 추가          | `catch`; 감싼 원인은 `{ cause }`로 보존                                     |
| 실패를 값으로 반환하는 API   | 판별 유니온으로 설계하고 판별자로 분기; 오류 값의 참·거짓으로 판정하지 않음 |
| 자원 정리                    | 지원 환경에서 `using`·`await using`; 사용 중인 작업 완료 후 정리            |

동기 throw와 Promise 거부의 구분이 유지되면 단순 전달의 `async`·`await`를 생략합니다.
`catch`·`finally`·자원 정리의 범위 안에서 완료를 기다려야 할 때는 `return await`를 유지합니다.

### TSDoc

공개 API·공유 도우미는 선언 바로 위에 TSDoc을 작성합니다.
짧은 요약은 한 줄로 쓰고, 상세 설명·블록 태그·예제가 있으면 여러 줄로 나눕니다.

| 위치                                                                                                  | 담을 정보                                 | 예제                                                                           |
| ----------------------------------------------------------------------------------------------------- | ----------------------------------------- | ------------------------------------------------------------------------------ |
| 요약·[`@remarks`](https://tsdoc.org/pages/tags/remarks/)                                              | 목적, 상세 동작·부수 효과                 | [입력 검증](references/type-patterns.md#외부-입력)                             |
| 옵션 속성·[`@defaultValue`](https://tsdoc.org/pages/tags/defaultvalue/)                               | 허용 범위·생략 의미·기본값                | 위 `JoinLabelsOptions`                                                         |
| `@typeParam`·[`@param`](https://tsdoc.org/pages/tags/param/)                                          | 타입 관계·입력 제약; 함수 인자의 기본값   | [재시도 입력](references/implementation-design.md#호출부에서-구분해야-할-의미) |
| [`@returns`](https://tsdoc.org/pages/tags/returns/)·[`@throws`](https://tsdoc.org/pages/tags/throws/) | 반환값·실패 값·완료 시점 / 예외·거부 조건 | [비동기 예제](references/async-patterns.md)                                    |
| [`@example`](https://tsdoc.org/pages/tags/example/)                                                   | 준비·호출·결과                            | 위 `joinLabels`                                                                |
| [`{@link}`](https://tsdoc.org/pages/tags/link/)·[`@see`](https://tsdoc.org/pages/tags/see/)           | 관련 선언·외부 자료                       | 옵션·결과 타입, 공식 API 문서                                                  |

표 순서로 필요한 그룹만 사용합니다. 그룹 사이와 각 `@example` 앞에 빈 주석 줄을 두고, 블록 태그는 새 줄에서 시작합니다.
클래스의 전체 사용 흐름은 [클래스 주석](references/implementation-design.md#인스턴스가-상태를-소유하는-예제)에, 개별 계약은 생성자·메서드에 둡니다.

사용 중단은 `@deprecated`에 대체 API·이전 방법을, 구현 선택의 이유는 코드 옆 `//`에 적습니다.

#### 참조 문법

TSDoc에서는 선언·외부 URL을 `{@link 대상 | 표시 이름}`으로 연결하고, 별도 참고 항목은 `@see`에 둡니다.

아래는 위 `joinLabels` 주석에서 참조를 표기하는 예제입니다.

```ts
/**
 * 목록 순서대로 문구를 이어 붙입니다. 구분자는 {@link JoinLabelsOptions | 옵션}으로 지정합니다.
 *
 * @see {@link https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/join | Array.join의 구분자 처리}
 */
```

## 결과 확인

설계안은 호출 예제와 제안한 계약을 대조하고, 선택 근거·미확인 조건·검증 방법을 남깁니다.
구현을 변경하면 공개 API·호출부·TSDoc을 대조하고 변경한 경계를 검증합니다.

| 변경        | 확인할 내용                                                 |
| ----------- | ----------------------------------------------------------- |
| 배럴        | 공개 이름·타입·초기화 동작과 내부 선언 노출                 |
| 파일 이동   | 순환 의존성·초기화 순서·상태 수명                           |
| 제네릭      | 결과·추론 타입·잘못된 입력 거부                             |
| 패키지 구성 | 소비자의 공개 경로 import·타입 검사·빌드·실행과 README 일치 |

실행·타입 검사용 코드 예제는 대상 환경에서 타입 검사합니다. 파일명이 표시되어 있으면 해당 이름을 사용합니다.
실행 예제는 동작도 검증합니다. 검증 후 임시 파일을 제거합니다.
코드·검사 도구 설정·패키지 구성을 변경하면 프로젝트의 `check`를 실행합니다. 포함된 단계는 중복 실행하지 않고, 결과·미실행 범위를 보고합니다.

## 참고

### 함수와 흐름

- [Google TypeScript: 이름 있는 함수의 선언](https://google.github.io/styleguide/tsguide.html#prefer-function-declarations-for-named-functions)
- [Google TypeScript: 선택 인자와 기본값 초기화](https://google.github.io/styleguide/tsguide.html#parameter-initializers)
- [Airbnb JavaScript: 반환 뒤 불필요한 else 제거](https://github.com/airbnb/javascript#blocks--no-else-return)

### 모듈과 타입

- [Google TypeScript: named export](https://google.github.io/styleguide/tsguide.html#exports)
- [Google TypeScript: 객체 계약에 interface 사용](https://google.github.io/styleguide/tsguide.html#prefer-interfaces-over-type-literal-aliases)
