# Teknofest Logistics Volume Prediction (Aşama 1: Günlük Temel Çözüm)

Bu proje, Teknofest Lojistik Anahat Optimizasyonu yarışması kapsamında, Türkiye geneli lojistik rotalarının **günlük yük hacimlerini (desi)** tahmin etmek için geliştirilmiş makine öğrenmesi boru hattının 1. Aşama (Temel İşlevli) kodlarını içermektedir.

Bu aşamada vardiya (sabah/akşam) ayrımı gözetilmeksizin, rotaların "günlük toplam" taşıma kapasiteleri zaman serisi algoritmaları ve XGBoost kullanılarak modellenmiştir.

---

## 🛠️ Özellik Mühendisliği (Feature Engineering) ve Sütun Mimarisi

Bir lojistik ağında yarının yükünü tahmin etmek, sadece geçmişteki sayılara bakmakla değil; o sayıların "hangi koşullarda" oluştuğunu anlamakla mümkündür. Bu nedenle ham veri işlenerek model için **dinamik, istatistiksel ve bağlamsal (contextual)** yeni sütunlar (features) üretilmiştir. 

`MODEL_READY_MASTER_DATASET_.xlsx` dosyasında bulunan ve modele yön veren sütunların mimari amaçları şunlardır:

### 1. Kimlik ve Hedef Değişkenler
* **`Rota_ID` / `Rota_Adi`:** Türkiye genelindeki rotaların kimlik bilgileri. `Rota_ID` algoritmanın işlemesi için sayısal olarak encode edilmiştir.
* **`Tarih`:** Zaman serisi dizilimi ve geçmiş verilerin (lag) doğru eşleşmesi için kullanılan ana zaman indeksi.
* **`Toplam Desi`:** Modelin tahmin etmeye çalıştığı (target) ana değer; yani o gün o rotada taşınan toplam hacim.

### 2. Gecikme (Lag) Özellikleri
Lojistik kargo akışında en belirgin desen haftalık döngülerdir (Örn: Pazartesi yükü genellikle bir önceki Pazartesi ile benzerdir).
* **`lag_7`:** Tam 1 hafta önceki (7 gün önce) gerçekleşen desi hacmi.
* **`lag_14`:** Tam 2 hafta önceki (14 gün önce) gerçekleşen desi hacmi.


### 3. Takvim ve Olay Özellikleri
* **`Haftanin_Gunu`, `Ayin_Gunu`, `Ay`:** Operasyonel mevsimselliği, ay sonu hedeflerini ve hafta içi/hafta sonu çalışma dinamiklerini modelin algılamasını sağlar.
* **`is_holiday`:** Tahmin edilen günün resmi veya dini tatil olup olmadığını belirtir.
* **`tatil_sonrasi_mi`:** Tatillerde biriken yükün patlama yaptığı "tatil dönüşü" günlerinin tespiti için eklenmiştir.
* **`Kriz_Mi`:** Doğal afet, sistem çökmesi gibi kargo hacmini anormal etkileyen günlerin etiketidir.

### 4. Dinamik İstatistiksel Özellikler (Rolling Features)
Rotalardaki genel eğilimi (trend) ve operasyonel istikrarı ölçmek için hesaplanmıştır:
* **`rolling_7_mean` / `rolling_14_mean`:** İlgili rotanın son 1 ve 2 haftalık hacim ortalaması. Kısa ve orta vadeli trendi yakalar.
* **`rolling_7_std` / `rolling_14_std`:** İlgili rotanın son 1 ve 2 haftalık standart sapması. (Rota stabil mi çalışıyor, yoksa sürekli büyük dalgalanmalar mı yaşıyor?)

### 5. Veri Kalitesi Özellikleri
* **`Sentetik_Mi`:** Lojistik verilerinde bazen eksik veya sistemsel hatadan dolayı 0 gelen günler olabilir. Bu günlerin matematiksel yöntemlerle (imputation) doldurulduğunu modele dürüstçe bildiren bir bayraktır. Model, sentetik veriye orijinal veri kadar güvenmemesi gerektiğini bu sayede öğrenir.

---

## ⚙️ Boru Hattı (Pipeline) ve İteratif Tahminleme Mantığı

Projenin makine öğrenmesi boru hattı, eğitimi ve geleceğe yönelik ardışık tahminlemeyi otomatize eden iki ana modülden oluşmaktadır:

### 1. Model Eğitimi (`model_train.py`)
Bu modül, tarihsel veriyi alarak şampiyon modeli üretmekten sorumludur:
*   **Target Encoding:** Kategorik `Rota_Adi` sütunu, hedef değişken (`Toplam Desi`) baz alınarak sayısal ağırlıklara dönüştürülür. Veri sızıntısını önlemek için encoder sadece eğitim verisine uyarlanır (`fit_transform`) ve diskte `.pkl` olarak saklanır.
*   **Hiperparametre Optimizasyonu ve Eğitim:** XGBoost algoritması, TimeSeriesSplit (Zaman Serisi Çapraz Doğrulaması) ve RandomizedSearchCV ile optimize edilir. En düşük hata payına (MAE/RMSE) sahip model `01_xgboost_sampiyon_model.pkl` olarak kaydedilir.

### 2. Gelecek Tahmini ve İteratif Mantık (`Model_Predict.py`)
Yarışma formatı gereği gelecekteki bir haftanın (Örn: 11-17 Mayıs) tahmin edilmesi için sıfırdan bir "Gelecek İskeleti" (Future Skeleton) oluşturulur[cite: 2]. Bu aşamada, lojistik operasyonların doğası gereği basit geçmiş veriler (lag) kullanmak modeli yanıltacağı için **Hibrit ve İteratif Lag Algoritması** geliştirilmiştir[cite: 2].

Script şu adımlarla çalışır:

1.  **Dinamik İskelet Üretimi:** 11-17 Mayıs tarihleri için boş bir veri çerçevesi oluşturulur ve `Haftanin_Gunu`, `Ay`, `Gun` gibi takvimsel özellikler koda dayalı olarak otomatik türetilir[cite: 2].
2.  **Kritik Aşama: İteratif ve Gün Eşlemeli Lag Hesabı:**
    Lojistikte 7 gün önceki veriyi (`lag_7`) referans almak standarttır. Ancak 7 gün öncesi bir resmi tatil veya kriz günü ise, bu referans bugünün tahminini bozar. Bunu çözmek için `temiz_lag_bul` adında özel bir fonksiyon yazılmıştır[cite: 2]:
    *   **İteratif Kriz/Tatil Atlama:** Algoritma 7 gün (veya 14 gün) geçmişe gider. Eğer o gün `is_holiday == 1`, `Kriz_Mi == 1` veya `tatil_sonrasi_mi == 1` ise, o veriyi çöpe atar ve **otomatik olarak bir 7 gün daha geriye gider**[cite: 2].
    *   Temiz (olağan) bir operasyon günü bulana kadar bu işlemi geçmişe dönük 5 defa (5 hafta) tekrarlar[cite: 2]. Böylece model, bayram durgunluklarını normal bir iş gününe kopyalamaktan kurtulur.
    *   **Gün Eşlemeli Fallback (Güvenlik Ağı):** Eğer aranan tarih veritabanında hiç yoksa, algoritma rastgele bir geçmiş veri almak yerine, ilgili rotanın **geçmişteki en son aynı gününe** (Örn: Tahmin edilecek gün Salı ise, geçmişteki en yakın Salı gününe) giderek veriyi çeker[cite: 2].
3.  **Enjeksiyon ve Tahmin:** Hazırlanan bu kusursuz ve temiz matris, önceden eğitilmiş Target Encoder ve XGBoost modeline sokulur[cite: 2].
4.  **Post-Processing:** Lojistikte eksi hacim olamayacağı için `np.maximum(tahminler, 0)` ile negatif değerler 0'a yuvarlanır ve sonuçlar formatlanarak teslim dosyası (`11_17_MAYIS_YARISMA_TESLIM_FINAL.xlsx`) olarak dışa aktarılır[cite: 2].
