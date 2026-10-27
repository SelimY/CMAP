# CMAP — ToDo (Ertelenen / Netleştirilecek Maddeler)

Bu dosya, ana prompt (`README.md`) kapsamına henüz alınmayan, sonraya bırakılan veya netleştirilmesi beklenen maddeleri tutar. Her madde karara bağlandığında fark promptuna dönüştürülür ve Archive ile ana prompt'a işlenir.

## B. Veri güvenliği ve hacim
2. **Yedek / geri alma:** Silmeler kalıcı ve geri alınamaz. Öneri: "Export copy" butonu veya her oturum başında otomatik yedek.
3. **Veri hacmi:** Tüm veritabanı bellekte tutulur ve her kayıtta tam export yapılır. Beklenen Item sayısı ve dosya boyutu belirlenmeli (öneri: ~50 MB hedef). Items tablosu için sayfalama veya sanal kaydırma kararı buna bağlı.
4. **İsim normalizasyonu ve Türkçe:** "Tüm boşluk" tanımı (NBSP, tab dahil mi?), Unicode normalizasyonu (NFC) ve Türkçe İ/ı, I/i harflerinin eşleşme kuralı tanımlanmalı (`toLowerCase()` "İ"yi "i̇" yapar). Card Name benzersizliği ve Enum tekilleştirmesini etkiler.

## C. Item varken Card / Property düzenleme
5. **Property silme ve tip değiştirme:** `item_values.property_id` FK'sinde ON DELETE tanımlı değil. Öneri: Item varsa tip kilitli; Property silme onaylı olur ve değerleri cascade ile siler.
6. **Daralan tipe özel ayarlar:** Number decimal→integer, Percent/Hex aralığı daralması, Date "Include time" değişimi için kural yok (yalnızca Text `max_length` için "mevcut değerler silinmez, düzenlerken uyarı" kuralı var). Öneri: aynı kuralın genelleştirilmesi.
7. **Property satırı silme:** Card düzenleme modalında satır silme gereksinimi ana prompt'ta açıkça yazılı değil; eklenmeli.
8. **Enum yönetimi:** Item silinince/değer değişince `usage_count` azalıyor mu? Kullanılmayan öneriyi silme/yeniden adlandırma yolu yok. ItemValue'da enum değerinin metin olarak tutulması önerilir.
9. **Değer taşımayan Card ve Tab kenar durumları:** Hiç değer Property'si olmayan Card'da New/form/tablo davranışı ve satır tanıtıcısı; Tab ile başlayan Card'da boş "Main" sekmesinin gizlenmesi; kullanıcının bir Tab'e "Main" adı vermesi.
10. **Alan kuralları:** Number'da negatif, ondalık hane sınırı, bilimsel gösterim; Hex'te çift hane (bayt) zorunluluğu; Date'te yıl aralığı; CheckBox'ın her zaman 0/1 yazıp yazmayacağı.
11. **Items tablosu ek davranışlar:** Arama kapsamı (hangi kolonlar; Cards listesi filtresi hangi alanlarda), yeni Item/düğümün listeye ekleniş konumu (öneri: sona).

## D. Bilinçli ertelenenler
12. **Map sekmesi** ("Coming soon").
13. **Key** alanının davranışı (Map fazı).
14. **Private** görünürlüğünün davranışı (Map fazı).
15. **Template** alanının kullanımı.
16. **Hamburger menü** içeriği.
17. **Item'ların klasörler arası taşınması** (kurallarıyla birlikte tanımlanacak).
18. **Supabase geçişi, Auth ve CoWork paylaşımı.**
19. **Slogan konumu:** header'da wordmark'ın "altında" mı "yanında" mı (şu an "altında/yanında").
