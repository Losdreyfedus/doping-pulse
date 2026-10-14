# 🧠 Doping Pulse: Durum Makinesi ve Geçiş Kuralları Spesifikasyonu (State Machine RFC)

**Doküman Kodu:** `RFC-PULSE-001`  
**Modül:** `Pulse State & Decision Engine`  
**Mimar:** Senior Backend Architect & MT Core Team  
**Hedef:** Öğrenci durum sınıflandırmasının deterministik, gürültüden arındırılmış (anti-flapping) ve esnek kural motoruyla modellenmesi.

---

## 1. Mimari Prensipler ve Problem Tanımı

Gerçek dünya EdTech sistemlerinde öğrenci davranışlarını izlerken yapılan en büyük backend hatası, basit `if-else` mantığıyla anlık durum değiştirmektir. 
Örneğin; dün çalışmayan öğrenciyi hemen "Kopmuş (ReStart)" ilan edip, ertesi gün 1 soru çözdü diye "Her şey yolunda (Steady)" durumuna almak **Durum Dalgalanmasına (State Flapping / Ping-Pong Effect)** yol açar. Bu da kullanıcıya sürekli tutarsız bildirimler gitmesine ve sistemin güvenilirliğini yitirmesine neden olur.

Pulse Karar Motoru şu 4 temel prensiple çalışır:
1. **Sliding Time Window (Kayan Zaman Penceresi):** Durumlar anlık tek bir olaya göre değil; 3, 7 ve 14 günlük hareketli ortalamalara ve trendlere göre değerlendirilir.
2. **Hysteresis & Cooldown (Geçiş Tamponu):** Bir durum değiştikten sonra sistem en az $N$ gün "soğuma (cooldown)" periyoduna girer; küçük dalgalanmalar durumu anında bozamaz.
3. **Kişiselleştirilmiş Taban Çizgisi (Personal Baseline):** Her öğrencinin "normali" farklıdır. Haftada 2 saat çalışan bir LGS öğrencisi ile haftada 25 saat çalışan bir YKS derece öğrencisi aynı mutlak saat eşiğiyle değerlendirilemez.
4. **Deterministik Önceliklendirme (Rule Precedence):** Çakışan kurallarda en kritik durum (örn: ReStart $\gt$ Boost) deterministik bir öncelik sırasıyla işletilir.

---

## 2. Durumlar ve Alt Durumlar (State & Sub-States)

Sistem 4 ana durum ve operasyonel derinlik için tanımlanmış alt durumlardan (sub-states) oluşur:

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

## 3. Durum Geçiş Matrisi (Formal Transition Matrix)

Aşağıdaki tablo, durumlar arası geçiş koşullarını, tetikleyici mekanizmasını ve koruma kurallarını tanımlar:

| Kaynak Durum ($S_{from}$) | Hedef Durum ($S_{to}$) | Tetikleyici ($Trigger$) | Guard Conditions (Ön Koşullar) | Cooldown / Gecikme |
| :--- | :--- | :--- | :--- | :--- |
| **[*]** | **ACTIVATE** | Event: `StudentEnrolledEvent` | Paket aktif, baseline aktivite yok | Yok (Başlangıç) |
| **ACTIVATE** | **STEADY** | Event: `MilestoneCompletedEvent` | **"Aha! Moment":** İlk video ($\ge \%80$ izleme) **VEYA** ilk test ($\ge 5$ soru) tamamlandı | Anında |
| **ACTIVATE** | **ACTIVATE_CRITICAL** | Batch (Nightly) | Kayıt üzerinden $\ge 7$ gün geçti AND toplam aktivite süresi $\le 0$ | 24 saat |
| **STEADY** | **BOOST** | Batch (Nightly) | Son 7 günlük hedef tamamlama $<\%50$ **VEYA** son 4 günde ardışık 2 gün inaktif | 48 saat Cooldown |
| **STEADY** | **RESTART** | Batch (Nightly) | Öğrencinin son 14 gün ortalaması $\ge 25\text{ dk/gün}$ **AND** son 4 gün ardışık sıfır aktivite | Anında (Kritik Kopuş) |
| **BOOST** | **STEADY** | Batch / Event | Son 3 gün üst üste günlük hedefler $\ge \%80$ tamamlandı | 72 saat stabilite |
| **BOOST** | **RESTART** | Batch (Nightly) | Boost durumunda iken ardışık 5 gün sıfır aktivite gerçekleşti | Anında |
| **RESTART** | **BOOST** | Event: `CheckInSubmittedEvent` | Check-in anketinde "Yavaşlayarak devam etmek istiyorum" seçildi | Anında |
| **RESTART** | **ACTIVATE** | Event: `CheckInSubmittedEvent` | Check-in anketinde "Sıfırdan baştan başlamak istiyorum" seçildi | Anında |
| **RESTART** | **STEADY** | Event / Batch | Müdahale sonrası son 4 günde hedefler başarıyla yerine getirildi | 4 gün stabilite |

---

## 4. Durum Bazlı Detaylı Kural Mantığı (Deep Dive)

### 4.1. 🟢 ACTIVATE Motoru
* **Problem:** Öğrenci satın aldı ancak platform çok karmaşık veya ilk adım çok büyük geliyor.
* **Tetikleyiciler:**
  * `Day 0-2 (Onboarding):` Hoş geldin mikro adımı: "Sadece 3 dakikalık 'Nasıl Çalışmalıyım?' videosunu izle."
  * `Day 3-6 (Stalled):` Uygulamayı açıyor ama derse girmiyor: "Bugün 1 ders seç ve sadece ilk 5 dakikasına bak."
  * `Day 7+ (Critical):` Hiç login olmadı: Push Notification + Velisine bilgilendirme hazırlığı.
* **Çıkış Kapısı (First Value Moment):** Öğrenci sisteme ilk anlamlı veriyi bıraktığı an (`video_duration >= 180s` VEYA `questions_solved >= 5`), anında `STEADY` durumuna yükseltilir.

### 4.2. 🟡 BOOST Motoru
* **Problem:** Kopmadı ama bocalıyor. Hedefler birikiyor, öğrenci suçluluk hissediyor ve kaçma eğilimi gösteriyor.
* **Metrikler:**
  * **Haftalık Yük Tamamlama Oranı (WCR):**
    $$\text{WCR} = \frac{\sum_{i=1}^{7} \text{CompletedTasks}_i}{\sum_{i=1}^{7} \text{TargetTasks}_i}$$
    Eğer $\text{WCR} < 0.50$ ise $\rightarrow$ `BOOST_LOW_PACE`.
  * **Seri Bozulması (Streak Break):** Öğrencinin geçmiş 10 günde en az 6 gün çalışması varken, ardışık 2 gün hiç girmemesi.
* **Müdahale Stratejisi:**
  * Asla *"Çok geridesin, daha çok çalışmalısın!"* denmez.
  * **Hedef Küçültme (Target Deceleration):** Günlük soru hedefi dinamik olarak $\%30 - \%50$ oranında otomatik kırpılır. Öğrenciye *"Bugün sadece 10 soru çözmen yeterli"* hissi verilir.
* **Çıkış Kapısı:** Ardışık 3 gün boyunca bu hafifletilmiş hedeflerin en az $\%80$'ini tamamladığında ritim oturmuş kabul edilip `STEADY` durumuna geçirilir.

### 4.3. 🔴 RESTART Motoru
* **Problem:** Düzenli çalışan öğrenci birden kayboldu. Bu öğrenciye sıradan bir "Hadi çalış" bildirimi atmak ters teper; çünkü arkasında bir hayat olayı vardır (Hastalık, okul sınav haftası, tükenmişlik/burnout veya zor bir konuya takılma).
* **Tespit Koşulu (Inactivity Drop):**
  * Öğrenci daha önce `STEADY` veya aktif bir profilde olmalıdır ($\text{Son 14 gün ortalama süre} \ge 25\text{ dk/gün}$).
  * Son $X$ gün (varsayılan: 4 gün) ardışık olarak sıfır aktivite kaydedilmelidir.
* **Proaktif Müdahale: Check-In Döngüsü (Feedback Loop):**
  1. Sisteme girdiğinde ana ekran kilitlenmez ancak en üstte şefkatli bir mini anket çıkar:  
     *"Birkaç gündür görüşemedik, her şey yolunda mı?"*
  2. Seçenekler:
     - 🅰️ *Okul yazılılarım / sınavlarım vardı.* $\rightarrow$ **Aksiyon:** 1 hafta boyunca Doping hedeflerini askıya al veya dondur.
     - 🅱️ *Son çalıştığım konuyu hiç anlamadım, moralim bozuldu.* $\rightarrow$ **Aksiyon:** O konunun temel seviye hap videosunu ve 3 kolay sorusunu öner.
     - 🅲️ *Çok yoruldum, tempom düştü.* $\rightarrow$ **Aksiyon:** Günlük hedefleri $\%60$ küçült, mikro ritme geç.
  3. **Yanıtsızlık Durumu (Unresponsive Escalation):** Check-in kartı 48 saat boyunca yanıtlanmazsa, durum `RESTART_UNRESPONSIVE` olur ve veli/koç panelinde risk bayrağı (`CHURN_RISK_HIGH`) kalkar.

### 4.4. ⚪ STEADY (On-Track & Baseline)
* Öğrencinin haftalık hedeflerinin $\ge \%70$'ini tutturduğu, çalışma aralıklarının 48 saati aşmadığı sağlıklı çalışma durumudur.
* **Önemli Kural:** Bu durumdaki öğrenciye "müdahale kartı" gösterilmez. Sadece tebrik, seviye atlama ve streak (gün serisi) sayaçları gösterilir.

---

## 5. Anti-Flapping (Dalgalanma Önleme) Algoritması

Durum motoru, öğrencinin geçişlerini denetlerken aşağıdaki **Sliding Window Buffer** algoritmasını işletir:

```
                  ┌─────────────────────────────────────┐
                  │ Sinyal Alındı (Event veya Gece İşi) │
                  └──────────────────┬──────────────────┘
                                     │
                                     ▼
                ┌─────────────────────────────────────────┐
                │ Cooldown Kontrolü:                      │
                │ Son durum değişikliğinden bu yana       │
                │ en az 48 saat geçti mi?                 │
                └────────────────────┬────────────────────┘
                                     │
                     ┌───────────────┴───────────────┐
                     │ EVET                          │ HAYIR
                     ▼                               ▼
       ┌───────────────────────────┐   ┌───────────────────────────┐
       │ Hedef durum mevcut durum- │   │ Durum Değiştirme YASAK    │
       │ la aynı mı?               │   │ (Sadece metrik güncelle)  │
       └─────────────┬─────────────┘   └───────────────────────────┘
                     │ FARKLI
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

## 6. Kural Motoru Değerlendirme Algoritması (Pseudo-Code)

Aşağıdaki mantık, her öğrenci değerlendirilirken çalışan deterministik fonksiyonu gösterir:

```python
def evaluate_student_pulse(student_id: UUID, context: PulseEvaluationContext) -> PulseEvaluationResult:
    current_state = context.current_state
    now = current_timestamp()

    # 1. Cooldown Kontrolü (Hysteresis Guard)
    # Acil durum istisnası: Aktiften ani kopuş (RESTART) cooldown dinlemez!
    is_in_cooldown = (now - context.state_updated_at).days < 2
    
    # 2. Metrik Hesaplamaları
    inactive_days = context.consecutive_inactive_days
    avg_minutes_last_14d = context.last_14d_daily_average_minutes
    completion_rate_last_7d = context.last_7d_target_completion_rate
    has_meaningful_activity = context.total_lifetime_watch_seconds > 180 or context.total_lifetime_questions > 5

    # -------------------------------------------------------------
    # KURAL 1: ACTIVATE Kontrolü
    # -------------------------------------------------------------
    if current_state == PulseState.ACTIVATE:
        if has_meaningful_activity:
            return PulseEvaluationResult(
                new_state=PulseState.STEADY,
                sub_state="STEADY_NORMAL",
                reason="FIRST_MEANINGFUL_MILESTONE_ACHIEVED"
            )
        elif context.days_since_enrollment >= 7 and inactive_days >= 7:
            return PulseEvaluationResult(
                new_state=PulseState.ACTIVATE,
                sub_state="ACTIVATE_CRITICAL",
                reason="ZERO_ACTIVITY_FIRST_WEEK"
            )
        return PulseEvaluationResult.no_change()

    # -------------------------------------------------------------
    # KURAL 2: RESTART Kontrolü (Öncelik: 1)
    # Düzenli çalışan bir öğrenci 4+ gün boyunca hiç girmedi mi?
    # -------------------------------------------------------------
    is_historically_active = avg_minutes_last_14d >= 25.0
    if is_historically_active and inactive_days >= 4:
        return PulseEvaluationResult(
            new_state=PulseState.RESTART,
            sub_state="RESTART_CHECKIN_PENDING",
            reason=f"SUDDEN_DROP_AFTER_{avg_minutes_last_14d}_MIN_BASELINE"
        )

    # Cooldown koruması: RESTART harici durum değişiklikleri 48 saat beklemeli
    if is_in_cooldown:
        return PulseEvaluationResult.no_change()

    # -------------------------------------------------------------
    # KURAL 3: BOOST Kontrolü (Öncelik: 2)
    # Hedef tamamlama <%50 mi veya son 4 günde 2 gün aksama var mı?
    # -------------------------------------------------------------
    if current_state in [PulseState.STEADY, PulseState.BOOST]:
        if completion_rate_last_7d < 0.50 or context.streak_broken_recently:
            if current_state != PulseState.BOOST:
                return PulseEvaluationResult(
                    new_state=PulseState.BOOST,
                    sub_state="BOOST_LOW_PACE" if completion_rate_last_7d < 0.50 else "BOOST_STREAK_BROKEN",
                    reason=f"WEEKLY_COMPLETION_RATE_{completion_rate_last_7d:.2f}"
                )

    # -------------------------------------------------------------
    # KURAL 4: STEADY'ye Dönüş (Öncelik: 3)
    # BOOST veya RESTART durumundaki öğrenci ritmini geri kazandı mı?
    # -------------------------------------------------------------
    if current_state in [PulseState.BOOST, PulseState.RESTART]:
        if context.consecutive_successful_days >= 3 and completion_rate_last_7d >= 0.70:
            return PulseEvaluationResult(
                new_state=PulseState.STEADY,
                sub_state="STEADY_NORMAL",
                reason="SUSTAINED_3_DAYS_CONSISTENT_PACE"
            )

    return PulseEvaluationResult.no_change()
```

---

## 7. Edge Caseler ve İstisnai Durumlar

1. **Sınav Haftaları & Resmi Tatiller (Holiday/Exam Exception):**  
   Milli Eğitim Bakanlığı (MEB) sınav haftalarında veya sömestr tatilinde tüm Türkiye genelinde çalışma süreleri düşer. Karar motoru genel takvim servisine (`AcademicCalendarService`) bağlanarak bu haftalarda eşikleri dinamik olarak $\%40$ gevşetir. Böylece on binlerce öğrenci haksız yere `RESTART` durumuna düşürülmez.
2. **Kasıtlı Dondurma (Student Pause Request):**  
   Öğrenci check-in anketinde veya ayarlar menüsünde "Bu hafta hasta/yoğunum" bildiriminde bulunmuşsa, öğrenciye `PAUSED` bayrağı konur ve motor bu öğrenciyi değerlendirme dışı (Bypass) bırakır.
3. **Müdahaleyi Kapatma (Dismissal Fatigue):**  
   Bir öğrenci sunulan mikro-görevi 3 gün üst üste kapatırsa (`DISMISSED`), sistem aynı şablonu göstermeyi durdurur ve müdahale kanalını push bildirimden sessiz arayüz kartına çeker.

