# project-template

pnpm·TypeScript 모노레포 템플릿입니다. 공통 개발 도구와 검사 환경을 제공하며, 실행 환경과 빌드는 패키지별로 구성합니다. 루트 [package.json](package.json)은 개발 도구 관리용 비공개 ESM 패키지입니다.

## 시작

[mise](https://mise.jdx.dev/getting-started.html)를 설치한 뒤 저장소 루트에서 실행합니다.

```sh
mise trust
mise install --locked
mise run setup
pnpm check
```

현재 테스트 파일은 없습니다. [Vitest 설정](vitest.config.ts)의 `projects` 패턴에 맞는 디렉터리가 있으면 `passWithNoTests`가 테스트 0개를 허용하지만, 일치하는 프로젝트 디렉터리 자체가 없으면 `No projects were found`로 실패합니다.

## 안내

- [AGENTS.md](AGENTS.md): 작업 명령·규칙, 저장소 구조, 갱신·검증 절차
- [mise.toml](mise.toml): 개발 도구 버전과 설치·갱신 명령 정의
- [CI 설정](.github/workflows/ci.yml): 자동 설치·검사 절차
