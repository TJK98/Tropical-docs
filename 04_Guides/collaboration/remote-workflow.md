# 원격 작업 가이드

> [↩ 워크플로우 규칙으로 돌아가기](../git-workflow.md)

본 문서는 **Team Aicemelt**의 *WarmUpdate 프로젝트*에서 **동일한 기능 브랜치를 여러 환경에서 끊김 없이 이어서 작업하기 위한 실무 가이드를 정리하여 팀 전체가 공유할 수 있도록 작성되었습니다.**

**작성자:** [왕택준](https://github.com/TJK98)

**문서 버전:** v0.1

**최종 수정일:** 2025.09.09

**대상 독자:**

* **개발자(프론트엔드/백엔드)**: 환경 전환 시 브랜치 동기화 및 충돌 방지 규칙 준수
* **팀 리더/리뷰어**: 팀원들의 원격 작업 흐름 점검 및 가이드 제공
* **신규 합류자**: 원격 환경에서도 동일한 규칙으로 작업을 이어가기 위한 온보딩 자료

---

## 원칙 요약

* 항상 **원격(Remote)이 진실의 단일 출처(Single Source of Truth)**
* 환경을 바꾸기 전에 **현재 브랜치 변경사항을 반드시 원격에 푸시**해야 함
* 다른 장소에서는 **항상 원격에서 가져와(start from remote)** 이어가야 함

---

## A. 새로운 작업 시작 (이전 작업 `dev`에 병합 완료된 경우)

1. 로컬 `dev` 최신화

```bash
git switch dev
git pull origin dev
```

2. 새 기능 브랜치 생성

```bash
git switch -c feature/새로운-기능명
```

> 항상 최신 `dev`에서 분기해야 함

---

## B. 진행 중인 기능 브랜치를 다른 장소에서 이어가기 (PR 전/리뷰 중)

### 1) 기존 환경(예: 회사)

* **모든 변경사항을 원격에 푸시**하여 로컬에만 남아있지 않도록 함

```bash
git push origin feature/작업중인-기능명
```

> 커밋 전 변경은 commit 후 push 권장
> 추적되지 않은 파일은 stash로 임시 저장 가능
>
> ```bash
> git stash push -u -m "환경 이동 전 임시저장"
> ```

### 2) 새 환경(예: 집)

* 원격에서 브랜치를 가져와 이어가기

```bash
git fetch origin
git switch feature/작업중인-기능명
git pull origin feature/작업중인-기능명
```

> ⚠️ Stash는 **PC 간 동기화되지 않음**
> → 기존 환경에서 `git stash list` 확인 후, **같은 PC에서만** `git stash pop` 가능

---

## C. PR 생성 직전 `dev` 최신화

1. `dev` 최신화

```bash
git switch dev
git pull origin dev
```

2. 작업 브랜치에 반영 (택 1)

```bash
# 병합 이력 보존 (권장)
git switch feature/기능명
git merge dev

# 깔끔한 이력 유지 (팀 정책에 따라 선택)
git switch feature/기능명
git rebase dev
```

> 충돌 발생 시 “충돌 처리 규칙” 참조
> rebase 이후 원격에 동일 브랜치가 있다면 반드시 `--force-with-lease` 사용
>
> ```bash
> git push --force-with-lease
> ```

---

### C-1. merge vs rebase 선택 기준

**권장 기본값 → `merge`**

* 안전함: 히스토리 재작성 없음 → 강제 푸시 불필요
* 팀 전략과 일치: 최종 병합은 Squash & Merge → 중간 이력의 깔끔함은 중요하지 않음
* 원격·다중 환경에서 충돌/실수 리스크 최소화

**`rebase`를 선택할 수 있는 경우**

* 아직 원격에 push하지 않은 로컬 브랜치
* 본인 혼자만 사용하는 feature 브랜치이며, 사전 공지 후 강제 푸시 감수 가능
* (고급) PR 직전 대화형 정리(`rebase -i`)로 WIP 커밋을 묶을 때

**주의: rebase의 위험성**

* 이미 원격에 있는 브랜치라면 강제 푸시 필요
* 다른 사람이 기반으로 작업 중일 경우 충돌/혼란 발생
* 보호 브랜치(`dev`, `main`)에는 절대 금지
* 반드시 `--force-with-lease` 사용 + 사전 공지 필수

**치트시트**

* 원격에 이미 존재 & 공유된 브랜치 → `merge`
* 원격에 올린 적 없음 & 혼자 쓰는 브랜치 → `rebase` 가능
* PR 직전 커밋 정리 필요 → `rebase -i` 후 강제 푸시(+공지)
* 안전·긴급 상황 → 무조건 `merge`

---

## D. 환경 이동 체크리스트

* [ ] 현재 브랜치를 원격에 push 했는가?
* [ ] 미커밋 변경을 commit 또는 stash 했는가?
* [ ] 새 환경에서 `git fetch → git switch → git pull` 했는가?
* [ ] PR 직전 `dev` 최신화를 수행했는가?

---

## E. 자주 겪는 문제 & 해결

**Q1. 집에서 가져왔는데 최신 코드가 아니다?**
→ `git fetch origin` 후 브랜치 명시 pull 필요

```bash
git pull origin feature/작업중인-기능명
```

**Q2. rebase 후 push가 막힌다?**
→ 이력 재작성 상황, 강제 푸시 필요

```bash
git push --force-with-lease
```

> feature 브랜치 한정, dev/main 금지

**Q3. Stash를 다른 PC에서 못 찾는다?**
→ stash는 로컬 전용, 동기화되지 않음
→ 원칙: 환경 이동 전 commit & push

**Q4. 충돌이 자주 발생한다?**
→ 작은 단위로 자주 push & PR 분리
→ PR 전 dev 최신화(merge/rebase) 습관화

---

## F. 빠른 명령어 모음 (QR)

**회사 → 집 이동 전 (feature 브랜치에서):**

```bash
git add .
git commit -m "chore: WIP 환경 이동 전 저장"
git push origin feature/작업중인-기능명
```

**집에서 이어받기:**

```bash
git fetch origin
git switch feature/작업중인-기능명
git pull origin feature/작업중인-기능명
```

**PR 직전 최신화 (택 1):**

```bash
# merge (기본 권장)
git switch dev && git pull origin dev
git switch feature/기능명 && git merge dev

# rebase (조건 충족 시만)
git switch dev && git pull origin dev
git switch feature/기능명 && git rebase dev
git push --force-with-lease
```
