# ⚡ Doping Pulse

> **"Öğrencinin çalışma yolculuğunda bulunduğu durumu anlayarak, ihtiyacı olan anda doğru müdahaleyi sunmak."**

Doping Hafıza MT Programı kapsamında tasarlanan **Doping Pulse**, öğrencinin platformdaki öğrenme davranışlarını, alışkanlıklarını ve kopuş sinyallerini gerçek zamanlı ve kümülatif olarak analiz eden; öğrenciyi büyük programlarla boğmak yerine doğru anda nokta atışı mikro-müdahaleler sunan bir **Akıllı Durum ve Karar Motoru (Backend State & Decision Engine)** projesidir.

---

## 🎯 Problem ve Vizyon

Öğrenciler Doping Hafıza'yı satın aldıklarında veya kullandıklarında karşılaşılan en temel 3 senaryo:
1. **Nereden başlayacağını bilememe:** Sistemi açmasına rağmen ilk adımın gözde büyümesi ve aksiyona geçememe.
2. **Ritmin kaybolması:** Çalışmaya devam edilmesine rağmen sürdürülebilir bir temponun yakalanamaması, hedeflerin gerisinde kalma.
3. **Kopuş ve Bırakma:** Düzenli çalışırken bir anda sistemden uzaklaşma ve bunun kalıcı bir terk edişe (churn) dönüşmesi.

Doping Pulse, bu durumları "öğrenciyi yargılamadan veya yetersiz hissettirmeden" proaktif olarak yönetir:

| Durum | Slogan | Tetikleyici Sinyal | Müdahale Yaklaşımı |
| :--- | :--- | :--- | :--- |
| **🟢 ACTIVATE** | *"Başlat."* | Kayıt var, aktivite yok | Net ve çok düşük eşikli mikro-görev ("Bugün sadece 1 video izle") |
| **🟡 BOOST** | *"Ritmini güçlendir."* | Düşük / düzensiz tempo | Hedefi küçültme, motivasyon desteği, hafifletilmiş program adaptasyonu |
| **🔴 RESTART** | *"Koptuysan yeniden başlat."* | Anlamlı ve ani kopuş | 2 soruluk mini check-in $\rightarrow$ kök nedeni anla $\rightarrow$ kişiselleştirilmiş dönüş |
| **⚪ STEADY** | *"Akışta kal."* | Hedef temposunda stabil | Pasif izleme, seri (streak) takibi, gereksiz bildirimle boğmama |

---

## 🏗️ Mimari & Backend Tasarımı Özeti

Bu depo, Doping Pulse'ın backend mimarisini **kodsuz sistem tasarımı (System Design / RFC)** standardında barındırır:

- **State Machine (Durum Makinesi):** 4 durumlu geçiş matrisi ve eşik değerleri (Thresholds).
- **Veri Modeli (Database / ERD):** PostgreSQL uyumlu, sliding-window metrik agregasyonuna dayalı 7 ilişkisel tablo (`pulse_student_states`, `pulse_daily_aggregates`, `pulse_student_interventions`, `pulse_checkin_responses` vb.).
- **Obje Modelleri & DTOs:** Frontend ve iç servisler için tip güvenli sözleşmeler.
- **RESTful API Şartnamesi:** Mobil/Web istemcileri ve dahili event/batch servisleri için endpoint spesifikasyonları.
- **Event-Driven & Batch Mimarisi:** Core monolite/mikroservislere yük bindirmeyen asenkron olay dinleme (Kafka) ve gece çalışan batch tarayıcı (Nightly Inactivity Scanner).

---

## 📂 Doküman Haritası

```text
doping-pulse/
├── README.md                           # Proje genel bakışı ve hızlı başlangıç (Bu dosya)
└── docs/
    └── doping-pulse-technical-spec.md  # Kapsamlı Backend Teknik Tasarım Belgesi (RFC)
```

> 📖 **Detaylı Teknik Şartname:** Tüm veritabanı şemaları, SQL DDL'leri, Mermaid durum/ERD diyagramları ve JSON API sözleşmeleri için [docs/doping-pulse-technical-spec.md](docs/doping-pulse-technical-spec.md) dosyasını inceleyin.

---

## 🚀 Yol Haritası (Sıradaki Adımlar)

- [x] **Faz 1:** Temel Durum Makinesi ve Kural Matrisinin Çıkarılması
- [x] **Faz 2:** Veritabanı Şeması (DDL, ERD, İndeks Stratejisi)
- [x] **Faz 3:** Obje Modelleri (DTOs) ve REST API Sözleşmeleri
- [ ] **Faz 4:** Veli & Koçluk Entegrasyon Senaryoları (Escalation Engine)
- [ ] **Faz 5:** OpenAPI 3.0 / Swagger YAML Spesifikasyonu
- [ ] **Faz 6:** Edge Case Analizi (Örn: Sınav haftası, tatil dönemleri, kasıtlı kapatmalar)

