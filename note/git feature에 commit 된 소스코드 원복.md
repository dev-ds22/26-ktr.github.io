### `feature/test1`의 커밋된 소스를 `origin/dev` 버전으로 원복하는 방법

먼저 최신 `origin/dev`를 가져옵니다.

```bash
git fetch origin
```

현재 브랜치 확인:

```bash
git branch --show-current
```

결과가 반드시:

```text
feature/test1
```

이어야 합니다.

### 1. 권장: 기존 `feature/test1` 커밋 이력은 유지하고 소스만 `origin/dev` 상태로 되돌리기

```bash
git restore --source=origin/dev --staged --worktree .
```

상태 확인:

```bash
git status
```

그 후 원복 내용을 새로운 커밋으로 남깁니다.

```bash
git add -A
git commit -m "Revert feature/test1 sources to origin/dev"
```

이 방식의 결과:

```text
origin/dev
   |
   A---B

feature/test1
   |
   A---B---C---D---E
           변경   원복 commit
```

즉 기존 `feature/test1`의 커밋 이력은 남고, **현재 파일 내용만 `origin/dev`와 동일하게 원복**됩니다.

### 2. 특정 파일만 `origin/dev` 버전으로 원복

예:

```bash
git restore --source=origin/dev --staged --worktree src/main/java/com/test/TestService.java
```

여러 파일:

```bash
git restore --source=origin/dev --staged --worktree \
src/main/java/com/test/TestService.java \
src/main/java/com/test/TestController.java
```

확인:

```bash
git diff --cached
```

커밋:

```bash
git commit -m "Restore files from origin/dev"
```

### 3. 특정 디렉터리만 원복

```bash
git restore --source=origin/dev --staged --worktree src/main/java/com/test/
```

### 4. `feature/test1` 자체를 완전히 `origin/dev`와 동일하게 만들고 기존 feature 커밋도 버리는 경우

이 경우에는 `restore`가 아니라 `reset`입니다.

```bash
git fetch origin
git switch feature/test1
git reset --hard origin/dev
```

이후:

```bash
git status
```

정상이라면:

```text
On branch feature/test1
nothing to commit, working tree clean
```

단, 이 방식은 `feature/test1`의 로컬 커밋을 브랜치 이력에서 제거합니다.

이미 remote의 `feature/test1`에도 push된 상태라면 remote까지 동일하게 하려면:

```bash
git push --force-with-lease origin feature/test1
```

`--force`보다 `--force-with-lease`가 안전합니다.

### 두 방식 차이

| 목적                                 | 명령                                                      | 기존 feature 커밋 |
| ---------------------------------- | ------------------------------------------------------- | ------------- |
| 소스만 `origin/dev`로 원복 후 다시 commit   | `git restore --source=origin/dev --staged --worktree .` | 유지            |
| 브랜치 자체를 `origin/dev`와 완전히 동일하게 초기화 | `git reset --hard origin/dev`                           | 제거            |
| 특정 파일만 원복                          | `git restore --source=origin/dev ... 파일명`               | 유지            |

### 현재 상황에서 권장

이미 `feature/test1`에 변경 내용을 **commit까지 한 상태이고, 그 변경을 취소한 기록도 남겨야 한다면** 아래 순서가 가장 안전합니다.

```bash
git fetch origin

git switch feature/test1

git restore --source=origin/dev --staged --worktree .

git status

git diff --cached

git commit -m "Restore feature/test1 to origin/dev source"
```

중요한 점은 `git restore .`만 사용하면 현재 HEAD 기준 원복이고, 원하는 **`origin/dev` 기준 원복**이 아니므로 반드시:

```bash
--source=origin/dev
```

를 지정해야 한다는 것입니다.
