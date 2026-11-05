# 에이전트 작업 지침

저장소 전체에 적용하는 변경·검증 규칙입니다. 명령은 저장소 루트에서 실행합니다.
최초 설치는 [README.md](README.md#시작)를 따릅니다.

## 작업 원칙

- 변경 전에 요청 범위·관련 파일·검증 방법을 확인합니다. 패키지 작업은 해당 README와 기존 코드를 먼저 읽습니다.
- 현재 브랜치에서 요청 범위만 수정하고 사용자 변경을 보존합니다. 여러 단계가 필요한 작업은 검증 가능한 단위로 나눕니다.
- 커밋·푸시·풀 리퀘스트는 사용자가 요청한 경우에만 수행합니다. 비밀정보와 로컬 환경 파일은 커밋하지 않습니다. 풀 리퀘스트 본문은 [템플릿](.github/pull_request_template.md)의 항목을 채웁니다.
- 잠금 파일과 생성물은 직접 편집하지 않습니다. 원본을 수정하고 해당 도구로 재생성합니다.

## 명령

| 명령                             | 용도                             |
| -------------------------------- | -------------------------------- |
| `mise run setup`                 | 의존성·스킬 설치                 |
| `mise run update`                | 개발 도구·의존성·외부 스킬 갱신  |
| `pnpm fix`                       | 린트 자동 수정·포맷              |
| `pnpm check`                     | 전체 검사                        |
| `pnpm knip`                      | 미사용 파일·의존성·내보내기 검사 |
| `pnpm test`                      | 전체 Vitest 테스트               |
| `pnpm test --project <패키지명>` | 패키지별 테스트                  |
| `pnpm exec vitest`               | 테스트 감시 모드                 |

명령 정의는 [package.json](package.json)과 [mise.toml](mise.toml)에서 확인합니다.
CI 실행 조건은 [CI 설정](.github/workflows/ci.yml)에서 확인합니다.

`pnpm check`는 포맷 검사 → Oxlint의 린트·컴파일러 진단(실험적인 `--type-check`) → 철자 검사 → 미사용 항목 검사 → 런타임 테스트 순서로 실행합니다.

tsconfig의 `include`·`exclude`만으로 Oxlint의 파일 탐색 범위가 제한되지는 않습니다.

## 저장소 구조

| 경로                                 | 역할                              |
| ------------------------------------ | --------------------------------- |
| `apps/*`                             | 앱 패키지                         |
| `libs/*`                             | 공유 라이브러리 패키지            |
| `tools/*`                            | 개발 도구 패키지                  |
| `.github/`                           | GitHub Actions·풀 리퀘스트 템플릿 |
| `.husky/`                            | Git 훅 원본 (`_/`는 생성물)       |
| `.vscode/`                           | VS Code 작업 영역 설정·권장 확장  |
| `.apm/skills/`                       | 로컬 스킬 원본                    |
| `.agents/skills/`, `.claude/skills/` | 스킬 배포본                       |
| 루트 설정 파일                       | 도구·워크스페이스·공통 검사 설정  |

패키지 검색 범위는 [pnpm-workspace.yaml](pnpm-workspace.yaml)을 따릅니다.
패키지 경로를 바꾸면 [Vitest 설정](vitest.config.ts)의 `projects`와 위 표도 함께 수정합니다.
실행 환경·프레임워크·빌드·배포는 패키지별로 구성합니다.

## 작성 규칙

### 파일·디렉터리

파일 배치는 패키지의 기존 구조를 따릅니다.

- 컴포넌트를 포함한 파일·디렉터리 이름은 `kebab-case`로 통일합니다. `README.md`·`AGENTS.md`·`SKILL.md` 등 정해진 문서명과 도구가 요구하는 이름은 유지합니다.

### 설정·문서

타입 검사는 [tsconfig.base.json](tsconfig.base.json), 린트는 [.oxlintrc.jsonc](.oxlintrc.jsonc), 포맷은 [.oxfmtrc.jsonc](.oxfmtrc.jsonc)와 [.editorconfig](.editorconfig), 철자는 [cspell.config.jsonc](cspell.config.jsonc)를 따릅니다.

미사용 파일·의존성·내보내기 검사는 [.knip.jsonc](.knip.jsonc)를 따릅니다.

- 설정 기본값은 생략합니다. 필요한 주석은 목적·유지 이유·공식 참고 링크 순서로 짧게 씁니다.
- 주석 블록 위에 빈 줄을 둡니다. 파일 최상단·연속 주석 내부는 제외하며, 객체 시작 직후는 포맷 도구의 결과를 따릅니다.
- 공통 규칙·저장소 관리 절차는 이 문서에, 재사용할 작업 절차·예제는 스킬에 둡니다. 스킬은 저장소 문서·다른 스킬에 의존하지 않도록 작성합니다.
- 소개·최초 설치는 루트 README, 패키지별 사용법은 해당 README에 둡니다. 주석을 지원하지 않는 설정의 근거도 해당 README에 남깁니다.
- 도구 버전은 `mise.toml`, 의존성은 각 `package.json`, 설치 버전은 잠금 파일에서 관리합니다. 문서에 버전을 중복 기록하지 않습니다.
- 설명은 한 곳에서 관리하고 변경 시 해당 정본을 갱신합니다.

## 작업별 절차

### 도구·의존성 갱신 (mise)

#### 일괄 갱신

전체 갱신 시 아래 명령을 순서대로 실행합니다. 개발 도구·워크스페이스 의존성은 메이저 버전까지, 외부 스킬은 선언된 ref 범위에서 갱신합니다.

```sh
mise run update
mise run setup
```

#### 도구 버전 직접 변경

[mise.toml](mise.toml)에서 해당 도구의 버전을 수정한 뒤 잠금 파일을 갱신하고 변경된 환경을 설치합니다.

```sh
mise lock --platform macos-arm64,linux-x64
mise install --locked
mise run setup
```

지원 플랫폼을 추가하면 위 명령의 `--platform` 값과 `mise.toml`의 `tasks.update`에 있는 플랫폼 목록을 함께 수정합니다.

### 패키지 의존성 변경 (pnpm)

변경 대상 패키지의 `package.json`을 수정합니다.

- 공유 버전은 [pnpm-workspace.yaml](pnpm-workspace.yaml)의 catalog에 선언하고 `catalog:`로 참조합니다.
- 내부 패키지는 `workspace:*`로 연결하고 진입점은 `exports`로 정의합니다.

선언을 변경한 뒤 `pnpm-lock.yaml`을 갱신합니다.

```sh
pnpm install
```

## 검증

### Git 훅

의존성 설치 시 `prepare` 스크립트가 Husky 훅을 등록합니다.
수동으로 재설정하려면 `pnpm run prepare`를 실행합니다. CI에서는 `HUSKY=0`으로 훅 설치를 생략합니다.

| 훅                              | 동작                                            | 설정                                           |
| ------------------------------- | ----------------------------------------------- | ---------------------------------------------- |
| [pre-commit](.husky/pre-commit) | 스테이징된 파일의 린트 자동 수정·포맷·철자 검사 | [lint-staged.config.ts](lint-staged.config.ts) |
| [commit-msg](.husky/commit-msg) | 커밋 메시지 형식 검사                           | [commitlint.config.ts](commitlint.config.ts)   |

lint-staged의 수정 결과는 스테이징에 반영하며 부분 스테이징을 보존합니다. pre-commit은 `pnpm check`를 대체하지 않습니다.

### 테스트

루트 [vitest.config.ts](vitest.config.ts)가 패키지별 테스트를 모아 실행합니다.

초기 테스트 동작과 허용 범위는 [README.md](README.md#시작)를 참고합니다. 테스트를 추가하면 루트의 `passWithNoTests`를 제거합니다.

## 완료 조건

1. `pnpm fix`로 린트·포맷을 정리합니다. 사용자 변경이 섞여 있으면 전체 자동 수정 대신 개별 도구로 이번 작업의 파일만 정리합니다.
2. 필요한 생성물을 재생성해 원본 변경과 함께 반영하고 관련 문서와의 일치 여부를 확인합니다.
3. `pnpm check`를 실행하고 이번 변경으로 발생한 실패를 해결합니다. 이 명령의 앞선 검사가 실패하면 이후 검사는 실행되지 않습니다.
4. 변경 내용·검증 결과·남은 제약을 보고합니다. 기존 실패와 실행하지 못한 검사는 원인을 구분해 밝힙니다.
