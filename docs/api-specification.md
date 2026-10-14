# 🌐 Doping Pulse: RESTful API ve Servis Sözleşmesi Spesifikasyonu (API RFC)

**Doküman Kodu:** `RFC-PULSE-003`  
**Modül:** `Pulse Gateway & Service Contracts`  
**Mimar:** Senior Backend Architect & MT Core Team  
**Protokol:** HTTPS / RESTful JSON / OpenAPI 3.0 Standardı  
**Base URL:** `https://api.dopinghafiza.com/pulse/v1`  

---

## 1. Mimari Prensipler ve Standartlar

### 1.1. Kimlik Doğrulama ve Yetkilendirme (Auth & Identity)
Tüm client istekleri **OAuth2 / JWT Bearer Token** taşımak zorundadır.  
İstek başlığı (Header):
```http
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```
* Backend, `student_id` bilgisini URL'den değil, güvenli bir şekilde JWT payload içerisindeki `sub` (subject claim) alanından çözer.

### 1.2. İdempotency (Çift İstek Koruması)
Ağ kopmaları veya kullanıcının butona çift basması durumunda mükerrer kayıt ve durum bozulmasını önlemek için tüm `POST` isteklerinde opsiyonel/zorunlu `Idempotency-Key` başlığı desteklenir.
```http
Idempotency-Key: 7c5a0b94-82a1-4235-9081-30c8df634f51
```

### 1.3. Standart Hata Formatı (RFC 7807 Problem Details)
Hatalı isteklerde tutarlı `application/problem+json` gövdesi döndürülür:
```json
{
  "type": "https://api.dopinghafiza.com/errors/intervention-already-completed",
  "title": "Conflict",
  "status": 409,
  "detail": "Bu müdahale kartı daha önce tamamlanmıştır veya süresi dolmuştur.",
  "instance": "/pulse/v1/interventions/8f7b3c2a-1122/events",
  "code": "INTERVENTION_EXPIRED_OR_COMPLETED",
  "timestamp": "2026-10-13T17:25:00Z"
}
```

### 1.4. Hız Sınırlama (Rate Limiting)
DDoS ve istemci döngülerini (infinite retry) önlemek için API Gateway seviyesinde Token Bucket algoritması uygulanır:
* **Client API'leri:** Öğrenci başına `60 istek / dakika` (Burst: 100)
* **Internal API'ler:** Servis başına `10.000 istek / dakika`

---

## 2. İstemci Uç Noktaları (Client-Facing Endpoints)

### 2.1. `GET /pulse/me/status`
Mobil veya web uygulaması açıldığında öğrencinin güncel nabız durumunu, serisini (streak) ve varsa gösterilmesi gereken aktif müdahale kartını çeker.

* **Yetki:** Öğrenci (`ROLE_STUDENT`)
* **Örnek Başarılı Yanıt (HTTP 200 OK):**
```json
{
  "studentId": "c3a1b8d2-45e0-4789-9a21-998877665544",
  "pulseState": "ACTIVATE",
  "pulseSubState": "ACTIVATE_STALLED",
  "consecutiveInactiveDays": 3,
  "streakDays": 0,
  "isPaused": false,
  "activeIntervention": {
    "interventionId": "018f3a2b-7c4d-7a89-9e12-3456789abcde",
    "templateId": "ACTIVATE_FIRST_5M_VIDEO",
    "type": "MICRO_TASK",
    "badgeText": "Günün Kolay Adımı",
    "title": "Bugün sadece bu 4 dakikalık videoyu izle",
    "description": "Programın gözünde büyümesine gerek yok. Sadece bir başlangıç yapıyoruz.",
    "cta": {
      "text": "Videoya Git (4 Dk)",
      "actionType": "DEEP_LINK",
      "payload": "doping://content/lesson/matematik/temel-kavramlar-1"
    },
    "dismissible": true,
    "expiresAt": "2026-10-14T20:59:59Z"
  }
}
```

* **Öğrenci STEADY (On-Track) İse Yanıt:**
```json
{
  "studentId": "c3a1b8d2-45e0-4789-9a21-998877665544",
  "pulseState": "STEADY",
  "pulseSubState": "STEADY_NORMAL",
  "consecutiveInactiveDays": 0,
  "streakDays": 5,
  "isPaused": false,
  "activeIntervention": null
}
```

---

### 2.2. `POST /pulse/interventions/{interventionId}/events`
Öğrencinin müdahale kartı üzerindeki yaşam döngüsü eylemlerini (görüntülendi, tıklandı, kapatıldı, tamamlandı) kaydeder.

* **Path Parametresi:** `interventionId` (UUIDv7)
* **İstek Gövdesi (Request Body):**
```json
{
  "eventType": "CLICK_CTA",
  "clientTimestamp": "2026-10-13T17:22:45Z",
  "metadata": {
    "screen": "HOME_DASHBOARD",
    "appVersion": "4.12.0",
    "platform": "IOS"
  }
}
```
* **Desteklenen `eventType` Değerleri:**
  * `IMPRESSION`: Kart ekranda en az 2 saniye görüntülendi.
  * `CLICK_CTA`: Aksiyon butonuna tıklandı ve hedef sayfaya yönlendirildi.
  * `DISMISS`: Öğrenci kartın sağ üst köşesindeki "x" butonuna basarak kapattı.
  * `COMPLETE`: Mikro-görev başarıyla tamamlandı (İstemci tarafından tetiklenir).
* **Başarılı Yanıt:** `HTTP 204 No Content`
* **Olası Hatalar:**
  * `404 Not Found`: Müdahale kartı bulunamadı veya öğrenciye ait değil.
  * `409 Conflict`: Kart zaten `DISMISSED` veya `EXPIRED` durumunda.

---

### 2.3. `GET /pulse/checkins/{surveyId}`
ReStart durumundaki öğrenciye sunulacak mini anketin güncel soru ve şıklarını getirir.

* **Path Parametresi:** `surveyId` (String, örn: `RESTART_CORE_V1`)
* **Örnek Başarılı Yanıt (HTTP 200 OK):**
```json
{
  "surveyId": "RESTART_CORE_V1",
  "title": "Küçük Bir Mola Mı?",
  "promptText": "Son günlerde görüşemedik, her şey yolunda mı? Seni ne zorladı?",
  "options": [
    {
      "key": "SCHOOL_EXAMS",
      "label": "Okul sınavlarım / yazılılarım çok yoğundu",
      "icon": "school_calendar"
    },
    {
      "key": "SUBJECT_DIFFICULTY",
      "label": "Son çalıştığım konuyu anlamadım, moralim bozuldu",
      "icon": "help_outline"
    },
    {
      "key": "BURNOUT",
      "label": "Çok yoruldum, tempomu koruyamadım",
      "icon": "battery_charging_20"
    },
    {
      "key": "OTHER",
      "label": "Diğer bir sebep",
      "icon": "more_horiz"
    }
  ]
}
```

---

### 2.4. `POST /pulse/checkins`
Öğrencinin ReStart anketine verdiği cevabı işler, kök nedeni veritabanına kaydeder ve programa anında adaptasyon uygular.

* **Headers:** `Idempotency-Key: <unique-uuid>`
* **İstek Gövdesi (Request Body):**
```json
{
  "interventionId": "018f3a2b-7c4d-7a89-9e12-3456789abcde",
  "surveyId": "RESTART_CORE_V1",
  "selectedOptionKey": "SCHOOL_EXAMS",
  "freeTextNote": "Fizik yazılım vardı, tüm hafta ona hazırlandım.",
  "requestedAdaptation": "REDUCE_LOAD_50_PERCENT"
}
```
* **Örnek Başarılı Yanıt (HTTP 200 OK):**
```json
{
  "success": true,
  "feedbackMessage": "Seni çok iyi anlıyoruz! Yazılı döneminde hedeflerini 1 hafta boyunca %50 hafiflettik.",
  "newPulseState": "BOOST",
  "newPulseSubState": "BOOST_LOW_PACE",
  "appliedAdaptation": {
    "actionType": "SCHEDULE_HALVED",
    "dailyTargetQuestions": 10,
    "dailyTargetDurationMinutes": 20,
    "validUntil": "2026-10-20T23:59:59Z"
  }
}
```

---

### 2.5. `POST /pulse/me/pause`
Öğrencinin kasıtlı olarak sisteme "Ben 3 gün yokum (hastalık/tatil/sınav)" diyerek Pulse motorunu geçici olarak dondurmasını sağlar (Bypass bayrağı).

* **İstek Gövdesi (Request Body):**
```json
{
  "durationDays": 4,
  "reason": "MEDICAL_OR_HOLIDAY"
}
```
* **Başarılı Yanıt (HTTP 200 OK):**
```json
{
  "isPaused": true,
  "pausedUntil": "2026-10-17T17:30:00Z",
  "message": "Pulse motoru 4 gün boyunca duraklatıldı. Bu süreçte kopuş uyarısı almayacaksın."
}
```

---

## 3. İç Servisler ve Asenkron İşçi Uç Noktaları (Internal / Engine APIs)

Bu uç noktalar sadece Doping'in iç ağı (VPC / Kubernetes Service Mesh) içerisinden `X-Internal-Service-Key` başlığı veya mTLS ile çağrılabilir.

### 3.1. `POST /internal/pulse/evaluate/{studentId}`
Core video veya test servislerinden bir olay (`activity.completed`) gerçekleştiğinde karar motorunu asenkron tetikler.

* **Headers:** `X-Internal-Service-Key: <internal-secret>`
* **İstek Gövdesi (Request Body):**
```json
{
  "eventSource": "LESSON_SERVICE",
  "eventType": "VIDEO_WATCHED",
  "eventPayload": {
    "lessonId": "geo-trigonometri-01",
    "watchedSeconds": 480,
    "isCompleted": true
  }
}
```
* **Örnek Başarılı Yanıt (HTTP 200 OK):**
```json
{
  "studentId": "c3a1b8d2-45e0-4789-9a21-998877665544",
  "evaluated": true,
  "stateChanged": true,
  "previousState": "ACTIVATE",
  "newState": "STEADY",
  "transitionReason": "FIRST_MEANINGFUL_MILESTONE_ACHIEVED",
  "auditLogId": 1049281
}
```

---

### 3.2. `POST /internal/pulse/batch/nightly-evaluation`
Her gece 03:00'te Kubernetes CronJob tarafından tetiklenen, öğrencilerin inaktivite günlerini hesaplayıp toplu olarak `BOOST` veya `RESTART` durumuna geçiren iş endpoint'i.

* **İstek Gövdesi (Request Body):**
```json
{
  "executionDate": "2026-10-13",
  "chunkSize": 500,
  "dryRun": false
}
```
* **Başarılı Yanıt (HTTP 202 Accepted):**
```json
{
  "jobId": "batch_pulse_nightly_20261013_0300",
  "status": "QUEUED",
  "message": "Gece taraması başlatıldı. İlerleyiş RabbitMQ üzerinden izlenebilir."
}
```

---

### 3.3. `GET /internal/pulse/students/{studentId}/timeline`
Koç paneli (Coaching CRM) veya veli destek uzmanları için öğrencinin son 30 gündeki durum değişikliklerini ve müdahale geçmişini döner.

* **Query Parametreleri:** `limit=20`
* **Örnek Başarılı Yanıt (HTTP 200 OK):**
```json
{
  "studentId": "c3a1b8d2-45e0-4789-9a21-998877665544",
  "currentState": "RESTART",
  "currentSubState": "RESTART_CHECKIN_PENDING",
  "timeline": [
    {
      "date": "2026-10-12T03:00:00Z",
      "fromState": "STEADY",
      "toState": "RESTART",
      "trigger": "CRON_NIGHTLY_INACTIVITY",
      "reason": "SUDDEN_DROP_AFTER_45_MIN_BASELINE"
    },
    {
      "date": "2026-10-01T14:15:00Z",
      "fromState": "ACTIVATE",
      "toState": "STEADY",
      "trigger": "EVENT_BUS_LESSON_WATCHED",
      "reason": "FIRST_MEANINGFUL_MILESTONE_ACHIEVED"
    }
  ]
}
```

