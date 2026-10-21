# 🎯 Doping Pulse: Statü ve Segment Bazlı Öneri Matrisi

**Doküman Kodu:** `RFC-PULSE-008`  
**Yazar:** Hamza Kar  
**Soru:** Öğrencilerin statüleri belli. Her statü kendi içinde segmentlere ayrılırsa, statü + segmente göre öğrencilere ne önermek mantıklı olur?

> Statü öğrencinin **şu an ne yaşadığını** söyler (başlayamadı, temposu düştü, koptu, yolunda). Segment ise **kim olduğunu** söyler. Aynı statüdeki iki öğrenci farklı öneri almalı: kopan bir LGS öğrencisiyle kopan bir 10. sınıf öğrencisinin ihtiyacı aynı değildir.

---

## 1. Segment Eksenleri

Üç eksen seçildi. Üçü de Doping'in zaten topladığı verilerden, ek bir anket sormadan hesaplanabilir.

| Eksen | Segmentler | Nasıl hesaplanır | Neden önemli |
| :--- | :--- | :--- | :--- |
| **Sınav takvimi** | `EXAM_NEAR` (sınava < 120 gün) · `EXAM_FAR` (sınav sınıfı, > 120 gün) · `SCHOOL_FOCUS` (ara sınıf: 5–7, 9–10, 11. sınıfın ilk dönemi) | Sınıf seviyesi + LGS/YKS tarihi | Öğrencinin neye çalıştığını ve zamanın ne kadar değerli olduğunu belirler |
| **Öğrenme tarzı** | `WATCHER` (video ağırlıklı) · `SOLVER` (soru ağırlıklı) · `BALANCED` | Son 14 günde: soru sayısı ÷ izlenen dakika. < 0,2 → `WATCHER`, > 1,0 → `SOLVER` | İlk adımın hangi formatta olacağını belirler |
| **Akademik durum** | `STRUGGLING` (doğruluk < %50) · `ON_LEVEL` (%50–75) · `ADVANCED` (> %75) | Son 14 günde: doğru ÷ çözülen soru (en az 20 soru; altındaysa `UNKNOWN`) | Önerinin zorluk seviyesini belirler |

* Eksenler **her gece** yeniden hesaplanır, statüden bağımsızdır.
* Veri yetersizse (yeni öğrenci, az soru) segment `UNKNOWN` olur ve o eksen öneri seçiminde yok sayılır.
* Eşik değerleri (120 gün, 0,2, %50 vb.) başlangıç değerleridir; pilot veriyle kalibre edilecektir.

---

## 2. Öneri Matrisi

Her statünün bir **temel önerisi** vardır (bugün RFC-004'te tanımlı olan). Segment bu temel öneriyi **değiştirir veya üzerine ekler**.

### 🟢 ACTIVATE: Henüz başlayamamış

*Temel öneri:* Düşük eşikli ilk adım.

| Segment | Öneri | Gerekçe |
| :--- | :--- | :--- |
| `EXAM_NEAR` | Sınavda en çok soru çıkan konudan 4 dakikalık video + koçla 10 dakikalık tanışma görüşmesi | Zaman kısıtlı; korkutmadan "en değerli ilk adım" gösterilir |
| `SCHOOL_FOCUS` | Okulda o hafta işlenen konudan video: *"Yarınki dersine 5 dakikada hazırlan"* | Öğrencinin bugünkü derdine çözüm; Doping'e okul üzerinden giriş kapısı |
| `WATCHER` | İlk adım kısa bir video | Alıştığı formatla başlamak sürtünmeyi azaltır |
| `SOLVER` | İlk adım 5 soruluk mini test | Aynı gerekçe; soru çözmeyi seven öğrenciye video "ders gibi" gelir |

### 🟡 BOOST: Temposu düşmüş

*Temel öneri:* Hedefler %40 hafifletilir.

| Segment | Öneri | Gerekçe |
| :--- | :--- | :--- |
| `STRUGGLING` | Hafifletmenin yanında zorlandığı konunun temel hap videosu + 3 kolay soru | Sorun tempoda değil, anlamada olabilir; sadece hafifletmek kök nedeni çözmez |
| `WATCHER` | *"İzlediğin videonun 3 sorusunu çöz, pekiştir"* | Pasif izlemeyi aktif öğrenmeye çevirir |
| `EXAM_NEAR` | Hafifletme **yerine** önceliklendirme: *"Bu hafta sadece en çok çıkan 3 konuya odaklan"* | Sınava 2 ay kala "yükünü azalttık" mesajı yanlış sinyal verir; yük azalır ama yön netleşir |
| `SCHOOL_FOCUS` + MEB yazılı haftası | "Yazılı haftası modu": YKS/LGS hedefleri bir haftalığına durur, okul konuları öne çıkar | Öğrenci okul sınavına çalışırken Doping planı onunla çatışmaz (bkz. RFC-001 §4.2 tempo kuralı) |

### 🔴 RESTART: Kopmuş

*Temel öneri:* Yargılamayan check-in; kopuş sebebi anketten gelir (RFC-004 §4). Segment, anketin üzerine ikinci bir katman ekler.

| Segment | Öneri | Gerekçe |
| :--- | :--- | :--- |
| `EXAM_NEAR` | Eskalasyon hızlanır: anket yanıtsızsa koç 48 yerine **24 saatte** devreye girer | Sınava yakın kaybedilen bir hafta, telafisi en zor haftadır |
| `ADVANCED` | *"Kaldığın yerden devam"*: yarım bıraktığı konu ve videodan başlatma | Başarılı öğrenci yolunu biliyor; ona en kısa dönüş yolu gösterilir |
| `STRUGGLING` | *"Temelden başla"*: koptuğu konunun bir önceki konusundan özet | Kopuşun muhtemel sebebi konu zorluğu; aynı yerden başlatmak aynı duvara çarptırır |

### ⚪ STEADY: Yolunda

*Temel öneri:* Müdahale yok, sadece streak. **Bildirim gönderilmez**; aşağıdaki öneriler yalnızca ana ekranda pasif kart olarak gösterilir.

> Bence en büyük fırsat bu statüde: bu öğrenciler bize en çok güvenen ve paketini yenileme ihtimali en yüksek olanlar.

| Segment | Öneri | Gerekçe |
| :--- | :--- | :--- |
| Uzun streak (≥ 14 gün) | Haftalık mini deneme, rozet | Alışkanlığı oturmuş öğrenciye yeni bir hedef |
| `STRUGGLING` (çalışıyor ama doğruluğu düşük) | *"Daha az ama doğru"*: yanlış yaptığı soruların tekrarı | Çok çalışıp sonuç alamamak ileride kopuşun habercisidir |
| `ADVANCED` | Zorluk seviyesini artırma | Sıkılan öğrenci de kopar |
| `EXAM_NEAR` | Deneme sınavı takvimi + eksik konu raporu | Sınav yaklaşırken neye çalışacağını görmek |

---

## 3. Birden Fazla Segmente Uyan Öğrenci

Bir öğrenci aynı anda üç eksende de bir segmentte olur (ör. `EXAM_NEAR` + `WATCHER` + `STRUGGLING`). Günde en fazla 1 kart kuralı (RFC-004 §5.1) korunduğu için tek bir öneri seçilmelidir:

1. **Öncelik sırası:** Sınav takvimi > Akademik durum > Öğrenme tarzı. Zaman kısıtı en acil olandır; anlamama sorunu formattan önce gelir.
2. **Öğrenme tarzı format belirler, içerik değil:** Seçilen öneri `WATCHER` için video, `SOLVER` için soru olarak sunulur. Örnek: `EXAM_NEAR` + `SOLVER` öğrencisine ACTIVATE'te "en çok çıkan konudan 5 soru" gösterilir.
3. `UNKNOWN` eksenler atlanır; hiçbir segment belli değilse statünün temel önerisi kullanılır.

---

## 4. Veri Modeli Değişiklikleri

Matris konu bazlı veri gerektirir ("zorlandığı konu", "en çok çıkan 3 konu"). Bugünkü `pulse_daily_aggregates` sadece günlük toplam tutar. Gerekenler:

```sql
-- Öğrencinin konu bazlı performansı (gece güncellenir, son 30 gün)
CREATE TABLE pulse_topic_stats (
    student_id          UUID NOT NULL,
    topic_id            VARCHAR(64) NOT NULL,
    questions_solved    SMALLINT NOT NULL DEFAULT 0,
    questions_correct   SMALLINT NOT NULL DEFAULT 0,
    last_lesson_id      VARCHAR(64) NULL,   -- "Kaldığın yerden devam" için
    last_watch_pct      SMALLINT NULL,
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    PRIMARY KEY (student_id, topic_id)
);

-- Segment sonucu durum tablosunda tutulur (her gece yeniden hesaplanır)
ALTER TABLE pulse_student_states
    ADD COLUMN segment_exam      VARCHAR(16) NOT NULL DEFAULT 'UNKNOWN',
    ADD COLUMN segment_style     VARCHAR(16) NOT NULL DEFAULT 'UNKNOWN',
    ADD COLUMN segment_academic  VARCHAR(16) NOT NULL DEFAULT 'UNKNOWN';
```

* "Sınavda en çok çıkan konular" listesi Doping içerik ekibinden gelen statik bir katalogdur; Pulse bunu üretmez.
* Şablon seçimi için `pulse_intervention_templates` tablosuna `target_segment` alanı eklenir; Müdahale Motoru önce statü + segmente uyan şablonu, yoksa statünün temel şablonunu seçer.

---

## 5. Örnek: Ece

Ece `EXAM_FAR` (YKS'ye > 120 gün), `BALANCED`, `ON_LEVEL`. Senaryodaki akış (`example-scenario-restart-journey.md`) temel önerilerle aynen geçerlidir. Farklı bir Ece düşünelim:

| Varyasyon | Değişen |
| :--- | :--- |
| Ece Mart ayında kopsaydı (`EXAM_NEAR`) | Check-in yanıtsız kalırsa koç 24 saatte devreye girer |
| Ece'nin doğruluğu %40 olsaydı (`STRUGGLING`) | Dondurma bitince "kaldığın yerden" değil, önceki konunun özetiyle döner |
| Ece 10. sınıf olsaydı (`SCHOOL_FOCUS`) | Yazılı haftasında kopmak yerine "yazılı haftası modu" önerilir ve RESTART'a hiç düşmeyebilirdi |

---

## 6. Nasıl Ölçeceğiz?

Segment önerileri de holdout deneyiyle test edilir (RFC-007 §5). Treatment grubunun bir kısmı segment önerisi, bir kısmı sadece statü temel önerisi alır. Segmentlemenin değeri, bu iki kol arasındaki fark kadardır. Fark yoksa sistem gereksiz yere karmaşıklaştırılmamalı.

---

## 7. Açık Sorular

1. **Koçlu paket:** Koç desteği olmayan pakette koç adımları ne olacak? Ayrı bir eksen mi, yoksa sadece eskalasyon kuralı mı?
2. **Veli ilgisi:** Veli uygulamasını aktif kullanan velinin öğrencisinde eskalasyon farklı mı olmalı?
3. **Eksik segment:** Doping'deki öğrenciler arasında bu üç eksende yakalanamayan bir tip var mı?
