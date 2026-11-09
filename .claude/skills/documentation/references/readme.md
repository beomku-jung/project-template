# README

프로젝트 시작·패키지 사용 안내를 작성할 때 읽습니다.
예제는 구성 템플릿입니다. API·명령·지원 환경·링크를 해당 저장소의 근거로 치환합니다. 검사 명령이 없으면 없다고 밝히고, 실행하지 않은 명령을 성공했다고 쓰지 않습니다.

## 루트

루트에는 최초 설치부터 성공 확인까지 두고, 개별 패키지의 API 설명은 해당 README로 연결합니다. 이미 패키지 README에 있는 사용법을 루트에 복제하지 않습니다.
아래 템플릿은 지정된 Node.js·pnpm이 있고 설치 후 `pnpm check`로 전체 검사하는 저장소를 가정합니다.

````markdown
# example-workspace

앱과 공유 라이브러리를 함께 개발하는 TypeScript 워크스페이스입니다.

## 시작

지정된 Node.js·pnpm을 준비하고 저장소 루트에서 실행합니다.

```sh
pnpm install
pnpm check
```

검사가 종료 코드 0으로 끝나면 성공입니다.

## 안내

- [공유 저장소](libs/example/README.md): 메모리 저장소 사용법과 개발 명령
````

## 패키지

패키지에는 소비자가 준비할 의존성·환경·공개 진입점·최소 사용 예제를 먼저 두고, 기여자가 쓸 검사·빌드·배포 명령은 개발 절로 분리합니다.
아래는 `libs/example/README.md`의 구성 예제이며 다음을 가정합니다.

- `createStore`가 초기값을 받고, `store.value`로 현재 값을 읽습니다.
- Node.js와 브라우저에서 TypeScript 소스를 직접 처리할 수 있으며, 저장소는 데이터를 외부에 저장하지 않습니다.
- 루트 `check`로 전체를 검사하고, 패키지와 같은 이름의 Vitest 프로젝트로 런타임 테스트를 실행합니다.

````markdown
# @project-template/example

앱에서 사용하는 메모리 저장소를 제공합니다. Node.js와 브라우저에서 사용할 수 있으며 데이터를 외부에 저장하지 않습니다.

## 사용법

TypeScript 소스를 직접 처리하는 실행 환경·번들러에서 사용합니다.

[저장소 설치](../../README.md#시작) 후 소비 패키지의 `package.json`에 의존성을 추가합니다. 저장소 루트에서 `pnpm install`을 실행합니다.

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

저장소 루트에서 다음 명령으로 이 패키지의 런타임 테스트만 실행합니다.

```sh
pnpm test --project @project-template/example
```
````

### 참고

- [Vitest 프로젝트 선택](https://vitest.dev/guide/projects.html#running-tests)

### 패키지 유형에 맞추기

| 유형      | 사용법에 담을 내용                             |
| --------- | ---------------------------------------------- |
| 앱        | 환경 변수·외부 서비스 준비 → 실행 → 접속·확인  |
| 개발 도구 | 명령·필수 인자·입력 → 출력 위치·형식·파일 변경 |

라이브러리는 설치 → 공개 진입점 → 호출 예제 → 결과 순서로 씁니다. 앱·도구는 위 표의 결과를 실제 화면·출력·파일 위치로 명시합니다. 배포나 데이터 변경 명령은 준비 권한·변경 대상도 명령 앞에 둡니다.

### 설명 배치

| 독자의 질문           | 작성 위치·예제                                             |
| --------------------- | ---------------------------------------------------------- |
| 어떤 진입점을 쓰나?   | 사용법: `/browser`·`/testing`의 용도·환경·초기화 조건      |
| 무엇을 준비하나?      | 준비·설정: 필수 옵션·생략 시 동작·`.env.example` 링크      |
| 왜 이 설정을 쓰나?    | 설계·설정 근거: 저장 방식·`sideEffects`의 이유와 해당 파일 |
| 어떻게 빌드·배포하나? | 개발 절에 명령, 별도 절에 상세 절차                        |
| 어떤 제약이 있나?     | 제약·문제 해결: 지원 범위·증상·원인·해결                   |

링크는 문서가 놓일 위치에서 해석합니다. 위 템플릿의 `../../README.md#시작`은 `libs/example/README.md` 기준이므로 다른 깊이에 복사하면 상대 경로도 바꿉니다.

## 참고

### 작성 안내

- [GitHub README 안내](https://docs.github.com/en/repositories/managing-your-repositorys-settings-and-features/customizing-your-repository/about-readmes)
- [npm 패키지 README 안내](https://docs.npmjs.com/about-package-readme-files/)

### 구성 예제

- [Fluent UI의 라이브러리 사용 예제](https://github.com/microsoft/fluentui/blob/master/packages/react-components/react-components/README.md)
- [create-vite의 도구 실행 예제](https://github.com/vitejs/vite/blob/main/packages/create-vite/README.md)
