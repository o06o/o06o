# o06 | Android & Flutter 개발자

> **현재** — 감정 대화와 수면 사운드 앱 **Sollang**의 Android 앱을 설계부터 배포 자동화까지 맡아 개발하고 있습니다. (2026.03~, 스토어 출시 심사 중)

5년간 모바일 제품의 기능 개발부터 아키텍처, 운영 안정성, 배포 자동화까지 다뤄 왔습니다.
Kotlin과 Jetpack Compose를 중심으로 사용자가 겪는 실패를 복구 가능한 상태로 만들고,
팀의 설계 기준을 코드와 자동 검증으로 남기는 일에 관심이 많습니다.

## 전문 분야

- **Android** — Kotlin, Jetpack Compose, Coroutines/Flow, Hilt, Room, Media3
- **Flutter** — Dart, Riverpod, go_router, Drift
- **Architecture** — state/action 경계, multi-module, Domain/Data API, dependency injection
- **Quality** — unit/UI test, architecture lint, GitHub Actions, Jenkins
- **Operations** — Crashlytics, Amplitude, WebSocket recovery, background playback

## 대표 엔지니어링 작업

### [Sollang Android Engineering](https://github.com/o06o/sollang-android-engineering)

감정 대화와 수면 사운드를 제공하는 모바일 제품에서 다룬 설계 문제를 공개 가능한 코드와 case study로 정리했습니다.

- process와 화면 lifecycle에 독립적인 Media3 백그라운드 재생·수면 타이머
- STOMP 연결 실패, 응답 timeout과 재연결 중 중복 응답 처리
- 고빈도 Compose state 분리와 draw/layout 단계 최적화
- Feature → Domain → Data 의존 방향과 architecture lint
- `dev`·`release/**` 검증 → Slack 승인 → Jenkins의 정확한 commit 배포
- SDK 독립 이벤트 계약과 DEV/PROD를 분리한 Amplitude 제품 분석

빠른 리뷰:
[타이머 코드](https://github.com/o06o/sollang-android-engineering/blob/main/src/main/kotlin/dev/o06o/sollang/timer/SleepTimerEngine.kt) ·
[회귀 테스트](https://github.com/o06o/sollang-android-engineering/blob/main/src/test/kotlin/dev/o06o/sollang/timer/SleepTimerEngineTest.kt) ·
[Compose 성능 사례](https://github.com/o06o/sollang-android-engineering/blob/main/docs/case-studies/compose-runtime-performance.md)
— 나머지는 [case studies](https://github.com/o06o/sollang-android-engineering/tree/main/docs/case-studies)

## AI 활용 크로스플랫폼 개발

기존 Android의 제품 계약을 SwiftUI·Swift Concurrency·Tuist 기반 iOS 앱으로 옮기면서 AI를 코드 생성기가 아니라
계약 탐색, 플랫폼 차이 분석, 테스트와 문서 동기화 도구로 사용했습니다.

제품 판단과 공개 API 경계는 사람이 결정하고, 확인되지 않은 동작은 추측해 구현하지 않는 방식을 유지했습니다.

→ [Android에서 iOS로: AI를 활용한 개발 시스템](https://github.com/o06o/sollang-android-engineering/blob/main/docs/ai-assisted-ios-development.md)

## 기술 기록

- [대용량 JSON 처리 중 Android OOM을 스트리밍 방식으로 바꾼 기록](troubleshooting/%5BAndroid%5D%20Json%20Memory%20%EA%B4%80%EB%A6%AC.md)
- [Compose 재구성·렌더링 성능 트러블슈팅](https://github.com/o06o/sollang-android-engineering/blob/main/docs/case-studies/compose-runtime-performance.md)

## 연락처

- Email: `o06.develop@gmail.com`
