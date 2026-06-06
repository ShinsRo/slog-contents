---
title: 'Java/Spring에서 HEIC 이미지 변환하기 — libheif JNI 직접 연동'
pubDate: 'May 29 2026'
tags: ['Java', 'JNI', 'Kotlin', 'Spring']
description: 'Java 생태계에 HEIC 변환 라이브러리가 없는 이유와, libheif C 라이브러리를 JNI로 직접 연동해 변환 시간을 8초에서 30ms로 줄인 경험을 공유한다.'
---

## 개요

신분증을 스마트폰으로 촬영해 업로드하는 기능을 개발하다 보면, iOS 사용자가 올리는 이미지 포맷이 `.heic`인 경우를 마주치게 된다. HEIC는 Apple이 iOS 11(2017)부터 카메라 기본 촬영 포맷으로 채택한 것으로, 같은 화질의 JPEG보다 파일 크기가 약 절반이다.

문제는 서버다. Java 생태계에는 HEIC를 변환할 수 있는 라이브러리가 사실상 없다. 처음에는 AWS Lambda로 해결을 시도했지만 처리 시간이 ~8초에 달해 UX에 영향이 컸다. 결국 libheif C 라이브러리를 JNI(Java Native Interface)로 직접 연동했고, 변환 시간은 30~40ms로 줄었다.

이 글에서는 Java에 HEIC 라이브러리가 없는 이유, JVM에서 선택할 수 있는 대안들, 그리고 JNI 직접 연동 구현을 다룬다.

---

## HEIC란 무엇인가

HEIC는 컨테이너 포맷(HEIF)과 코덱(HEVC)의 조합이다.

- **HEIF**(High Efficiency Image File Format): ISO BMFF(Base Media File Format) 기반 컨테이너. MP4와 같은 컨테이너 규격 위에 이미지를 담는다.
- **HEVC**(High Efficiency Video Coding, H.265): HEIF 컨테이너 안에 담기는 코덱. 영상 압축 기술을 이미지에 적용해 JPEG 대비 약 50% 파일 크기를 절감한다.

| | JPEG | HEIC |
|---|---|---|
| 컨테이너 | JFIF/Exif | HEIF (ISO BMFF) |
| 코덱 | DCT | HEVC (H.265) |
| 색공간 | sRGB | Display P3 등 넓은 색역 지원 |
| 파일 크기 | 기준 | ~50% 절감 (같은 품질) |
| Java 표준 지원 | ✅ | ❌ |

iPhone이 기본 포맷으로 채택한 이후, iOS 사용자가 업로드하는 이미지에서 HEIC는 피할 수 없는 포맷이 됐다.

---

## Java에서 HEIC를 다루기 어려운 이유

### 문제는 컨테이너가 아니라 코덱이다

HEIF 컨테이너 자체는 Nokia가 비상업적 사용에 한해 특허를 무상 제공한다. 진짜 문제는 컨테이너 안에 담긴 **HEVC 코덱**이다.

HEVC는 MPEG LA, Via LA 등 여러 특허 풀의 적용을 받는다. 상용 소프트웨어가 HEVC 디코더를 번들링하려면 각 특허 풀로부터 라이선스를 취득해야 한다. 이 구조 때문에 주요 Java 이미지 라이브러리들은 HEIC 지원을 회피하거나 선택적으로만 제공한다.

> HEVC 라이선스를 두고 업계에서는 "a bag of hurt"라고 부른다. 단일 협상 창구가 없고, 특허 풀이 분산되어 있어 라이선스 취득 자체가 복잡하다.

### TwelveMonkeys는?

TwelveMonkeys ImageIO는 Java 표준 ImageIO를 확장해 WebP, TIFF, PSD 등 수십 가지 포맷을 추가해주는 사실상의 표준 라이브러리다. 하지만 HEIC는 [GitHub Issue #976](https://github.com/haraldk/TwelveMonkeys/issues/976)에서 확인할 수 있듯 2026년 현재도 지원되지 않는다. HEVC 특허가 직접적인 이유다.

libheif는 C/C++ 라이브러리로(LGPL), HEVC 디코딩 책임을 사용자에게 넘기는 방식으로 오픈소스를 유지한다. Java 생태계에는 이런 방식의 바인딩조차 없다. 직접 JNI로 연결하는 것이 현실적인 선택지다.

---

## JVM에서 쓸 수 있는 HEIC 변환 방식 비교

### 1. Java 표준 ImageIO

JPEG, PNG, GIF, BMP는 기본 지원하지만 HEIC는 플러그인 없이는 불가능하다. TwelveMonkeys를 포함한 플러그인 생태계에서도 HEIC 지원 플러그인이 없다.

### 2. ImageMagick (ProcessBuilder / im4java)

ImageMagick은 200여 종의 포맷을 지원하는 C 기반 도구다. Java에서는 `ProcessBuilder`로 외부 프로세스를 호출하거나 im4java 래퍼를 사용한다.

```kotlin
// ProcessBuilder 방식 예시
val process = ProcessBuilder("convert", "input.heic", "output.jpg").start()
process.waitFor()
```

**단점:**
- 변환할 이미지를 파일로 디스크에 써야 한다 (메모리 스트림 처리 불가)
- 서버에 ImageMagick이 설치되어 있어야 한다 → 배포 환경 의존성 증가
- 외부 프로세스 fork 오버헤드

### 3. AWS Lambda

S3 업로드 이벤트를 트리거로 Lambda에서 변환하는 패턴이다. Node.js의 `sharp` 라이브러리(libvips 기반)는 HEIC를 잘 처리한다.

**단점:**
- **콜드 스타트 포함 처리 시간 ~8초.** 업로드 → Lambda 트리거 → 변환 → 완료 신호를 폴링하거나 대기해야 하는 구조가 되어 UX에 직접 영향을 준다.
- 비동기 파이프라인(S3 이벤트 → Lambda → 결과 수신)을 별도로 설계해야 해서 아키텍처가 복잡해진다.

### 4. FFM API (Foreign Function & Memory API)

JNI의 후속으로 설계된 현대적인 네이티브 연동 방식이다. JNI보다 안전하고 사용하기 편하며, C 헤더 파일 없이 네이티브 함수를 직접 호출할 수 있다.

하지만 **JDK 22에서야 GA(정식 출시)** 됐다. 당시 프로젝트가 JDK 1.8을 사용하고 있었기 때문에 선택지에 없었다. JDK 22 이상을 사용한다면 JNI 대신 FFM을 고려할 만하다.

### 5. libheif + JNI (직접 연동)

libheif는 strukturag가 개발한 C/C++ HEIC 디코딩 라이브러리로 LGPL 라이선스다. JNI로 직접 연동하면 JVM에서 네이티브 속도로 변환할 수 있다.

**선택 이유:**
- Lambda 대비 변환 속도 **~200배 개선** (8초 → 30~40ms)
- 외부 프로세스 · 외부 서비스 의존 없음
- 정적 링크 빌드로 서버에 별도 설치 불필요

트레이드오프는 **크로스 플랫폼 빌드 복잡성**이다. 각 타깃 플랫폼마다 C 컴파일 환경을 구성하고 shared object를 별도로 빌드해야 한다.

---

## 왜 libheif + JNI를 선택했나

의사결정 흐름은 아래와 같았다.

```
오퍼레이터 수동 변환
  → AWS Lambda 도입 (~8초)
    → libheif JNI 직접 연동 (30~40ms)
```

Lambda ~8초는 단순히 "느리다"의 문제가 아니었다. 신분증 업로드 후 즉시 서류 심사 화면으로 넘어가는 플로우에서, 변환 완료를 기다리는 비동기 구조는 UX를 해쳤다. 동기적으로 처리해서 업로드 응답에 변환 결과를 포함시키는 구조가 필요했다. JNI 직접 연동이 그 요건을 만족하는 유일한 선택지였다.

---

## 구현 상세

### JNI 인터페이스 설계

JNI 함수명은 `Java_패키지_클래스명_메서드명` 규칙을 따른다. 패키지 구분자(`.`)는 `_`로 치환되고, 클래스명과 메서드명이 이어진다.

```java
// [주의] 패키지·클래스명·메서드명 변경 시 C 헤더도 같이 수정해야 합니다.
public class HeicToJpgConverterJNI {
    public native byte[] convertFromBytes(byte[] heicBytes);
}
```

**Kotlin이 아닌 Java로 작성한 이유:** Kotlin 클래스는 컴파일 시 이름이 변형될 수 있다(`$`, `Kt` 접미사 등). JNI 시그니처는 컴파일된 클래스명을 기반으로 결정되므로, Kotlin에서 선언하면 C 헤더의 함수명과 매칭이 깨질 수 있다. Java 클래스는 시그니처가 안정적이다.

### C 구현체 (heic_to_jpg.c)

libheif API로 HEIC를 RGB 픽셀 데이터로 디코딩하고, libjpeg로 JPEG 90% 품질로 인코딩한다. 파일 I/O 없이 메모리 버퍼만 사용한다.

```c
JNIEXPORT jbyteArray JNICALL
Java_kr_co_lendit_jni_libheif_HeicToJpgConverterJNI_convertFromBytes(
    JNIEnv *env, jobject obj, jbyteArray heicBytes
) {
    // 1. JVM 바이트 배열 → C 포인터
    jsize heic_size = (*env)->GetArrayLength(env, heicBytes);
    jbyte* heic_data = (*env)->GetByteArrayElements(env, heicBytes, NULL);

    // 2. libheif로 HEIC 디코딩 (메모리에서 직접 읽기)
    struct heif_context* ctx = heif_context_alloc();
    heif_context_read_from_memory(ctx, (const void*)heic_data, heic_size, NULL);

    struct heif_image_handle* handle;
    heif_context_get_primary_image_handle(ctx, &handle);

    struct heif_image* img;
    heif_decode_image(handle, &img,
                      heif_colorspace_RGB, heif_chroma_interleaved_RGB, NULL);

    // 3. RGB 픽셀 → libjpeg로 JPEG 인코딩 (메모리 출력)
    int stride;
    uint8_t* data = heif_image_get_plane(img, heif_channel_interleaved, &stride);
    unsigned long jpeg_size;
    unsigned char* jpeg_data = encode_jpeg(data, width, height, stride, &jpeg_size);

    // 4. C → JVM 반환 + 메모리 해제
    jbyteArray result = (*env)->NewByteArray(env, jpeg_size);
    (*env)->SetByteArrayRegion(env, result, 0, jpeg_size, (jbyte*)jpeg_data);

    free(jpeg_data);
    heif_image_release(img);
    heif_image_handle_release(handle);
    heif_context_free(ctx);
    (*env)->ReleaseByteArrayElements(env, heicBytes, heic_data, JNI_ABORT);

    return result;
}
```

흐름을 정리하면:

```
JVM(byte[]) 
  → GetByteArrayElements (C 포인터 취득)
    → heif_context_read_from_memory
    → heif_context_get_primary_image_handle
    → heif_decode_image (RGB)
    → encode_jpeg (libjpeg, quality 90)
  → NewByteArray + SetByteArrayRegion (JVM 반환)
  → 메모리 해제 (heif_*, free, ReleaseByteArrayElements)
```

메모리 해제 순서가 중요하다. `heif_image_release` → `heif_image_handle_release` → `heif_context_free` 순으로 안쪽부터 해제해야 한다.

### Kotlin 래퍼 (HeicToJpgConverter)

Spring `@Component`로 등록되며, 두 가지 핵심 설계가 있다.

**1. 런타임 플랫폼 감지**

```kotlin
private val soPath: String by lazy {
    val osArch = System.getProperty("os.arch")
    val osName = System.getProperty("os.name")

    val (system, suffix) = when {
        "Mac" in osName -> "_mac" to ".dylib"
        profileIdentifier.isProduction -> "_correcto" to ".so"
        else -> "_linux" to ".so"
    }

    val arch = when (osArch) {
        "aarch64" -> "_arm64"
        else -> "_x86"
    }

    "$SO_PATH$system$arch$suffix"
}
```

`os.arch`, `os.name` 시스템 프로퍼티로 런타임에 적절한 바이너리를 선택한다. 개발 환경(Mac)과 운영 환경(Rocky Linux)이 다른 상황에서 코드 변경 없이 동작한다.

**2. JAR 리소스 → 임시 파일 로딩**

`System.loadLibrary()`는 `java.library.path`에 있는 파일만 로드할 수 있다. JAR 리소스로 번들된 `.so`/`.dylib`는 이 경로에 없으므로, 임시 파일로 추출한 뒤 `System.load(절대경로)`로 로드한다.

```kotlin
private fun loadLibraryFromResources(resourcePath: String) {
    val inputStream = this::class.java.classLoader.getResourceAsStream(resourcePath)
        ?: throw UnsatisfiedLinkError("네이티브 라이브러리를 찾을 수 없습니다: $resourcePath")

    val tempFile = File.createTempFile("temp_heic_to_jpg", ".so").apply {
        deleteOnExit()
    }

    FileOutputStream(tempFile).use { output ->
        inputStream.use { input -> input.copyTo(output) }
    }

    System.load(tempFile.absolutePath)
}
```

**3. Graceful failure**

JNI 로드 실패가 애플리케이션 부팅을 막지 않도록 처리한다.

```kotlin
private val jni: HeicToJpgConverterJNI? = try {
    loadLibraryFromResources(soPath)
    HeicToJpgConverterJNI()
} catch (e: UnsatisfiedLinkError) {
    log.error(e) { "링크 라이브러리를 찾을 수 없습니다 - $soPath" }
    null
}

fun convertFromBytes(heicBytes: ByteArray): ByteArray? {
    return jni?.convertFromBytes(heicBytes)
}
```

`jni`가 `null`이면 변환 결과도 `null`이다. 서비스 레이어에서 `null` 반환 시 원본 HEIC 파일을 그대로 저장하는 fallback이 동작한다.

### 크로스 플랫폼 빌드

이 구현 전체에서 가장 험난한 부분이다. 빌드는 총 3단계로 이뤄진다.

```
1. JNI 헤더 생성 (javac -h)
     ↓
2. 플랫폼별 gcc 컴파일 (정적 링크)
     ↓
3. 결과물을 JAR 리소스에 번들
```

#### 1단계: JNI 헤더 파일 생성

C 코드가 호출할 함수 시그니처를 Java 컴파일러가 자동으로 만들어준다.

```bash
javac -h . HeicToJpgConverterJNI.java
```

생성된 헤더 파일은 아래와 같다. 함수명이 패키지 경로를 `_`로 구분해 그대로 따른다.

```c
// kr_co_lendit_jni_libheif_HeicToJpgConverterJNI.h (자동 생성)
JNIEXPORT jbyteArray JNICALL
Java_kr_co_lendit_jni_libheif_HeicToJpgConverterJNI_convertFromBytes
  (JNIEnv *, jobject, jbyteArray);
```

C 소스코드(`heic_to_jpg.c`)는 이 헤더를 `#include`하고, 위 시그니처를 그대로 구현한다. **패키지명·클래스명·메서드명 중 하나라도 바뀌면 헤더와 시그니처가 불일치해 런타임에 `UnsatisfiedLinkError`가 난다.** 헤더는 반드시 재생성해야 한다.

#### 2단계: 플랫폼별 gcc 컴파일

세 가지 타깃 환경을 지원한다.

| 타깃 | 빌드 환경 | 출력물 |
|---|---|---|
| Mac arm64 | arm64 Mac 로컬 | `heic_to_jpg_mac_arm64.dylib` |
| Mac x86 | x86 Mac 로컬 | `heic_to_jpg_mac_x86.dylib` |
| Rocky Linux x86 | **Docker** (Rocky Linux 8.10 컨테이너) | `heic_to_jpg_linux_x86.so` |

의존 라이브러리(libheif, libde265, libjpeg, libstdc++)를 모두 **정적 링크**한다. 동적 링크하면 서버에 해당 버전의 라이브러리가 설치되어 있어야 하는데, 배포 환경을 강제할 수 없다. 정적 링크로 `.so` 하나에 모든 의존성을 포함시키면 서버에 별도 설치가 필요 없다.

```bash
# Rocky Linux 8.10 빌드
gcc -shared -o heic_to_jpg_linux_x86.so heic_to_jpg.c -fPIC \
  -I/usr/include/libheif \
  -I/usr/lib/jvm/java-1.8.0/include \
  -I/usr/lib/jvm/java-1.8.0/include/linux \
  /build/libheif/build/libheif/libheif.a \
  /usr/local/lib64/libde265.a \
  /usr/local/lib64/libjpeg.a \
  /usr/lib/gcc/x86_64-redhat-linux/8/libstdc++.a \
  -static -static-libgcc -static-libstdc++
```

**Rocky Linux는 로컬에서 직접 빌드할 수 없어 Docker 컨테이너를 띄워서 진행했다.** 같은 환경에서 빌드하지 않으면 glibc 버전 차이로 런타임에 심볼을 못 찾는 문제가 생길 수 있다.

플랫폼별로 겪은 주요 난관은 두 가지였다.

- **정적 라이브러리 경로**: 플랫폼마다 libheif, libde265, libjpeg의 `.a` 파일 위치가 다르다. 각 환경에서 직접 경로를 찾아야 한다.
- **libstdc++ 정적 링크 깨짐**: `libstdc++.a` 위치가 gcc 버전 디렉터리 아래에 있어 찾기 까다롭고, 링크 순서가 잘못되면 심볼 미정의 오류(`undefined reference to`)가 난다. 링크 순서는 의존하는 쪽을 앞에 써야 한다.

#### 3단계: JAR 리소스 번들

빌드된 바이너리를 `src/main/resources/jni/heif/` 아래에 그대로 놓으면 Gradle 빌드 시 JAR에 포함된다. 별도 설정 없이 `classLoader.getResourceAsStream()`으로 꺼낼 수 있다.

```
src/main/resources/
└── jni/heif/
    ├── heic_to_jpg.c                  ← C 소스 (참고용)
    ├── heic_to_jpg_mac_arm64.dylib
    ├── heic_to_jpg_mac_x86.dylib
    ├── heic_to_jpg_linux_x86.so
    └── heic_to_jpg_correcto_x86.so    ← 운영 Alpine 컨테이너용
```

운영 환경이 Alpine Linux 기반 Docker 이미지(`correcto`)였기 때문에, Rocky Linux 빌드와 별도로 Alpine 환경용 바이너리도 추가로 빌드했다. Alpine은 glibc 대신 musl libc를 사용해 Rocky Linux 빌드와 호환되지 않는다.

---

## 서버 변환 vs 프론트 변환

JavaScript에서도 HEIC를 변환할 수 있다. `heic2any` 라이브러리가 대표적이다.

```javascript
import heic2any from "heic2any";

const jpegBlob = await heic2any({ blob: heicFile, toType: "image/jpeg" });
```

브라우저에서 바로 변환하면 서버 왕복 없이 미리보기를 즉시 렌더링할 수 있다는 장점이 있다. 하지만 프론트 변환에는 몇 가지 한계가 있다.

| | 프론트 변환 (heic2any) | 서버 변환 (JNI) |
|---|---|---|
| 번들 크기 | +2.7MB | 없음 |
| 처리 시간 | UI 블로킹 1~2초 | 30~40ms |
| iOS 호환성 | HEIC tiling variant에서 간헐적 실패 | 안정적 |
| 색공간 | Display P3 → sRGB 변환 정확도 브라우저별 편차 | 제어 가능 |
| 보안 | 클라이언트 신뢰 필요 | 서버에서 일관 처리 |

**신분증 같은 민감한 서류 업로드에는 서버 변환이 필수다.** 클라이언트가 이미지를 가공한 후 업로드하는 것을 막을 수 없고, 변환 결과의 정합성을 KYC 워크플로우에서 보장할 수 없다.

프론트 변환이 유효한 경우는 **업로드 전 미리보기** 용도다. 사용자가 업로드할 이미지를 확인하는 단계에서 서버 왕복 없이 빠르게 보여줄 수 있다. 실제 저장 · 처리는 원본 또는 서버 변환 결과를 기준으로 해야 한다.

---

## 마치며

Java 생태계에서 HEIC를 다루기 어려운 근본 원인은 기술적 한계가 아니라 **HEVC 특허 라이선스 구조**다. 주요 라이브러리들이 지원을 기피하는 이유이며, libheif + JNI 직접 연동이 현실적인 돌파구가 된다.

구현 복잡성의 핵심은 알고리즘이 아니라 **크로스 플랫폼 빌드 환경 구성**이다. 변환 로직 자체는 단순하지만, 세 가지 타깃 환경을 각각 빌드하고 정적 링크 순서를 맞추는 과정이 험난하다. 빌드 스크립트와 Docker 환경을 문서화해두면 이후 유지보수 비용을 크게 줄일 수 있다.
