---
name: apm
description: 에이전트 스킬·MCP 서버를 탐색·비교·추천하고 APM 패키지와 도구 통합을 관리합니다. 스킬·MCP 검색, 에이전트 기능 확장, apm.yml·로컬 SKILL.md 작성·수정·배포, MCP·LSP·플러그인 연결 요청에 사용합니다.
---

# APM 통합 관리

명령은 `apm.yml`과 `mise.toml`이 있는 디렉터리에서 mise로 실행합니다. 스킬 검색에는 Skills CLI를, 설치·배포에는 APM을 사용합니다.

## 작업 선택

| 요청                       | 읽을 절                                    |
| -------------------------- | ------------------------------------------ |
| 작업에 맞는 스킬·MCP 찾기  | [스킬·MCP 검색](#스킬mcp-검색)             |
| 스킬 패키지 설치·갱신·제거 | [APM 패키지 관리](#apm-패키지-관리)        |
| 로컬 스킬 작성·정제·배포   | [로컬 스킬 작성·배포](#로컬-스킬-작성배포) |
| MCP 서버 연결              | [MCP 연결](#mcp-연결)                      |
| LSP·플러그인 등 연결       | [기타 통합](#기타-통합)                    |

## 스킬·MCP 검색

이미 설치된 스킬과 연결된 도구로 해결할 수 있는지 확인한 뒤, 필요한 기능에 따라 검색합니다.

| 필요한 기능                       | 검색 대상                   |
| --------------------------------- | --------------------------- |
| 작업 절차·작성 규칙·전문 지식     | 스킬 — Skills CLI·skills.sh |
| 외부 서비스·데이터·실행 도구 연결 | MCP — 공식 Registry·APM     |
| 작업 절차와 외부 연결             | 스킬과 MCP 함께 검토        |

후보는 원본을 기준으로 비교합니다.

- 지원 기능·실행 조건·출처·유지보수 상태를 확인합니다.
- 실행 명령·연결 도구·데이터 전송 범위를 검토합니다. 작업과 무관한 지시나 비밀값 노출 요구가 있으면 제외합니다.
- 조회에 실패하면 검색어·출처를 바꾸고 공식 문서와 교차 확인합니다.

### 스킬

[skills.sh](https://skills.sh/)와 Skills CLI로 검색합니다.

```sh
mise exec -- pnpm dlx skills find react performance
```

### MCP

[공식 MCP Registry](https://registry.modelcontextprotocol.io/)와 서비스 제공자의 문서를 확인합니다. Registry 등록 자체가 코드 품질이나 보안 심사를 보장하지는 않습니다.

```sh
mise exec -- apm mcp search github --limit 5
mise exec -- apm mcp show "<server-name>"
```

`<server-name>`은 검색 결과의 전체 식별자로 바꿉니다. 검색은 문자열 일치 방식이므로 서비스명·기능명을 검색어로 사용합니다.

- 필요한 조회·수정 도구와 읽기 전용 사용 가능 여부를 확인합니다.
- 공식 배포·커뮤니티 구현을 구분하고 stdio·HTTP 방식, 클라이언트 지원, 인증·권한·비용을 비교합니다.
- 사설 서버는 조직의 레지스트리·문서에서 확인합니다.

#### 참고

- [공식 Registry의 역할](https://modelcontextprotocol.io/registry/about)
- [APM MCP 명령](https://microsoft.github.io/apm/reference/cli/mcp/)

## APM 패키지 관리

`apm.yml`의 의존성·배포 대상과 설치 상태를 확인합니다. 잠금 파일은 APM 명령으로 갱신합니다.

| 작업               | 명령                                                       |
| ------------------ | ---------------------------------------------------------- |
| 설치된 의존성 확인 | `mise exec -- apm deps list`                               |
| 패키지 설치        | `mise exec -- apm install "<owner/repo>"`                  |
| 특정 스킬 설치     | `mise exec -- apm install "<owner/repo>" --skill "<name>"` |
| 특정 패키지 갱신   | `mise exec -- apm update "<package>"`                      |
| 전체 의존성 갱신   | `mise exec -- apm update`                                  |
| 패키지 제거        | `mise exec -- apm uninstall "<package>"`                   |

`<package>`는 의존성 목록의 식별자로 바꿉니다. 전체 갱신은 요청받은 경우에 실행하며, 갱신 범위는 선언된 ref를 따릅니다.

## 로컬 스킬 작성·배포

### 작성

`includes: auto`를 사용하는 프로젝트에서는 `.apm/skills/<name>/SKILL.md`에 스킬 원본을 작성합니다. 이름에는 소문자·숫자·하이픈을 사용하고 폴더명과 맞춥니다.

대표 요청별 산출물을 정하고 [작성·정제 기준](references/skill-authoring.md)에 따라 선택 조건·예제·검증 방법을 연결합니다.

### 배포

스킬 배포본에만 있는 수정은 원본과 비교해 반영한 뒤 아래 명령으로 다시 배포합니다. `apm.yml`을 수정한 경우에도 같은 명령을 사용합니다.

```sh
mise exec -- apm install
```

스킬 이름을 바꾸거나 삭제한 뒤 남은 배포본은 사용자 수정 여부와 소유 관계를 확인해 정리하고 다시 배포합니다.

## MCP 연결

다음 명령은 `apm.yml`의 `dependencies.mcp`에 서버를 추가하고 대상별 설정을 배포합니다.
Codex의 프로젝트 설정에는 `.codex/` 디렉터리가 필요합니다.

예제의 이름·경로·주소는 서버에 맞춥니다. 인증은 환경 변수 참조나 도구의 인증 절차로 처리하고 비밀값은 기록하지 않습니다.

### 레지스트리

```sh
mise exec -- apm install --mcp "<registry-name>"
```

### 로컬 실행 (stdio)

```sh
mise exec -- apm install --mcp example -- node /absolute/path/server.js
```

### 원격 서버 (HTTP)

```sh
mise exec -- apm install --mcp example --transport http --url https://example.com/mcp
```

참고: [MCP 서버 설치](https://microsoft.github.io/apm/consumer/install-mcp-servers/)

## 기타 통합

요청한 기능의 지원 대상과 설정 방식은 해당 문서에서 확인합니다.

- [LSP 서버 설치·대상별 지원](https://microsoft.github.io/apm/consumer/install-lsp-servers/)
- [Copilot Agent Plugins 설치](https://microsoft.github.io/apm/consumer/copilot-agent-plugins/)
- [프롬프트 작성·실행](https://microsoft.github.io/apm/producer/author-primitives/prompts/)
- [지침·에이전트 작성·통합](https://microsoft.github.io/apm/producer/author-primitives/instructions-and-agents/)

## 결과 확인

| 작업           | 확인·보고할 내용                                                               |
| -------------- | ------------------------------------------------------------------------------ |
| 후보 추천      | 이름·원본 링크·선택 이유·연결 조건·제약; 적합한 후보가 없으면 확인 범위와 대안 |
| 스킬 작성·수정 | 대표 요청의 선택 조건·본문·참고 문서 연결, 링크 검사·예제 검증                 |
| 설치·배포      | APM이 갱신한 선언·잠금 파일과 대상별 생성물, 지원 차이·생략 이유               |
| MCP 연결       | `.codex/config.toml`·`.mcp.json`과 대상 도구의 연결 상태                       |

로컬 스킬은 `.apm/skills/`의 원본과 `.agents/skills/`·`.claude/skills/`의 배포본을 비교합니다. 실패는 기존 문제와 이번 변경의 영향을 구분해 보고합니다.

## 참고

- [APM 설치](https://microsoft.github.io/apm/reference/cli/install/)
- [APM 갱신](https://microsoft.github.io/apm/reference/cli/update/)
