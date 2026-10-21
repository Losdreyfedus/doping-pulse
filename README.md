# ⚡ Doping Pulse

> **"Öğrencinin çalışma yolculuğunda bulunduğu durumu anlayarak, ihtiyacı olan anda doğru müdahaleyi sunmak."**

Doping Hafıza MT Programı kapsamında tasarlanan **Doping Pulse**, öğrencinin platformdaki öğrenme alışkanlıklarını, ritmini ve kopuş sinyallerini gerçek zamanlı ve kümülatif olarak analiz eden; öğrenciyi büyük programlarla boğmak yerine doğru anda nokta atışı mikro-müdahaleler sunan bir **Akıllı Karar Motoru** mimarisidir.

Bu doküman seti, kod yazmadan uçtan uca bir backend tasarımı olarak hazırlanmıştır: ilişkisel veri modelleri, obje modelleri, RESTful API sözleşmeleri, state machine kuralları ve event-driven akışlar.

**Yazar:** Hamza Kar

---

## 🎯 Problem ve Çözüm Çatısı

Öğrenciler Doping Hafıza ekosisteminde ilerlerken karşılaşılan 3 temel kırılma noktası:
1. **🟢 Activate (Başlat):** Satın aldı / sisteme girdi ancak nereden başlayacağını bilemiyor veya ilk adım gözünde büyüyor.  
   *Müdahale:* Net, düşük eşikli mikro-görev ("Bugün sadece 4 dakikalık giriş videosunu izle").
2. **🟡 Boost (Ritmini Güçlendir):** Çalışmaya devam ediyor ancak hedeflerinin altında kalıyor, tempo düzensizleşiyor.  
   *Müdahale:* Suçluluk hissettirmeden hedefleri dinamik küçültme (%40 hafifletme), ritmi koruma desteği.
3. **🔴 ReStart (Koptuysan Yeniden Başlat):** Düzenli çalışırken anlamlı ve ani bir kopuş yaşıyor (kişisel eşiğe göre 4–7 gün hiç çalışmama).  
   *Müdahale:* Yargılamayan 2 soruluk check-in ("Seni ne zorladı?") → kök nedeni anla → kişiselleştirilmiş program uyarlaması.
4. **⚪ Steady (Akışta Kal):** Hedef temposunda stabil ilerleyen öğrenci.  
   *Müdahale:* Rahatsız etmeme ilkesi; kart gösterilmez, sadece streak takibi yapılır.

---

## 📂 Teknik Dokümantasyon Haritası

Tüm backend altyapısı **Separation of Concerns** ilkesiyle bağımsız RFC dokümanlarına ayrılmıştır:

```text
doping-pulse/
├── README.md                                  # Proje genel bakışı ve teknik harita (Bu dosya)
├── .gitignore                                 # Git versiyon kontrolü hariç tutma kuralları
└── docs/
    ├── doping-pulse-technical-spec.md         # Çatı Sistem Tasarım Belgesi
    ├── state-machine-and-rules.md             # RFC-001: Durum Makinesi, Hysteresis & Anti-Flapping
    ├── database-schema-and-indexing.md        # RFC-002: PostgreSQL DDL, UUIDv7, Bölümleme & İndeksler
    ├── api-specification.md                   # RFC-003: RESTful API Sözleşmesi, Idempotency & RFC 7807
    ├── intervention-engine-and-surveys.md     # RFC-004: Mikro-Görev Şablonları & ReStart Anket Matrisi
    ├── coaching-and-guardian-escalation.md    # RFC-005: Veli Bildirimi & Koçluk CRM Eskalasyonu
    ├── event-driven-messaging-architecture.md# RFC-006: Apache Kafka Mesajlaşma, Strict Ordering & DLQ
    ├── openapi-swagger-spec.yaml              # Canlı OpenAPI 3.0 / Swagger UI & Postman Sözleşmesi
    ├── example-scenario-restart-journey.md    # Uçtan uca örnek senaryo: Ece'nin RESTART yolculuğu
    ├── segments-and-recommendations.md        # RFC-008: Statü + segment bazlı öneri matrisi
    └── monitoring-alerting-and-slas.md        # RFC-007: Gözlemlenebilirlik, Prometheus, Grafana & SLA
```

---

## 🏗️ Modüler Teknik Mimari Özeti

| Modül Kodu | Doküman | Odak Noktası | Öne Çıkan Mimari Karar |
| :--- | :--- | :--- | :--- |
| **RFC-001** | [State Machine & Kurallar](docs/state-machine-and-rules.md) | Durum Geçişleri | **Anti-Flapping:** 48 saatlik cooldown tamponu, sliding window ve kişisel baseline. |
| **RFC-002** | [Veritabanı & İndeksler](docs/database-schema-and-indexing.md) | Veri Katmanı | **PostgreSQL 16+ & UUIDv7:** Aylık deklaratif range partitioning, mobil için kısmi indeksler (`WHERE status = 'ASSIGNED'`). |
| **RFC-003** | [API Spesifikasyonu](docs/api-specification.md) | Servis Sözleşmesi | **Zero-Trust & Idempotency:** JWT claim çözümleme, mükerrer istek koruması (`Idempotency-Key`), RFC 7807 `ProblemDetails` hata standardı. |
| **RFC-004** | [Müdahale Motoru](docs/intervention-engine-and-surveys.md) | İçerik Önerisi | **Survey-to-Action Matrix:** Anket cevaplarına göre hedefleri dondurma veya hazırlık videosu atama; günde max 1 kart ve dismissal backoff kuralı. |
| **RFC-005** | [Veli & Koç Eskalasyonu](docs/coaching-and-guardian-escalation.md) | İnsan Faktörü | **"Şikayet Etme, Çözüm Sun":** 4 kademeli alarm (Öğrenci → Koç → Veli → CRM Outbound); öğrenci derse girince otomatik çözümleme. |
| **RFC-006** | [Event-Driven Altyapı](docs/event-driven-messaging-architecture.md) | Mesajlaşma (Kafka) | **Strict In-Order per Student:** Partition Key = `studentId`, fire-and-forget ile sıfır senkron bağımlılık, Transactional Outbox, transaction içi idempotent tüketici, 3x exponential backoff & DLQ. |
| **RFC-007** | [Gözlemlenebilirlik & SLA](docs/monitoring-alerting-and-slas.md) | SRE & İzleme | **%99.95 Uptime & Sub-30ms Latency:** Prometheus durum dağılım sayaçları, Grafana dönüşüm hunileri, Sev-1 PagerDuty alarmları, %10 holdout ile uplift ölçümü. |
| **OpenAPI** | [Swagger YAML](docs/openapi-swagger-spec.yaml) | Entegrasyon | **Etkileşimli Test:** Swagger Editor ve Postman uyumlu, tam şemalı, örnek payload'lu canlı API kontratı. |
| **Senaryo** | [Örnek Senaryo: Ece](docs/example-scenario-restart-journey.md) | Uçtan Uca Doğrulama | **Gün gün veri izi:** Bir öğrencinin kopuşu, check-in yanıtı ve dönüşü boyunca her tabloya yazılan satırlar ve her API yanıtı. |
| **RFC-008** | [Segment ve Öneriler](docs/segments-and-recommendations.md) | Kişiselleştirme | **Statü + Segment:** Sınav takvimi, öğrenme tarzı ve akademik duruma göre her statüde farklı öneri; çakışma önceliği ve konu bazlı veri modeli. |

---

## 🚀 Hızlı Başlangıç & İnceleme

1. **Sistemin kuşbakışı tasarımını incelemek için:** [docs/doping-pulse-technical-spec.md](docs/doping-pulse-technical-spec.md)
2. **API uçlarını Swagger'da görüntülemek için:** [docs/openapi-swagger-spec.yaml](docs/openapi-swagger-spec.yaml) dosyasını [editor.swagger.io](https://editor.swagger.io) adresine yapıştırın.

