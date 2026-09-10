
# Desi Tahmin — Temel İşlevler Aşaması

**TEKNOFEST 2026 — Hepsiburada Mid-Mile Linehaul Optimizasyonu**  
Temel İşlevler Aşaması · talep / hacim tahmini katmanı

Rota bazlı günlük kargo hacmini (**desi**) tarihsel veriden öğrenip yarışma haftası (**11–17 Mayıs 2026**) için tahmin eden XGBoost hattı. Tek model kullanılır; vardiya ayrımı yoktur. Çıktı Excel, aynı paketteki OR-Tools sevk planlayıcısına gider.

Gelişmiş çözüm aşamasındaki sabah/akşam izole modeller ve recursive 7 günlük döngü burada yoktur. Bu katman **tek şampiyon model + lag/rolling özellikler + akıllı lag geri çekilmesi** ile temel teslimi üretir.

---

## Ne işe yarar?

Planlayıcı her rota ve gün için bir desi ister. Bu depo onu üretir:

1. `Model_Train.ipynb` tarihsel master Excel ile modeli eğitir, encoder + XGBoost’u `.pkl` kaydeder.
2. `Model_Predict.py` 11–17 Mayıs 2026 iskeletini üretir, `lag_7` / `lag_14` değerlerini geçmişten (kriz/tatil atlayarak) doldurur, tahmin yazar.
3. Çıktı `11_17_MAYIS_YARISMA_TESLIM_FINAL.xlsx` → `Tarih`, `Rota_Adi`, `Tahmin_Edilen_Desi`.

Hedef kolon: **`Toplam_Desi`**. Tahmin negatif olamaz (`np.maximum(..., 0)`). Log dönüşümü **kullanılmaz**; model doğrudan desi uzayında eğitilir.

---

## Mimari

```
MODEL1_READY_MASTER_DATASET_2_ESKİ.xlsx
        │  sentetik satırlar atılır
        │  son 7 gün hold-out test
        ▼
TargetEncoder (Rota_ID)  +  XGBRegressor
        │
        ├── target_encoder.pkl
        └── xgboost_sampiyon_model.pkl
                    │
MODEL_READY_MASTER_DATASET_.xlsx
        │  11–17 Mayıs iskeleti
        │  temiz_lag_bul (kriz/tatil atla, aynı gün fallback)
        ▼
11_17_MAYIS_YARISMA_TESLIM_FINAL.xlsx
        │
        ▼
optimizasyon_v2_son_hafta.py   (sevk planı)
```

---

## Dosyalar

| Dosya | Rol |
|---|---|
| `Model_Train.ipynb` | Eğitim, RandomizedSearchCV, metrik, `.pkl` kayıt. Colab. |
| `Model_Predict.py` | Yarışma haftası tahmini. Script’in bulunduğu klasördeki Excel ve pkl’leri okur. |
| `MODEL1_READY_MASTER_DATASET_2_ESKİ.xlsx` | Eğitim girdisi (notebook yolu). |
| `MODEL_READY_MASTER_DATASET_.xlsx` | Tahmin girdisi (`Model_Predict.py`). |
| `target_encoder.pkl` / `01_target_encoder.pkl` | `Rota_ID` hedef ortalaması. |
| `xgboost_sampiyon_model.pkl` / `01_xgboost_sampiyon_model.pkl` | Şampiyon XGBoost. |

Eğitim varsayılan kayıt adları `xgboost_sampiyon_model.pkl` ve `target_encoder.pkl`. Tahmin betiği `01_` önekli adlar bekler. Dosyaları aynı isimle kopyalayın veya `Model_Predict.py` içindeki sabitleri güncelleyin.

---

## Sütunlar

İsimler master Excel’den gelir. Eğitimde `Toplam_Desi` ve gizlenen kolonlar düşülür; kalan her şey XGBoost’a gider.

### Hedef

| Sütun | Amaç |
|---|---|
| `Toplam_Desi` | O rota, o gündeki hacim. Eğitimde `y`, tahminde üretilmez (çıktıda `Tahmin_Edilen_Desi`). |

### Modele giren özellikler (bu koşudaki nihai liste)

Notebook çıktısı:

`Rota_ID`, `lag_7`, `lag_14`, `is_holiday`, `tatil_sonrasi_mi`, `Kriz_Mi`, `Ay`, `Ayin_Gunu`, `Haftanin_Gunu`, `rolling_7_mean`, `rolling_7_std`, `rolling_14_mean`, `rolling_14_std`

| Sütun | Anlamı | Neden var |
|---|---|---|
| `Rota_ID` | Çıkış–varış çifti kimliği. Kategorik. | Rotanın tarihsel ortalama hacmini taşır. `TargetEncoder(smoothing=10)` ile eğitim desi ortalamasına çevrilir; test/tahminde yalnız `transform`. |
| `lag_7` | 7 gün önceki (mümkünse “temiz” gün) desi. | En güçlü sinyal (gain sıralamasında 1.). Aynı haftanın aynı gün karakteri. |
| `lag_14` | 14 gün önceki temiz desi. | İkinci haftalık tekrar. |
| `is_holiday` | Resmi tatil bayrağı. | Tatilde hacim düşer. Gain’de 2. |
| `tatil_sonrasi_mi` | Tatil ertesi. | Birikmiş yük / toparlanma. |
| `Kriz_Mi` | Kriz / bozulma günü. | Anormal hacmi işaretler; lag hesabında da atlanır. |
| `Ay`, `Ayin_Gunu`, `Haftanin_Gunu` | Takvim. | Sezon ve hafta içi/sonu. `Haftanin_Gunu` pandas `dayofweek` (0–6) veya 1–7; tahmin betiği master’daki max’e göre kaydırır. |
| `rolling_7_mean`, `rolling_7_std` | 7 günlük kayan ortalama / std. | Kısa dönem seviye ve oynaklık. |
| `rolling_14_mean`, `rolling_14_std` | 14 günlük kayan ortalama / std. | Daha uzun pencere. |

Tahminde encoder’da varsa ayrıca `Çıkış Transfer Merkezi` / `Varış Transfer Merkezi` de encode edilebilir; bu koşuda modele yalnızca `Rota_ID` girmiş.

Gain (eğitim koşusu, büyükten küçüğe): `lag_7` ≫ `is_holiday` > `tatil_sonrasi_mi` > `Haftanin_Gunu` > `Rota_ID` > `lag_14` > `Kriz_Mi` > takvim ve rolling’ler.

### Bilerek dışarıda bırakılanlar

| Sütun | Neden modelde yok |
|---|---|
| `Tarih` | Split ve iskelet için; ham tarih ezber üretir. |
| `Rota_Adi` | Metin rota adı; model `Rota_ID` kullanır. Çıktı raporunda tutulur. |
| `Sentetik_Mi` | Sentetik satırlar eğitimden **tamamen silinir** (`== 0` kalanlar). Kolon da `X`’ten düşülür. |
| `eski_lag_*` | Denetim kopyaları (`eski_lag_7_tatil`, `eski_lag_7_kriz`, …). Sızıntı / çift sayım. |

Eksik `Toplam_Desi` satırları `dropna` ile atılır.

---

## Model nasıl eğitilir?

`Model_Train.ipynb` → `train_xgboost_colab_pro_nolog`:

1. Excel yükle, `Toplam Desi` → `Toplam_Desi`, `Tarih` datetime.
2. `Sentetik_Mi == 1` satırlarını at (bu koşuda 590 satır).
3. Hedefi boş olanları at.
4. **Zaman hold-out:** `max(Tarih) − 6 gün` kesim. Train: kesimden önce. Test: kesim ve sonrası (son 7 gün).  
   Bu koşu: train bitiş **2026-05-03**, test **2026-05-04 … 2026-05-10**.
5. `X`: gizlenen kolonlar düşülmüş tablo. `y`: ham `Toplam_Desi` (log yok).
6. `TargetEncoder` yalnız train’de `fit_transform`.
7. **XGBRegressor** `objective='reg:squarederror'`, `tree_method='hist'`, mümkünse `device='cuda'`, olmazsa CPU.
8. **RandomizedSearchCV:** 150 aday × 3-fold = 450 fit. Skor `neg_mean_absolute_error`. `cv=3` sklearn varsayılan KFold’dur (TimeSeriesSplit değil).
9. Test tahmini `max(pred, 0)`. Encoder ve model `joblib` kaydı. Gain tablosu + gerçek vs tahmin Excel.

Hiperparametre ızgarası:

| Parametre | Adaylar |
|---|---|
| `n_estimators` | 100 … 500 (50’şer) |
| `learning_rate` | 0.01, 0.02, 0.03, 0.05, 0.08, 0.1, 0.15 |
| `max_depth` | 4–9 |
| `subsample` | 0.65, 0.75, 0.85, 0.95, 1.0 |
| `colsample_bytree` | 0.65, 0.75, 0.85, 0.95, 1.0 |
| `min_child_weight` | 1, 3, 5, 7, 10 |
| `gamma` | 0, 1, 5, 10 |

Bu koşudaki şampiyon:

`n_estimators=400`, `learning_rate=0.08`, `max_depth=5`, `subsample=0.65`, `colsample_bytree=1.0`, `min_child_weight=7`, `gamma=10`.

### Hold-out metrikler (notebook)

| Metrik | Değer |
|---|---|
| MAE | 2622.00 desi |
| RMSE | 3848.51 desi |
| R² | 0.8815 |
| WAPE | %16.89 |
| 1 − WAPE | %83.11 |

WAPE = Σ\|y − ŷ\| / (Σy + ε).

---

## Tahmin (`Model_Predict.py`) — akıllı lag

Yarışma penceresi kodda sabittir: **2026-05-11 … 2026-05-17**.

1. Master okunur, kolon adları normalize edilir (`Tarih`, `Rota_Adi`).
2. Her rota için **son tarihli satır** klonlanır; 7 güne yayılır. Takvim kolonları (`Haftanin_Gunu`, `Ay`, `Gun`) o güne yazılır. `is_holiday`, `Kriz_Mi`, `tatil_sonrasi_mi` iskelette **0**.
3. `lag_7` ve `lag_14` için `temiz_lag_bul`:
   - Hedef günden N gün geriye bak.
   - Kayıt kriz, tatil veya tatil ertesi ise **7 gün daha geri** git (en fazla 5 deneme).
   - Hâlâ yoksa aynı `dayofweek` üzerindeki en son geçmiş desi.
   - O da yoksa rotanın son bilinen desisi; yoksa 0.
4. Modelin `feature_names_in_` sırasına hizala, eksik kolon 0.
5. Encoder `transform` → predict → `max(0)` → 2 ondalık yuvarla → Excel.

Bu, eksik gün ve tatil lag’inin şişirmesini kesmek için “smart fallback”tır. Gelişmiş aşamadaki gün gün recursive enjeksiyon yoktur; 7 gün **paralel** tahmin edilir, lag’ler yalnızca tarihsel master’dan gelir.

---

## Kurulum ve çalıştırma

Python 3.8+ (3.10+ önerilir). GPU eğitim için CUDA’lı XGBoost; yoksa notebook CPU’ya düşer.

```bash
pip install pandas numpy xgboost scikit-learn category_encoders openpyxl joblib
```

Eğitim (Colab veya Jupyter): `Model_Train.ipynb` içinde `GIRDI_MATRISI` yolunu kendi Excel’inize çekin.

Tahmin (pkl’ler ve master, script ile aynı klasörde):

```bash
python Model_Predict.py
```

Çıktı: `11_17_MAYIS_YARISMA_TESLIM_FINAL.xlsx`.

Eğitim 150 × 3 fit sürdüğü için dakikalar–saatler sürebilir.

---

## Klasör önerisi (GitHub)

```
DESI-TAHMIN/
├── README.md
├── Model_Train.ipynb
├── Model_Predict.py
├── MODEL_READY_MASTER_DATASET_.xlsx    # büyük; LFS veya release
├── 01_target_encoder.pkl
└── 01_xgboost_sampiyon_model.pkl
```

Excel ve pickle’ları Git LFS veya release’e koyun.

---

## Tasarım seçimleri (kısa)

- **Tek model:** Temel aşamada vardiya yok; günlük rota hacmi yeterli.
- **Saf desi, log yok:** Ters dönüşüm hatası olmasın; MAE doğrudan operasyonel sapma.
- **Sentetik satır silme:** Modelin uydurma hacmi öğrenmesini keser.
- **Target encoding:** Yüksek kardinaliteli rota; encoder yalnız train’de fit.
- **Temiz lag:** Tatil/kriz gününün desisi gelecek haftaya kopyalanmasın.
- **MAE odaklı 150 iterasyonluk arama:** Temel teslimde geniş hiperparametre taraması.

Optimizasyon ve maliyet rakamları bu klasörün dışında, `OPTİMİZASYON/` altındadır.
=======
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
Yarışma formatı gereği gelecekteki bir haftanın (Örn: 11-17 Mayıs) tahmin edilmesi için sıfırdan bir "Gelecek İskeleti" (Future Skeleton) oluşturulur[cite: 2]. Bu aşamada, lojistik operasyonların doğası gereği basit geçmiş veriler (lag) kullanmak modeli yanıltacağı için **Hibrit ve İteratif Lag Algoritması** geliştirilmiştir.

Script şu adımlarla çalışır:

1.  **Dinamik İskelet Üretimi:** 11-17 Mayıs tarihleri için boş bir veri çerçevesi oluşturulur ve `Haftanin_Gunu`, `Ay`, `Gun` gibi takvimsel özellikler koda dayalı olarak otomatik türetilir[cite: 2].
2.  **Kritik Aşama: İteratif ve Gün Eşlemeli Lag Hesabı:**
    Lojistikte 7 gün önceki veriyi (`lag_7`) referans almak standarttır. Ancak 7 gün öncesi bir resmi tatil veya kriz günü ise, bu referans bugünün tahminini bozar. Bunu çözmek için `temiz_lag_bul` adında özel bir fonksiyon yazılmıştır.
    *   **İteratif Kriz/Tatil Atlama:** Algoritma 7 gün (veya 14 gün) geçmişe gider. Eğer o gün `is_holiday == 1`, `Kriz_Mi == 1` veya `tatil_sonrasi_mi == 1` ise, o veriyi çöpe atar ve **otomatik olarak bir 7 gün daha geriye gider**
    *   Temiz (olağan) bir operasyon günü bulana kadar bu işlemi geçmişe dönük 5 defa (5 hafta) tekrarlar[cite: 2]. Böylece model, bayram durgunluklarını normal bir iş gününe kopyalamaktan kurtulur.
    *   **Gün Eşlemeli Fallback (Güvenlik Ağı):** Eğer aranan tarih veritabanında hiç yoksa, algoritma rastgele bir geçmiş veri almak yerine, ilgili rotanın **geçmişteki en son aynı gününe** (Örn: Tahmin edilecek gün Salı ise, geçmişteki en yakın Salı gününe) giderek veriyi çeker.
3.  **Enjeksiyon ve Tahmin:** Hazırlanan bu kusursuz ve temiz matris, önceden eğitilmiş Target Encoder ve XGBoost modeline sokulur.
4.  **Post-Processing:** Lojistikte eksi hacim olamayacağı için `np.maximum(tahminler, 0)` ile negatif değerler 0'a yuvarlanır ve sonuçlar formatlanarak teslim dosyası (`11_17_MAYIS_YARISMA_TESLIM_FINAL.xlsx`) olarak dışa aktarılır.

