# CropViewController (Swift 래퍼)

> `TOCropViewController` 를 완전히 캡슐화하여 Swift 친화적 인터페이스를 제공하는 래퍼 프레임워크.
> 코어 라이브러리 설명 → [TOCropViewController.md](TOCropViewController.md)

## 1. 개요

| 항목 | 내용 |
|---|---|
| 언어 | Swift 5 (`SWIFT_VERSION = 5.0`, SwiftPM tools 5.9) |
| 최소 지원 | iOS 12.0 |
| 배포 | CocoaPods(`CropViewController`), SwiftPM(product `CropViewController`), Carthage |
| 소스 | `Swift/CropViewController/CropViewController.swift` (741줄, 단일 파일), `CropViewController.h`, `Info.plist` |
| 의존성 | `TOCropViewController` (Obj-C 코어) — `#if canImport(TOCropViewController)` 로 import |
| 번들 ID | `dev.tim.CropViewController` |

> Swift 프로젝트는 **Obj-C 코어를 포함**해서 배포된다 (podspec `source_files` 에 `Objective-C/TOCropViewController/**` 포함). 따라서 Obj-C 쪽 수정이 곧 Swift 사용자에게도 반영된다.

## 2. 구조

```
Swift/
├── CropViewController/            # 프레임워크 (CropViewController.swift 하나)
├── CropViewControllerExample/     # 샘플 앱 (ViewController.swift 233줄, AppDelegate)
└── CropViewControllerTests/       # XCTest (약 14개, Bridging Header 사용)
```

`CropViewController` (`UIViewController`) 가 내부에 `TOCropViewController` 인스턴스(`toCropViewController`)를 **자식 VC** 로 추가하고, 모든 프로퍼티·메서드를 위임(forwarding)한다.

핵심 포인트:
- `typealias` 로 Obj-C 타입 재노출: `CropViewControllerAspectRatioPreset`, `CropViewControllerToolbarPosition`, `CropViewCroppingStyle`
- 프로토콜 `CropViewControllerDelegate` (`@MainActor @objc`, 모든 메서드 `optional`): `cropViewControllerDidTapDone`, `didCropImageToRect`, `didCropToImage`, `didCropToCircularImage`, `didFinishCancelled`
- `viewWillAppear` 에서 자식 뷰를 지연 추가, `viewDidLayoutSubviews` 에서 frame 동기화
- 상태바/제스처 지연 설정은 `childForStatusBarStyle`, `childForStatusBarHidden`, `childForScreenEdgesDeferringSystemGestures` 로 자식에게 위임
- `modalPresentationStyle = .fullScreen`, `transitioningDelegate = toCropViewController` (커스텀 애니메이션 유지)

## 3. 델리게이트 ↔ 클로저 브리지 (주의 ★)

Obj-C 코어는 델리게이트와 블록 콜백이 별개이다. Swift 래퍼는 `delegate` 가 설정되면 `setUpDelegateHandlers()` 에서 **델리게이트가 구현한 셀렉터에 대해서만** `onDid…` 클로저를 설치한다.

- `delegate` 는 `weak`. 클로저 내부도 `[weak self, weak delegate]` 로 캡처 (3.2.0 에서 retain cycle 수정).
- 델리게이트가 이미 해제된 상태에서 콜백이 오면 `performDefaultDismissal()` 로 pop/dismiss 하여 화면에 남지 않게 함.
- `bridgedCallbacks: Set<BridgedCallback>` 로 **래퍼가 설치한 슬롯만** 추적 → `delegate` 교체/해제 시 이전 브리지만 제거하고, 호스트가 직접 할당한 클로저는 보존.
- `onDidTapDone` 브리지는 다른 크롭 콜백과 **함께** 호출되므로 dismiss fallback 없음 (의도된 설계).
- 새 콜백을 추가할 때는: ① Obj-C 코어에 블록 프로퍼티 추가 → ② Swift 래퍼에 `public var onDidXxx` 위임 → ③ `BridgedCallback` enum 케이스 + 브리지 로직 → ④ 프로토콜에 `@objc optional` 메서드 → ⑤ 테스트.

## 4. 공개 API (래퍼가 위임하는 항목)

Obj-C 코어와 1:1 대응. 이름 차이만 주의.

| Swift | Obj-C |
|---|---|
| `init(image:)`, `init(croppingStyle:image:)` | `initWithImage:`, `initWithCroppingStyle:image:` |
| `presentAnimatedFrom(_:fromView:fromFrame:setup:completion:)` | `presentAnimatedFromParentViewController:fromView:fromFrame:setup:completion:` (코어의 `NS_SWIFT_NAME` 은 `presentAnimatedFrom(_:view:frame:...)`) |
| `dismissAnimatedFrom(_:withCroppedImage:toView:toFrame:setup:completion:)` | `dismissAnimatedFromParentViewController:withCroppedImage:...` |
| `onDidCropToRect` / `onDidCropToCircleImage` / `onDidCropImageToRect` | 동일 이름 블록 프로퍼티 |
| `title` (override) | 코어 `title` 위임 |

그 외: `image`, `imageCropFrame`, `angle`, `croppingStyle`, `aspectRatioPreset`, `allowedAspectRatios`, `aspectRatioLockEnabled`, `aspectRatioLockDimensionSwapEnabled`, `resetAspectRatioEnabled`, `toolbarPosition`, `reverseContentLayout`, 각종 `…ButtonHidden`, `showOnlyIcons`, `showCancelConfirmationDialog`, `doneButtonTitle/Color`, `cancelButtonTitle/Color`, `activityItems`, `applicationActivities`, `excludedActivityTypes`, `cropView`, `toolbar`, `hidesNavigationBar`, `commitCurrentCrop()`, `resetCropViewLayout()`, `setAspectRatioPreset(_:animated:)`.

## 5. 사용 예

```swift
let vc = CropViewController(image: image)          // 원형: CropViewController(croppingStyle: .circular, image: image)
vc.delegate = self
present(vc, animated: true)

func cropViewController(_ cropViewController: CropViewController,
                        didCropToImage image: UIImage, withRect cropRect: CGRect, angle: Int) {
    // 결과 이미지 처리 후 직접 dismiss 하거나, 구현하지 않으면 자동 dismiss
}

// 클로저 방식
vc.onDidCropToRect = { image, rect, angle in … }
vc.onDidFinishCancelled = { finished in … }
```

## 6. 빌드 / 테스트

- 스킴: `CropViewController`, `CropViewControllerExample` (`TOCropViewControllerExample.xcodeproj` 내)
- 테스트 파일: `Swift/CropViewControllerTests/CropViewControllerTests.swift` (Bridging Header 로 Obj-C 내부 접근)
- CI 는 현재 `TOCropViewControllerTests` 스킴만 실행 → Swift 테스트는 필요 시 로컬에서 실행

```bash
xcodebuild test -project TOCropViewControllerExample.xcodeproj -scheme CropViewController \
  -destination 'platform=iOS Simulator,name=iPhone 17 Pro'
```

## 7. 수정 시 주의사항

- 래퍼는 얇게 유지: 로직은 Obj-C 코어에, Swift 쪽은 위임과 타입 브리징만.
- Xcode 26 모듈 검증 이슈가 과거에 있었음(3.2.0 수정) → `CropViewController.h` umbrella 헤더/모듈 설정을 건드릴 때 주의.
- Obj-C 헤더에서 `nullable` 을 바꾸면 Swift 호출부(옵셔널 언래핑)가 깨질 수 있음 (예: `gridOverlayView`, `doneTextButton`).
- Obj-C 에서 forward declaration 만 있는 클래스의 프로퍼티는 Swift 에서 보이지 않음 → 공개 헤더에서 `#import` 할 것 (`TOCropOverlayView.h` 사례, #653).
- 버전은 `CropViewController.podspec` 과 `MARKETING_VERSION` 을 코어와 **동일하게** 유지 (현재 3.2.0).
- 파일 헤더: MIT 라이선스 주석 + `Copyright 2017-2026 Timothy Oliver`.

---

# 부록 (심화 분석)

## A. 래퍼 구현 세부

- `toCropViewController` 는 `internal let ... !` — 같은 모듈(테스트 포함 `@testable`)에서만 접근.
- `setUpCropController()`: `modalPresentationStyle = .fullScreen`, 자식 VC 추가(`addChild`), `transitioningDelegate = toCropViewController`, 코어 `delegate = self`(래퍼가 코어 델리게이트 역할), `didMove(toParent:)`.
- 코어의 `TOCropViewControllerDelegate` 를 래퍼가 직접 구현하지 않고, **블록 콜백 슬롯**(`onDid…`)을 통해 `CropViewControllerDelegate` 로 중계한다. 코어는 "블록 또는 델리게이트가 하나라도 있으면 자동 dismiss 안 함" 규칙이므로, 래퍼는 델리게이트가 사라진 경우에도 `performDefaultDismissal()` 로 규칙을 보완한다.
- `hasCancelHandler` 류 분기 없이, 델리게이트가 구현한 셀렉터만 `responds(to:)` 로 검사하여 브리지 설치.
- `public var delegate` 의 `didSet` 에서 `setUpDelegateHandlers()` 호출 → 델리게이트 교체 시 자동 재설정.

## B. 콜백 우선순위 정리

| 상황 | 결과 |
|---|---|
| 델리게이트만 설정 | 구현한 메서드만 브리지 → 호출 |
| 클로저만 직접 할당 | 래퍼는 건드리지 않음, 그대로 호출 |
| 둘 다 | 같은 슬롯이면 **나중에 설정된 쪽이 덮어씀**(브리지는 `delegate` 설정 시점에 설치). 슬롯이 다르면 모두 호출 |
| 델리게이트 해제(`nil`) | 래퍼가 설치한 브리지만 제거, 직접 할당 클로저 유지 |
| 델리게이트 deinit 후 콜백 | `performDefaultDismissal()` (didTapDone 제외) |

> 실무 팁: 델리게이트와 클로저를 섞어 쓸 때는 `delegate` 를 먼저 설정하고, 그 다음 직접 클로저를 할당하면 의도가 명확하다.

## C. 샘플 앱 (`Swift/CropViewControllerExample`)

- `ViewController.swift`(233줄): 이미지 피커 → `CropViewController` 표시 → 델리게이트로 결과 반영, 원형 크롭/애니메이션 present 시연.
- `AppDelegate.swift`, `Main.storyboard`, `LaunchScreen.storyboard`, `CropViewController-Bridging-Header.h`.
- 새 래퍼 API 를 추가하면 이 샘플에 사용 예를 넣어 수동 검증한다.

## D. 테스트 (`CropViewControllerTests.swift`, 약 14개)

- Bridging Header(`CropViewControllerTests-Bridging-Header.h`)로 Obj-C 코어 내부 접근.
- 주요 검증 대상: 프로퍼티 위임(getter/setter 왕복), 델리게이트 브리지 설치/해제, delegate weak 해제 시 retain cycle 없음, 교체된 델리게이트 미호출.
- 새 래퍼 프로퍼티 추가 시 "set → 코어 값 확인 → get" 패턴 테스트를 추가한다.

## E. 새 프로퍼티 추가 레시피 (Obj-C → Swift)

1. Obj-C `TOCropViewController.h` 에 `@property` 추가 (nullability, 문서 주석 포함). 구현부(.m)에서 기본값/툴바·뷰 반영.
2. `CropViewController.swift` 에 위임 프로퍼티 추가:
   ```swift
   public var newOption: Bool {
       set { toCropViewController.newOption = newValue }
       get { return toCropViewController.newOption }
   }
   ```
3. 주석은 기존과 동일하게 `/** … Default is … */` 형식.
4. Obj-C·Swift 테스트 각각 1개 이상, 샘플 앱 반영, `CHANGELOG.md` 기록.
5. 최근 3.2.0 에서 "getter 가 항상 기본값을 반환" 류 버그(#651)가 있었으므로 **setter/getter 왕복 테스트 필수**.

## F. 배포 메타

- `CropViewController.podspec`: `source_files = Swift/CropViewController/**/*.{h,swift}` + Obj-C 소스 전체, `swift_version = 5.0`, 리소스 번들 `TOCropViewControllerBundle`.
- SwiftPM: `CropViewController` 타깃은 `TOCropViewController` 타깃에 의존, `sources: ["CropViewController.swift"]` 만 컴파일(헤더/Info.plist 제외).
- `TOCropViewController.podspec` 과 **버전 동기화** 필수 (fastlane `release` 가 두 podspec + `MARKETING_VERSION` 을 함께 갱신).
