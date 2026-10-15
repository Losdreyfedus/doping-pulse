# 🧭 Doping Pulse: Uçtan Uca Örnek Senaryo, Ece'nin RESTART Yolculuğu

**Doküman Türü:** Örnek senaryo. Kuralları değil, kuralların tek bir öğrenci üzerinde nasıl çalıştığını gösterir.  
**Yazar:** Hamza Kar  
**Kullanılan kurallar:** RFC-001 (state machine, kişisel baseline), RFC-002 (tablolar), RFC-003 (API), RFC-004 (anket ve dondurma), RFC-006 (outbox)

> Bu doküman normatif değildir. Kural ile senaryo arasında bir fark görülürse RFC geçerlidir. Tüm ID'ler ve değerler örnektir.

---

## 1. Kurgu

| | |
| :--- | :--- |
| **Öğrenci** | Ece, YKS öğrencisi, `studentId = c3a1b8d2-45e0-4789-9a21-998877665544` |
| **Normali** | 28 Eylül – 11 Ekim arası her gün ~90 dakika çalışıyor. Durumu `STEADY`. |
| **Olay** | 12 Ekim'de okulda yazılı haftası başlıyor. Ece yazılılarına okul notlarından çalışıyor; 4 gün boyunca Doping'e hiç girmiyor. |
| **Deney grubu** | `TREATMENT` (müdahale alıyor) |

---

## 2. Zaman Çizelgesi

```mermaid
gantt
    title Ece, 28 Eylül – 28 Ekim 2026
    dateFormat YYYY-MM-DD
    axisFormat %d.%m
    section Ece'nin davranışı
    Her gün ~90 dk         :done, b1, 2026-09-28, 14d
    Yazılı haftası 0 dk    :crit, b2, 2026-10-12, 4d
    Hedefler dondurulmuş   :b3, 2026-10-16, 8d
    4 başarılı gün         :done, b4, 2026-10-24, 4d
    section Pulse durumu
    STEADY                 :s1, 2026-09-28, 16d
    BOOST                  :active, s1b, 2026-10-14, 2d
    RESTART                :crit, s2, 2026-10-16, 12d
    STEADY                 :s3, 2026-10-28, 1d
```

| # | Zaman | Ece ne yaptı? | Motor ne gördü? | Durum → Alt durum | Yazılan kayıt / olay |
| :---: | :--- | :--- | :--- | :--- | :--- |
| 1 | 28 Eyl – 11 Eki | Her gün ~90 dk çalıştı | Günlük özetler birikti | `STEADY` / `STEADY_NORMAL` | `pulse_daily_aggregates`: günde 1 satır |
| 2 | 12 – 13 Eki | Yazılılarına okul notlarından çalıştı, Doping'e girmedi | Ters streak (`consecutive_inactive_days`): 1 → 2 | `STEADY` | Sadece günlük özet satırları (0 dk) |
| 2b | **14 Eki 03:00** | Uyuyor | Ters streak 2 ve seriden önceki 10 günün 10'unda aktif → seri bozulması | `BOOST` / `BOOST_STREAK_BROKEN` | Durum + audit log + outbox, ardından hedef hafifletme kartı |
| 2c | 14 – 15 Eki | Doping'e girmedi | Kartın push'u 09:00 – 09:30'da gitti, açılmadı. 15 Eki 03:00'te ters streak 3 < 4 | değişmedi | Kartın 24 saati doldu → `EXPIRED` |
| 3 | **16 Eki 03:00** | Uyuyor | Ters streak 4 ≥ kişisel eşik 4 | `RESTART` / `RESTART_CHECKIN_PENDING` | Durum + audit log + outbox, ardından check-in kartı |
| 3b | 16 Eki 09:00 – 09:30 | Okulda | Sessiz saatler bitti | değişmedi | Ertelenen push gönderilir: "Birkaç gündür görüşemedik, her şey yolunda mı?" |
| 4 | 16 Eki 12:30 | Bildirime dokunup uygulamayı açtı | Aktif check-in kartı var | değişmedi | `GET /pulse/me/status` |
| 5 | 16 Eki 12:32 | "Okul sınavlarım vardı" seçti | `SCHOOL_EXAMS` → `ACTION_FREEZE_GOALS` | `RESTART` / `RESTART_ADAPTED` | `POST /pulse/checkins`, anket yanıtı, dondurma |
| 6 | 17 – 23 Eki | Sınavlarına çalıştı | Öğrenci `PAUSED`, değerlendirme dışı; ters streak 4'te donar | değişmedi | Bildirim yok, kart yok, veliye mesaj yok |
| 7 | 24 – 27 Eki | Her gün hedefinin ≥ %80'ini tamamladı | `consecutive_successful_days`: 1 → 4 | değişmedi | Günlük özet satırları |
| 8 | **28 Eki 03:00** | Uyuyor | 4 başarılı gün ≥ çıkış eşiği 4 | `STEADY` / `STEADY_NORMAL` | Durum + audit log + outbox |

---

## 3. Adım Adım Veri İzi

### Ön Adım: Seri bozulması (14 Eki 03:00)

Ece 12 ve 13 Ekim'de Doping'e girmedi; ters streak 2. Gece batch'i şunları görür (RFC-001 §4.2, §4.6):

| Kontrol | Değer | Sonuç |
| :--- | :--- | :--- |
| RESTART eşiği | Ters streak 2 < kişisel eşik 4 | Henüz RESTART değil |
| Cooldown | Son durum değişikliği 20 Eylül'de | Cooldown yok |
| Seri bozulması | Seriden önceki 10 günde (2 – 11 Eki) 10 aktif gün (eşik ≥ 6 ✅) + ardışık 2 gün inaktif | **`BOOST` / `BOOST_STREAK_BROKEN`** |

* Durum tablosu (`version` 7 → 8, `state_reason = STREAK_BROKEN_2_DAYS`), audit log ve outbox tek transaction'da yazılır.
* Müdahale Motoru bir hedef hafifletme kartı üretir: *"Bugün yükünü hafiflettik, sadece 15 soru yeter."* Kartın ömrü 24 saattir. Push sessiz saatler yüzünden 09:00 – 09:30 arasında gider.
* Ece kartı açmaz; kartın süresi 15 Eki 03:00'te dolar (`EXPIRED`).
* 15 Eki 03:00'te ters streak 3'tür. Eşik 4'e ulaşılmadığı için değişiklik olmaz.

> **Neden bu ara adım önemli?** Ece'nin ters streak'i BOOST'a geçince sıfırlanmaz; kopuşun başından sayılmaya devam eder. Bu yüzden BOOST'tan RESTART'a geçiş de aynı kişisel eşiği (4) kullanır. Ara bir duruma girmek RESTART'ı geciktirmez.

### Adım 1: Gece batch'i Ece'yi değerlendirir (16 Eki 03:00)

Ece şu an `BOOST` durumunda. Motor önce `PulseEvaluationContext` objesini oluşturur (teknik spec §4.3). Kopuşun başladığı gün 12 Ekim olduğu için baseline penceresi bu günden **önceki** 14 gündür (RFC-001 §4.5):

| Değer | Hesap | Sonuç |
| :--- | :--- | :--- |
| `consecutive_inactive_days` (ters streak) | 12, 13, 14, 15 Ekim | **4** |
| Baseline penceresi | 12 Ekim − 14 gün … 12 Ekim − 1 gün | 28 Eyl – 11 Eki |
| `activeDays` | Penceredeki aktif gün sayısı | **14** (eşik ≥ 4 ✅) |
| `avgMinutes` | toplam izleme dakikası ÷ 14 | **90** (eşik ≥ 10 ✅) |
| `maxGapDays` | Penceredeki iki aktif gün arasındaki en uzun inaktif seri | **0** |
| `restartThresholdDays` | `clamp(0 + 2, 4, 7)` | **4** |
| Karar | 4 ≥ 4 ve alışkanlık var | **RESTART**. BOOST'a geçişin 48 saatlik cooldown'u RESTART'ı engellemez. |

> **Neden sabitlenmiş pencere?** Pencere "son 14 gün" olsaydı içinde 4 tane 0 dakikalık gün olurdu ve ortalama 90'dan 64'e düşerdi. Ece kaybolmaya devam ettikçe ortalaması düşer ve en kritik öğrenci alarmdan çıkardı. Kopuş öncesine sabitlenen pencere, Ece'nin *normalini* ölçmeye devam eder.

### Adım 2: Tek transaction, üç kayıt

**`pulse_student_states`**: Ece'nin satırı optimistic locking ile güncellenir (`WHERE version = 8`):

| Kolon | Önce | Sonra |
| :--- | :--- | :--- |
| `current_state` | `BOOST` | `RESTART` |
| `current_sub_state` | `BOOST_STREAK_BROKEN` | `RESTART_CHECKIN_PENDING` |
| `previous_state` | `STEADY` | `BOOST` |
| `state_reason` | `STREAK_BROKEN_2_DAYS` | `INACTIVE_4D_PERSONAL_THRESHOLD_4D` |
| `state_updated_at` | 2026-10-14 03:00 | 2026-10-16 03:00 |
| `consecutive_inactive_days` | 3 | 4 (gece batch'i 15 Ekim'i ekledi) |
| `restart_threshold_days` | `NULL` | **4** (eskalasyon günleri buna göre sayılır) |
| `version` | 8 | **9** |

**`pulse_state_audit_logs`**: "Ece neden RESTART'a düştü?" sorusunun cevabı burada:

```json
{
  "student_id": "c3a1b8d2-45e0-4789-9a21-998877665544",
  "from_state": "BOOST", "from_sub_state": "BOOST_STREAK_BROKEN",
  "to_state": "RESTART", "to_sub_state": "RESTART_CHECKIN_PENDING",
  "trigger_source": "CRON_NIGHTLY_INACTIVITY",
  "transition_reason": "INACTIVE_4D_PERSONAL_THRESHOLD_4D",
  "metrics_snapshot": {
    "consecutiveInactiveDays": 4,
    "anchoredBaseline": {
      "windowStart": "2026-09-28", "windowEnd": "2026-10-11",
      "activeDays": 14, "avgMinutes": 90, "maxGapDays": 0, "restartThresholdDays": 4
    }
  }
}
```

**`pulse_outbox`**: Olay Kafka'ya doğrudan değil, aynı transaction içinde outbox'a yazılır. Outbox Relay onu `pulse.student.state_changed` topic'ine yayınlar (RFC-006 §6):

```json
{
  "eventId": "0199ea1c-2e8b-7c01-8a44-1f3e9d0b2c77",
  "eventType": "pulse.student.state_changed",
  "schemaVersion": "1.0.0",
  "timestamp": "2026-10-16T03:00:12+03:00",
  "aggregateId": "c3a1b8d2-45e0-4789-9a21-998877665544",
  "payload": {
    "studentId": "c3a1b8d2-45e0-4789-9a21-998877665544",
    "previousState": "BOOST", "newState": "RESTART",
    "previousSubState": "BOOST_STREAK_BROKEN", "newSubState": "RESTART_CHECKIN_PENDING",
    "transitionReason": "INACTIVE_4D_PERSONAL_THRESHOLD_4D",
    "assignedInterventionId": null,
    "cooldownUntil": "2026-10-18T03:00:12+03:00"
  }
}
```

> Üç kayıt ya birlikte yazılır ya hiçbiri yazılmaz. "Ece RESTART oldu ama kimse haberdar olmadı" ya da "veliye hiç gerçekleşmemiş bir durum için SMS gitti" durumları oluşamaz.

### Adım 3: Müdahale Motoru check-in kartını üretir

Müdahale Motoru olayı tüketir. Ece `TREATMENT` grubunda olduğu için kart üretilir; `HOLDOUT` grubunda olsaydı bu adım atlanırdı (RFC-004 §1).

**`pulse_student_interventions`**:

| Kolon | Değer |
| :--- | :--- |
| `intervention_id` | `0199ea1c-2f40-7a3e-9c51-7d2b8e4f6a10` (UUIDv7) |
| `template_id` | `RESTART_CHECKIN_CORE_V1` |
| `pulse_state` | `RESTART` |
| `status` | `ASSIGNED` |
| `assigned_at` | 2026-10-16 03:00 |
| `expires_at` | 2026-10-18 03:00 (**48 saat**, check-in kartı istisnası, RFC-004 §5.1) |

`pulse_student_states.active_intervention_id` bu karta işaret eder. 14 Ekim'deki BOOST kartının süresi zaten dolmuştu. Dolmamış olsaydı bile check-in kartı onu `EXPIRED` yapıp yerine geçerdi; "günde 1 aktif kart" kuralının tek istisnası budur (RFC-004 §5.1).

**Push bildirimi:** Kart saat 03:00'te üretildiği için sessiz saatlere (21:00 – 09:00) denk gelir. Kart uygulamada hemen görünür, ama push gece gönderilmez; 09:00 – 09:30 arasında rastgele bir dakikada gider (RFC-004 §5.1). Ece o saatten önce uygulamayı açıp anketi yanıtlamış olsaydı push iptal edilirdi.

### Adım 4: Ece uygulamayı açar (16 Eki 12:30)

`GET /pulse/me/status`. Öğrenci kimliği JWT'den okunur; sorgu PK lookup ve kısmi indeksle cevaplanır (hedef P95 ≤ 30 ms):

```json
{
  "studentId": "c3a1b8d2-45e0-4789-9a21-998877665544",
  "pulseState": "RESTART",
  "pulseSubState": "RESTART_CHECKIN_PENDING",
  "consecutiveInactiveDays": 4,
  "streakDays": 0,
  "isPaused": false,
  "activeIntervention": {
    "interventionId": "0199ea1c-2f40-7a3e-9c51-7d2b8e4f6a10",
    "templateId": "RESTART_CHECKIN_CORE_V1",
    "type": "CHECK_IN_SURVEY",
    "badgeText": "Hızlı Check-in",
    "title": "Birkaç gündür görüşemedik, her şey yolunda mı?",
    "description": "Ne olduğunu söylersen programını sana göre ayarlayalım.",
    "cta": { "text": "Anlat (10 sn)", "actionType": "MODAL_SURVEY", "payload": "RESTART_CORE_V1" },
    "dismissible": true,
    "expiresAt": "2026-10-18T03:00:12+03:00"
  }
}
```

Kart görüntülendiğinde mobil `IMPRESSION` olayı gönderir ve kart `VIEWED` olur. Ece butona basınca mobil `GET /pulse/checkins/RESTART_CORE_V1` ile 5 seçeneği çeker (RFC-003 §2.3).

### Adım 5: Ece anketi yanıtlar (16 Eki 12:32)

`POST /pulse/checkins`:

```http
Authorization: Bearer <jwt>
Idempotency-Key: 5f1c7c2e-8a3b-4d9e-b1f0-2c6a9e7d4b31
```
```json
{
  "interventionId": "0199ea1c-2f40-7a3e-9c51-7d2b8e4f6a10",
  "surveyId": "RESTART_CORE_V1",
  "selectedOptionKey": "SCHOOL_EXAMS",
  "freeTextNote": "Fizik ve kimya yazılılarım vardı."
}
```

Sunucu eylemi seçenekten türetir: `SCHOOL_EXAMS` → `ACTION_FREEZE_GOALS(days: 7)` (RFC-004 §4). Yanıt:

```json
{
  "success": true,
  "feedbackMessage": "Sınav haftanı biliyoruz! Doping hedeflerini 7 gün dondurduk, sınavlarına odaklan.",
  "newPulseState": "RESTART",
  "newPulseSubState": "RESTART_ADAPTED",
  "appliedAdaptation": {
    "actionType": "ACTION_FREEZE_GOALS",
    "dailyTargetQuestions": 0,
    "dailyTargetDurationMinutes": 0,
    "validUntil": "2026-10-23T12:32:00+03:00"
  }
}
```

Aynı transaction'da yazılanlar:

| Tablo | Değişiklik |
| :--- | :--- |
| `pulse_checkin_responses` | Yeni satır: `selected_option_key = SCHOOL_EXAMS`, `triggered_adaptation = ACTION_FREEZE_GOALS` |
| `pulse_daily_aggregates` | 16 Ekim satırı `was_paused = true`. Dondurma 12:32'de başladığı için bu gün ters streak'i artırmaz; 17–23 Ekim satırları da gece batch'inde aynı şekilde işaretlenir. |
| `pulse_student_interventions` | Check-in kartı `COMPLETED` |
| `pulse_student_states` | `current_sub_state = RESTART_ADAPTED`, `is_paused = true`, `paused_until = 23 Eki 12:32`, `adaptation_expires_at = 28 Eki 12:32` (dondurma bitişi + 5 gün toparlanma penceresi), `active_intervention_id = null`, `version = 10` |

> **Çift tıklama:** Ece butona iki kez basarsa ikinci istek aynı `Idempotency-Key` ile gelir ve sunucu ilk yanıtı aynen döndürür. Hedefler iki kez dondurulmaz, ikinci bir anket satırı oluşmaz.

### Adım 6: Dondurma haftası (17 – 23 Eki)

Her gece motor Ece'yi değerlendirir, ama ilk kural `is_paused AND paused_until > now` olduğu için hiçbir şey değişmez (RFC-001 §6, Kural 0). Kart üretilmez, bildirim gitmez. Ece sınavlarına odaklanır.

Ters streak da dondurma süresince **4'te donar** (RFC-001 §4.6). Dondurma olmasaydı 19 Ekim'de 7'ye ulaşacak ve izin almış olan Ece'nin velisine "7 gündür çalışmıyor" mesajı gidecekti.

### Adım 7: Dönüş ve STEADY (24 – 28 Eki)

Dondurma 23 Ekim'de biter, hedefler normale döner. Ece 24–27 Ekim arasında her gün hedefinin ≥ %80'ini tamamlar.

| Nightly batch | `consecutive_successful_days` | Toparlanma penceresi (28 Eki 12:32'ye kadar) | Karar |
| :--- | :---: | :--- | :--- |
| 25 Eki 03:00 | 1 | açık | değişiklik yok |
| 26 Eki 03:00 | 2 | açık | değişiklik yok |
| 27 Eki 03:00 | 3 | açık | değişiklik yok |
| **28 Eki 03:00** | **4** | açık | **`STEADY`** (`SUSTAINED_4_SUCCESSFUL_DAYS`) |

Ters streak dondurma boyunca 4'te kalır; dondurmanın bittiği 23 Ekim de yarısı dondurmada geçtiği için sayılmaz. Ece 24 Ekim'de çalışınca ters streak 25 Ekim 03:00'teki batch'te 0'a döner. 28 Ekim'de durum tablosu (`restart_threshold_days` yeniden `NULL` olur), audit log ve outbox yine tek transaction'da yazılır. Ece artık müdahale görmez, sadece streak sayacını görür.

---

## 4. Ya Başka Türlü Olsaydı?

| Senaryo | Ne değişir? | Sonuç |
| :--- | :--- | :--- |
| **Ece ankete hiç yanıt vermeseydi** | Dondurma olmaz, ters streak saymaya devam eder. 18 Eki 03:00'te kartın 48 saati dolar (ters streak 6 = E + 2); saatlik batch durumu `RESTART_UNRESPONSIVE` yapar. | Koçun paneline "Kopuş Riski" görevi düşer (Tier 2). 19 Ekim'de ters streak E + 3 = 7 olur ve veliye suçlamayan bir bilgilendirme gider (Tier 3). 26 Ekim'de E + 10 = 14'te çağrı merkezi arar (Tier 4). Ece arada bir gün bile çalışırsa merdiven sıfırlanır (RFC-001 §4.6). |
| **"Çok yoruldum" (`BURNOUT`) seçseydi** | Dondurma yerine hedefler 5 gün boyunca %60 hafifletilir. | `BOOST` / `BOOST_BURNOUT_RISK`. 3 başarılı günden sonra `STEADY`'ye döner. |
| **Dönüşte 4 gün tutturamasaydı** | 28 Eki 12:32'de toparlanma penceresi kapanır. | Sonraki nightly batch'te `BOOST`. Ece cezalandırılmaz, hafifletilmiş programla devam eder. |
| **Ece `HOLDOUT` grubunda olsaydı** | Durum geçişleri aynen olur (BOOST ve RESTART hesaplanır, audit log yazılır), ama Ön Adım'daki hedef hafifletme kartı ve Adım 3'teki check-in kartı üretilmez. | Kart, bildirim ve dondurma olmaz. `GET /pulse/me/status`, `activeIntervention: null` döner. Bu fark, RFC-007 §5'teki uplift ölçümünün temelidir. |
| **Ece hafta sonu çalışan Mert olsaydı** | Baseline: 14 günde 4 aktif gün (iki hafta sonu), ort. ~17 dk/gün, `maxGapDays = 5` (pazartesi–cuma). | Kişisel eşik `clamp(5 + 2, 4, 7) = 7` gün. Hafta içindeki 5 günlük normal boşlukta alarm çalmaz; 7. sessiz günde, yani bir hafta sonunu da kaçırdığında RESTART olur. Seriden önceki 10 günde 6 aktif günü olmadığı için 2. günde seri bozulması da tetiklenmez. Eskalasyon da göreli sayılır: koç 9., veli 10., çağrı merkezi 17. günde. |
| **Ece yazılılarına Doping'den çalışsaydı** | Ece 4 gün boyunca günde ~2 saat okul konularına çalışır. Aktif gün içerikten bağımsız olduğu için `consecutive_inactive_days = 0`; YKS planındaki görevleri yapmadığı için WCR %50'nin altına düşer, ama tempo oranı (3 × 90 + 4 × 120) ÷ (90 × 7) ≈ 1,2 olur (normalinden fazla çalışıyor). | **RESTART da BOOST da tetiklenmez**, Ece `STEADY` kalır. BOOST'a girmek için WCR < %50 **ve** tempo oranı < %70 gerekir (RFC-001 §4.2). Böylece en yoğun çalıştığı haftada Ece'ye "temposu düştü" muamelesi yapılmaz. |

---

## 5. Modellemenin Netleştirdiği Kurallar

Bu senaryo gün gün işletilirken altı kural boşluğu bulundu ve ilgili RFC'lerde düzeltildi:

| Bulgu | Düzeltme |
| :--- | :--- |
| `SCHOOL_EXAMS` dondurması bittiği gece öğrencinin 4 başarılı günü olamayacağı için, dondurulan her öğrenci otomatik `BOOST`'a düşüyordu. | `adaptation_expires_at` = adaptasyon bitişi + 5 günlük toparlanma penceresi (RFC-001 §3, RFC-004 §4). |
| Kartların ömrü 24 saatti, ama `RESTART_UNRESPONSIVE` 48 saat bekliyordu. Kart, öğrenci "yanıtsız" sayılmadan önce siliniyordu. | `CHECK_IN_SURVEY` kartının TTL'i 48 saat (RFC-004 §5.1). |
| BOOST kuralı yalnızca plandaki görevlerin tamamlanma oranına (WCR) bakıyordu. Yazılılarına Doping'den okul konularıyla çalışan bir öğrencinin WCR'ı düşüyor ve sistem onu, en yoğun çalıştığı haftada "temposu düştü" diye BOOST'a alıyordu. | BOOST girişine ikinci koşul: son 7 günün çalışma süresi öğrencinin normalinin %70'inin altında olmalı (RFC-001 §4.2). "Plana uymuyor" ile "çalışmıyor" artık ayrı durumlar. |
| BOOST'tan RESTART'a geçiş eşiği en az 5 gündü. Her gün çalışan bir öğrenci 2. günde seri bozulması yüzünden BOOST'a girdiği için RESTART'a 4. değil 5. günde geçiyordu; ara bir durum RESTART'ı geciktiriyordu. | Ters streak durumdan bağımsız sayılır; BOOST'tan RESTART'a geçiş de STEADY ile aynı kişisel eşiği kullanır (RFC-001 §4.6). |
| Veli ve çağrı merkezi eskalasyonu sabit günlerdeydi (7. ve 14. gün). Kişisel eşiği 7 olan öğrencinin velisine, öğrenci RESTART'a düştüğü gün mesaj gidiyordu. | Eskalasyon günleri kişisel eşiğe göre sayılır: veli E + 3, çağrı merkezi E + 10 (RFC-005 §2). |
| Ters streak dondurma süresince de artıyordu. Check-in'de "sınavlarım var" diyerek izin alan bir öğrencinin velisine, dondurmanın ortasında "7 gündür çalışmıyor" mesajı gidiyordu. | Dondurmada ters streak durur ve eskalasyon çalışmaz (RFC-001 §4.6, RFC-005 §2). |
