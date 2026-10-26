# GitHub에서 하네스 검증하기

회사 PC에 Python, PowerShell 또는 Codex를 설치하지 않고 브라우저로 자동 검증을 실행할 수 있습니다.

## 1. API 키 없는 자동 검증

1. 저장소에 쓰기 권한이 있는 GitHub 계정으로 로그인합니다.
2. 저장소의 **Actions → Validate harness**를 엽니다.
3. 포크의 워크플로가 비활성화되어 있으면 GitHub가 표시하는 활성화 안내를 따릅니다.
4. **Run workflow → Branch: main → Run workflow**를 선택합니다.
5. 실행 결과에서 **validate (3.11)**과 **validate (3.13)**이 모두 성공했는지 확인합니다.

직접 링크: https://github.com/faerie25/codex-harness/actions/workflows/validate.yml

이 워크플로는 Ubuntu에서 다음 검사를 실행합니다.

- Python 단위·통합 테스트: `python3 -m unittest discover -s tests -v`
- 저장소 구조: `python3 scripts/validate.py --project .`
- 예제 구조: `python3 scripts/validate.py --project examples/service-migration`
- 공백 오류: `git diff --check` (기존 작업 트리 검사이며 커밋 전체의 공백 품질을 증명하지는 않습니다.)

OpenAI 인증정보는 필요하지 않으며 AI 모델을 호출하지 않습니다. GitHub Actions 자체의 이용 한도·과금은 GitHub 계정 정책에 따릅니다. 성공은 코드 테스트와 정적 구조 검증의 성공이며, 실제 AI 에이전트 실행의 성공을 뜻하지 않습니다.

## 2. 실제 AI 에이전트 실행 검증 — 별도 절차

`scripts/live_smoke.py`는 기본 Actions에 포함되지 않습니다. 인증된 클라우드 Codex CLI 환경에서 별도로 실행합니다.

필요 조건:

- Python 3.11 이상과 Git 체크아웃(심볼릭 링크 보존).
- 설치·인증된 Codex CLI 및 모델 사용 권한·이용 한도.
- 프로젝트의 `.codex/config.toml` 및 `.codex/agents/*.toml`을 인식하는 새 프로젝트 세션.
- 실제 사용자 정의 worker 2개와 QA 역할을 생성할 수 있는 네이티브 하위 에이전트 기능.
- 모델 서비스로의 네트워크 연결과 임시 프로젝트의 파일 작성·프로세스 실행 권한.
- 해당 CLI에서 `exec --strict-config --json --sandbox workspace-write` 등 스크립트가 사용하는 옵션 지원 확인.

인증된 신뢰할 수 있는 클라우드 CLI에서 저장소 루트를 작업 경로로 설정한 뒤, 아직 존재하지 않는 대상 경로로 실행합니다.

```bash
codex --version
codex login status
python3 scripts/live_smoke.py run --target _workspace/live-tests/NEW_UNIQUE_ID
```

스크립트의 기본 제한시간은 1,200초입니다. 병렬 워커, 의도된 차단, 완료 거부, 입력 수정 후 부분 재개, 기존 결과 재사용 및 독립 QA를 검사합니다. 성공 여부는 실제 종료코드와 `_workspace/verification.json`, 네이티브 실행 기록을 함께 검토합니다. 스크립트의 자체 검증은 네이티브 역할·동시 실행·통신의 출처를 독립적으로 증명하지 않으므로 실제 도구 기록도 확인해야 합니다. 하위 에이전트 생성 실패를 부모의 순차 실행으로 대신해 통과 처리하지 않습니다.

## 3. 인증정보 보호

- 키, 토큰, `auth.json`은 저장소·PR·채팅·Actions 로그·산출물에 넣지 않습니다.
- 일반 자동 테스트에는 OpenAI Secret을 추가하지 않습니다.
- 별도 GitHub Actions AI 실행을 구성할 경우 OpenAI 공식 문서가 권고하는 `openai/codex-action`과 GitHub Actions Secrets를 사용합니다. 저장소 코드를 실행하는 job 전체 환경변수에 API 키를 노출하지 않습니다.
- 공개 저장소 CI에 ChatGPT 계정의 `auth.json`을 복사하는 방식은 사용하지 않습니다.
- API 키 인증과 ChatGPT 계정 인증은 구분합니다. GitHub 연결 인증만으로 Codex 모델 인증이 제공되지는 않습니다.
- 신뢰하는 클라우드 실행환경의 인증 수단으로 직접 로그인하며, 사용량·과금은 해당 인증 방식에 따릅니다.

공식 참고(2026-10-26 확인):
- https://learn.chatgpt.com/docs/auth
- https://learn.chatgpt.com/docs/non-interactive-mode

## 4. 이번 변경과 검증 범위 (2026-10-26)

- 원격 기준 커밋: `80f8217b5b6d3f220a31f05f9c0adaf09dac40f4`.
- 수동 실행 추가 커밋: `8aab5adcd72fcec77ca8c8571edaa617fe372a8a`.
- 워크플로에 `workflow_dispatch`만 추가하고 기존 push/PR, Ubuntu, Python 3.11/3.13 및 읽기 권한 설정을 유지했습니다.
- 이전 Windows 패치는 별도 작업 폴더에 미커밋 상태로 존재하며, 이번 변경은 설치기·검증기·테스트·CHANGELOG를 수정하지 않습니다. Windows 패치 적용 후에도 검증을 다시 실행해야 합니다.
- 이 작업의 클라우드 셸(Python 3.12.14)에서 테스트 97개, 저장소·예제 구조 검증, `git diff --check`가 통과했습니다.
- GitHub Actions 원격 실행과 실제 Codex 에이전트 실행은 이 기록 작성 시점에 미실행입니다. GitHub 브라우저 인증이 없어 원격 수동 실행을 진행하지 못했습니다.
