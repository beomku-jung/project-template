# README

## 루트

설치 후 `pnpm check`로 전체 검사를 실행하는 저장소의 예제입니다.

````markdown
# example-workspace

앱과 공유 라이브러리를 함께 개발하는 TypeScript 워크스페이스입니다.

## 시작

저장소에 지정된 Node.js·pnpm을 준비한 뒤 루트에서 실행합니다.

```sh
pnpm install
pnpm check
```

검사가 종료 코드 0으로 끝나면 개발 환경이 준비된 상태입니다.

## 안내

- [공유 저장소](libs/example/README.md): 메모리 저장소 사용법과 개발 명령
````

명령·링크는 실제 저장소에 맞추고, 검사가 없거나 알려진 제한이 있으면 밝힙니다.

## 패키지

`libs/example/README.md`에 적용하는 예제입니다.

- API: 초기값을 받는 `createStore`와 현재 값을 읽는 `store.value`
- 검사: 루트 `check` 스크립트로 전체 검사, 패키지 이름과 같은 Vitest 프로젝트로 런타임 테스트

````markdown
# @project-template/example

앱에서 사용하는 메모리 저장소를 제공합니다. Node.js와 브라우저에서 사용할 수 있으며 데이터를 외부에 저장하지 않습니다.

## 사용법

TypeScript 소스를 직접 처리하는 실행 환경·번들러에서 사용합니다.

[저장소 설치](../../README.md#시작)를 마친 뒤 소비 패키지의 `package.json`에 의존성을 추가하고 루트에서 `pnpm install`을 실행합니다.

```json
{
  "dependencies": {
    "@project-template/example": "workspace:*"
  }
}
```

```ts
import { createStore } from "@project-template/example";

const store = createStore("ready");

console.log(store.value); // "ready"
```

## 개발

전체 검사는 저장소 루트에서 실행합니다.

```sh
pnpm check
```

해당 패키지의 런타임 테스트만 실행하려면 저장소 루트에서 다음 명령을 사용합니다.

```sh
pnpm test --project @project-template/example
```
````

### 참고

- [Vitest 프로젝트 선택](https://vitest.dev/guide/projects.html#running-tests)

### 패키지 유형에 맞추기

| 유형      | 사용법에 담을 예제                                            |
| --------- | ------------------------------------------------------------- |
| 앱        | 환경 변수·외부 서비스 준비 → 실행 명령 → 접속 주소·확인 방법  |
| 개발 도구 | 명령·필수 인자·입력 → 출력 위치·형식과 파일 변경 등 실행 효과 |

### 설명 배치·구조 정리

| 독자의 질문                | 작성 위치·예제                                             |
| -------------------------- | ---------------------------------------------------------- |
| 어떤 진입점을 쓰나?        | 사용법: `/browser`·`/testing`의 용도·환경·초기화 조건      |
| 실행 전에 무엇이 필요한가? | 준비·설정: 필수 옵션·생략 시 동작·`.env.example` 링크      |
| 왜 이 설정을 쓰나?         | 설계·설정 근거: 저장 방식·`sideEffects`의 이유와 해당 파일 |
| 어떻게 빌드·배포하나?      | 짧은 명령은 개발 절, 상세 절차는 같은 README의 별도 절     |
| 어떤 제약이 있나?          | 제약·문제 해결: 지원 범위·증상·원인·해결                   |

## 참고

### 작성 안내

- [GitHub README 안내](https://docs.github.com/en/repositories/managing-your-repositorys-settings-and-features/customizing-your-repository/about-readmes)
- [npm 패키지 README 안내](https://docs.npmjs.com/about-package-readme-files/)

### 구성 예제

- [Fluent UI의 라이브러리 사용 예제](https://github.com/microsoft/fluentui/blob/master/packages/react-components/react-components/README.md)
- [create-vite의 도구 실행 예제](https://github.com/vitejs/vite/blob/main/packages/create-vite/README.md)
