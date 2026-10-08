# TEST_GitHubActions

`release`(배포 전용) + `release-x.y.z`(QA 작업) 브랜치 모델과 GitHub Actions 배포 파이프라인을 검증하는 테스트 레포.
빌드는 시뮬레이터까지만 하고 Firebase / TestFlight 업로드는 stub 이다.

## 구성

| 경로 | 역할 |
|---|---|
| `Project.swift` | Tuist 4.12.1. 앱 `TestApp` + 위젯 `TestWidget` |
| `TestApp/Resources/Info.plist`, `TestWidget/Info.plist` | 버전 값 리터럴. `bump` lane 이 두 파일을 함께 수정 |
| `fastlane/Fastfile` | `bump`, `firebase`(stub), `test_flight`(stub) |
| `.github/workflows/release-start.yml` | `workflow_dispatch(version)` → develop 에서 `release-x.y.z` 생성 + `x.y.z (0)` 커밋 push |
| `.github/workflows/deploy.yml` | `release` push → bump +1 커밋 push → 빌드 → stub 업로드. dispatch 로 `firebase` / `test_flight` 선택 |
| `.github/workflows/release-note.yml` | `master` push 시 최근 20개 커밋에서 `release:` 커밋을 찾아 GitHub Release 생성 |

## 브랜치

- `develop`: 기능 개발
- `release-x.y.z`: QA 작업. 수정 PR 의 base
- `release`: 배포 전용. `release-x.y.z` 에서 오는 merge commit 과 CI 의 bump 커밋만
- `master`: 기본 브랜치, 상용. `release` 에서 오는 merge commit 만

## 검증 시나리오

1. Actions → Release Start → version `1.0.1` 실행
   - `release-1.0.1` 브랜치 생성, github-actions[bot] 커밋 `Bump version to iOS 1.0.1 (0)`
   - 다른 워크플로가 트리거되지 않음
2. PR `release-1.0.1 → release` 를 **Create a merge commit** 으로 머지
   - Deploy 실행, `Bump version to iOS 1.0.1 (1)` 커밋이 `release` 에 push 됨
   - 그 push 로 Deploy 가 다시 실행되지 않음 (루프 없음)
   - `xcodebuild -version` 이 `Xcode 26.6`
3. `release-1.0.1` 에 커밋 2개 추가 → PR 머지 → `(2)`
   - 머지 직후 바로 한 번 더 머지하면 이전 run 이 취소되는지 (`cancel-in-progress`)
4. Actions → Deploy → `test_flight` 를 `release` 에서 실행 → `(3)`
   - 같은 것을 `develop` 에서 실행하면 첫 step 에서 실패
5. `release-1.0.1` 에 빈 커밋 `release: 1.0.1` (본문 = 릴리즈 노트) → PR 머지 → `(4)`
   - PR `release → master` merge commit → Release Note 워크플로가 태그 `1.0.1` 과 GitHub Release 생성
6. PR `release → develop` (back-merge) 머지 → Release Start `1.0.2` → PR `release-1.0.2 → release`
   - 충돌 없이 `1.0.2 (1)` 로 넘어감
7. 실패 재현: `release-1.0.2` 에서 로컬로 `bundle exec fastlane ios bump` 후 push → PR 에 Info.plist 충돌 표시

## 로컬

```bash
mise install
mise x -- tuist generate
bundle install
bundle exec fastlane ios bump version:1.0.0 build:0
```
