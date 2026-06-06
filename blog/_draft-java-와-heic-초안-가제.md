주제: Java Spring 진영에서 HEIC 대응
목적: 내가 했던 HEIC -> JPG 변환 방법(libheif + JNI) 소개하는 블로그 포스팅

AI 작업 순서

1. lendit 프로젝트 확인 및 작업한 거 코드로 확인
2. draft 파일에 확인한 것 정리. - 언제든 다시 참조할 수 있도록 레퍼런스 남기기
    - 커밋에 의사결정이 남아있다면 그것도 정리해두기
    - ex. 당시 람다로 변환했으나 느려서 jni 로 변경
3. 현재 JVM 기준, 이미지 변환 방식 조사 및 정리 + 레퍼런스
4. HEIC 포맷이 자바에서 변환하는 라이브러리가 없는 이유
5. 서버 변환과 프론트 변환 차이. 둘다해야하는가에 대한 고찰 정리
6. 목차 정하기
7. 블로그 글 작성 시작

---

## [Step 2] 코드 리뷰 & 커밋 타임라인

### 핵심 파일 위치
- `lendit/app/core/src/main/java/kr/co/lendit/jni/libheif/HeicToJpgConverter.kt`
- `lendit/app/core/src/main/java/kr/co/lendit/jni/libheif/HeicToJpgConverterJNI.java`
- `lendit/app/core/src/main/resources/jni/heif/heic_to_jpg.c`
- `lendit/app/core/src/main/java/kr/co/lendit/user/document/UserDocumentServiceImpl.kt`
- 바이너리: `jni/heif/heic_to_jpg_{mac_arm64,mac_x86,linux_x86,correcto_x86}.{dylib,so}`

### 설계 포인트

**HeicToJpgConverterJNI.java — Java로 작성한 이유**
Kotlin 클래스는 컴파일 후 이름이 변형될 수 있어 JNI 함수 시그니처(`Java_패키지_클래스_메서드`)가 깨질 수 있다. Java로 유지해야 C 헤더와 매칭이 안전하다.

**System.loadLibrary vs System.load**
`System.loadLibrary()`는 `java.library.path`에 있는 파일만 로드한다. JAR 리소스에 번들된 `.so`/`.dylib`는 해당 경로에 없으므로, 임시 파일로 추출 후 `System.load(절대경로)`로 로드한다.

**Graceful failure**
JNI 로드 실패 시 `UnsatisfiedLinkError`를 catch해 `jni = null`로 처리. 변환 불가 시 원본 파일을 그대로 업로드하는 fallback 동작.

**content-type 기반 분기 (UserDocumentServiceImpl)**
`convertToJpgIfIsHeic(file)` — `file.contentType == "image/heic"`일 때만 변환. 변환 성공 시 `HeicToJpgMultipartFile`(wrapper)로 교체, 실패 시 원본 반환.

**C 코드 흐름 (heic_to_jpg.c)**
1. `heif_context_read_from_memory` — 파일 I/O 없이 메모리에서 HEIC 파싱
2. `heif_context_get_primary_image_handle` → `heif_decode_image` — RGB 디코딩
3. `encode_jpeg` (libjpeg, quality 90, 메모리 출력)
4. `NewByteArray` + `SetByteArrayRegion` — JVM 반환

### 커밋 타임라인 (lendit)

| 날짜 | 커밋 | 내용 |
|---|---|---|
| 2025-03-10 | `44b2dd8b` | feat: libheif JNI 작성 for arm64 — amd64 리눅스는 TODO |
| 2025-03-12 | `b9287a12` | **PR #11978** libheif JNI 작성과 신분증 이미지파일 변환 (arm64/x86 Mac + Rocky Linux) |
| 2025-03-12 | `4233cc10` | PR #11980 Correcto 8 알파인 도커 이미지용 빌드 파일 추가 |
| 사이 커밋들 | 여러 개 | 정적 링크 깨짐 수정, 로그 정리, 빌드 오류 수정 등 |

### 의사결정 타임라인 (이력서 + 커밋 조합)

Lambda → JNI 전환 커밋이 git에 직접 남아있지 않음. 이력서 기준:
- 기존: 오퍼레이터 수동 변환 구조
- 중간: AWS Lambda 도입 → 처리 시간 ~8초
- 최종: libheif JNI → 30~40ms (약 200배 개선)

Lambda가 느린 이유: 콜드 스타트 + 비동기 파이프라인 (S3 이벤트 → Lambda → 결과 대기)

---

## [Step 3] JVM 이미지 변환 방식 조사

| 방식 | HEIC 지원 | 특이사항 |
|---|---|---|
| Java 표준 ImageIO | ❌ | 기본 포맷만 (JPEG, PNG, GIF, BMP) |
| TwelveMonkeys ImageIO | ❌ | [Issue #976](https://github.com/haraldk/TwelveMonkeys/issues/976) — 특허 이유로 미지원 |
| ImageMagick (ProcessBuilder/im4java) | ✅ | 외부 프로세스, 파일 기반, 배포 환경 의존 |
| AWS Lambda (Node sharp) | ✅ | 콜드 스타트 포함 ~8초, 비동기 파이프라인 복잡성 |
| libheif + JNI | ✅ | 30~40ms, 정적 링크 번들, 빌드 복잡 |

---

## [Step 4] Java에 HEIC 변환 라이브러리가 없는 이유

- HEIF **컨테이너** 자체는 ISO BMFF 기반, Nokia가 비상업적 사용 특허 무상 제공
- 문제는 내부 **코덱(HEVC/H.265)**: MPEG LA, Via LA 등 특허 풀 적용
- HEVC 디코더를 소프트웨어에 번들하려면 상용 라이선스 취득 필요
- TwelveMonkeys, Apache Imaging 등 오픈소스 라이브러리가 기피하는 직접적 이유
- libheif (C, LGPL)는 라이선스 책임을 사용자에게 넘김으로써 오픈소스로 유지

참고: https://www.mpegla.com/wp-content/uploads/HEVCweb.pdf, https://news.ycombinator.com/item?id=23264295

---

## [Step 5] 서버 변환 vs 프론트 변환

**heic2any (JS 클라이언트 라이브러리)**
- 브라우저에서 HEIC → JPEG/PNG/GIF 변환
- 번들 크기 +2.7MB
- Canvas 렌더링 시 메인 스레드 블로킹 (~1-2초)
- iOS Safari: HEIC tiling variant(멀티 이미지 컨테이너)에서 간헐적 실패
- Display P3 색공간 → 브라우저별 색 관리 편차 발생 가능

**서버 변환이 필요한 이유 (신분증 업로드 컨텍스트)**
- 클라이언트 변환 결과를 신뢰할 수 없음 (KYC 정합성)
- 이미지 조작 여지 차단
- 일관된 품질(90% JPEG) 보장

**두 개 다 써야 하는 경우**
- 업로드 전 미리보기(preview): heic2any 클라이언트 변환
- 실제 저장/처리: 서버 변환 필수

---

## [Step 6] 목차

```
## 개요
## HEIC란 무엇인가
## Java에서 HEIC를 다루기 어려운 이유
  ### 문제는 컨테이너가 아니라 코덱이다
  ### TwelveMonkeys는?
## JVM에서 쓸 수 있는 HEIC 변환 방식 비교
## 왜 libheif + JNI를 선택했나
## 구현 상세
  ### JNI 인터페이스 설계
  ### C 구현체 (heic_to_jpg.c)
  ### Kotlin 래퍼 (HeicToJpgConverter)
  ### 크로스 플랫폼 빌드
## 서버 변환 vs 프론트 변환
## 마치며
```
