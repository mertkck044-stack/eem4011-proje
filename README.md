# Görüntü küçüldükçe Otsu ile lezyon sınırı ne zaman bozuluyor? (HAM10000, melanom ve nevüs)

EEM4011 dersi projesi. Gri tonlama + Otsu eşikleme ile lezyon bölütleme yapılır, sonuç uzman maskesine göre Dice ile ölçülür. Makine öğrenmesi (sınıflandırıcı eğitimi) kullanılmaz.

## Veri Seti

* **Ad:** HAM10000 (dermoskopi görüntüleri + metadata) ve `HAM10000_segmentations_lesion_tschandl` (uzman lezyon maskeleri)
* **İndirme:** https://dataverse.harvard.edu/dataset.xhtml?persistentId=doi:10.7910/DVN/DBW86T
* **Veri bu depoda yoktur.** Kod yalnız gereken 4 dosyayı indirir: metadata (`meta.csv`), maskeler, görüntü parça 1 ve parça 2.

## Yöntem (Kısaca)

1. Metadata'dan yalnız `mel` ve `nv` seçilir. Lezyon başına ilk görüntü alınır (`image_id` sıralı). Nevüste yalnız `dx_type == 'histo'` kalır.
2. Her kaynak (`dataset`) için nevüs sayısı, o kaynağın melanom sayısına eşitlenir (`random_state=42`). Sonuç: 614 melanom + 614 nevüs (1228 görüntü).
3. **Hat:** gri -> Otsu (gri < eşik) -> merkez bölge -> delik doldurma. Merkez piksel arka plandaysa en büyük bölge seçilir (ilk iyileştirme).
4. **Ölçüt:** uzman maskesine göre Dice, melanom ve nevüs ayrı.
5. **Küçültme deneyi:** k = 2, 4, 8, 16, 32. İki yol: süzgeçsiz (`INTER_NEAREST`) ve önce Gauss (sigma = 0.5*k) + `INTER_AREA`. Tahmin maskesi `INTER_NEAREST` ile orijinal boyuta büyütülür.
6. **Kontroller:** (1) tavan (uzman maskesini aynı biçimde küçültüp büyütmek), (2) yalnız süzgeç (Gauss var, küçültme yok), (3) x ekseni olarak lezyonun eşdeğer çapı (piksel).

## Kodu Nasıl Çalıştıracağım

Kod Google Colab'da hücre hücre çalıştırılır (Python, `opencv-python`, `numpy`, `pandas`, `scipy`, `matplotlib`; Colab'da hazır gelir).

1. Yeni bir Colab defteri aç.
2. Veri indirme ve açma hücresini çalıştır (4 dosya, `img/` ve maske klasörü oluşur).
3. Seçim hücresini çalıştır (`secim.csv` üretilir; kaynak başına mel/nv: 329, 225, 44, 16; toplam 614/614 çıkmalı).
4. Baseline hücresini çalıştır, sonra yedek kuralı değiştiren hattı (hat3) çalıştır.
5. Küçültme deneyi için `deney(k)` fonksiyonunu tanımla ve k = 2, 4, 8, 16, 32 için çalıştır.

> **Not:** Colab oturumu kapanınca dosyalar ve değişkenler silinir. Her yeni oturumda 2. adımdan başlanmalıdır.

## Ana Sonuçlar

Sayılar Colab çıktılarından alınmıştır. Dice ortalamasıdır (aynı 1228 görüntü).

### Baseline ve İlk İyileştirme

| Sınıf | Baseline | Yedek kural = en büyük bölge |
| :--- | :--- | :--- |
| **Melanom** | 0.623 | 0.698 |
| **Nevüs** | 0.735 | 0.749 |

* Baseline'da merkez piksel eşikten açık kalınca hat küçük bir leke seçiyordu. Yedek kural bunu düzeltti.
* Denenip işe yaramayan: 7x7 açma + kapama (ortalama Dice her iki sınıfta yaklaşık 0.005 düştü).

### Küçültme Deneyi (k = 1 referansı: mel 0.698, nv 0.749)

* **"Bozulma" ölçütü:** k = 1'e göre ek kayıp > 0.10. Bu ölçüt **hiçbir k'de, hiçbir yolda, iki sınıfta da aşılmadı.**
* En büyük kayıp: süzgeçsiz yol, nevüs, k = 32 (eşdeğer çap medyanı 7.0 px): 0.080.
* Tavan (maskeyi küçültüp büyütmenin üst sınırı) k = 2'de 0.99 iken k = 32'de mel 0.874, nv 0.817. Küçük lezyonda maskenin kendisi çözünürlükte kayboluyor.
* Süzgeçli yol, süzgeçsizden her k'de yüksek. Fark en çok k = 32'de: yaklaşık 0.064 (hem mel hem nv).
* Yalnız-süzgeç yolu k = 16'ya kadar süzgeçli yolla neredeyse aynı. Bu, kazancın büyük kısmının aliasing'i önlemekten çok yumuşatmadan geldiğini düşündürüyor.

## Sınırlamalar

* Seçim kuralı, yedek kural ve ölçümler aynı 1228 görüntüde yapıldı. Ayrı test kümesi yok, genelleme iddiası yoktur.
* Süzgeçsiz yol `INTER_NEAREST`, süzgeçli yol Gauss + `INTER_AREA` kullandı. Bu yüzden "süzgeçli - süzgeçsiz" farkı yalnız Gauss'a bağlanamaz.
* Eşdeğer çap sınıf medyanıdır. Medyandan çok daha küçük lezyonlar ayrı incelenmedi.
* Colab oturumu sıfırlandığı için görüntü başına sonuç dosyaları kayboldu. K deneyi tablosu ve şekil, kaydedilmiş çıktılardan elle kuruldu. Görüntü başına eşleştirilmiş fark hesaplanmadı ve anlamlılık testi yapılmadı.
