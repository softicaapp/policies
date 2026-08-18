# 공통 기능 구현 가이드 (묵상한줄 → AI 기도메이트 적용용)

---

## 1. 다른 앱 홍보 섹션

### Firestore 설정
컬렉션: `apps_promotion` / 문서: 자동 ID

| 필드 | 타입 | 예시 |
|------|------|------|
| appId | string | `"prayer"` ← 앱마다 다르게 |
| name_ko | string | `"묵상한줄"` |
| name_en | string | `"Muksang"` |
| name_es | string | `"Muksang"` |
| description_ko | string | `"매일 성경 한 줄 묵상"` |
| description_en | string | `"Daily Bible in One Line"` |
| description_es | string | `"Lectura bíblica diaria"` |
| storeUrl | string | 플레이스토어 URL |
| isActive | boolean | `true` |
| order | number | `1` |

### 생성 파일
```
lib/features/settings/models/promo_app.dart
lib/features/settings/providers/promo_provider.dart
```

### promo_provider.dart 핵심 코드
```dart
final promoAppsProvider = FutureProvider<List<PromoApp>>((ref) async {
  final snapshot = await FirebaseFirestore.instance
      .collection('apps_promotion')
      .where('appId', isEqualTo: 'prayer')  // ← 앱마다 변경
      .where('isActive', isEqualTo: true)
      .orderBy('order')
      .get();
  return snapshot.docs.map((doc) => PromoApp.fromMap(doc.data())).toList();
});
```

### app_strings.dart 추가 문자열
```dart
final String sectionOtherApps;
// ko: '다른 앱도 확인해보세요'
// en: 'Check Out Our Other Apps'
// es: 'Nuestras Otras Apps'
```

### settings_screen.dart 추가
- `_buildPromoSection()` 메서드 추가
- `_buildSectionHeader(context, s.sectionOtherApps)` + `_buildPromoSection()` 호출

---

## 2. 사용자 활동 추적 (lastActive)

### 동작
앱 시작 시 `users/{uid}` 문서에 `lastActive` 타임스탬프 자동 업데이트

### user_repository.dart 추가
```dart
Future<void> updateLastActive(String uid) async {
  await _firestore.collection('users').doc(uid).update({
    'lastActive': FieldValue.serverTimestamp(),
  });
}
```

### auth_provider.dart 추가
```dart
final lastActiveUpdaterProvider = Provider<void>((ref) {
  final userId = ref.watch(currentUserIdProvider);
  if (userId.isEmpty) return;
  final repo = ref.read(userRepositoryProvider);
  repo.updateLastActive(userId).catchError((_) {});
});
```

### app.dart 추가
```dart
ref.watch(lastActiveUpdaterProvider); // build() 안에 추가
```

---

## 3. 개발자에게 의견 보내기

### settings_screen.dart 추가
```dart
Future<void> _sendFeedback() async {
  final uri = Uri(
    scheme: 'mailto',
    path: 'hyunsoo.kim@softica.co.kr',
    query: 'subject=앱이름 피드백',  // ← 앱마다 변경
  );
  if (!await launchUrl(uri)) {
    if (mounted) {
      ScaffoldMessenger.of(context).showSnackBar(
        SnackBar(content: Text(ref.read(strProvider).pageCannotOpen)),
      );
    }
  }
}
```

About 섹션에 타일 추가:
```dart
_buildSettingsTile(
  icon: Icons.mail_outline,
  title: s.sendFeedback,
  onTap: _sendFeedback,
),
```

### app_strings.dart 추가 문자열
```dart
final String sendFeedback;
// ko: '개발자에게 의견 보내기'
// en: 'Send Feedback'
// es: 'Enviar comentarios'
```

---

## 4. Firebase Analytics

### pubspec.yaml 추가
```yaml
firebase_analytics: ^11.4.0
```

### main.dart 추가
```dart
import 'package:firebase_analytics/firebase_analytics.dart';

// Firebase.initializeApp() 바로 아래에
FirebaseAnalytics.instance.setAnalyticsCollectionEnabled(true);
```

---

## 5. 앱 업데이트 팝업

### Firestore 설정
컬렉션: `app_config` / 문서 ID: `prayer` (앱마다 고정 ID)

| 필드 | 타입 | 설명 |
|------|------|------|
| minVersion | string | `"1.0.1"` - 이 버전 미만이면 팝업 |
| forceUpdate | boolean | `true`면 닫기 불가 강제 업데이트 |
| storeUrl | string | 플레이스토어 URL |

### pubspec.yaml 추가
```yaml
package_info_plus: ^8.0.0
```

### 생성 파일
```
lib/features/settings/providers/update_provider.dart
lib/core/widgets/update_dialog.dart
```

### update_provider.dart 핵심 코드
```dart
final updateInfoProvider = FutureProvider<AppUpdateInfo>((ref) async {
  final doc = await FirebaseFirestore.instance
      .collection('app_config')
      .doc('prayer')  // ← 앱마다 변경
      .get();
  // 버전 비교 로직...
});
```

### app.dart 추가
```dart
import 'core/widgets/update_dialog.dart';

// MaterialApp.router의 builder 파라미터에 추가
builder: (context, child) => UpdateChecker(child: child ?? const SizedBox()),
```

### app_strings.dart 추가 문자열
```dart
final String updateTitle;
final String updateMessage;
final String updateNow;
final String updateLater;
// ko: '업데이트 안내' / '새로운 버전이 출시되었습니다. 업데이트 후 이용해 주세요.' / '업데이트' / '나중에'
// en: 'Update Available' / 'A new version is available. Please update to continue.' / 'Update Now' / 'Later'
// es: 'Actualización disponible' / 'Hay una nueva versión disponible...' / 'Actualizar' / 'Después'
```

### 테스트 방법
Firestore에서 `minVersion`을 현재 앱 버전보다 높게 설정하면 팝업 확인 가능

---

## 체크리스트 (새 앱 적용 시)

- [ ] `apps_promotion` 문서 추가 (`appId: 'prayer'`)
- [ ] `promo_provider.dart` - `appId == 'prayer'` 로 변경
- [ ] `app_config/prayer` 문서 생성
- [ ] `update_provider.dart` - `.doc('prayer')` 로 변경
- [ ] `_sendFeedback()` - 메일 제목 변경
- [ ] Firebase Analytics 초기화 확인
- [ ] `app_strings.dart` - 위 문자열 5종 추가
