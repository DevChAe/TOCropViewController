# CLAUDE.md

## ⛔ 최우선 작업 규칙 (모든 규칙보다 우선, 예외 없음)

**사용자가 직접 "진행" 또는 "수정하라"고 말하기 전에는 어떤 작업도 하지 않는다.**

1. **모든 요청에 계획 먼저**: 요청의 종류와 크기에 상관없이(수정, 빌드, 테스트, 실행, 조회, 커밋, 푸시, PR, 문서 작업 모두) 먼저 계획을 안내한다. 예외는 없다.
2. **계획에 포함할 내용**: 무엇을, 어느 파일·대상에서, 어떻게 할지, 영향 범위, 실행할 명령, 외부 영향(푸시·PR 등)을 적는다.
3. **승인 기준**: 사용자가 직접 "진행" 또는 "수정하라"는 취지로 말한 경우에만 실행한다. 그런 언급이 없으면 절대 진행하지 않는다.
4. **승인으로 인정하지 않는 것**: 요청 자체("커밋해줘" 등), "1" 같은 선택지 답변, 침묵, 이전 승인의 연장은 승인이 아니다. 선택 답변을 받으면 세부 계획을 다시 안내하고 승인을 받는다.
5. **승인 범위**: 승인은 안내한 계획 한 건에만 유효하다. 계획과 다르게 하거나 범위를 넓혀야 하면 중단하고 다시 안내한다.
6. **모호하면 묻기**: 승인인지 불분명하면 실행하지 않고 되묻는다.
7. **커밋·푸시·PR**: 일반 작업과 동일하게 계획(대상 저장소·브랜치·포함 파일)을 안내하고 승인을 받는다.
8. **규칙 위반 시**: 위반 사실을 즉시 알리고, 추가 작업 없이 상태를 보고한다.

이 저장소는 Tim Oliver의 **TOCropViewController**(Obj-C 코어)와 **CropViewController**(Swift 래퍼)를 커스터마이징하는 프로젝트다. 작업 브랜치는 `customize`, 원본 추적은 `origin/main`.

상세 분석 문서 (먼저 읽을 것):
- [docs/TOCropViewController.md](docs/TOCropViewController.md) — Obj-C 코어: 구조, 아키텍처, Done/Cancel 흐름, iOS 26 규칙, 함정
- [docs/CropViewController.md](docs/CropViewController.md) — Swift 래퍼: 델리게이트↔클로저 브리지, 프로퍼티 추가 레시피

## 프로젝트 구성

| 프로젝트 | 경로 | 언어 |
|---|---|---|
| TOCropViewController | `Objective-C/TOCropViewController/` | Objective-C (ARC), iOS 12+ |
| CropViewController | `Swift/CropViewController/CropViewController.swift` | Swift 5, 코어를 얇게 위임 |
| Xcode 프로젝트 | `TOCropViewControllerExample.xcodeproj` (프레임워크·샘플·테스트 공용) | |
| 패키징 | `Package.swift`, `TOCropViewController.podspec`, `CropViewController.podspec` | |

## 명령어

```bash
# Obj-C 테스트 (CI와 동일 스킴)
xcodebuild test -project TOCropViewControllerExample.xcodeproj -scheme TOCropViewControllerTests \
  -destination 'platform=iOS Simulator,name=iPhone 17 Pro'

# Swift 래퍼 테스트
xcodebuild test -project TOCropViewControllerExample.xcodeproj -scheme CropViewController \
  -destination 'platform=iOS Simulator,name=iPhone 17 Pro'

# fastlane (CI 방식)
bundle install && TEST_SCHEME=TOCropViewControllerTests bundle exec fastlane test
```

> 로컬 Xcode 가 deployment target 12.0 을 거부하면 위 명령 끝에 `IPHONEOS_DEPLOYMENT_TARGET=15.0` 을 덧붙인다.

## 수정 규칙 (필수)

1. **`include/*.h` 는 심볼릭 링크**다(SwiftPM용). 기존 헤더 수정은 자동 반영되지만, 새 공개 헤더를 추가하면 `include/` 에 링크를 만든다.
2. **API 추가 순서**: Obj-C `.h` → `.m` → `CropViewController.swift` 위임 프로퍼티 → 테스트(Obj-C·Swift 둘 다, setter/getter 왕복 포함) → 샘플 앱 → `CHANGELOG.md`.
3. **iOS 26 분기**는 `TOCropViewUsesGlassDesign()` 또는 `#if defined(__IPHONE_26_0)` 가드를 사용한다. Xcode 16에서도 빌드돼야 한다.
4. **Swift 래퍼는 얇게**: 로직은 Obj-C 코어에, Swift는 위임·타입 브리징만. 델리게이트 클로저는 `[weak self, weak delegate]` 로 캡처한다.
5. 공개 헤더에는 `nullable/nonnull`, Doxygen 주석, 필요 시 `NS_SWIFT_NAME` 을 단다. `nullable` 변경은 Swift 호출부를 깨뜨릴 수 있다.
6. 문자열 추가 시 `Resources/*.lproj` 전부 반영하고, 번들 접근은 `TO_CROP_VIEW_RESOURCE_BUNDLE_FOR_OBJECT` 를 쓴다.
7. 포맷은 `.clang-format`, 파일 헤더는 기존 MIT 라이선스 주석 형식을 유지한다.
8. 버전 변경 시 두 podspec + `MARKETING_VERSION`(pbxproj) + `CHANGELOG.md` 를 함께 수정한다 (현재 3.2.0).
9. **레이아웃은 safe area 만 사용**: 툴바·타이틀·크롭 영역 배치 시 `view.safeAreaInsets` 의 좌/우/상/하를 모두 고려한다 (폴더블·듀얼스크린 대응).
10. 새 파일 추가 시 Xcode 프로젝트, podspec glob, `include/` 링크, SwiftPM 경로를 같이 확인한다.

## 작업 방식

- 계획 안내 → 사용자의 명시적 승인 → 작업 순서를 지킨다 (위 최우선 규칙).
- 변경 후에는 관련 테스트를 실행하고 결과를 그대로 보고한다.
- 분석 문서(`docs/`)의 내용이 바뀌는 수정을 하면 문서도 함께 갱신한다.
