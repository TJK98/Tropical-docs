# 고급 Git 가이드

> [↩ 워크플로우 규칙으로 돌아가기](./git-workflow.md)

본 문서는 **Team Aicemelt**의 *WarmUpdate 프로젝트*에서 **숙련자 또는 특정 상황에서만 필요한 심화 Git 작업 방법을 정리하여 팀 전체가 공유할 수 있도록 작성되었습니다.**

**작성자:** [왕택준](https://github.com/TJK98)

**문서 버전:** v0.1

**최종 수정일:** 2025.09.09

**대상 독자:**

* **숙련 개발자**: 기본 Git 사용법을 넘어 심화 기능이 필요한 경우
* **팀 리더/관리자**: 충돌 해결, 브랜치 정리, 배포 등 고급 Git 작업을 수행해야 하는 경우
* **신규 합류자(고급 사용자)**: 팀 내 Git 활용을 빠르게 심화 수준으로 익히고 싶은 경우

---

## 브랜치 전략(팀 합의 반영)

* **기본(디폴트) 브랜치: `dev`**
* **보호 브랜치: `dev`, `main` (둘 다 보호)**
* **작업 흐름**: `dev`에서 **feature 브랜치**를 분기 → PR로 `dev`에 병합 → 검증 후 `main`으로 릴리스 병합
* **머지 전략 표**

| From → To           | 전략                           | 비고                            |
| ------------------- | ---------------------------- | ----------------------------- |
| `feature/*` → `dev` | **Squash & Merge**           | 리뷰/CI 필수, 커밋 메시지는 PR 제목/본문 사용 |
| `dev` → `main`      | **Merge Commit (`--no-ff`)** | 릴리스 성격 유지, 체인지로그/릴리스 노트 동반    |
| hotfix/* → `dev`   | **Squash & Merge**           | 빠른 수습 후 `dev` 안정화             |
| hotfix/* → `main`  | **Merge Commit (`--no-ff`)** | 배포 추적성 확보                     |

> **보호 규칙(요약)**: 직접 푸시 금지, `--force` 금지, `--force-with-lease`는 보호 브랜치에 금지, 필수 리뷰 ≥ 1, CI 통과 필수.

---

## 1) Rebase 규칙 (feature 한정)

**언제**

* PR 전 **히스토리 정리**(WIP → 의미 단위로 squash)
* 긴 수명 feature를 **`dev` 기준으로 최신화**할 때
* 충돌을 **사전** 해결해 리뷰를 매끄럽게 하고 싶을 때

**대체 시나리오**

* 단순 최신화: `git merge origin/dev`
* 특정 커밋만 필요: `git cherry-pick <sha>`
* 이미 공유된 브랜치 정리: rebase 지양, 다음 PR에서 squash

**팀 공지** *(강제푸시 필요할 때만)*

```
[사전 공지] feature/auth-jwt rebase -i 및 강제푸시 예정
목적: WIP 8→3 정리, dev 최신화/충돌 해소
시간: 14:00–14:10 KST
영향: 로컬 재동기화 필요 (fetch; reset --hard)
담당/검토: 팀장(이름), 팀원(이름)
```

```
[완료 보고] feature/auth-jwt rebase -i 완료, CI 통과
동기화:
  git fetch --all
  git checkout feature/auth-jwt
  git reset --hard origin/feature/auth-jwt
```

**수행 절차**

1. 백업: `git branch backup/$(date +%F)-feature-auth`
2. 최신화: `git fetch origin`
3. 대화형: `git rebase -i origin/dev` → `s/fixup`로 정리
4. 충돌 해결: 수정 → `git add -A` → `git rebase --continue`
5. 로컬 테스트/빌드/단위테스트
6. 필요 시 푸시: `git push --force-with-lease` *(공유 rebase 시 사전 공지 필수)*
7. PR 생성(이유/정리 내용 기입)

**후속 조치**

* CI 녹색 확인, 리뷰어에 영향 없음 알림
* 동료 로컬 동기화 안내(사후 템플릿)

**금지**

* `dev`/`main` 등 **공유 브랜치 rebase**
* 무공지 강제푸시

---

## 2) Revert 규칙 (공유 브랜치 포함)

**언제**

* `dev`/`main`에 잘못된 변경이 **이미** merge되어 **안전 롤백**이 필요할 때
* 이력을 보존하며 되돌려야 할 때

**대체 시나리오**

* 즉시 수정 가능: **fix-forward**(추가 커밋)
* 여러 커밋 얽힘: `revert -n` 누적 후 1회 커밋

**팀 공지**

* 사전(선택): 문제 요약 + 대상 커밋/PR 링크 + 배포 영향
* 사후(필수): revert 커밋/PR 링크, 배포/모니터링 상태

**수행 절차**

1. 대상 커밋/PR 확인, 영향 범위 정리
2. 단일 커밋: `git revert <sha>`
3. 머지 커밋: `git revert -m 1 <merge_sha>` *(부모1=기준 브랜치)*
4. 여러 커밋:

   ```
   git revert -n <sha1> <sha2> ...
   git commit -m "Revert: <범위/이유>"
   ```

5. 테스트/CI 통과 → PR 생성 → 승인 후 병합
6. 배포 파이프라인 모니터링

**후속 조치**

* 회귀 원인(RCA) 메모
* 재적용 조건/계획 문서화

**주의**

* `-m` 부모 지정 오류는 반대 방향 롤백을 유발

---

## 3) Push 강제 옵션

**원칙**

* `--force` **금지**
* 불가피 시 **`--force-with-lease`만** 허용 *(보호 브랜치에서는 금지)*

**언제**

* feature 브랜치 rebase 후 **이미 리모트에 존재하는 동일 브랜치**를 갱신해야 할 때

**대체 시나리오**

* 강제푸시 대신 **새 브랜치**로 PR 재개
* 다음 PR에서 squash로 정리

**수행/후속**

```
git push --force-with-lease
# 사후 공지 + 동기화 가이드 공유 필수
```

---

## 4) Reset --hard

**언제**

* 로컬 변경/인덱스를 **모두 폐기**해야 할 때
* 원격 상태와 **완전 일치**시킬 때

**대체 시나리오**

* 특정 파일만: `git restore --source=<sha> -- <path>`
* 워킹 변경 유지: `git reset --soft/--mixed`
* 보류: `git stash push -m "…"`

**팀 공지**

* 로컬 전용은 공지 불필요
* 타인에게 지시 시 **백업·명령·영향**을 함께 제공

**수행 절차**

1. 백업 브랜치: `git branch backup/pre-reset-$(date +%F)`
2. 리모트 기준 일치:

   ```
   git fetch origin
   git checkout <branch>
   git reset --hard origin/<branch>
   ```

3. 필요 시 생성물 정리: `git clean -ndx` → 확인 후 `git clean -fdx`

**후속 조치**

* 재빌드/재설치 확인, IDE 캐시 재기동

**금지**

* 원격 히스토리에 영향을 주는 reset 지시

---

## 5) Commit --amend

**언제**

* **푸시 전** 작은 수정/메시지 정정, 파일 누락 추가

**대체 시나리오**

* 푸시 후: **새 커밋**으로 fix-forward
* 여러 커밋 정리: PR의 squash merge

**수행/후속**

```
git add <files>
git commit --amend
```

* 푸시 후 amend 했다면: `--force-with-lease` + rebase 공지 규칙 적용

**금지**

* 푸시 후 무공지 amend + 강제푸시

---

## 6) Cherry-pick

**언제**

* `release`/`hotfix` 흐름에서 **특정 수정 커밋만** 이식할 때
* 다중 브랜치 간 부분 적용

**대체 시나리오**

* 변경 폭이 크면: **작은 PR로 분리** 후 정상 머지
* 의존 얽힘: revert 후 재작업 또는 원 브랜치 병합

**팀 공지**

* 다량/연속 cherry-pick 시 사전 알림 (영향 범위/테스트 계획)
* 사후에 원 SHA → 대상 SHA 매핑 공유

**수행 절차**

```
# 단일/다중 (순서 주의)
git cherry-pick <sha1> <sha2> ...
# 충돌 시
# 파일 수정 → add → 계속
git cherry-pick --continue
# 취소
git cherry-pick --abort
```

**후속 조치**

* 테스트/CI 통과 후 PR
* 체인지로그에 원 커밋 링크 기록

**주의**

* 커밋 간 **순서/의존** 깨지기 쉬움 → 가능한 **작게** 적용

---

## 7) Merge 전략 & Squash 정책

**브랜치별 정책(본 팀 합의)**

* **`feature/*` → `dev`**: **Squash & Merge**
* **`dev` → `main`**: **Merge Commit (`--no-ff`)**
* **`hotfix/*` → `dev`**: **Squash & Merge** *(필요 시)*
* **`hotfix/*` → `main`**: **Merge Commit (`--no-ff`)**

**대체 시나리오**

* OSS/감사 요구 시: Merge Commit 보존
* 실험 브랜치: 공유 전 한정으로 Rebase & FF 허용

**수행 절차**

* **Squash & Merge**: PR 제목/본문 → 커밋 메시지로 사용

  * 제목 예: `[feat]: 사용자 알림 추가 (#123)`
  * 본문: 핵심 변경 2–3줄 + 브레이킹 체인지/마이그레이션 안내
* **Merge Commit**: `--no-ff`로 출처 유지, 릴리스 노트 동반

**후속 조치**

* 병합 후 파이프라인/모니터링 확인
* 히스토리 가독성·롤백 용이성 정기 점검

**금지**

* 보호 브랜치 직접 커밋, 리뷰/CI 우회
* 빌드 산출물 대량 포함 PR(아티팩트는 릴리스 자산 사용)

---

## 부록 A — 공지 템플릿 요약

**사전 공지(히스토리 영향 작업 공통)**

```
[사전 공지] 히스토리 영향 작업 예정
브랜치: <feature/...>
작업: rebase -i / amend+강제푸시 / cherry-pick 다량
이유: <간단명료>
시간: HH:MM–HH:MM KST
영향: 로컬 재동기화 필요 (fetch; reset --hard)
담당/검토: <이름>/<이름>
```

**사후 보고**

```
[완료 보고] 작업 완료 및 CI 통과
링크: <PR/커밋>
변경 요약: <한 줄>
동기화:
  git fetch --all
  git checkout <branch>
  git reset --hard origin/<branch>
주의: <있으면 한 줄>
```

---

## 부록 B — 로컬 동기화 가이드(공지에 첨부)

```bash
git fetch --all
git checkout <branch>
git reset --hard origin/<branch>
# 필요 시
git clean -ndx   # Dry-run(삭제 목록 확인)
git clean -fdx   # 확정 삭제
```
