# TOCropViewController (Objective-C 코어 라이브러리)

> 이미지(`UIImage`)를 자르고(crop) 90도 단위로 회전시키는 `UIViewController` 서브클래스. iOS 사진 앱 편집기와 유사한 UX.
> 원작자: Tim Oliver · 라이선스: MIT · 현재 버전: **3.2.0** · 브랜치: `customize` (원본 `main` 대비 변경 없음, 커스터마이징 시작점)

## 1. 개요

| 항목 | 내용 |
|---|---|
| 언어 | Objective-C (ARC) |
| 최소 지원 | iOS 12.0 (Mac Catalyst / visionOS는 비공식) |
| 배포 | CocoaPods(`TOCropViewController`), SwiftPM, Carthage, 수동 복사 |
| 소스 위치 | `Objective-C/TOCropViewController/` |
| 리소스 | `Resources/` – 28개 언어 `*.lproj/TOCropViewControllerLocalizable.strings`, `PrivacyInfo.xcprivacy` |
| 코드 규모 | 약 4,700줄 (`TOCropView.m` 1,795 / `TOCropViewController.m` 1,353 / `TOCropToolbar.m` 781) |
| Swift 래퍼 | 별도 프로젝트 → [CropViewController.md](CropViewController.md) |

## 2. 디렉터리 구조

```
Objective-C/TOCropViewController/
├── TOCropViewController.{h,m}        # 메인 VC (공개 API, 델리게이트, 프레젠테이션)
├── Views/
│   ├── TOCropView.{h,m}              # 크롭 엔진: 스크롤뷰/크롭박스/제스처/회전/종횡비
│   ├── TOCropToolbar.{h,m}           # 하단/상단 툴바 (iOS 26 Liquid Glass 대응)
│   ├── TOCropOverlayView.{h,m}       # 그리드 라인 오버레이
│   └── TOCropScrollView.{h,m}        # 터치 훅이 있는 UIScrollView
├── Models/
│   ├── TOCropViewControllerAspectRatioPreset  # 종횡비 프리셋(원본/정사각/3:2/5:3/4:3/5:4/7:5/16:9)
│   ├── TOCropViewControllerTransitioning      # 커스텀 present/dismiss 애니메이션
│   ├── TOCroppedImageAttributes               # 크롭 프레임·각도·원본 크기 값 객체
│   └── TOActivityCroppedImageProvider         # 공유 시트용 UIActivityItemProvider
├── Categories/UIImage+CropRotate.{h,m}        # croppedImageWithFrame:angle:circularClip:
├── Constants/TOCropViewConstants.h            # 열거형, 리소스 번들 헬퍼, iOS 26 판별
├── include/                                   # SwiftPM 공개 헤더(원본 헤더로의 심볼릭 링크) + module.modulemap
├── Resources/ , Supporting/Info.plist
```

> ℹ️ `include/*.h` 는 원본 헤더를 가리키는 **심볼릭 링크**다. 원본을 수정하면 자동 반영된다. 단, **새 공개 헤더를 추가**하면 `include/` 에 링크를 만들고 `module.modulemap`/Xcode/podspec 도 확인해야 한다.

## 3. 아키텍처

### 3.1 레이어
```
TOCropViewController (UIViewController)
 ├─ TOCropView (UIView)  ── 크롭 로직 전부
 │    ├─ TOCropScrollView + backgroundImageView   (이동/줌 되는 원본)
 │    ├─ overlayView (딤) + translucencyView      (반투명 효과)
 │    ├─ foregroundContainerView + foregroundImageView (크롭박스 영역만 클립되어 "구멍"처럼 보이게 함)
 │    └─ gridOverlayView (TOCropOverlayView)       (원형 크롭 시 nil)
 └─ TOCropToolbar (UIView) ── 취소/완료/회전/리셋/종횡비 버튼
```

핵심 아이디어(README): 딤 뷰에 구멍을 뚫는 대신, 배경에 스크롤뷰+이미지, 그 위에 크롭 크기로 클립된 이미지 복사본을 정렬시켜 "구멍"처럼 보이게 함 → 화면 뷰 수 최소화.

### 3.2 크롭 결과 전달 (3가지 경로)
1. **델리게이트** `TOCropViewControllerDelegate` (전부 `@optional`)
   - `didCropImageToRect:angle:` (좌표만)
   - `didCropToImage:withRect:angle:`
   - `didCropToCircularImage:withRect:angle:`
   - `didFinishCancelled:` — 구현하지 않으면 VC가 스스로 dismiss/pop
2. **블록 콜백** `onDidTapDone`, `onDidFinishCancelled`, `onDidCropImageToRect`, `onDidCropToRect`, `onDidCropToCircleImage`
3. **공유 시트** `showActivitySheetOnDone = YES` → `TOActivityCroppedImageProvider` 사용

### 3.3 TOCropView 주요 책임
- 초기 레이아웃 `performInitialSetup` / `layoutInitialImage`
- 크롭박스 드래그: `updateCropBoxFrameWithGesturePoint:` (엣지/코너 판정 `cropEdgeForPoint:`)
- 종횡비: `setAspectRatio:animated:`, 잠금(`aspectRatioLockEnabled`, `aspectRatioLockDimensionSwapEnabled`), `minimumAspectRatio`
- 회전: `rotateImageNinetyDegreesAnimated:clockwise:completion:`, `angle`(0/90/180/270, ±360 이상도 처리), 디바이스 회전 `prepareforRotation` → `performRelayoutForRotation`
- 상태 복원: `imageCropFrame` + `angle` 을 present 전에 설정
- 리셋: `canBeReset` 변화를 `TOCropViewDelegate` 로 통지, `resetLayoutToDefaultAnimated:`
- 편집 중 자동 재정렬 타이머: `startResetTimer`(`cropAdjustingDelay`)

### 3.4 프레젠테이션
- 기본: `presentViewController:` (모달) 또는 `UINavigationController` push
- 커스텀 애니메이션: `presentAnimatedFromParentViewController:fromView:fromFrame:setup:completion:` 및 이미지/각도/복원 프레임을 받는 오버로드, `dismissAnimatedFromParentViewController:...` (`TOCropViewControllerTransitioning` 사용)

## 4. 공개 API 요약 (`TOCropViewController`)

| 분류 | 프로퍼티 / 메서드 |
|---|---|
| 생성 | `initWithImage:`, `initWithCroppingStyle:image:` (`TOCropViewCroppingStyleDefault/Circular`) |
| 상태 | `image`, `imageCropFrame`, `angle`, `croppingStyle`, `aspectRatioPreset`, `minimumAspectRatio`, `cropView`, `toolbar`, `titleLabel` |
| 종횡비 | `aspectRatioLockEnabled`, `aspectRatioLockDimensionSwapEnabled`, `resetAspectRatioEnabled`, `allowedAspectRatios` (3.0.0 추가), `setAspectRatioPreset:animated:` |
| 툴바 | `toolbarPosition`(Bottom/Top), `reverseContentLayout`, `showOnlyIcons`(iOS 26+ 에선 무시), `doneButtonTitle/Color`, `cancelButtonTitle/Color` |
| 버튼 숨김 | `rotateButtonsHidden`, `rotateClockwiseButtonHidden`, `resetButtonHidden`, `aspectRatioPickerButtonHidden`, `doneButtonHidden`, `cancelButtonHidden` |
| 동작 | `showActivitySheetOnDone`, `activityItems`, `applicationActivities`, `excludedActivityTypes`, `showCancelConfirmationDialog`, `hidesNavigationBar` |
| 명령 | `commitCurrentCrop`, `resetCropViewLayout` |

## 5. iOS 26 (Liquid Glass) 대응 규칙 ★

- 판별은 **반드시** `TOCropViewUsesGlassDesign()` (`TOCropViewConstants.h`) 사용. 내부적으로 `#ifdef __IPHONE_26_0` + `@available(iOS 26.0, *)`.
- iOS 26 전용 심볼은 `#if defined(__IPHONE_26_0)` 로 감싸 **Xcode 16(구 SDK)에서도 빌드**되게 유지 (최근 커밋 `f71823b` 가 이 방식으로 통일).
- iOS 26 에서는 툴바가 아이콘 전용 → `doneTextButton`/`cancelTextButton` 은 `nullable`.
- `TOCropView.gridOverlayView` 도 `nullable` (원형 크롭 시 없음).
- 디스플레이 코너 반경 조회는 private 키 → 실패해도 안전하게 폴백해야 함.

## 6. 빌드 / 테스트 / CI

- **Xcode 프로젝트**: `TOCropViewControllerExample.xcodeproj` (Obj-C·Swift 공용)
  - 타깃: `TOCropViewController`(framework), `CropViewController`(framework), `TOCropViewControllerExample`, `CropViewControllerExample`(앱), `TOCropViewControllerExample-Extension`(Share Extension), `TOCropViewControllerTests`, 스위프트 테스트
  - `IPHONEOS_DEPLOYMENT_TARGET = 12.0`, `MARKETING_VERSION = 3.2.0`
- **SwiftPM**: `Package.swift` (tools 5.9, iOS 12) — 라이브러리 `TOCropViewController`, `CropViewController`
- **테스트**: `Objective-C/TOCropViewControllerTests/TOCropViewControllerTests.m` (XCTest, 약 24개). private 상태는 클래스 익스텐션으로 노출해 검증, `UIScrollView.isDragging` 은 method swizzling 으로 스텁.
- **CI**: GitHub Actions(`macos-26`, Ruby 3.4) → `bundle exec fastlane test` (스킴 `TOCropViewControllerTests`, 기본 시뮬레이터 `iPhone 17 Pro`)
- **릴리스**: Buildkite → `fastlane release` (CHANGELOG 항목 필수, podspec 2개 + `MARKETING_VERSION` 버전 범프, 태그/GitHub Release/`pod trunk push`)

```bash
# 로컬 테스트 (fastlane)
bundle install && TEST_SCHEME=TOCropViewControllerTests bundle exec fastlane test
# 또는 xcodebuild
xcodebuild test -project TOCropViewControllerExample.xcodeproj -scheme TOCropViewControllerTests \
  -destination 'platform=iOS Simulator,name=iPhone 17 Pro'
```

## 7. 코딩 컨벤션 (수정 시 반드시 준수)

- `.clang-format` 기준 포맷, 파일 헤더에 MIT 라이선스 주석 + `Copyright 2015-2026 Timothy Oliver` 형식 유지.
- 공개 헤더는 `nullable/nonnull` 어노테이션과 Doxygen 스타일 `/** @param */` 주석 작성.
- Swift 노출 이름은 `NS_SWIFT_NAME(...)` 으로 지정 (예: `presentAnimatedFrom(_:view:frame:setup:completion:)`).
- `#pragma mark - 섹션 -` 로 구획.
- 문자열 추가 시 **28개 `*.lproj` 모두** 반영하고, 리소스 접근은 `TO_CROP_VIEW_RESOURCE_BUNDLE_FOR_OBJECT` 사용 (CocoaPods/SPM/Carthage 번들 위치 차이 흡수).
- 사용자 가시 변경은 `CHANGELOG.md` 최상단 버전 블록(Fixed / Changed / Enhancements / Added)에 PR 링크와 함께 기록.
- 새 파일/헤더 추가 시: Xcode 프로젝트, podspec glob, `include/` 심볼릭 링크, SwiftPM 경로를 함께 점검.

## 8. 알려진 변경 이력 (최근)

| 버전 | 날짜 | 요지 |
|---|---|---|
| 3.2.0 | 2026-07-28 | iOS 26 회전/툴바 버그, 각도 ±360 무한루프, 델리게이트 retain cycle, HDR 보존, `TOCropOverlayView.h` 공개 헤더화 등 다수 수정 |
| 3.1.2 | 2026-04-07 | `onDidTapDone` 추가, Xcode 16 호환 |
| 3.1.0/3.1.1 | 2025-10~11 | iOS 26 게이트, 툴바 크기/iPadOS 타이틀바 이슈 |
| 3.0.0 | 2025-09-30 | Liquid Glass 디자인, `allowedAspectRatios` |

이후 커밋: `3d78d7d` 공백 정리, `f71823b` iOS 26 체크 표준화(Xcode 16 회귀 수정).

## 9. 수정·개발 작업 체크리스트

1. `git switch customize` 에서 작업 (원본 추적은 `origin/main`).
2. 공개 API 변경 → `.h` + Swift 래퍼(`CropViewController.swift`) + README/CHANGELOG 동시 갱신.
3. iOS 26 분기 코드는 `TOCropViewUsesGlassDesign()` / `__IPHONE_26_0` 가드 확인.
4. 테스트 추가(`TOCropViewControllerTests.m`) 후 위 `xcodebuild test` 실행.
5. 버전을 올릴 때는 podspec 2개 + `MARKETING_VERSION` + CHANGELOG 를 같이 수정.

---

# 부록 (심화 분석)

## A. 뷰 컨트롤러 라이프사이클 동작

| 시점 | 동작 |
|---|---|
| `init` | `overrideUserInterfaceStyle = Dark` 고정, `modalPresentationStyle = FullScreen`, `modalTransitionStyle = CrossDissolve`, `hidesNavigationBar = true`. Mac Catalyst 는 툴바 기본 위치 Top, 그 외 Bottom |
| `viewDidLoad` | `transitioningDelegate = self`, cropView/toolbar 프레임 설정, 툴바 버튼 블록 연결(`__weak` 캡처), 원형이면 `clampButtonHidden = YES` |
| `viewWillAppear` | 네비게이션 스택이면 navBar/toolBar 숨기고 상태 저장(`viewWillDisappear` 에서 복원). 모달이면 배경 이미지뷰를 숨겨 전환 성능 확보. 사전 설정된 `aspectRatioPreset` 적용 |
| `viewDidAppear` | `simpleRenderMode = NO`(블러 재활성화), 상태바 페이드, 그리드 오버레이/배경 이미지 페이드 인 |
| `viewWillDisappear` | 상태바 전환 플래그, 네비게이션 바 상태 복원 |

- 레이아웃 판별: `verticalLayout`(세로 레이아웃 여부)에 따라 툴바가 하단/상단(세로) 또는 좌측(가로)에 배치됨. 세로 툴바 폭은 `kTOCropViewControllerToolbarHeight = 44`.
- iOS 26: 가로에서는 `UIViewLayoutRegion`(corner adaptation)으로 iPadOS 창 컨트롤 영역 회피, 세로에서는 전체화면 여부에 따라 하단 인셋을 20/8pt 로 보정(디바이스 코너 반경 private API 사용 → 실패 시 안전 폴백 필요).

## B. Done / Cancel 처리 흐름 (`doneButtonTapped`)

1. `cropView.imageCropFrame`, `cropView.angle` 취득.
2. `onDidTapDone` 이 있으면 **main queue async** 로 호출.
3. `showActivitySheetOnDone == YES` → `TOActivityCroppedImageProvider` + `TOCroppedImageAttributes` + 사용자 `activityItems` 로 `UIActivityViewController`(팝오버, 앵커=`toolbar.doneButtonFrame`) 표시. 완료 시에만 `didFinishCancelled:NO` 호출, 아무 핸들러도 없으면 기본 dismiss. **이 경로에서는 crop 콜백이 호출되지 않는다.**
4. 그 외: Done 버튼 비활성화(텍스트/아이콘 둘 다, 이중 탭 방지 — #651) 후
   - `didCropImageToRect` / `onDidCropImageToRect`: **동기** 호출
   - 원형 + circular 핸들러 존재 → `croppedImageWithFrame:angle:circularClip:YES` 생성 후 0.03초 뒤 호출
   - 일반 + `didCropToImage` 핸들러 존재 → 각도 0 이고 크롭 영역이 전체 이미지면 **원본 이미지를 그대로 반환**, 아니면 생성 후 0.03초 뒤 호출
   - 어떤 핸들러도 없으면 `presentingViewController` 를 dismiss
   - 이미지 콜백 이후 Done 버튼 재활성화(화면을 유지하는 호스트가 재커밋 가능)
5. 델리게이트와 블록이 **둘 다** 구현돼 있으면 둘 다 호출된다.

Cancel: `showCancelConfirmationDialog` 가 YES 면 액션시트("Delete Changes"/"Cancel") 확인 후 `dismissCropViewController` → `didFinishCancelled:YES`(델리게이트/블록, 없으면 pop 또는 dismiss).

## C. 이미지 생성 알고리즘 (`UIImage+CropRotate`)

`croppedImageWithFrame:angle:circularClip:`
- `UIGraphicsImageRenderer` (크기=크롭 프레임, `scale = 원본 scale`).
- `format.opaque = !hasAlpha && !circular` → 불투명 이미지는 알파 제거(용량·성능).
- iOS 17+: 원본이 HDR 이면 HDR 포맷으로 렌더링(톤매핑 방지, #651).
- 순서: (원형이면 타원 클립) → CTM 을 `-frame.origin` 이동 → angle≠0 이면 회전 후 회전된 bounds 만큼 보정 이동 → `drawAtPoint:CGPointZero`.
- 결과는 `orientation = Up` 으로 재래핑.
- **크롭 프레임 좌표계 = 회전이 적용된 이미지 좌표계**(원본 이미지 기준). 복원 시 `imageCropFrame` + `angle` 을 쌍으로 저장/복원해야 한다.

## D. TOCropView 내부 상태 / 상수

| 상수 | 값 | 의미 |
|---|---|---|
| `kTOCropViewPadding` | 14pt | 크롭 영역과 뷰 가장자리 간 패딩 (`cropViewPadding`) |
| `kTOCropTimerDuration` | 0.8s | 조작 종료 후 자동 재정렬 대기 (`cropAdjustingDelay`) |
| `kTOCropViewMinimumBoxSize` | 42pt | 크롭박스 최소 크기 |
| `kTOMaximumZoomScale` | 15 | 최대 줌 (`maximumZoomScale`, 내부 `baseMaximumZoomScale` 로 파생) |
| `kTOCropOverLayerCornerWidth` | 20pt | 오버레이 모서리 핸들 길이 |
| 배경색 | white 0.12 | `TOCROPVIEW_BACKGROUND_COLOR` (딤 overlay 는 alpha 0.35) |

- 내부 열거형 `TOCropViewOverlayEdge` (None/TopLeft/Top/…/Left): 터치 지점으로 어느 변/모서리를 잡았는지 판정.
- 회전 상태 보존: `cropBoxLastEditedSize/Angle/ZoomScale/MinZoomScale/MaxZoomScale`, 디바이스 회전 전 상태 `rotationContentOffset/ContentSize/BoundFrame`.
- 리셋 판정: `originalCropBoxSize`, `originalContentOffset` 과 비교해 `canBeReset` 갱신 (`checkForCanReset`).
- 초기화 전 설정된 값(`restoreAngle`, `restoreImageCropFrame`)은 `performInitialSetup` 에서 한 번만 적용 (`initialSetupPerformed`).
- 원형 모드: `aspectRatio = 1:1`, `cropBoxResizeEnabled = NO`, `gridOverlayView`/팬 제스처 **생성 안 함**.
- 제스처: 그리드 팬이 실패해야 스크롤뷰 팬이 동작 (`requireGestureRecognizerToFail`).
- 편집 중(`editing`)에는 블러(translucency)가 숨고 조작 종료 후 타이머가 지나면 크롭 영역을 중앙 정렬(`moveCroppedContentToCenterAnimated:`).
- 회전 중(`rotateAnimationInProgress`)에는 입력 무시.

## E. 툴바 (`TOCropToolbar`)

- 텍스트 버튼(Cancel/Done)과 아이콘 버튼이 이원화(`doneTextButton`/`doneIconButton`, `cancelTextButton`/`cancelIconButton`). 현재 보이는 취소 버튼은 `visibleCancelButton`.
- iOS 26 + 26 SDK: **아이콘 전용 + Liquid Glass 캡슐**. `showOnlyIcons` 는 무시(`API_DEPRECATED ios(7.0, 18.0)`).
- 버튼: 회전(반시계 기본, 시계 선택), 리셋(`resetButtonEnabled`), 종횡비(clamp, `clampButtonGlowing` 으로 잠금 표시).
- 모든 버튼 동작은 블록 프로퍼티(`doneButtonTapped` 등)로 노출 → VC 가 연결.
- `reverseContentLayout`(RTL/역순), `backgroundViewOutsets`, `statusBarHeightInset` 로 레이아웃 보정.
- 디버그: `TOCROPTOOLBAR_DEBUG_SHOWING_BUTTONS_CONTAINER_RECT` 매크로(0/1).

## F. 현지화 / 프라이버시

- 문자열 키(7개): `Done`, `Cancel`, `Reset`, `Original`, `Square`, `Delete Changes`, `Yes`, `No` (테이블명 `TOCropViewControllerLocalizable`).
- 지원 언어(28+): ar, ca, cs, da-DK, de, en, es, fa, fa-IR, fi, fr, hu, id, it, ja, ko, ms, nl, pl, pt, pt-BR, ro, ru, sk, tr, uk, vi, zh-Hans, zh-Hant (+ Base). **한국어(`ko`)는 이미 포함** — 커스텀 문구가 필요하면 `ko.lproj` 만 수정.
- `PrivacyInfo.xcprivacy`: 추적/수집 데이터 없음 선언 (리소스 번들 `TOCropViewControllerBundle` 에 포함).

## G. 샘플 앱 / 확장

- `Objective-C/TOCropViewControllerExample/ViewController.m`: `UIImagePickerController` 로 이미지 선택 → 이미지 탭 시 `TOCropViewController` 표시(`showCropViewController`), 델리게이트로 결과를 `UIImageView` 에 반영, 공유 버튼. 크롭 스타일(기본/원형) 전환과 애니메이션 present/dismiss 시연.
- `TOCropViewControllerExample-Extension`: Share Extension 샘플(`ShareViewController`). 앱 그룹/entitlements 파일 2개 존재.
- 기능 확인용 최소 코드는 이 샘플을 참고해 신규 옵션을 추가하는 것이 가장 빠름.

## H. 알려진 함정 / 주의점

1. 새 공개 헤더 추가 시 `include/` 심볼릭 링크 누락 → SwiftPM 빌드만 깨짐.
2. 델리게이트·블록 동시 구현 시 **중복 호출**; 콜백 안에서 직접 dismiss 하지 않으면 화면이 남음(핸들러가 하나라도 있으면 자동 dismiss 안 됨).
3. `onDidFinishCancelled`, `onDidCropImageToRect`, `onDidCropToRect`, `onDidCropToCircleImage` 는 헤더상 `strong` 블록 프로퍼티(`copy` 아님) — 수정 시 `copy` 로 통일할지 검토 대상.
4. `angle` 는 90 단위만 유효. 360 이상/음수는 정규화 처리됨(3.2.0).
5. `showActivitySheetOnDone` 사용 시 crop 델리게이트가 호출되지 않는다.
6. `UIAlertController` 액션시트는 iPad 에서 `sourceView` 필요(취소 확인 다이얼로그는 `visibleCancelButton`, 종횡비 선택은 `clampButtonFrame` 기준).
7. 디스플레이 코너 반경 private API 사용 — App Store 심사/향후 OS 변경에 취약, 가드 유지.

## I. 커스터마이징 이력 (`customize` 브랜치)

### 좌우 Safe Area 반영 (폴더블/듀얼스크린 대응)
- 문제: 세로 레이아웃에서 툴바와 타이틀 라벨이 화면 전체 폭 기준으로 배치되어, 좌우 safe area(힌지·컷아웃)를 침범.
- 수정 (`TOCropViewController.m`):
  - `frameForToolbarWithVerticalLayout:` 세로 레이아웃 — 툴바 폭/원점을 `safeAreaInsets.left/right` 기준으로 계산 (일반/글래스 모두). 글래스 풀스크린은 가용 폭 안에서 코너 반경만큼 줄여 중앙 정렬.
  - `adjustToolbarInsets` — 툴바 콘텐츠는 inset 되지만 배경은 `backgroundViewOutsets.left/right` 로 화면 끝까지 확장.
  - `frameForTitleLabelWithSize:verticalLayout:` — 좌우 safe area 사이에서 중앙 정렬 (가로 레이아웃에서 우측 inset 누락도 수정).
  - 크롭 영역(`cropRegionInsets`)은 기존에 `safeAreaInsets` 좌우를 이미 반영하고 있어 변경 없음.
- 테스트: `testPortraitLayoutStaysWithinLeftAndRightSafeArea` (`additionalSafeAreaInsets` 로 좌 40 / 우 60 가정, 툴바·타이틀이 범위 내인지 검증). 수정 전에는 실패, 수정 후 통과 확인.
- 로컬 테스트 시 Xcode 가 deployment target 12.0 을 거부하면 `IPHONEOS_DEPLOYMENT_TARGET=15.0` 을 명령행에 덧붙인다 (프로젝트 파일은 수정하지 않음).
