# AAB 빌드 가이드 (말씀한줄)

## 사전 준비
- Flutter SDK 설치 완료
- Android Studio 또는 JDK 설치 완료 (keytool 사용을 위해)

---

## 1단계 — Keystore 생성 (최초 1회만)

터미널(PowerShell 또는 CMD)에서 실행:

```bash
keytool -genkey -v -keystore C:\Users\suris\upload-keystore.jks ^
  -keyalg RSA -keysize 2048 -validity 10000 ^
  -alias upload
```

입력 항목:
| 항목 | 설명 |
|------|------|
| 키 저장소 비밀번호 | 기억하기 쉬운 비밀번호 설정 (재입력 확인) |
| 키 비밀번호 | 위와 동일하게 설정해도 됨 |
| 이름 (이름과 성) | 본인 이름 또는 앱 이름 (영문) |
| 조직 단위 | 생략 가능 (Enter) |
| 조직 이름 | softica (또는 생략) |
| 구/군/시 | Seoul |
| 시/도 | Seoul |
| 국가 코드 | KR |

> ⚠️ **keystore 파일은 절대 분실하지 마세요.**
> 잃어버리면 앱 업데이트를 올릴 수 없습니다.
> 클라우드(Google Drive, iCloud 등)에 백업해두세요.

---

## 2단계 — key.properties 파일 생성

`android/key.properties` 파일을 새로 만들고 아래 내용 입력:

```
storePassword=1단계에서_설정한_비밀번호
keyPassword=1단계에서_설정한_비밀번호
keyAlias=upload
storeFile=C:\\Users\\suris\\upload-keystore.jks
```

> ⚠️ 이 파일은 절대 Git에 올리지 마세요.
> `.gitignore`에 `android/key.properties`가 포함되어 있는지 확인하세요.

---

## 3단계 — build.gradle.kts 확인

`android/app/build.gradle.kts` 파일이 이미 아래와 같이 설정되어 있어야 합니다:

```kotlin
val keyPropertiesFile = rootProject.file("key.properties")
val keyProperties = Properties()
if (keyPropertiesFile.exists()) {
    keyProperties.load(keyPropertiesFile.inputStream())
}

// ... (중략)

signingConfigs {
    create("release") {
        keyAlias = keyProperties["keyAlias"] as String?
        keyPassword = keyProperties["keyPassword"] as String?
        storeFile = keyProperties["storeFile"]?.let { file(it) }
        storePassword = keyProperties["storePassword"] as String?
    }
}

buildTypes {
    release {
        signingConfig = signingConfigs.getByName("release")
    }
}
```

✅ 이미 설정 완료된 상태입니다.

---

## 4단계 — AAB 빌드

프로젝트 루트 디렉토리(`c:\Users\suris\Desktop\DEV\muksang`)에서 실행:

```bash
flutter build appbundle --release
```

빌드 완료 후 파일 위치:
```
build/app/outputs/bundle/release/app-release.aab
```

빌드 시간: 약 3~10분 (첫 빌드는 더 오래 걸릴 수 있음)

---

## 5단계 — Play Console에 업로드

1. [Google Play Console](https://play.google.com/console) 접속
2. 앱 선택 → **프로덕션** (또는 내부 테스트)
3. **새 버전 만들기** 클릭
4. `app-release.aab` 파일 업로드
5. 버전 정보 입력 후 검토 제출

---

## 버전 업데이트 시

`pubspec.yaml`의 버전 번호를 올리고 다시 빌드:

```yaml
version: 1.0.1+2   # 앞: 사용자에게 보이는 버전 / +뒤: 빌드 번호 (매번 증가)
```

```bash
flutter build appbundle --release
```

> Play Console은 같은 versionCode(+뒤 숫자)를 두 번 올릴 수 없습니다.
> 업데이트마다 반드시 +뒤 숫자를 1씩 올려야 합니다.

---

## 문제 해결

| 오류 | 해결 방법 |
|------|----------|
| `keytool: command not found` | JDK 설치 후 환경변수 PATH에 추가 |
| `Keystore file not found` | `key.properties`의 `storeFile` 경로 확인 |
| `Wrong password` | `key.properties`의 비밀번호 확인 |
| `versionCode already exists` | `pubspec.yaml`의 `+뒤 숫자` 증가 |
