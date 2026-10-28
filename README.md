# Haber Grubu Konu Sınıflandırma (Çok Sınıflı)

İngilizce forum mesajlarının dört konudan hangisine ait olduğunu tahmin eden bir makine öğrenmesi çalışması. TF-IDF ve Logistic Regression kullanıldı.

## Veri

- **Kaynak:** 20 Newsgroups veri seti (scikit-learn `fetch_20newsgroups`).
- **Seçilen 4 kategori:** `comp.graphics`, `rec.sport.hockey`, `sci.space`, `talk.politics.misc`.
- **Temizlik:** Mesajların başlık, imza ve alıntı kısımları silindi (`remove=("headers", "footers", "quotes")`). Böylece model kategori ipuçlarını bu kısımlardan değil, mesajın asıl metninden öğrenmek zorunda kalır.
- **Bölme:** Veri setinde hazır gelen train (2.242 mesaj) ve test (1.492 mesaj) kullanıldı. Train'in %20'si `stratify` ile validation olarak ayrıldı.

| Parça | Mesaj sayısı | Kullanım |
|---|---|---|
| Train | 1.793 | Modeli eğitmek |
| Validation | 449 | Kontrol ve model karşılaştırma |
| Test | 1.492 | Son ölçüm (yalnızca bir kez bakıldı) |

Sınıflar büyük ölçüde dengeli (train'de hockey 600, space 593, graphics 584, politics 465).

## Yöntem

1. **Metni sayıya çevirme:** `TfidfVectorizer` (küçük harf, İngilizce stop word'ler çıkarıldı, en az 2 mesajda geçen kelimeler). Sözlükte 12.401 özellik oluştu.
2. **Sızıntıyı önleme:** Vektörleştirici yalnızca train üzerinde `fit` edildi; validation ve test'e sadece `transform` uygulandı.
3. **Model:** `LogisticRegression` (çok sınıflı). Her kelime için her konuya ait bir ağırlık öğrenir.

## Model karşılaştırması (5-fold çapraz doğrulama, macro F1)

| Model | Ortalama F1 | Sapma |
|---|---|---|
| Logistic Regression | 0,896 | 0,011 |
| Naive Bayes | 0,892 | 0,015 |
| Linear SVM | 0,897 | 0,008 |

Üç model arasındaki fark (0,001 - 0,005), çapraz doğrulama sapmasından küçük. Yani modeller pratikte eşit performans gösteriyor. **Logistic Regression** seçildi, çünkü olasılık değeri verir (belirsiz tahminleri ayırt etmek için gerekli) ve skoru diğerlerinden anlamlı biçimde geride değil.

Çapraz doğrulamada her turda vektörleştirici baştan eğitilsin diye `Pipeline` kullanıldı.

## Sonuçlar

| | Accuracy | Macro F1 |
|---|---|---|
| Validation | 0,895 | 0,89 |
| Test | 0,883 | 0,88 |

Test sonucu validation'a çok yakın; model validation'a aşırı uyum sağlamamış.

Test, kategori bazında:

| Kategori | Precision | Recall | F1 |
|---|---|---|---|
| comp.graphics | 0,92 | 0,90 | 0,91 |
| rec.sport.hockey | 0,97 | 0,91 | 0,94 |
| sci.space | 0,78 | 0,90 | 0,83 |
| talk.politics.misc | 0,88 | 0,80 | 0,84 |

Test confusion matrix (satır: gerçek, sütun: tahmin):

|  | graphics | hockey | space | politics |
|---|---|---|---|---|
| **graphics** | 352 | 2 | 30 | 5 |
| **hockey** | 4 | 364 | 20 | 11 |
| **space** | 21 | 2 | 353 | 18 |
| **politics** | 4 | 9 | 49 | 248 |

## Hata analizi

- **Ana hata:** Politika mesajlarının önemli bir kısmına "uzay" deniyor (validation'da 7, test'te 49 mesaj). Bu eğilim iki sette de görüldü. Uzay sınıfının precision değeri bu yüzden en düşük (test'te 0,78).
- **Validation'daki hatalı mesajlara bakış:** Hatalı mesajların bir kısmı boş veya çok kısa ("_TOO_ many.", sadece boşluk). Bu mesajlarda konuyu belirleyen kelime yok, bir insan da bunları sınıflandıramaz.
- **Boş mesaj testi:** Boş bir mesaj için model olasılıkları 0,21 ile 0,29 arasında verdi (uzay 0,286 ile önde). Yani bilgisiz girdide model neredeyse rastgele tahmin yapıyor ve küçük bir farkla uzay'ı seçiyor.
- **Etiket belirsizliği:** `talk.politics.misc` karışık (misc) bir kategori. Örneğin cinsellik anketi gibi siyasetle sınırlı ilgisi olan bir mesaj bu kategoride yer alıyor. Bazı hatalar modelden değil, etiketin kendisinden kaynaklanıyor olabilir.

Not: Boş mesaj testi tek bir sonuca dayanıyor; "bilgisiz mesajlarda uzay'a kayma" bir hipotez olarak sunulmuştur ve tüm hataları açıklamaz.

## Sınırlılıklar ve olası iyileştirmeler

- Maksimum olasılığı düşük (örneğin 0,5'in altında) olan mesajlar için "belirsiz" denilebilir (çekimser kalma).
- Çok kısa veya boş mesajlar eğitim ve değerlendirmeden filtrelenebilir.
- Önceden eğitilmiş bir dil modeli (BERT gibi) kelime sıklığı yerine anlama baktığı için politika ve uzay gibi örtüşen konularda daha iyi olabilir.
- Veri İngilizce ve eski forum mesajlarından oluşuyor; güncel metinlerde performans farklı olacaktır.

## Dosyalar

- `haber_kategorisi.ipynb`: Tüm adımları içeren notebook (Google Colab'da çalıştırıldı).
- `requirements.txt`: Gerekli kütüphaneler.
