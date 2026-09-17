# SettlementML

**Sıvılaşma kaynaklı bina oturması ve dönmesi için yeniden üretilebilir Python kodu.**

[English quick start](README_EN.md) · [Veri ve kolon sözlüğü](docs/DATA_SCHEMA.md) · [Model kartı](docs/MODEL_CARD.md) · [Yeniden üretilebilirlik](docs/REPRODUCIBILITY.md) · [Sürüm değişiklikleri](CHANGELOG.md)

Bu depo, **core girdilerle çalışan, ham hedefli ve örnek ağırlıksız ExtraTrees** modelini, veri hazırlamayı, `AnalizNo` gruplu çapraz doğrulamayı, dış vaka değerlendirmesini ve şekil/tablo üretimini içerir.

Modelin öğrendiği iki hedef:

```text
uy_min_cm
uy_fark_cm = uy_max_cm - uy_min_cm
```

Diğer çıktılar bu tahminlerden hesaplanır:

```text
uy_max_pred = uy_min_pred + uy_fark_pred
T_pred = degrees(atan(uy_fark_pred / (100 * B_m)))
```

B metre, deplasmanlar santimetre, T derecedir. `T` bağımsız bir üçüncü regresyon hedefi değildir. Öğrenilen fiziksel büyüklükler negatif çıkarsa sıfırdan aşağıya izin verilmez; sonuçları iyi göstermek için üstten kırpma yapılmaz.

## Önce hangi yolu kullanmalıyım?

| Amacınız | Yol |
|---|---|
| Hazır eğitilmiş modele Excel/CSV verip sonuç almak | **A. Hazır modelle tahmin** |
| Veriyi hazırlamak, doğrulamak ve modeli yeniden eğitmek | **B. Baştan çalışma** |
| Feature engineering, ANN, hedef dönüşümleri ve farklı algoritmaları karşılaştırmak | **C. Ablation ve katsayılı modeller** |

**Önemli sürüm notu:** Arşivdeki gerçek eğitilmiş model **60 ağaç, `random_state=1777`** içeriyor. Eski eğitim sarmalayıcısında farklı olarak 500 ağaç/777 yazıyordu. Bu sürüm, kayıtlı modeli temel alarak konfigürasyonu tek yerde birleştirir. 500 ağaç kullanmak mümkündür, fakat farklı bir deneydir. Ayrıca eski `upper_response` sınıflandırıcı adı düzeltildi; ayrıntılar [CHANGELOG](CHANGELOG.md) içindedir.

## 1. Kurulum

ZIP'i açın. Terminalde **README.md dosyasının bulunduğu ana klasöre** geçin. Klasör içinde yeniden `Downloads/...` yazarak alt klasör aramayın.

Önerilen Python: **3.13**. Paket Python **3.11–3.13** için düzenlenmiştir; yayın kontrolü Python **3.13.5 / Linux** üzerinde çalıştırılmıştır. Windows/macOS için aşağıdaki komutlar sağlanır; bu sistemlerde yerel yürütme kontrolü yapılmış olduğu iddia edilmez. GPU, Microsoft Excel veya LibreOffice gerekmez.

### Windows PowerShell

```powershell
py -3.13 -m venv .venv
.\.venv\Scripts\python.exe -m pip install -r requirements.txt
.\.venv\Scripts\python.exe -m settlementml doctor
```

**Aktivasyon zorunlu değildir.** `Activate.ps1` için execution-policy değiştirmeniz gerekmez. Aşağıdaki komutlarda `python` yerine `.\.venv\Scripts\python.exe` kullanabilirsiniz.

### Linux / macOS

```bash
python3 -m venv .venv
.venv/bin/python -m pip install -r requirements.txt
.venv/bin/python -m settlementml doctor
```

### Conda kullananlar

```bash
conda create -n settlementml python=3.13 -y
conda activate settlementml
python -m pip install -r requirements.txt
python -m settlementml doctor
```

Mevcut araştırma ortamınızı değiştirmek yerine ayrı bir ortam kullanın. `.joblib` dosyalarını yalnızca güvendiğiniz kaynaktan yükleyin. `requirements.txt` kayıtlı modelin kütüphane sürümlerini sabitler; farklı scikit-learn sürümünü sessizce kullanmayın. [Resmî model saklama dokümanı](https://scikit-learn.org/stable/model_persistence.html).

## A. Hazır modelle tahmin

### 2. Örnek girdiyi çalıştırın

```bash
python -m settlementml predict --input data/examples/prediction_long.csv --model models/final.joblib --out outputs/demo
python -m settlementml figures --run outputs/demo
```

Açılacak dosyalar:

```text
outputs/demo/predictions.csv        Tüm girdiler + tahminler + bantlar + uyarılar
outputs/demo/prediction_audit.json  Girdi/model dosya özetleri ve dönüşüm kayıtları
outputs/demo/figures/               PNG ve SVG grafikler
outputs/demo/report.html            Tarayıcıda açılan grafik galerisi
```

`prediction_long.csv` ve Excel şablonundaki doldurulmuş satırlar **geliştirme verisinden alınan kullanım örnekleridir**, bağımsız dış test değildir.

### 3. Kendi Excel dosyanızı verin

`data/examples/Input_Template.xlsx` dosyasının **Inputs** sayfasını doldurun. İlk satır kolon başlıkları olarak kalmalıdır. Örnek satırları kendi binalarınızla değiştirin.

```bash
python -m settlementml predict --input data/examples/Input_Template.xlsx --sheet Inputs --model models/final.joblib --out outputs/my_cases
python -m settlementml figures --run outputs/my_cases
```

Yeni dosyanın adı farklıysa yalnızca `--input` değerini değiştirin. Dosya adında boşluk varsa çift tırnak kullanın.

```bash
python -m settlementml predict --input "data/Benim Vakalarim.xlsx" --sheet Inputs --model models/final.joblib --out outputs/my_cases
```

XLSX okuyucu kaydedilmiş hücre değerlerini okur; makro veya formül yürütmez. `H_B` boşsa kod `H_m/B_m` olarak hesaplar. Diğer zorunlu formül hücrelerinin önbelleği boşsa değer olarak kaydedin veya CSV kullanın. `.xls` ve `.xlsm` desteklenmez.

### 4. Girdi formatı

**Bir satır = bir bina.** Birbirini etkileyen iki binada iki satır, üç binada üç satır kullanılır. Aynı `CaseID` içindeki binaların `BuildingID` ve konumları farklı olmalıdır.

| Alan | Gerekli bilgi |
|---|---|
| `CaseID`, `BuildingID` | Vaka ve bina kimliği; model girdisi değildir |
| `Senaryo` | `Ayrık (Tekil) Bina`, `İki Bina Ayrık`, `İki Bina Bitişik`, `Üç Bina Ayrık`, `Üç Bina Bitişik` |
| `ProfilNo` | `P1` veya `P2` |
| `CAV_cm_s`, `EtkinSure_D5_D95_s`, `AriasIntensity_Ia` | Deprem girdileri, veri kaynağıyla aynı birimlerde |
| `B_m`, `H_m` | Temel genişliği ve bina yüksekliği, metre |
| `H_B` | H/B; boş bırakılabilir, kod hesaplar; hatalı girilmiş oran reddedilir |
| `Q_kPa` | Temel basıncı/yük girdisi, kPa; kat sayısı değildir |
| `w_m` | Ayrık çoklu binalarda pozitif mesafe. Tekil/bitişik için model kodu `0` |
| `Df_m` | `1 = Bodrumsuz`, `3 = Bodrumlu` |
| `BinaKonumu` | Tekilde `Sol Bina`; ikilide `Sol Bina`, `Sag Bina`; üçlüde bunlara `Orta Bina` eklenir |

`w_m=0` bitişik senaryoda bir **model temsil kodudur**; fiziksel aralığın kesin sıfır olduğunu göstermez. Varsa gerçek fiziksel boşluğu `w_physical_m`/`Notes` alanında saklayın. Program bitişik vakayı kendiliğinden ayrık olarak yeniden sınıflandırmaz.

Her bir komşunun B/H/Q/Df değerleri farklıysa ayrı satırlar kullanılmalıdır. Kod bunu kabul eder ve kapsam notu üretir; mevcut modelin **komşu binaların ayrı özelliklerini açık girdiler olarak öğrenmediği** unutulmamalıdır.

Homojen bir sistem için tek satır girip tüm bina konumlarını açtırmak mümkündür:

```bash
python -m settlementml predict --input data/examples/prediction_wide.csv --format wide --model models/final.joblib --out outputs/wide_demo
```

Wide yöntemi bina bazında farklı geometri/yük veya gözlenen çıktı varsa kullanılmaz. Varsayılan long yöntem tam vaka kontrolü yapar. Tek bir bina konumunu özellikle sorgulamak için `--allow-partial-cases` kullanılabilir; bu, tam vaka değerlendirmesinin yerine geçmez.

### 5. Çıktılar ve bantların anlamı

| Çıktı | Anlamı |
|---|---|
| `uy_min_cm_pred`, `uy_fark_cm_pred` | Doğrudan öğrenilen iki büyüklüğün tahmini |
| `uy_max_cm_pred`, `T_deg_pred` | Fiziksel bağıntıyla türetilen çıktılar |
| `*_lo90`, `*_hi90` | OOF hata dağılımından gelen nominal %90 **tanısal** bant |
| `high_response_probability`, `high_response_flag` | `T > 5°` etiketi için yardımcı model skoru ve karar; hasar/güvenlik olasılığı değildir |
| `interval_source` | Global veya üst-response hata havuzundan gelen bant |
| `outside_training_range_flag`, `outside_training_features` | Hangi sayısal girdiler marjinal eğitim sınırları dışında? |
| `unseen_training_levels`, `scope_notes` | Sınırlar içinde olsa bile görülmemiş değerler ve ortak kombinasyon uyarıları |

Bantlar nokta tahminini değiştirmez; ikinci bir regresyon modeline geçiş yapılmaz. **Tanısal bant, ±25/50/75 cm tolerans analizi ve güven aralığı aynı kavramlar değildir.** `lo90/hi90` adındaki 90, yeni her vakada %90 kapsama garantisi vermez. Ayrıntılar: [Model kartı](docs/MODEL_CARD.md).

### 6. Gerçek gözlem varsa değerlendirin

İsteğe bağlı kolonlar:

```text
observed_uy_min_cm, observed_uy_max_cm, observed_T_deg, observed_status
```

`observed_status=numeric`: verilen gözlem sayısal karşılaştırmaya alınır.

`observed_status=non-observable` veya `unknown`: tahmin gösterilir, sayısal hata hesabına alınmaz. Gözlemlenebilir kanıt olmadığı için girilen 0/0/0 kesin sıfır değildir; program bunları otomatik öğrenme hedefi yapmaz. Gerçek sayısal sıfır varsa `numeric` kullanılabilir.

Tüm dış vakaları hazır modelle çalıştırmak:

```bash
python -m settlementml predict --input data/examples/external_cases.csv --model models/final.joblib --out outputs/external_predictions
python -m settlementml evaluate --predictions outputs/external_predictions/predictions.csv --out outputs/external
python -m settlementml figures --run outputs/external
```

Bu örnek dosyada **14 bina** bulunur; iki belirsiz 0/0/0 kayıt nitel olarak korunur ve **12 sayısal kayıt** hata hesabına girer. Kötü tahmin veya geometri sınırı dışındaki kayıtlar değerlendirmeden çıkarılmaz. İki/üç binalı vaka figürlerinde tüm bina konumları gösterilir. `CASE 6` içindeki C34/C35/C36 bağımsız tekil binalardır.

## B. Baştan çalışma

### 7. Ön işleme

Paketle gelen `data/training_qc.csv`, uzman kontrolünden geçmiş **2.325 bina-gözlemi ve 1.099 benzersiz AnalizNo** içerir. 1.120, eski analiz programındaki toplam kimlik sayısıdır; bu nihai tablo için grup sayısı değildir.

```bash
python -m settlementml preprocess --input data/training_qc.csv --out outputs/preprocessed
```

Bu adım kolon adlarını, kategorileri, zorunlu sayısal değerleri, H/B ve hedef tutarlılığını denetler; `Yerlesim`, `BinaSayisi`, `BodrumDurumu` ve Δuy üretir. Çıktılar `training_long.csv`, `preprocess_audit.json`, `data_audit.json` ve senaryo sayılarıdır. **One-hot kodlayıcı bu aşamada tüm veri üzerinde eğitilmez**; her eğitim fold'unda ayrıca fit edilir.

Varsayılan akış önceden onaylanmış tabloyu kullanır; tekrar hedef filtresi uygulamaz. Eski A:U, üç başlık satırlı orijinal Excel'i yeniden işlemek gerekiyorsa:

```bash
python -m settlementml preprocess --input data/original.xlsx --format original --start-row 4 --qc-policy configs/legacy_expert_qc.json --out outputs/original_import
```

Bu açıkça seçilen tarihsel kalite-kontrol politikası ve çıkarılan kayıtlar audit dosyasına yazılır. Politikadaki eşikler evrensel fiziksel/hasar sınırları değildir. Orijinal veri kaynağı ve geçmiş işlem kaydı: [DATA_PROVENANCE.json](data/DATA_PROVENANCE.json).

### 8. GroupKFold doğrulama

```bash
python -m settlementml validate --input outputs/preprocessed/training_long.csv --out outputs/validation
```

Aynı AnalizNo içindeki binalar birlikte tutulur. Her kayıt yalnızca kendi test fold'unda tahminlenir. Temel dosyalar:

```text
oof_predictions.csv                     Her kaydın gözlem, OOF tahmin ve fold'u
fold_audit.csv                          Her fold'da ortak AnalizNo kontrolü
regression_metrics.csv                  uy_min / Δuy / uy_max / T için MAE, MedAE, RMSE, R², bias
tolerance_rates.csv                     Mutlak hata toleransları, pay/payda ve oran
calibration_diagnostics_NOT_independent.csv
validation_manifest.json               Tüm ayarlar, sürümler, veri imzası, kantiller, uyarılar
```

İç OOF hatalarından kantil hesaplayıp aynı hatalar üzerinde kapsama ölçmek **bağımsız bant doğrulaması değildir**. Ek, daha sıkı bir bant kontrolü için:

```bash
python -m settlementml validate --input outputs/preprocessed/training_long.csv --out outputs/validation --nested-bands
```

Burada her dış fold'un bant kalibrasyonu yalnızca o fold'un eğitim gruplarında yapılan iç GroupKFold hatalarından elde edilir. Ek dosyalar: `nested_interval_predictions.csv`, `nested_interval_coverage.csv`. Bu test de kendiliğinden bir conformal/güvenlik garantisi oluşturmaz.

### 9. Son modeli eğitin

```bash
python -m settlementml train --input outputs/preprocessed/training_long.csv --validation outputs/validation --out outputs/models/final.joblib
```

Bütün geliştirme verisi kullanılarak tahmin modeli fit edilir. Model dosyasının yanında okunabilir `final.metadata.json` ve MDI önem tablosu yazılır. Kod, eğitim verisinin ve konfigürasyonunun kalibrasyon dosyasıyla eşleştiğini kontrol eder. Birlikte değerlendirilen değişkenler arasındaki bağımlılıklar nedeniyle MDI bir nedensel etki ölçüsü değildir.

Yeniden eğittiğiniz modelle tahmin için `--model outputs/models/final.joblib` kullanın. Hazır referans modelin yolu `models/final.joblib` olarak kalır ve üzerine yazılmaz.

### 10. Şekil ve tablolar

Tablolar validate/evaluate sırasında CSV olarak üretilir; ayrıca grafik ve HTML galerisi:

```bash
python -m settlementml figures --run outputs/validation
python -m settlementml figures --run outputs/external
```

Dört hedefin ayrı parity grafikleri, tolerans kademeleri, senaryo özetleri, her tam vaka için gözlenen/tahmin edilen oturma ve dönme grafikleri üretilir. Farklı birimler tek sayısal eksene sıkıştırılmaz; `uy_max/20` gibi görselleştirme ölçeklemesi kullanılmaz.

### 11. Hepsini tek komutla çalıştırın

```bash
python -m settlementml all --out outputs/full_run
```

Bu komut ön işleme → doğrulama → eğitim → dış vaka tahmini → değerlendirme → grafik sırasını çalıştırır. Ablation otomatik olarak eklenmez. Daha sıkı bant kontrolü için sonuna `--nested-bands` ekleyin.

## C. Ablation ve katsayılı modeller

### 12. Feature engineering, hedef dönüşümü ve ANN

```bash
python -m settlementml ablation --suite features --out outputs/ablation_features
python -m settlementml ablation --suite targets --out outputs/ablation_targets
python -m settlementml ablation --suite ann --out outputs/ablation_ann
python -m settlementml figures --run outputs/ablation_features
```

Feature deneyi core + tekil eklemeler + selected + full engineered temsilleri karşılaştırır. Hedef deneyinde raw, log1p, asinh50 ve norm_log1p; ANN deneyinde 64–32 ve 128–64 katmanları değerlendirilir. Ek değişkenler sadece girdilerden üretilir. Final varsayılan model core/raw/unweighted olarak kalır.

ANN'lerde bu sürümde gizli satır bazlı `early_stopping` bölmesi yerine sabit eğitim bütçesi kullanılır. Bu nedenle yeni ANN sonuçları eski tabloların birebir tekrarı diye etiketlenmez. Yakınsama uyarıları kaydedilir, bastırılarak yok edilmez.

Kesilen bir deney, veri ve ayarlar eşleşiyorsa devam ettirilebilir:

```bash
python -m settlementml ablation --suite ann --out outputs/ablation_ann --resume
```

### 13. XGBoost, LightGBM ve CatBoost

```bash
python -m pip install -r requirements-optional.txt
python -m settlementml ablation --suite families --out outputs/ablation_families
```

CatBoost kategorileri doğal biçimde alır. Diğer ailelerde ortak one-hot kategorik kodlama kullanılır. Bütün deneyleri çalıştırmak için `--suite all` kullanılır. Kurulu olmayan opsiyonel kütüphaneler `SKIPPED_OPTIONAL_DEPENDENCY` olarak kaydedilir; çalışmış gibi raporlanmaz.

Önemli ablation çıktıları: `ablation_metrics.csv`, `run_status.csv`, `ablation_manifest.json`, her konfigürasyonun OOF tahminleri. Feature karşılaştırmalarında ayrıca AnalizNo bazlı bootstrap ile keşifsel eşleştirilmiş MAE farkları verilir. Bunlar çoklu karşılaştırma düzeltmeli doğrulayıcı testler veya nested model-seçim kanıtı değildir.

### 14. Katsayılı regresyon ve denklem çıktısı

```bash
python -m settlementml coefficients --input data/training_qc.csv --out outputs/coefficients
python -m settlementml formula-predict --input data/examples/prediction_long.csv --formula outputs/coefficients/formula.json --out outputs/formula_predictions.csv
```

Yöntem: **kantil düğümlü toplamsal parçalı doğrusal regresyon, ridge cezası ile**. Karşılaştırma modeli de lineer ridge regresyondur. Bu, adaptif MARS algoritması değildir ve ExtraTrees tahminlerini taklit ederek eğitilmez; sayısal gözlem hedeflerine fit edilir.

`coefficients.csv`, `knots.csv`, `formula.json`, `baseline_metrics.csv` ve `formula_export_check.json` üretilir. JSON, standardize katsayıların yanı sıra ham-baz katsayılarını ve doğru sabit terim düzeltmesini içerir. Böylece tam denklem yeniden uygulanabilir; yalnızca en büyük birkaç katsayı ile tahmin yapıldığı iddia edilmez.

## Run butonu ile kullanım

IDE'nizin Python yorumlayıcısını bu projenin `.venv` ortamı olarak seçin. Dosyalar çalışma klasörünü kendileri proje köküne alır.

| Dosya | İşlev |
|---|---|
| `RUN_00_INSTALL.py` | Seçili Python ortamına temel bağımlılıkları kurar |
| `RUN_01_PREPROCESS.py` | Veri hazırlama ve denetim |
| `RUN_02_VALIDATE.py` | GroupKFold doğrulama |
| `RUN_03_TRAIN.py` | Yeniden eğitim |
| `RUN_04_PREDICT.py` | Hazır modelle dış örnek CSV tahmini; model yolu dosyada değiştirilebilir |
| `RUN_PREDICT_EXCEL.py` | Excel dosya/sayfa adını düzenleyip tahmin |
| `RUN_05_EVALUATE_CASES.py` | Gözlenen/tahmin edilen değerlerin karşılaştırılması |
| `RUN_06_FIGURES_TABLES.py` | Üretilmiş validation/external çıktılarından grafik |
| `RUN_07_FEATURE_ABLATION.py` | Tekil/blok feature deneyi |
| `RUN_08_ANN_ABLATION.py` | ANN ve hedef dönüşümü |
| `RUN_09_COEFFICIENTS.py` | Katsayılı regresyon ve denklem dışa aktarımı |
| `RUN_ALL.py` | Ablation hariç tam akış; kendi eğittiği yeni modeli kullanır |

## Klasör yapısı

```text
SettlementML_GitHub/
├── README.md / README_EN.md
├── configs/                 Tek final konfigürasyonu ve açık tarihsel QC politikası
├── data/
│   ├── training_qc.csv       Onaylanmış geliştirme verisi
│   ├── DATA_PROVENANCE.json  Kaynak ve SHA256 kayıtları
│   └── examples/            Excel/CSV şablonları ve tüm dış vakalar
├── settlementml/             Python paketi ve CLI
├── models/                  Değiştirilmemiş hazır nokta modeli, metadata
├── docs/                    Veri/model kartları, protokol ve yayın kontrol listesi
├── reference/               Eski sonuçlar, yeni sürüm kontrol çıktıları ve manifestleri
├── tests/                   Otomatik kontroller
├── scripts/                 Sürüm/örnek çıktı denetimleri
├── .github/workflows/       Windows/Linux test matrisi tanımı
└── outputs/                 Kullanıcı çalıştırınca oluşur; Git'e eklenmez
```

## Arşiv sonuçlarını nasıl kontrol ederim?

```bash
python -m settlementml reference-check
python -m pip install -r requirements-dev.txt
python -m pytest
python scripts/verify_release.py
```

Arşivdeki OOF tahminlerinden önceki MAE değerleri yeniden hesaplanabilir. Bununla **orijinal OOF modellerinin aynı tohumlarla baştan fit edildiği** iddia edilmez; önceki dosyalar tam fold-tohum kaydını içermiyordu. Bu sürüm her yeni deneyin tohumlarını ve veri imzasını kaydeder. Eski sonuçlarla bu sürümde çalıştırılmış sonuçlar ayrı klasörlerdedir. Ayrıntılar: [REPRODUCIBILITY.md](docs/REPRODUCIBILITY.md).

## Bilimsel kapsam

Eğitimde B yalnızca 9 ve 12 m, H/B 1–2 aralığındadır; üç binalı sistemlerde yalnızca Df=1 temsil edilmiştir. Başka değerlerle yazılım tahmin verebilir, fakat bu eğitim desteği genişledi demek değildir. Marjinal sınırlar içinde kalan fakat görülmemiş ortak geometri/yük kombinasyonları da vardır. Kapsam notları bu nedenle yalnızca tek bir range bayrağından ibaret değildir.

Dış vaka performansı, nümerik geliştirme OOF başarımından ayrı raporlanır. Varsayılan değerlendirme tüm vakaları korur. İyi görünen sonuçları seçip genel başarı gibi sunmak, dış sonuçlara bakarak bant genişletmek veya belirsiz gözlemleri sıfıra dönüştürmek uygulanmaz. ±25/50/75 cm oranları keşifsel tolerans analizi olarak adlandırılır; mühendislik kabul sınırları olduğu varsayılmaz.

## GitHub'a yayımlama

ZIP dosyasının kendisini tek dosya olarak yüklemek yerine **içindeki proje klasörünün içeriğini** depoya koyun. `README.md` depo kökünde bulunmalıdır. `outputs/` ve sanal ortam `.gitignore` ile dışarıda tutulur.

Yayımlamadan önce [PUBLICATION_CHECKLIST.md](docs/PUBLICATION_CHECKLIST.md) içindeki veri/model paylaşım izni, lisans ve makale atıf bilgilerini ekipçe tamamlayın. Hazırlama sırasında sizin adınıza veri lisansı, DOI, yazar listesi veya hayali GitHub bağlantısı oluşturulmadı.

## Resmî yazılım kaynakları

- [ExtraTreesRegressor](https://scikit-learn.org/stable/modules/generated/sklearn.ensemble.ExtraTreesRegressor.html)
- [GroupKFold](https://scikit-learn.org/stable/modules/generated/sklearn.model_selection.GroupKFold.html)
- [Model saklama ve sürüm güvenliği](https://scikit-learn.org/stable/model_persistence.html)
- [MLPRegressor](https://scikit-learn.org/stable/modules/generated/sklearn.neural_network.MLPRegressor.html)
- [GitHub dosya boyutu kuralları](https://docs.github.com/en/repositories/working-with-files/managing-large-files/about-large-files-on-github)

Bu belgeler yazılım API davranışlarının kaynaklarıdır; projenin deneysel sonuçları depodaki CSV ve manifest dosyalarına dayanır.
