---
type: 지식
created: 2026-08-20
updated: 2026-08-20
tags: [Git, GitHub, Fork, 버전관리, 개발환경]
aliases: [Git 사용법, Fork 사용법, Git 명령어 정리]
source: ChatGPT 대화 정리
---

# Git과 Fork 클라이언트 기본 사용법

## 요약

- Git은 파일의 변경 이력을 **커밋(Commit)** 단위로 저장하고, 브랜치를 이용해 여러 작업을 분리한 뒤 합칠 수 있는 분산 버전 관리 시스템이다.
- GitHub는 Git 저장소를 원격에서 공유하기 위한 서비스이며 Git 자체와는 별개의 개념이다.
- 가장 중요한 흐름은 `Working Directory → Stage(Index) → Commit → Push`다.
- Fork는 Git을 GUI로 사용할 수 있게 해주는 클라이언트다. Fork의 버튼을 이해하려면 각 버튼이 어떤 Git 명령에 대응하는지 같이 이해하는 것이 좋다.
- 처음에는 `Status`, `Stage`, `Commit`, `Fetch`, `Pull`, `Push`, `Branch`, `Stash`의 차이를 정확히 아는 것이 중요하다.
- `Force Push`, `Reset --hard`, 이미 공유한 커밋의 Rebase 같은 작업은 이력을 바꿀 수 있으므로 의미를 이해하기 전에는 사용하지 않는다.

## 내용

## 1. Git이란

Git은 **분산 버전 관리 시스템(Distributed Version Control System)** 이다.

파일을 단순히 백업하는 것이 아니라 변경 이력을 저장하고, 여러 브랜치에서 독립적으로 작업한 뒤 합칠 수 있게 해준다.

```text
Git
└─ 버전 관리 프로그램

GitHub
└─ Git 저장소를 원격으로 보관하고 협업하는 서비스

Fork
└─ Git 명령을 GUI로 사용할 수 있게 해주는 Git 클라이언트
```

Perforce를 사용하다 Git으로 넘어오는 경우 가장 먼저 느끼는 차이는 다음과 같다.

- 일반적인 파일 수정 전에 Checkout이 필요하지 않다.
- Commit이 먼저 **내 로컬 저장소**에 만들어진다.
- GitHub에 반영하려면 Commit 이후 별도로 Push해야 한다.
- Branch 생성과 전환 비용이 매우 낮아 기능 단위 작업을 분리하기 쉽다.

## 2. Git의 핵심 구조

```text
파일 수정
  ↓
Working Directory
  ↓ git add / Fork의 Stage
a
Staging Area (Index)
  ↓ git commit
Local Repository
  ↓ git push
Remote Repository (GitHub 등)
```

### Working Directory

현재 실제로 파일을 수정하고 있는 작업 공간이다.

### Staging Area / Index

**다음 Commit에 어떤 변경을 넣을지 선택하는 공간**이다.

Git에서는 파일을 수정했다고 바로 Commit 대상이 되지 않는다. 먼저 Stage해야 한다.

### Local Repository

`git commit`으로 만들어진 Commit은 우선 내 PC의 Git 저장소에 기록된다.

### Remote Repository

GitHub 같은 원격 저장소다. 로컬 Commit을 다른 사람과 공유하려면 Push한다.

---

## 3. Fork 화면과 Git 명령 대응

| Fork에서 보이는 개념 | Git 명령 | 의미 |
| --- | --- | --- |
| Changes | `git status`, `git diff` | 현재 수정된 파일 확인 |
| Stage | `git add` | 다음 Commit에 포함할 변경 선택 |
| Unstage | `git restore --staged` | Stage한 변경을 다시 제외 |
| Commit | `git commit` | Stage된 내용을 로컬 이력으로 저장 |
| Fetch | `git fetch` | 원격 상태만 가져오기 |
| Pull | `git pull` | 원격 변경을 가져와 현재 브랜치에 반영 |
| Push | `git push` | 로컬 Commit을 원격으로 전송 |
| Branch | `git branch`, `git switch` | 작업 흐름 분리 및 이동 |
| Merge | `git merge` | 다른 브랜치의 이력을 현재 브랜치에 합침 |
| Rebase | `git rebase` | Commit의 기반을 다른 Commit 위로 다시 배치 |
| Stash | `git stash` | Commit하지 않은 작업을 임시 보관 |

Fork의 UI 명칭이나 세부 옵션 위치는 버전에 따라 조금 달라질 수 있다.

---

## 4. 상태 확인 — Status / Changes

### 명령어

```bash
git status
```

현재 브랜치, 수정된 파일, Stage된 파일, 추적되지 않은 파일 등을 확인한다.

Git 사용 중 상태가 헷갈리면 가장 먼저 실행할 명령어다.

```bash
git diff
```

아직 Stage하지 않은 변경 내용을 확인한다.

```bash
git diff --staged
```

이미 Stage한 변경, 즉 다음 Commit에 들어갈 내용을 확인한다.

Fork에서는 보통 `Changes` 화면에서 파일과 Diff를 함께 확인할 수 있다.

---

## 5. Stage — 다음 Commit에 넣을 변경 선택

Stage는 Git을 처음 사용할 때 가장 낯선 개념 중 하나다.

예를 들어 파일 세 개를 수정했다고 하자.

```text
Login.cs
ZoneActor.cs
README.md
```

이 중 `ZoneActor.cs`만 먼저 Commit하고 싶다면 해당 파일만 Stage하면 된다.

### 특정 파일 Stage

```bash
git add ZoneActor.cs
```

### 여러 파일 Stage

```bash
git add ZoneActor.cs ZoneManager.cs
```

### 현재 경로의 변경 Stage

```bash
git add .
```

편리하지만 관계없는 변경까지 같이 들어갈 수 있으므로 Commit 전에 반드시 확인한다.

### 모든 변경 Stage

```bash
git add -A
```

추가, 수정, 삭제된 모든 변경을 Stage한다.

### 이미 추적 중인 파일의 수정/삭제만 Stage

```bash
git add -u
```

새로 생성된 Untracked 파일은 추가하지 않는다.

### 파일 일부만 Stage

```bash
git add -p
```

한 파일에 서로 다른 목적의 수정이 섞여 있을 때 변경 덩어리(hunk) 단위로 Commit을 나눌 수 있다.

Fork의 장점 중 하나가 이 **Partial Staging**을 GUI에서 하기 쉽다는 것이다. 파일 전체가 아니라 변경 블록이나 필요한 부분만 Stage해 하나의 파일에서도 Commit을 분리할 수 있다.

### Unstage

잘못 Stage했다고 수정 내용 자체를 지울 필요는 없다.

```bash
git restore --staged ZoneActor.cs
```

Stage에서만 제외되고 Working Directory의 수정 내용은 유지된다.

### 추천 습관

```text
파일 수정
→ Diff 확인
→ 필요한 파일/변경만 Stage
→ Staged Diff 확인
→ Commit
```

`Stage All → 바로 Commit`을 반복하기보다 **Commit 하나에 하나의 의미 있는 변경을 넣는다**는 기준으로 Stage를 사용하는 것이 좋다.

---

## 6. Commit — 로컬 이력 저장

### 기본 Commit

```bash
git commit -m "Add ZoneActor"
```

Commit은 단순 저장이 아니라 **의미 있는 작업 단위의 스냅샷**이라고 생각하면 된다.

```text
A  Initial project
↓
B  Add ZoneActor
↓
C  Fix ZoneActor lifecycle
```

좋은 Commit은 나중에 다음 작업을 쉽게 만든다.

- 코드 리뷰
- 문제 발생 지점 추적
- 특정 변경 Revert
- Cherry-pick
- 기능 단위 Merge/Rebase

### 마지막 Commit 수정 — Amend

```bash
git commit --amend
```

마지막 Commit의 내용이나 메시지를 수정한다.

아직 Push하지 않은 개인 Commit에는 편리하지만, **이미 다른 사람과 공유한 Commit을 Amend하면 Commit ID가 바뀐다.** 공유된 이력에는 주의해서 사용한다.

---

## 7. Fetch — 원격 상태만 확인

```bash
git fetch
```

Fetch는 원격 저장소의 최신 Commit과 브랜치 정보를 가져오지만 **현재 작업 브랜치에는 바로 합치지 않는다.**

```text
GitHub
  ↓ Fetch
origin/main 갱신

내 main
변경 없음
```

따라서 원격 변경을 먼저 확인하고 싶을 때 안전하게 사용할 수 있다.

### 특정 Remote Fetch

```bash
git fetch origin
```

### 삭제된 원격 브랜치 정보 정리

```bash
git fetch --prune
```

원격에서 삭제된 브랜치의 오래된 Remote-tracking reference를 정리한다.

Fork에서는 Fetch 버튼을 눌러 원격 상태를 갱신한 뒤 Commit Graph에서 로컬과 원격의 차이를 확인하는 방식이 편하다.

---

## 8. Pull — 원격 변경을 현재 브랜치에 반영

Git의 Pull은 단순 다운로드가 아니다.

개념적으로 다음과 같다.

```text
git pull
≈
git fetch
+
Merge 또는 Rebase 등으로 현재 브랜치에 통합
```

### 기본 사용

```bash
git pull
```

Upstream이 설정된 현재 브랜치에서 최신 변경을 가져온다.

```bash
git pull origin main
```

`origin`의 `main`을 가져와 현재 브랜치에 반영한다.

### Merge 방식 Pull

```bash
git pull --no-rebase
```

원격 이력과 로컬 이력이 갈라졌다면 Merge 방식으로 합친다.

```text
A --- B --- C  origin/main
      \
       D --- E local

Pull + Merge

A --- B --- C ------- M
      \             /
       D --------- E
```

### Rebase 방식 Pull

```bash
git pull --rebase
```

로컬 Commit을 원격 최신 Commit 뒤로 다시 배치한다.

```text
기존

A --- B --- C  origin/main
      \
       D --- E local

Rebase 후

A --- B --- C --- D' --- E'
```

Commit Graph가 직선으로 깔끔해지는 장점이 있다.

하지만 Rebase는 Commit을 다시 만들기 때문에 Commit ID가 바뀐다. **이미 Push해서 다른 사람과 공유한 Commit을 함부로 Rebase하지 않는다.**

### Fast-forward만 허용

```bash
git pull --ff-only
```

현재 브랜치를 단순히 앞으로 이동할 수 있을 때만 Pull하고, 로컬과 원격 이력이 갈라졌다면 실패한다.

자동 Merge Commit을 만들고 싶지 않을 때 안전한 선택이 될 수 있다.

---

## 9. Fork Pull 창의 주요 옵션

Fork Pull 창에서 자주 보게 되는 옵션을 Git 개념과 연결해서 이해한다.

### Rebase instead of merge

체크하지 않음:

```text
Pull → Fetch → Merge 방식 통합
```

개념적으로:

```bash
git pull --no-rebase
```

체크함:

```text
Pull → Fetch → Rebase 방식 통합
```

개념적으로:

```bash
git pull --rebase
```

#### 언제 체크할까?

개인 Feature Branch에서 아직 공유하지 않은 Commit을 최신 `main` 위로 정리하고 싶을 때 유용하다.

#### 언제 조심할까?

이미 Push해서 다른 사람이 해당 Commit을 기준으로 작업하고 있다면 Rebase는 이력을 다시 작성하므로 주의한다.

### Stash and reapply local changes

Pull을 하려는데 아직 Commit하지 않은 수정사항이 Working Directory에 있을 때 사용한다.

개념적으로 다음 흐름이다.

```text
현재 수정사항 임시 Stash
→ Pull
→ Stash했던 수정사항 다시 적용
```

Git의 `--autostash` 동작과 유사한 목적이다.

예:

```bash
git pull --rebase --autostash
```

#### 장점

작업 중인 파일을 일부러 임시 Commit하지 않고도 Pull할 수 있다.

#### 주의점

Pull 이후 Stash를 다시 적용하는 과정에서 동일한 부분이 바뀌었다면 Conflict가 발생할 수 있다.

따라서 단순히 '편한 옵션'이라기보다 **내 미완성 변경을 잠깐 치워두고 Pull한 뒤 다시 적용하는 옵션**으로 이해한다.

---

## 10. Push — 로컬 Commit을 원격으로 전송

Commit을 했다고 GitHub에 올라간 것은 아니다.

```text
Commit
= 내 PC의 Git 저장소에 기록

Push
= 내 로컬 Commit을 GitHub 같은 Remote에 전달
```

### 기본 Push

```bash
git push
```

현재 브랜치에 Upstream이 설정되어 있다면 해당 원격 브랜치로 Push한다.

### 특정 Remote / Branch Push

```bash
git push origin main
```

### 새 브랜치 최초 Push + Upstream 설정

```bash
git push -u origin feature/zone-actor
```

`-u`는 `--set-upstream`의 축약형이다.

이후에는 해당 로컬 브랜치가 `origin/feature/zone-actor`를 추적하므로 단순히 다음처럼 사용할 수 있다.

```bash
git push
git pull
```

Fork에서 새 로컬 브랜치를 처음 Push할 때 Tracking/Upstream 관련 선택지가 보이는 이유가 이것이다.

### Tag Push

특정 Tag:

```bash
git push origin v1.0.0
```

모든 Tag:

```bash
git push --tags
```

### Dry Run

```bash
git push --dry-run
```

실제로 전송하지 않고 어떤 Push가 수행될지 확인한다.

---

## 11. Force Push

일반 Push는 원격 이력을 안전하게 앞으로 이동할 수 없으면 거부된다.

Rebase, Amend, Reset 등으로 이미 Push한 Commit 이력을 바꾸면 일반 Push가 실패할 수 있다.

### Force Push

```bash
git push --force
```

또는:

```bash
git push -f
```

원격 브랜치를 강제로 현재 로컬 이력으로 바꾼다.

**다른 사람의 Commit까지 덮어쓸 수 있으므로 공동 브랜치에서는 매우 위험하다.**

### Force with lease

강제 Push가 꼭 필요하다면 일반 `--force`보다 다음이 상대적으로 안전하다.

```bash
git push --force-with-lease
```

내가 마지막으로 확인한 이후 원격 브랜치가 예상과 다르게 변경됐다면 Push를 거부해 다른 사람의 변경을 실수로 덮어쓸 가능성을 낮춘다.

### 권장 기준

| 상황 | 권장 |
| --- | --- |
| 일반 Push | `git push` |
| 새 개인 Branch 첫 Push | `git push -u origin <branch>` |
| 개인 Feature Branch를 Rebase 후 다시 Push | 상황을 확인한 뒤 `--force-with-lease` |
| main / develop 등 공유 브랜치 | Force Push 사용하지 않음 |
| 이유를 모르겠는데 Push가 거부됨 | Force부터 누르지 말고 원인 확인 |

Fork에서 `Force Push` 관련 옵션을 볼 때도 같은 기준으로 판단한다.

---

## 12. Upstream / Tracking Branch

Git에는 로컬 브랜치와 원격 브랜치가 따로 존재한다.

```text
로컬
feature/zone-actor

원격 추적
origin/feature/zone-actor
```

두 브랜치의 연결 관계를 Tracking 또는 Upstream 관계라고 생각하면 된다.

```bash
git branch -vv
```

현재 로컬 브랜치가 어떤 Remote branch를 추적하는지 확인할 수 있다.

Upstream을 직접 설정하려면:

```bash
git branch --set-upstream-to=origin/main main
```

대부분 새 브랜치를 처음 Push할 때 다음 명령으로 자동 설정한다.

```bash
git push -u origin feature/test
```

---

## 13. Branch

### 현재 Branch 확인

```bash
git branch
```

### Branch 생성과 동시에 이동

```bash
git switch -c feature/zone-actor
```

### 기존 Branch로 이동

```bash
git switch main
```

### Branch 삭제

```bash
git branch -d feature/zone-actor
```

병합되지 않은 브랜치를 강제 삭제하는 `-D`도 있지만, 변경을 잃을 수 있으므로 이유 없이 사용하지 않는다.

Fork에서는 Commit Graph에서 Branch 위치를 시각적으로 보기 쉬우므로 CLI보다 브랜치 관계를 이해하기 편하다.

---

## 14. Merge

현재 브랜치에 다른 브랜치를 합친다.

```bash
git switch main
git merge feature/zone-actor
```

여기서 중요한 것은 **어느 브랜치에서 Merge 명령을 실행하느냐**다.

```text
현재 브랜치: main
git merge feature/test

→ feature/test의 변경을 main에 합친다.
```

Merge 과정에서 동일한 부분을 양쪽에서 수정했다면 Conflict가 발생할 수 있다.

---

## 15. Rebase

현재 브랜치의 Commit 기반을 다른 브랜치의 최신 Commit 위로 옮긴다.

```bash
git switch feature/zone-actor
git rebase main
```

Merge와 결과 코드가 같을 수 있어도 Commit History를 만드는 방식이 다르다.

```text
Merge
A---B---C------M
     \        /
      D------E

Rebase
A---B---C---D'---E'
```

Rebase는 Commit Hash를 변경할 수 있으므로 **공유하기 전 개인 작업 정리**에 사용하는 것이 이해하기 쉽다.

---

## 16. Stash

작업 중인 변경을 Commit하지 않고 잠깐 치워두고 싶을 때 사용한다.

### 저장

```bash
git stash push -m "WIP ZoneActor"
```

### Untracked 파일까지 포함

```bash
git stash push -u -m "WIP ZoneActor"
```

### 목록 확인

```bash
git stash list
```

### 다시 적용하고 Stash 목록에서도 제거

```bash
git stash pop
```

### 다시 적용하지만 Stash는 유지

```bash
git stash apply
```

`pop`은 적용 후 성공하면 Stash 항목을 제거하고, `apply`는 항목을 남긴다는 차이가 있다.

Fork의 `Stash and reapply local changes` 같은 옵션을 이해하려면 이 개념을 알고 있는 것이 좋다.

---

## 17. Clone

이미 GitHub에 존재하는 저장소를 처음 받아올 때 사용한다.

```bash
git clone <repository-url>
```

단순 파일 다운로드가 아니라 Commit History, Branch 정보와 Git 저장소 설정을 함께 가져온다.

---

## 18. 실전 작업 흐름

`ZoneActor` 기능을 개발한다고 가정한다.

### 1. main 최신 상태 확인

Fork에서 `main`을 Checkout한 뒤 Fetch/Pull한다.

CLI라면:

```bash
git switch main
git pull
```

### 2. 작업 Branch 생성

```bash
git switch -c feature/zone-actor
```

Fork에서는 `New Branch`로 생성한다.

### 3. 코드 수정

```text
ZoneActor.cs
ZoneManager.cs
```

### 4. 변경 확인

```bash
git status
git diff
```

Fork에서는 `Changes` 화면에서 확인한다.

### 5. 필요한 변경만 Stage

```bash
git add ZoneActor.cs ZoneManager.cs
```

또는 Fork에서 파일/변경 블록을 Stage한다.

### 6. Staged Diff 확인

```bash
git diff --staged
```

### 7. Commit

```bash
git commit -m "Add ZoneActor"
```

### 8. 최초 Push

```bash
git push -u origin feature/zone-actor
```

### 9. 이후 추가 작업

```bash
git add <files>
git commit -m "Fix ZoneActor lifecycle"
git push
```

### 10. GitHub에서 Pull Request

```text
feature/zone-actor
→ Pull Request
→ Review
→ main Merge
```

---

## 19. Perforce와 개념 비교

완전히 1:1 대응하지는 않지만 처음 이해할 때 참고할 수 있다.

| Perforce | Git | 참고 |
| --- | --- | --- |
| Depot | Remote Repository | GitHub 등의 원격 저장소 |
| Workspace | Working Directory | 로컬 작업 공간 |
| Changelist | Commit과 유사 | Git은 Stage를 이용해 Commit 구성 |
| Submit | Commit + Push와 유사 | Git에서는 두 과정이 분리됨 |
| Sync | Pull과 일부 유사 | Git Pull은 Fetch + 통합 과정 포함 |
| Stream | Branch와 일부 유사 | 구현 구조는 다름 |
| Checkout | 일반적으로 필요 없음 | Git은 바로 파일 수정 가능 |

Git의 `checkout` 명령은 Perforce의 Checkout과 의미가 다르다.

브랜치 이동에는 현대 Git에서 다음처럼 `switch`를 사용하는 것이 이해하기 쉽다.

```bash
git switch main
```

---

## 20. 자주 쓰는 명령어 빠른 정리

| 목적 | 명령어 |
| --- | --- |
| 상태 확인 | `git status` |
| 수정 내용 확인 | `git diff` |
| Stage된 내용 확인 | `git diff --staged` |
| 특정 파일 Stage | `git add <file>` |
| 부분 Stage | `git add -p` |
| Unstage | `git restore --staged <file>` |
| Commit | `git commit -m "message"` |
| 원격 상태만 갱신 | `git fetch` |
| 삭제된 원격 브랜치 정보 정리 | `git fetch --prune` |
| Pull | `git pull` |
| Rebase 방식 Pull | `git pull --rebase` |
| Fast-forward만 Pull | `git pull --ff-only` |
| Push | `git push` |
| 새 Branch 최초 Push | `git push -u origin <branch>` |
| 상대적으로 안전한 강제 Push | `git push --force-with-lease` |
| Branch 확인 | `git branch` |
| Branch 생성 | `git switch -c <branch>` |
| Branch 이동 | `git switch <branch>` |
| Merge | `git merge <branch>` |
| Rebase | `git rebase <branch>` |
| Stash | `git stash push` |
| Stash 복원 | `git stash pop` |
| Commit Graph 확인 | `git log --oneline --graph --all` |

---

## 21. Fork에서 처음 사용할 때 권장 기준

### Stage

- 수정했다고 무조건 Stage All부터 하지 않는다.
- Diff를 확인하고 하나의 Commit에 필요한 변경만 Stage한다.
- 한 파일에 서로 다른 변경이 섞여 있으면 Partial Stage를 활용한다.

### Pull

처음에는 다음 기준이 안전하다.

- 작업 파일이 깨끗한 상태에서 Pull하는 습관을 들인다.
- `Rebase instead of merge`는 Rebase 의미를 이해한 뒤 사용한다.
- `Stash and reapply local changes`는 미완성 변경을 임시 보관했다가 다시 적용하는 기능이라는 것을 알고 사용한다.

### Push

- 일반적으로 Force 옵션 없이 Push한다.
- 새 Branch의 첫 Push에서는 원격 Tracking branch 설정을 확인한다.
- Push가 거부됐다고 바로 Force Push하지 않는다.
- Force가 정말 필요하면 공동 작업 여부를 확인하고 가능하면 `force-with-lease` 방식의 의미를 먼저 이해한다.

### Commit

- 한 Commit에는 하나의 의미 있는 변경을 넣는다.
- 메시지만 보고도 무엇을 바꿨는지 알 수 있게 작성한다.

예:

```text
Add ZoneActor
Fix ZoneActor lifecycle
Add Redis connection retry
Remove unused packet handler
```

---

## 22. 위험하거나 주의할 명령

### Force Push

```bash
git push --force
```

원격 이력을 덮어쓸 수 있다.

### Hard Reset

```bash
git reset --hard <commit>
```

Working Directory의 변경까지 사라질 수 있다.

### Clean

```bash
git clean -fd
```

추적되지 않은 파일/폴더를 삭제할 수 있다.

### 공유된 Commit Rebase

이미 다른 사람이 사용 중인 Commit을 Rebase하면 Commit Hash가 바뀌어 협업 이력이 꼬일 수 있다.

처음 Git을 사용할 때는 **오류가 나면 Force/Hard 옵션으로 해결하려 하지 말고 `git status`와 Commit Graph부터 확인하는 습관**을 들이는 것이 좋다.

---

## 참고 자료

- Git Pull 공식 문서: https://git-scm.com/docs/git-pull
- Git Push 공식 문서: https://git-scm.com/docs/git-push
- Git Add 공식 문서: https://git-scm.com/docs/git-add
- Git Fetch 공식 문서: https://git-scm.com/docs/git-fetch
- Git Stash 공식 문서: https://git-scm.com/docs/git-stash
- Fork 공식 사이트: https://git-fork.com/
- Fork Windows Release Notes: https://git-fork.com/releasenoteswin

## 관련 문서

- [[지식_인덱스]]
