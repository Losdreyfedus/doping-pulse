# 📊 Doping Pulse: Observability, İzleme ve SLA Spesifikasyonu

**Doküman Kodu:** `RFC-PULSE-007`  
**Modül:** `Pulse Monitoring, Alerting & Site Reliability Engineering (SRE)`  
**Yazar:** Hamza Kar  
**Teknoloji Yığını:** Prometheus, Grafana, OpenTelemetry, Alertmanager, Loki  
**Tasarım Hedefi:** Sistemin sağlığını, karar motorunun doğruluğunu ve yüzbinlerce öğrencinin durum dağılımını gerçek zamanlı izlemek.

---

## 1. SLI, SLO ve SLA Hedefleri

Doping Pulse, öğrenci ana ekran deneyimini doğrudan etkileyen kritik bir servis olduğu için aşağıdaki SLI hedefleri garanti edilir:

| SLI | SLO | Ölçüm Penceresi | Açıklama |
| :--- | :--- | :--- | :--- |
| **Availability** | **≥ %99.95** | 30 Günlük Kayan Pencere | HTTP 5xx hata oranının %0.05'in altında kalması. |
| **Mobil Ana Ekran P95 Latency** | **≤ 30 ms** | 5 Dakikalık Ortalama | `GET /pulse/me/status` isteğinin 95. yüzdelik yanıt süresi. |
| **Mobil Ana Ekran P99 Latency** | **≤ 75 ms** | 5 Dakikalık Ortalama | En uç tail latency. |
| **Olay İşleme Latency'si** | **≤ 250 ms (P99)** | Anlık | Video/Test bittiği andan durum güncellenmesine kadar geçen süre. |
| **Nightly Batch Süresi** | **≤ 45 dakika** | Her Gece (03:00 - 03:45) | 500.000 aktif öğrencinin inaktivite analizinin tamamlanma süresi. |

---

## 2. Temel Prometheus Metrikleri

Uygulama, `/metrics` endpoint'i üzerinden Prometheus formatında hem iş metrikleri hem de sistem ve altyapı metrikleri sunar:

### 2.1. İş Metrikleri

```prometheus
# Anlık öğrenci durum dağılımı (Gauge)
pulse_students_by_state{state="ACTIVATE", sub_state="ACTIVATE_STALLED"} 14205
pulse_students_by_state{state="BOOST", sub_state="BOOST_LOW_PACE"} 28410
pulse_students_by_state{state="RESTART", sub_state="RESTART_CHECKIN_PENDING"} 8912
pulse_students_by_state{state="STEADY", sub_state="STEADY_NORMAL"} 185420

# Durum geçiş sayacı (Counter)
pulse_state_transitions_total{from="ACTIVATE", to="STEADY", trigger="MILESTONE_COMPLETED"} 1450
pulse_state_transitions_total{from="STEADY", to="RESTART", trigger="NIGHTLY_INACTIVITY"} 320

# Müdahale dönüşüm hunisi (Intervention Funnel Counters)
pulse_intervention_events_total{template_id="ACTIVATE_FIRST_5M", type="IMPRESSION"} 8400
pulse_intervention_events_total{template_id="ACTIVATE_FIRST_5M", type="CLICK_CTA"} 3200
pulse_intervention_events_total{template_id="ACTIVATE_FIRST_5M", type="COMPLETE"} 2150
pulse_intervention_events_total{template_id="ACTIVATE_FIRST_5M", type="DISMISS"} 920

# ReStart anketi kök neden dağılımı (Counter)
pulse_checkin_responses_total{reason="SCHOOL_EXAMS"} 4120
pulse_checkin_responses_total{reason="SUBJECT_DIFFICULTY"} 1840
pulse_checkin_responses_total{reason="BURNOUT"} 1210
pulse_checkin_responses_total{reason="TECHNICAL_ISSUE"} 85
```

### 2.2. Sistem ve Altyapı Metrikleri

```prometheus
# API Gecikme Histogramı
pulse_http_request_duration_seconds_bucket{le="0.01", endpoint="/pulse/me/status"} 140291
pulse_http_request_duration_seconds_bucket{le="0.05", endpoint="/pulse/me/status"} 189210

# Kafka Tüketici Birikmesi (Consumer Lag)
pulse_kafka_consumer_lag_records{topic="learning.lesson.watched", partition="0"} 12

# PostgreSQL Bağlantı Havuzu
pulse_db_connections_active 18
pulse_db_connections_idle 32
pulse_db_connections_max 100
```

---

## 3. Grafana Dashboard Tasarımı

Operasyon ve ürün ekipleri için 3 ana dashboard paneli kurgulanmıştır:

```mermaid
graph TD
    subgraph D1 ["Dashboard 1 · Executive"]
        D1A["Öğrenci Durum Dağılımı"]
        D1B["Durum Geçiş Trendleri"]
        D1C["Günlük Kurtarılan Öğrenci Sayısı (Restart -> Steady)"]
    end

    subgraph D2 ["Dashboard 2 · Ürün Hunisi"]
        D2A["Müdahale Kartı Tıklanma Oranı (CTR %)"]
        D2B["Kart Tamamlama Oranı %"]
        D2C["Kopuş Nedenleri Dağılımı"]
    end

    subgraph D3 ["Dashboard 3 · SRE"]
        D3A["P95 & P99 HTTP Latency (ms)"]
        D3B["Kafka Consumer Lag & Throughput"]
        D3C["PostgreSQL Query Execution Times & Lock Waits"]
    end
```

---

## 4. Alarm Kuralları ve Eskalasyon Matrisi

Alertmanager üzerinde tanımlı kurallar önem derecesine göre PagerDuty veya Slack kanalına yönlendirilir:

### 4.1. Sev-1: Kritik, PagerDuty ile nöbetçi mühendis uyandırılır
1. **API Latency Degradation:**  
   * Koşul: `histogram_quantile(0.99, sum(rate(pulse_http_request_duration_seconds_bucket[5m])) by (le)) > 0.15` (150ms üzeri P99).  
   * Süre: 3 dakika boyunca sürerse.
2. **Yüksek 5xx Hata Oranı:**  
   * Koşul: `sum(rate(pulse_http_requests_total{status=~"5.."}[5m])) / sum(rate(pulse_http_requests_total[5m])) > 0.01` (> %1).
3. **Kafka Consumer Deadlock / Massive Lag:**  
   * Koşul: `sum(pulse_kafka_consumer_lag_records) > 25000` (Kuyrukta 25 binden fazla işlenmemiş olay birikmesi).

### 4.2. Sev-2: Uyarı, Slack #pulse-alerts kanalına düşer
1. **Anormal ReStart Sıçraması:**  
   * Koşul: Günlük `STEADY` → `RESTART` geçiş adedi son 7 günün ortalamasından %35 daha yüksekse. (Bir ders videosunun bozuk olması veya sistemik bir hata ihtimaline işaret eder).
2. **Dead-Letter Queue (DLQ) Uyarısı:**  
   * Koşul: `pulse_dlq_messages_total > 5` (Hata kuyruğuna mesaj düştüğü an).
3. **Gece Taraması Süre Aşımı:**  
   * Koşul: Gece inaktivite tarama işi 60 dakikadan uzun sürerse.
4. **Outbox Relay Lag:**  
   * Koşul: `pulse_outbox_oldest_unpublished_age_seconds > 60` (Durum değişti ama downstream'e ulaşmadı; check-in bildirimi ve koç alarmları gecikiyor. Bkz. RFC-006 §6).

---

## 5. Etki Ölçümü ve Holdout Tasarımı

Bölüm 2'deki funnel metrikleri (CTR, tamamlama oranı) Pulse'ın *çalıştığını* gösterir ama *işe yaradığını* göstermez. RESTART'tan STEADY'ye dönen bir öğrenci kartımız olmasa da dönecek olabilir; buna regression to the mean denir. Asıl soru şudur: **"Pulse olmasaydı ne olurdu?"** Bu soruyu yanıtlamanın tek güvenilir yolu kontrollü bir karşılaştırmadır.

### 5.1. Holdout Grubu Tasarımı
* **Atama:** `hash(studentId + salt) mod 100 < 10` olan öğrenciler `HOLDOUT` grubuna girer (%10). Atama deterministiktir ve **kalıcıdır**: öğrenci kayıt anında atanır, grubu değişmez ve sonuç `pulse_student_states.experiment_arm` alanına yazılır.
* **Shadow Mode:** Holdout öğrencileri için state machine normal çalışır; durumlar hesaplanır ve audit log yazılır. Yalnızca Müdahale Motoru kart, bildirim ve hedef uyarlaması üretmez (RFC-004 §1). Böylece iki grupta "RESTART'a düşen öğrenci" **aynı kuralla** tanımlanır ve karşılaştırma elma ile elma arasında yapılır.
* **Etik Sınır:** Holdout öğrencisi mevcut Doping deneyiminden hiçbir şey kaybetmez; yalnızca *yeni* Pulse müdahalelerini görmez. Pulse'tan önce de var olan insan süreçleri (koç görüşmeleri, retansiyon aramaları) her iki grupta aynen devam eder.

### 5.2. Metrik Hiyerarşisi
| Tip | Metrik | Tanım |
| :--- | :--- | :--- |
| **North Star** | RESTART Kurtarma Oranı | RESTART'a giren öğrencilerden 14 gün içinde `STEADY`'ye dönenlerin oranı (Treatment − Holdout = **uplift**) |
| İkincil | ACTIVATE → Aha! Süresi | Kayıttan ilk anlamlı aktiviteye kadar geçen medyan gün |
| İkincil | 30 Günlük Aktif Gün | Öğrenci başına son 30 gündeki aktif gün sayısı (RFC-001 §4.5 tanımı) |
| İkincil | Paket Yenileme / Churn | Dönem sonu yenileme oranı (gecikmeli; çeyreklik raporlanır) |
| **Guardrail** | Bildirim Kapatma Oranı | Push izni kapatan öğrencilerin oranı. Treatment'ta anlamlı artış varsa deney durdurulur. |
| **Guardrail** | Dismiss Oranı / Veli Şikayeti | Kart kapatma oranı ve veli destek kaydı sayısı |

### 5.3. Örneklem Büyüklüğü (Örnek Hesap)
*Varsayımlar (pilot verisiyle güncellenecek):* RESTART'tan 14 günde kendiliğinden dönüş oranı p = %30; tespit etmek istediğimiz minimum etki δ = 5 puan; α = 0,05; güç = %80.

```text
n (grup başına) ≈ 16 · p · (1 − p) ÷ δ²  =  16 · 0,21 ÷ 0,0025  ≈  1.350
```

Günde ~320 STEADY → RESTART geçişi (§2.1) ve %10 holdout ile holdout grubu günde ~32 öğrenci biriktirir. Bu da **~6 haftalık** bir deney süresi demektir. Süre kısaltılmak istenirse holdout oranı geçici olarak %20'ye çıkarılabilir; ACTIVATE ve BOOST akışları daha yüksek hacimli olduğu için daha erken sonuç verir.

### 5.4. Analiz Kuralları
* Sonuçlar **intention-to-treat** prensibiyle analiz edilir: kartı görmeyen veya kapatan treatment öğrencisi de treatment grubunda sayılır.
* Metrikler deney başlamadan önce sabitlenir. Deney sırasında birincil metrik değiştirilmez (p-hacking koruması).
* Kural parametreleri (eşikler, hafifletme oranları) değiştiğinde yeni bir deney sürümü (`experiment_version`) açılır ve eski sürümün verisiyle karıştırılmaz.
