# 🧠 Doping Pulse: State Machine ve Geçiş Kuralları Spesifikasyonu

**Doküman Kodu:** `RFC-PULSE-001`  
**Modül:** `Pulse State & Decision Engine`  
**Yazar:** Hamza Kar  
**Hedef:** Öğrenci durum sınıflandırmasının deterministik, anti-flapping korumalı ve esnek kural motoruyla modellenmesi.

---

## 1. Mimari Prensipler ve Problem Tanımı

Gerçek dünya EdTech sistemlerinde öğrenci davranışlarını izlerken yapılan en büyük backend hatası, basit `if-else` mantığıyla anlık durum değiştirmektir. 
Örneğin; dün çalışmayan öğrenciyi hemen "Kopmuş (ReStart)" ilan edip, ertesi gün 1 soru çözdü diye "Her şey yolunda (Steady)" durumuna almak **flapping** sorununa yol açar. Bu da kullanıcıya sürekli tutarsız bildirimler gitmesine ve sistemin güvenilirliğini yitirmesine neden olur.

Pulse Karar Motoru şu 4 temel prensiple çalışır:
1. **Sliding Window:** Durumlar anlık tek bir olaya göre değil; 3, 7 ve 14 günlük hareketli ortalamalara ve trendlere göre değerlendirilir.
2. **Histerezis & Cooldown:** Bir durum değiştikten sonra sistem en az N gün cooldown periyoduna girer; küçük dalgalanmalar durumu anında bozamaz.
3. **Kişisel Baseline:** Her öğrencinin "normali" farklıdır. Haftada 2 saat çalışan bir LGS öğrencisi ile haftada 25 saat çalışan bir YKS derece öğrencisi aynı mutlak saat eşiğiyle değerlendirilemez.
4. **Kural Önceliği:** Çakışan kurallarda en kritik durum (örn: ReStart > Boost) deterministik bir öncelik sırasıyla işletilir.

---

## 2. Durumlar ve Alt Durumlar

Sistem 4 ana durum ve operasyonel derinlik için tanımlanmış alt durumlardan oluşur:

```
[ACTIVATE]  --> Henüz çalışma alışkanlığı başlamamış
   ├── ACTIVATE_ONBOARDING (0 - 2. gün: Yeni kayıt, ilk keşif)
   ├── ACTIVATE_STALLED    (3 - 6. gün: Sisteme giriyor ama başlamıyor)
   └── ACTIVATE_CRITICAL   (7+ gün: Tamamen hareketsiz, alarm seviyesi)

[BOOST]     --> Çalışıyor ama ritim/tempo yetersiz veya kırılgan
   ├── BOOST_STREAK_BROKEN (Seri bozulması: Düzenli giderken 2 gün aksatan)
   ├── BOOST_LOW_PACE      (Tempo düşüklüğü: Hedefinin <%50'sinde kalan)
   └── BOOST_BURNOUT_RISK  (Aşırı yük: Hedefleri sürekli yarım bırakan)

[RESTART]   --> Daha önce aktifken belirgin ve anlamlı kopuş yaşayan
   ├── RESTART_CHECKIN_PENDING (Öğrenciye anket açıldı, yanıt bekleniyor)
   ├── RESTART_ADAPTED         (Anket yanıtlandı, hafifletilmiş programa geçildi)
   └── RESTART_UNRESPONSIVE    (Ankete 48 saat yanıt vermedi, eskalasyon adımı)

[STEADY]    --> Ritim yakalamış, hedef temposunda ilerleyen (Müdahalesiz)
   ├── STEADY_NORMAL   (Ritimde ve hedefinde)
   └── STEADY_STREAK_HERO (Yüksek performans, seri yakalamış)
```

---

## 3. Durum Geçiş Matrisi

Aşağıdaki tablo, durumlar arası geçiş koşullarını, tetikleyici mekanizmasını ve koruma kurallarını tanımlar:

| Kaynak Durum | Hedef Durum | Tetikleyici | Ön Koşullar | Cooldown / Gecikme |
| :--- | :--- | :--- | :--- | :--- |
| **[*]** | **ACTIVATE** | Event: `StudentEnrolledEvent` | Paket aktif, baseline aktivite yok | Yok (Başlangıç) |
| **ACTIVATE** | **STEADY** | Event: `MilestoneCompletedEvent` | **"Aha! moment"** (bkz. §4.1): Bir videonun ≥ %80'i **ve** en az 180 sn'si izlendi **VEYA** kümülatif ≥ 5 soru çözüldü | Anında |
| **ACTIVATE** | **ACTIVATE_CRITICAL** | Nightly batch | Kayıt üzerinden ≥ 7 gün geçti AND toplam aktivite süresi = 0 | 24 saat |
| **STEADY** | **BOOST** | Nightly batch | Son 7 günlük hedef tamamlama (WCR) < %50 **VEYA** seri bozulması (önceki 10 günde ≥ 6 aktif gün varken ardışık 2 gün inaktif) | 48 saat Cooldown |
| **STEADY** | **RESTART** | Nightly batch | **Kişisel baseline** (bkz. §4.5): Kopuş öncesi 14 günde ≥ 4 aktif gün **AND** ortalama ≥ 10 dk/gün **AND** ardışık inaktif gün ≥ `restart_threshold_days` | Anında (Cooldown'u deler) |
| **BOOST** | **STEADY** | Batch / Event | Ardışık 3 "başarılı gün" (o günün *efektif/hafifletilmiş* hedefinin ≥ %80'i tamamlandı) | 72 saat stabilite |
| **BOOST** | **RESTART** | Nightly batch | Ardışık inaktif gün ≥ max(5, `restart_threshold_days`) | Anında (Cooldown'u deler) |
| **RESTART** | **BOOST** | Event: `CheckInSubmittedEvent` | Anket yanıtı `BURNOUT` (hedefler 5 gün %60 hafifletilir, `BOOST_BURNOUT_RISK`) | Anında |
| **RESTART** | **ACTIVATE** | Event: `CheckInSubmittedEvent` | Anket yanıtı `OTHER` (temiz sayfa, `ACTIVATE_ONBOARDING`) | Anında |
| **RESTART_CHECKIN_PENDING** | **RESTART_ADAPTED** | Event: `CheckInSubmittedEvent` | Anket yanıtı `SCHOOL_EXAMS`, `SUBJECT_DIFFICULTY` veya `TECHNICAL_ISSUE` (eylemler: RFC-004 §4) | Anında |
| **RESTART_CHECKIN_PENDING** | **RESTART_UNRESPONSIVE** | Saatlik batch | Check-in kartı 48 saat yanıtsız | 48 saat |
| **RESTART** | **STEADY** | Event / Batch | Ardışık 4 "başarılı gün" | 4 gün stabilite |
| **RESTART_ADAPTED** | **BOOST** | Nightly batch | Adaptasyon süresi (`paused_until` / eylem süresi) doldu ve 4 başarılı gün koşulu sağlanmadı | Adaptasyon bitimi |

---

## 4. Durum Bazlı Detaylı Kural Mantığı

### 4.1. 🟢 ACTIVATE Motoru
* **Problem:** Öğrenci satın aldı ancak platform çok karmaşık veya ilk adım çok büyük geliyor.
* **Tetikleyiciler:**
  * `Day 0-2 (Onboarding):` Hoş geldin mikro adımı: "Sadece 3 dakikalık 'Nasıl Çalışmalıyım?' videosunu izle."
  * `Day 3-6 (Stalled):` Uygulamayı açıyor ama derse girmiyor: "Bugün 1 ders seç ve sadece ilk 5 dakikasına bak."
  * `Day 7+ (Critical):` Hiç login olmadı: Push Notification + Velisine bilgilendirme hazırlığı.
* **Çıkış Kapısı:** Öğrenci sisteme ilk anlamlı veriyi bıraktığı an, anında `STEADY` durumuna yükseltilir. Kanonik tanım (tüm RFC'ler bu tanıma referans verir):
  ```
  is_aha_moment = (lesson.completion_percentage >= 80 AND lesson.watched_seconds >= 180)
                  OR (lifetime_questions_solved >= 5)
  ```
  180 sn alt sınırı, 1 dakikalık tanıtım videosunun %80'ini izlemenin "anlamlı aktivite" sayılmasını engeller.

### 4.2. 🟡 BOOST Motoru
* **Problem:** Kopmadı ama bocalıyor. Hedefler birikiyor, öğrenci suçluluk hissediyor ve kaçma eğilimi gösteriyor.
* **Metrikler:**
  * **Haftalık Yük Tamamlama Oranı (WCR):**
    ```text
    WCR = son 7 günde tamamlanan görev sayısı ÷ son 7 günde hedeflenen görev sayısı
    ```
    WCR < 0.50 ise → `BOOST_LOW_PACE`.
  * **Seri Bozulması:** Öğrencinin geçmiş 10 günde en az 6 gün çalışması varken, ardışık 2 gün hiç girmemesi.
* **Müdahale Stratejisi:**
  * Asla *"Çok geridesin, daha çok çalışmalısın!"* denmez.
  * **Hedef Hafifletme:** Günlük soru hedefi varsayılan olarak %40 kırpılır (`factor: 0.6`; şablon `config_payload`'ından %30 – %50 aralığında ayarlanabilir). Öğrenciye *"Bugün sadece 15 soru çözmen yeterli"* hissi verilir. `BURNOUT` anket yanıtında bu oran %60'tır (`factor: 0.4`, bkz. RFC-004 §4).
  * **Efektif Hedef Kuralı:** WCR ve "başarılı gün" hesapları her zaman o günün **efektif (hafifletilmiş)** hedefine göre yapılır. Aksi halde hafifletilmiş hedefi tamamlayan öğrenci orijinal hedefe göre başarısız görünür ve `BOOST`'tan hiç çıkamaz.
* **Çıkış Kapısı:** Ardışık 3 gün boyunca bu hafifletilmiş hedeflerin en az %80'ini tamamladığında ritim oturmuş kabul edilip `STEADY` durumuna geçirilir.

### 4.3. 🔴 RESTART Motoru
* **Problem:** Düzenli çalışan öğrenci birden kayboldu. Bu öğrenciye sıradan bir "Hadi çalış" bildirimi atmak ters teper; çünkü arkasında bir hayat olayı vardır (Hastalık, okul sınav haftası, burnout veya zor bir konuya takılma).
* **Tespit Koşulu:** Öğrencinin **kendi normaline göre** anlamlı bir kopuş yaşaması gerekir. Mutlak bir dakika eşiği kullanılmaz; hesaplama §4.5'teki Kişisel Baseline algoritmasıyla yapılır:
  * Öğrencinin kopuş öncesinde gerçek bir çalışma alışkanlığı olmalıdır (baseline penceresinde ≥ 4 aktif gün, ortalama ≥ 10 dk/gün).
  * Ardışık inaktif gün sayısı öğrencinin kişisel eşiğini (`restart_threshold_days`, 4–7 gün arası) aşmalıdır.
* **Proaktif Müdahale: Check-In Döngüsü:**
  1. Sisteme girdiğinde ana ekran kilitlenmez ancak en üstte şefkatli bir mini anket çıkar:  
     *"Birkaç gündür görüşemedik, her şey yolunda mı?"*
  2. Anket 2 sorudan oluşur: **(1)** kök neden seçimi (zorunlu, tek seçim), **(2)** serbest not (opsiyonel, max 500 karakter). Seçenekler ve tetiklenen eylemler **kanonik olarak RFC-004 §4 Survey-to-Action Matrix**'te tanımlıdır; özet:
     - `SCHOOL_EXAMS` → 7 gün hedef dondurma → `RESTART_ADAPTED`
     - `SUBJECT_DIFFICULTY` → ön koşul hap video + 3 kolay soru → `RESTART_ADAPTED`
     - `BURNOUT` → hedefler 5 gün %60 hafifletilir → `BOOST` (`BOOST_BURNOUT_RISK`)
     - `TECHNICAL_ISSUE` → yüksek öncelikli destek bileti, 3 gün değerlendirme dışı → `RESTART_ADAPTED`
     - `OTHER` → temiz sayfa → `ACTIVATE` (`ACTIVATE_ONBOARDING`)

     Eylem, öğrencinin seçtiği nedenden sunucu tarafında türetilir; istemci ayrıca bir eylem talep etmez.
  3. **Yanıtsızlık Durumu:** Check-in kartı 48 saat boyunca yanıtlanmazsa, durum `RESTART_UNRESPONSIVE` olur ve veli/koç panelinde risk bayrağı (`CHURN_RISK_HIGH`) kalkar.

### 4.4. ⚪ STEADY Motoru
* Öğrencinin haftalık hedeflerinin ≥ %70'ini tutturduğu, çalışma aralıklarının 48 saati aşmadığı sağlıklı çalışma durumudur.
* **Önemli Kural:** Bu durumdaki öğrenciye "müdahale kartı" gösterilmez. Sadece tebrik, seviye atlama ve streak sayaçları gösterilir.

### 4.5. 📐 Kişisel Baseline Hesaplaması

**Problem (Pencere Seyrelmesi):** "Son 14 günün ortalaması ≥ X AND son 4 gün sıfır" kuralı kendi kendini sabote eder. Ortalama penceresi inaktif günleri de içerdiği için öğrenci kayboldukça ortalaması düşer. Böylece en uzun süre kaybolan, yani en kritik öğrenci RESTART koşulundan *çıkar*. Ayrıca mutlak bir dakika eşiği, haftada 2–3 gün çalışan düşük tempolu (örn. LGS) öğrenciyi hiçbir zaman RESTART'a düşürmez.

**Çözüm: Kopuş Öncesine Sabitlenmiş Baseline Penceresi**

```
Zaman ekseni (gün):

   ◄────── Baseline Penceresi (14 gün) ──────►◄── İnaktif Seri ──►
   [d-14-k ...........................d-1-k ][ d-k ......... d-1 ][ bugün ]
                                              ▲
                                     streak_start (son aktif günün ertesi)
```

* `k = consecutive_inactive_days`. Baseline, inaktif seri başlamadan **önceki** 14 günden hesaplanır. Seri uzadıkça pencere geriye kayar ama ölçtüğü şey değişmez: öğrencinin *kopmadan önceki* normali.
* **Aktif gün:** `watch_seconds >= 300 OR questions_solved >= 5` olan gün (uygulamayı açıp kapatmak aktif gün sayılmaz).

**Hesaplanan Değerler:**

| Değer | Formül | Amacı |
| :--- | :--- | :--- |
| `baseline_active_days` | Penceredeki aktif gün sayısı | Gerçek bir alışkanlık var mı? (Eşik: ≥ 4) |
| `baseline_avg_minutes` | toplam izleme dakikası ÷ 14 | Gürültü filtresi (Eşik: ≥ 10 dk/gün) |
| `baseline_max_gap` | Penceredeki iki aktif gün arasındaki en uzun inaktif seri | Öğrencinin *normal* boşluk uzunluğu |
| `restart_threshold_days` | `clamp(baseline_max_gap + 2, 4, 7)` | Kişisel kopuş eşiği |

**Örnekler:**

| Profil | Baseline Davranışı | `max_gap` | Kişisel Eşik | Sonuç |
| :--- | :--- | :---: | :---: | :--- |
| YKS derece öğrencisi | Her gün ~90 dk | 0 | **4 gün** | 4. sessiz günde RESTART |
| Düzenli LGS öğrencisi | Gün aşırı ~30 dk | 1 | **4 gün** | 4. sessiz günde RESTART |
| Hafta sonu çalışanı | Haftada 2 gün ~60 dk (ort. ~17 dk/gün) | 4 | **6 gün** | Normal 4–5 günlük boşluğunda alarm *çalmaz*, 6. günde RESTART |
| Seyrek kullanıcı | 14 günde 2 aktif gün | — | — | Alışkanlık yok → RESTART değil; BOOST/WCR kurallarıyla izlenir |

**Performans Notu:** Hesap, `pulse_daily_aggregates` üzerinde öğrenci başına tek bir indeksli aralık sorgusudur (en fazla 14 + 7 = 21 satır). `idx_daily_aggregates_window` indeksiyle karşılanır (RFC-002 §3.2).

---

## 5. Anti-Flapping Algoritması

Durum motoru, öğrencinin geçişlerini denetlerken aşağıdaki **Sliding Window Buffer** algoritmasını işletir:

```
                  ┌─────────────────────────────────────┐
                  │ Sinyal Alındı (Event veya Gece İşi) │
                  └──────────────────┬──────────────────┘
                                     │
                                     ▼
                ┌─────────────────────────────────────────┐
                │ Acil Durum İstisnası:                   │
                │ Hedef durum RESTART mı?                 │
                └────────────────────┬────────────────────┘
                     ┌───────────────┴───────────────┐
                     │ EVET (Cooldown delinir)       │ HAYIR
                     │                               ▼
                     │       ┌─────────────────────────────────────────┐
                     │       │ Cooldown Kontrolü:                      │
                     │       │ Son durum değişikliğinden bu yana       │
                     │       │ en az 48 saat geçti mi?                 │
                     │       └────────────────────┬────────────────────┘
                     │            ┌───────────────┴───────────────┐
                     │            │ EVET                          │ HAYIR
                     ▼            ▼                               ▼
       ┌───────────────────────────┐   ┌───────────────────────────┐
       │ Hedef durum mevcut durum- │   │ Durum Değiştirme YASAK    │
       │ la aynı mı?               │   │ (Sadece metrik güncelle)  │
       └─────────────┬─────────────┘   └───────────────────────────┘
                     │ FARKLI (AYNI ise no-op: tekrar event/kart üretilmez)
                     ▼
       ┌───────────────────────────┐
       │ Öncelik Değerlendirmesi:  │
       │ 1. RESTART (En yüksek)    │
       │ 2. BOOST                  │
       │ 3. STEADY / ACTIVATE      │
       └─────────────┬─────────────┘
                     │
                     ▼
       ┌───────────────────────────┐
       │ Yeni Durumu Kaydet &      │
       │ Audit Log Oluştur         │
       └───────────────────────────┘
```

---

## 6. Kural Motoru Değerlendirme Algoritması

Aşağıdaki mantık, her öğrenci değerlendirilirken çalışan deterministik fonksiyonu gösterir:

```python
# Kanonik parametreler (tek kaynak: bu RFC). Değerler config servisinden okunur.
AHA_MIN_COMPLETION_PCT     = 80
AHA_MIN_WATCH_SECONDS      = 180
AHA_MIN_QUESTIONS          = 5
COOLDOWN_HOURS             = 48
BOOST_ENTRY_WCR            = 0.50
BOOST_EXIT_SUCCESS_DAYS    = 3
RESTART_EXIT_SUCCESS_DAYS  = 4
BASELINE_MIN_ACTIVE_DAYS   = 4
BASELINE_MIN_AVG_MINUTES   = 10.0

def evaluate_student_pulse(student_id: UUID, ctx: PulseEvaluationContext) -> PulseEvaluationResult:
    state, sub_state = ctx.current_state, ctx.current_sub_state
    now = current_timestamp()

    # 0. Değerlendirme dışı (PAUSED): Sınav dondurma, teknik sorun, öğrenci isteği
    if ctx.is_paused and ctx.paused_until > now:
        return PulseEvaluationResult.no_change()

    is_in_cooldown = (now - ctx.state_updated_at) < timedelta(hours=COOLDOWN_HOURS)
    inactive_days  = ctx.consecutive_inactive_days
    wcr_7d         = ctx.last_7d_effective_completion_rate   # Efektif (hafifletilmiş) hedefe göre
    success_days   = ctx.consecutive_successful_days         # Efektif hedefin >= %80'i tamamlanan ardışık gün

    # -------------------------------------------------------------
    # KURAL 1: ACTIVATE Kontrolü (Aha! Moment, bkz. §4.1)
    # -------------------------------------------------------------
    if state == PulseState.ACTIVATE:
        is_aha_moment = (
            (ctx.best_lesson_completion_pct >= AHA_MIN_COMPLETION_PCT
                and ctx.best_lesson_watched_seconds >= AHA_MIN_WATCH_SECONDS)
            or ctx.lifetime_questions_solved >= AHA_MIN_QUESTIONS
        )
        if is_aha_moment:
            return transition(PulseState.STEADY, "STEADY_NORMAL", "FIRST_MEANINGFUL_MILESTONE_ACHIEVED")
        if ctx.days_since_enrollment >= 7 and ctx.lifetime_active_seconds == 0:
            return transition(PulseState.ACTIVATE, "ACTIVATE_CRITICAL", "ZERO_ACTIVITY_FIRST_WEEK")
        return PulseEvaluationResult.no_change()

    # -------------------------------------------------------------
    # KURAL 2: RESTART Girişi (Öncelik: 1, cooldown'u deler)
    # Baseline kopuş öncesine sabitlenmiş pencereden gelir (§4.5)
    # -------------------------------------------------------------
    if state in (PulseState.STEADY, PulseState.BOOST):
        b = ctx.anchored_baseline   # inaktif seri başlamadan önceki 14 gün
        has_habit = (b.active_days >= BASELINE_MIN_ACTIVE_DAYS
                     and b.avg_minutes >= BASELINE_MIN_AVG_MINUTES)
        threshold = b.restart_threshold_days                  # clamp(max_gap + 2, 4, 7)
        if state == PulseState.BOOST:
            threshold = max(5, threshold)
            has_habit = True                                  # BOOST zaten aktif geçmişi ima eder
        if has_habit and inactive_days >= threshold:
            return transition(PulseState.RESTART, "RESTART_CHECKIN_PENDING",
                              f"INACTIVE_{inactive_days}D_PERSONAL_THRESHOLD_{threshold}D")

    # Cooldown koruması: RESTART girişi dışındaki tüm geçişler 48 saat bekler
    if is_in_cooldown:
        return PulseEvaluationResult.no_change()

    # -------------------------------------------------------------
    # KURAL 3: BOOST Girişi (Öncelik: 2)
    # -------------------------------------------------------------
    if state == PulseState.STEADY:
        if wcr_7d < BOOST_ENTRY_WCR:
            return transition(PulseState.BOOST, "BOOST_LOW_PACE", f"WEEKLY_COMPLETION_RATE_{wcr_7d:.2f}")
        if ctx.streak_broken_recently:   # önceki 10 günde >= 6 aktif gün + ardışık 2 gün inaktif
            return transition(PulseState.BOOST, "BOOST_STREAK_BROKEN", "STREAK_BROKEN_2_DAYS")

    # -------------------------------------------------------------
    # KURAL 4: STEADY'ye Dönüş (Öncelik: 3)
    # -------------------------------------------------------------
    if state == PulseState.BOOST and success_days >= BOOST_EXIT_SUCCESS_DAYS:
        return transition(PulseState.STEADY, "STEADY_NORMAL", "SUSTAINED_3_SUCCESSFUL_DAYS")
    if state == PulseState.RESTART and success_days >= RESTART_EXIT_SUCCESS_DAYS:
        return transition(PulseState.STEADY, "STEADY_NORMAL", "SUSTAINED_4_SUCCESSFUL_DAYS")

    # -------------------------------------------------------------
    # KURAL 5: Adaptasyon Süresi Doldu (RESTART_ADAPTED -> BOOST)
    # -------------------------------------------------------------
    if sub_state == "RESTART_ADAPTED" and ctx.adaptation_expires_at <= now:
        return transition(PulseState.BOOST, "BOOST_LOW_PACE", "ADAPTATION_ENDED_WITHOUT_RECOVERY")

    return PulseEvaluationResult.no_change()

# Not: transition(), hedef (state, sub_state) mevcut değerle aynıysa no_change() döner.
# Böylece RESTART'taki öğrenci her gece yeniden RESTART'a "geçirilmez", mükerrer kart/event üretilmez.
# Anket yanıtı kaynaklı geçişler (RESTART -> BOOST / ACTIVATE / RESTART_ADAPTED) bu fonksiyonda değil,
# CheckInSubmittedEvent işleyicisinde (RFC-004 §4) yapılır.
```

---

## 7. Edge Caseler ve İstisnai Durumlar

1. **Sınav Haftaları & Resmi Tatiller:**  
   Milli Eğitim Bakanlığı (MEB) sınav haftalarında veya sömestr tatilinde tüm Türkiye genelinde çalışma süreleri düşer. Karar motoru genel takvim servisine (`AcademicCalendarService`) bağlanarak bu haftalarda eşikleri dinamik olarak %40 gevşetir. Böylece on binlerce öğrenci haksız yere `RESTART` durumuna düşürülmez.
2. **Kasıtlı Dondurma:**  
   Öğrenci check-in anketinde veya ayarlar menüsünde "Bu hafta hasta/yoğunum" bildiriminde bulunmuşsa, öğrenciye `PAUSED` bayrağı konur ve motor bu öğrenciyi değerlendirme dışı bırakır.
3. **Müdahaleyi Kapatma:**  
   Bir öğrenci sunulan mikro-görevi 3 gün üst üste kapatırsa (`DISMISSED`), sistem aynı şablonu göstermeyi durdurur ve müdahale kanalını push bildirimden sessiz arayüz kartına çeker.

