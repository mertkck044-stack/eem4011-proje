# Görüntü küçüldükçe Otsu ile lezyon sınırı ne zaman bozuluyor? (HAM10000, melanom ve nevüs)

EEM4011 dersi projesi. Gri tonlama + Otsu eşikleme ile lezyon bölütleme yapılır, sonuç uzman maskesine göre Dice ile ölçülür. Makine öğrenmesi (sınıflandırıcı eğitimi) kullanılmaz.

## Veri seti

- **Ad:** HAM10000 (dermoskopi görüntüleri + metadata) ve `HAM10000_segmentations_lesion_tschandl` (uzman lezyon maskeleri)
- **İndirme:** https://dataverse.harvard.edu/dataset.xhtml?persistentId=doi:10.7910/DVN/DBW86T
- **Veri bu depoda yoktur.** Kod yalnız gereken 4 dosyayı indirir: metadata (`meta.csv`), maskeler, görüntü parça 1 ve parça 2.

## Yöntem (kısaca)

1. Metadata'dan yalnız `mel` ve `nv` seçilir. Lezyon başına ilk görüntü alınır (`image_id` sıralı). Nevüste yalnız `dx_type == 'histo'` kalır.
2. Her kaynak (`dataset`) için nevüs sayısı, o kaynağın melanom sayısına eşitlenir (`random_state=42`). Sonuç: 614 melanom + 614 nevüs (1228 görüntü).
3. Hat: gri → Otsu (gri < eşik) → merkez bölge → delik doldurma. Merkez piksel arka plandaysa **en büyük bölge** seçilir (ilk iyileştirme).
4. Ölçüt: uzman maskesine göre Dice, melanom ve nevüs ayrı.
5. Küçültme deneyi: k = 4, 8, 16, 32 (küçük görüntü genişliği 150, 75, 37, 18 piksel). İki yol: süzgeçsiz (`INTER_NEAREST`) ve önce Gauss (sigma = 0.5·k) + `INTER_AREA`. Tahmin maskesi `INTER_NEAREST` ile orijinal boyuta büyütülür.
6. Kontroller: (1) tavan (uzman maskesini aynı biçimde küçültüp büyütmek), (2) yalnız süzgeç (Gauss var, küçültme yok), (3) x ekseni olarak lezyonun eşdeğer çapı (piksel).

## Depodaki dosyalar

- `Untitled0.ipynb`: Colab defteri (tüm kod hücreleri)
- `grafik.png`: küçültme deneyinin şekli (x: lezyon eşdeğer çapı, y: Dice, melanom ve nevüs ayrı panel)
- `README.md`: bu dosya

## Kodu nasıl çalıştıracağım

Kod Google Colab'da hücre hücre çalıştırılır (Python, `opencv-python`, `numpy`, `pandas`, `scipy`, `matplotlib`; Colab'da hazır gelir).

1. `Untitled0.ipynb` dosyasını Colab'da aç (ya da yeni bir defter aç ve hücreleri sırayla yapıştır).
2. Veri indirme ve açma hücresini çalıştır (4 dosya, `img/` ve maske klasörü oluşur).
3. Seçim hücresini çalıştır (`secim.csv` üretilir; kaynak başına mel/nv: 329, 225, 44, 16; toplam 614/614 çıkmalı).
4. Baseline hücresini çalıştır, sonra yedek kuralı değiştiren hattı (`hat3`) çalıştır.
5. Küçültme deneyi için `seg`, `buyut` ve `deney(k)` fonksiyonlarını tanımla ve k = 4, 8, 16, 32 için çalıştır.

> Not: Colab oturumu kapanınca dosyalar ve değişkenler silinir. Her yeni oturumda 2. adımdan başlanmalıdır.

## Ana sonuçlar

Sayılar Colab çıktılarından alınmıştır. Dice ortalamasıdır (aynı 1228 görüntü).

**Baseline ve ilk iyileştirme**

| | Baseline | Yedek kural = en büyük bölge |
|---|---|---|
| Melanom | 0.623 | 0.698 |
| Nevüs | 0.735 | 0.749 |

- Baseline'da merkez piksel eşikten açık kalınca hat küçük bir leke seçiyordu. Yedek kural bunu düzeltti.
- Denenip işe yaramayan: 7×7 açma + kapama (ortalama Dice her iki sınıfta yaklaşık 0.005 düştü).

**Küçültme deneyi** (k = 1 referansı: mel 0.698, nv 0.749)

| k | Genişlik (px) | Eşdeğer çap, medyan mel / nv (px) | Tavan mel / nv | Süzgeçsiz mel / nv | Süzgeçli mel / nv | Yalnız süzgeç mel / nv |
|---|---|---|---|---|---|---|
| 4 | 150 | 83.4 / 57.4 | 0.984 / 0.977 | 0.698 / 0.749 | 0.703 / 0.759 | 0.703 / 0.757 |
| 8 | 75 | 41.6 / 28.7 | 0.968 / 0.954 | 0.693 / 0.746 | 0.708 / 0.763 | 0.710 / 0.760 |
| 16 | 37 | 20.7 / 14.3 | 0.935 / 0.906 | 0.683 / 0.728 | 0.717 / 0.764 | 0.716 / 0.764 |
| 32 | 18 | 10.2 / 7.0 | 0.874 / 0.817 | 0.644 / 0.669 | 0.708 / 0.733 | 0.728 / 0.776 |

- "Bozulma" ölçütü: k = 1'e göre ek kayıp > 0.10 (sınıf ortalaması üzerinden). Bu ölçüt **denenen k'lerde (4, 8, 16, 32), hiçbir yolda, iki sınıfta da aşılmadı**.
- En büyük kayıp: süzgeçsiz yol, nevüs, k = 32 (eşdeğer çap medyanı 7.0 px): 0.080.
- Boş maske sayısı her k'de, her yolda ve iki sınıfta 0.
- Tavan (maskeyi küçültüp büyütmenin üst sınırı) k = 4'te mel 0.984, nv 0.977 iken k = 32'de mel 0.874, nv 0.817. Küçük lezyonda maskenin kendisi çözünürlükte kayboluyor.
- Süzgeçli yol, süzgeçsizden her k'de yüksek. Fark k büyüdükçe artıyor (mel / nv: k = 4'te +0.005 / +0.010, k = 8'de +0.015 / +0.017, k = 16'da +0.034 / +0.036, k = 32'de +0.064 / +0.064).
- Yalnız-süzgeç yolu k = 16'ya kadar süzgeçli yolla neredeyse aynı. Bu, kazancın büyük kısmının aliasing'i önlemekten çok yumuşatmadan geldiğini düşündürüyor. k = 32'de yalnız-süzgeç daha yüksek, yani küçültmenin kendisi süzgeçli yola kayıp ekliyor.

## Sınırlamalar

- Seçim kuralı, yedek kural ve ölçümler aynı 1228 görüntüde yapıldı. Ayrı test kümesi yok, genelleme iddiası yoktur.
- Süzgeçsiz yol `INTER_NEAREST`, süzgeçli yol Gauss + `INTER_AREA` kullandı. Bu yüzden "süzgeçli − süzgeçsiz" farkı yalnız Gauss'a bağlanamaz.
- Küçültme deneyi k = 4, 8, 16, 32 ile sınırlı. Daha küçük çaplar denenmedi. Eşdeğer çap sınıf medyanıdır, medyandan çok daha küçük lezyonlar ayrı incelenmedi.
- Defter baştan çalıştırıldı. Seçim sayıları (329, 225, 44, 16; 614/614), baseline (mel 0.6226, nv 0.7345), yedek kural (mel 0.6984, nv 0.7487) ve `deney(4)`, `deney(8)`, `deney(16)`, `deney(32)` çıktıları önceki sonuçlarla aynı çıktı.
- Küçültme tablosundaki sayılar çıktılardan 3 haneye yuvarlanarak yazıldı. Görüntü başına sonuç dosyası yoktur.
- Görüntü başına eşleştirilmiş fark hesaplanmadı ve anlamlılık testi yapılmadı. Ortalama farklar görüntüler arası std'nin (yaklaşık 0.21–0.24) altındadır.
