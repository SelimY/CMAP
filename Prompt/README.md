# CMAP — Yazılım Geliştirme Prompt'u

## 1. Özet
CMAP, kullanıcıların kendi tanımladıkları dinamik "Card" şablonları (form/kayıt tipleri) üzerinden "Item" adında yapılandırılmış kayıtlar oluşturup yönetebildiği, **tek-kullanıcılı ve tamamen offline-first** çalışan bir web uygulamasıdır. Kullanıcı, önce bir Card (şablon) tanımlar; bu Card'a ait Property'ler (alanlar) belirler; ardından bu Card'ı kullanarak istediği kadar Item (gerçek veri kaydı) oluşturur. Tüm veri, kullanıcının kendi diskindeki tek bir `.sqlite` dosyasında tutulur; sunucu tarafı bir backend veya hesap sistemi yoktur. Kapsam: Card tanımlama/yönetimi, veritabanı dosyası yönetimi ve Items sayfası (klasör ağacı + Item tablosu + Item formu). Map sekmesi sonraki fazda tanımlanacaktır.

## 2. Hedef Kullanıcı ve Kullanım Senaryosu
- Şimdilik uygulama tek-kullanıcılı ve offline-first'tir; tüm veri, kullanıcının seçtiği/oluşturduğu tek bir `.sqlite` dosyasında yerel olarak saklanır. Hesap/giriş sistemi yoktur.
- **Gelecek faz için not (Supabase geçişi):** İleride veritabanı katmanının Supabase (PostgreSQL + Auth + RLS) üzerine taşınması ve "CoWork" adında bir paylaşım özelliği (bir kullanıcının başka bir kullanıcıya belirli bir Card'a okuma/yazma erişimi vermesi) eklenmesi planlanmaktadır. Şimdilik bu geçiş UI olarak geliştirilmeyecek; yalnızca şema ve veri erişim katmanı bu geçişi kolaylaştıracak şekilde tasarlanacaktır (bkz. Bölüm 5 ve 6).

## 3. Temel Özellikler (Fonksiyonel Gereksinimler)

### 3.1 Genel Sayfa Yapısı
1. Sayfa header'ı solda: SW logosu, adı ("CMAP"), yanında versiyon rozeti (numara kodda tek bir sabitte tutulur, bkz. Bölüm 5), altında/yanında slogan.
2. Header sağında, **veritabanı durumuna göre** değişen bir alan bulunur (bkz. 3.2) ve her zaman görünen hamburger menü ikonu (içerik şimdilik boş/placeholder) ile tema seçim (dark/light) ikon butonu yer alır.
3. Header ve tab düzeni, paylaşılan ekran görüntülerindeki (Ekler bölümüne bak) genel yerleşim mantığına (logo + isim + versiyon rozeti solda; altında kırmızı alt çizgili aktif tab göstergesi olan tab menüsü) **görsel referans** olarak uygulanmalıdır.
4. **Tüm arayüz metinleri, buton etiketleri, hata/onay/uyarı mesajları ve bilgilendirme yazıları İngilizce olmalıdır.** Türkçe metin kullanılmamalıdır.

### 3.2 Veritabanı Yönetimi (Database Management)
5. Uygulama ilk açıldığında (henüz bir `.sqlite` dosyası seçilmemişken), header'ın sağında **"Open Database"** ve **"New Database"** butonları gösterilir; ana içerik alanında da bu iki seçeneği sunan bir karşılama ekranı/mesajı bulunur. Bu durumda Map/Items/Cards tabları gösterilmez.
6. **"New Database"** akışı: Kullanıcıya varsayılan dosya adı olarak **`CMAP.sqlite`** önerilir (kullanıcı dilerse değiştirebilir); tarayıcının "save file" diyaloğu açılır, boş şema (bkz. Bölüm 6) ile yeni bir `.sqlite` dosyası oluşturulur.
7. **"Open Database"** akışı: Tarayıcının dosya seçim diyaloğu açılır, kullanıcı mevcut bir `.sqlite` dosyası seçer, dosya sql.js ile belleğe yüklenir.
8. Bir veritabanı dosyası açıldığında/oluşturulduğunda: "Open Database"/"New Database" butonları kaybolur, yerine **o anki dosya adı** (ör. "CMAP.sqlite") header'da küçük bir rozet olarak gösterilir; yanında manuel **"Save"** (anlık export) ve **"Switch Database"** (mevcut dosyayı kapatıp yeniden seçim ekranına dönme) ikon butonları yer alır. Bu noktadan sonra Map/Items/Cards tabları görünür hale gelir.
9. **Kalıcılık (File System Access API):** Seçilen/oluşturulan dosyanın `FileSystemFileHandle`'ı uygulama genelinde (global state) tutulur. Her CRUD işleminden sonra (1500 ms debounce ile, bkz. 3.2.1) `db.export()` çıktısı, aynı handle üzerinden `createWritable()` → `write()` → `close()` adımlarıyla diske otomatik yazılır. Elle **"Save"** butonu her zaman UI'da bulunur ve anlık export tetikler.
10. File System Access API yalnızca Chromium tabanlı tarayıcılarda (Chrome, Edge) desteklenir. Desteklenmeyen tarayıcılarda uygulama şu **fallback** moduna düşmelidir: veritabanı dosyası bir `Blob` olarak `<a download>` ile dışa aktarılır, açma işlemi ise `<input type="file">` ile dosya seçtirilerek yapılır (otomatik diske yazma yapılamaz; kullanıcı her değişiklikten sonra manuel "Save/Download" ile dışa aktarmalıdır — bu durum kullanıcıya UI'da açıkça belirtilmelidir).

### 3.2.1 Dosya Doğrulama, Dosya Kimliği, Hatırlama ve Kayıt Güvenliği
**Dosya doğrulama (yabancı/geçersiz dosya):** "Open Database" ile seçilen dosya SQLite olarak açılamıyorsa, CMAP tabloları (`cards`, `properties` vb.) içermiyorsa (boş SQLite dosyası dahil) veya `schema_version` uygulamanın desteklediğinden **yüksekse** dosya yüklenmez ve İngilizce hata gösterilir: "This file is not a CMAP database." (yabancı/boş/bozuk) veya "This database was created by a newer version of CMAP." (daha yeni şema). Hata sonrası uygulama karşılama ekranına döner; mevcut açık veritabanı (varsa) etkilenmez.

**Dosya kimliği ve şema sürümü:** Her veritabanında `meta` tablosu bulunur (bkz. Bölüm 6): `db_id` (UUID) ve `schema_version`. "New Database" bu kayıtları oluşturur. Eski dosyalarda (CMAP tabloları var, `meta` yok) idempotent şema yükseltmesi sonrası yeni bir `db_id` üretilir ve `schema_version` mevcut şemanın sürümüne ayarlanır; yükseltme adımları `schema_version`'a göre sırayla uygulanır. `PRAGMA user_version` kullanılmaz. localStorage'daki tüm kayıtlar (aç/kapa durumu, seçili düğüm, seçili Project) **dosya adıyla değil `db_id` ile** anahtarlanır.

**Son dosyayı hatırlama:** Başarılı Open/New sonrası `FileSystemFileHandle` ve dosya adı **IndexedDB**'de saklanır. Karşılama ekranında, saklı handle varsa "Open Database" / "New Database" yanında **"Reopen last: <dosya adı>"** butonu gösterilir. Tıklanınca (kullanıcı hareketi içinde) `queryPermission({mode:'readwrite'})`, gerekirse `requestPermission` çağrılır; izin alınırsa dosya yüklenir. Handle geçersizse (dosya taşınmış/silinmiş), izin reddedilmişse veya dosya yukarıdaki doğrulamaya takılırsa İngilizce hata gösterilir ve saklı handle temizlenir. "Switch Database" ile kapatılan dosya da "Reopen last" olarak sunulur. File System Access API desteklenmeyen tarayıcılarda bu özellik gösterilmez.

**Kayıt güvenliği (autosave):**
- Her değişiklikten **1500 ms** sonra otomatik kayıt yapılır. "Save" butonu, "Switch Database", yeni dosya açma ve sayfa gizlenmesi (`visibilitychange` → hidden) bekleyen kaydı **hemen** tetikler. Yazmalar sıraya alınır, aynı anda tek yazma yapılır.
- Header'da dosya adı rozetinin yanında canlı durum gösterilir: **"Saved"**, **"Saving…"**, **"Unsaved changes"**, **"Save failed"** (her biri ayrı renk/ikon).
- Yazma hatasında bellekteki veri korunur, durum "Save failed" olur, hata mesajı gösterilir; sonraki değişiklikte veya "Save" ile yeniden denenir.
- Kaydedilmemiş değişiklik varken sekme/pencere kapatılırsa tarayıcının standart `beforeunload` uyarısı gösterilir. Fallback modunda (otomatik yazma yok) değişiklik yapıldığı andan itibaren durum "Unsaved changes" olur ve kullanıcı "Save/Download" yapana kadar bu uyarı geçerlidir.

**Dış değişiklik algılama:** Her başarılı yükleme/kayıttan sonra dosyanın `lastModified` değeri saklanır; her yazmadan önce ve pencere odağı geri geldiğinde güncel değerle karşılaştırılır. Dosya uygulama dışında değişmişse otomatik yazma **durdurulur** ve modal gösterilir: "The database file was changed outside CMAP." Seçenekler: **"Overwrite file"** (bellektekini yaz), **"Reload from disk"** (bellektekini at, diskten yeniden yükle), **"Save a copy…"** (bellektekini başka bir dosyaya kaydet; mevcut dosya olduğu gibi kalır). Kullanıcı seçene kadar yazma yapılmaz, durum "Unsaved changes" kalır.

### 3.3 Ana Navigasyon
11. Bir veritabanı açıldıktan/oluşturulduktan sonra ekranda 3 tab belirir: **Map**, **Items**, **Cards** (Ekran Görüntüsü 3'teki genel yerleşime görsel referans). Bir veritabanı her açıldığında/oluşturulduğunda `cards` tablosunda **en az bir kayıt** varsa (disable edilmişler dahil) **Items** sekmesi, hiç kayıt yoksa **Cards** sekmesi seçili açılır; sonrasında kullanıcının sekme seçimi oturum boyunca korunur.
12. **Cards** ve **Items** tablarının fonksiyonları tanımlıdır (bkz. 3.4 ve 3.7). **Map** tabı arayüzde görünmeli, tıklanabilir olmalı, ancak içerik olarak "Coming soon" durumu göstermelidir.

### 3.4 Cards Tab — Card Listeleme
13. Cards tabına geçildiğinde, kayıtlı Card'ları listeleyen bir tablo gösterilmeli.
14. Tablonun hemen üzerinde: bir filtre/arama alanı (sol) ve "New Card" butonu (sağ).
15. Tablo kolonları:
    - Card Name
    - Title
    - # of Properties (yalnızca değer taşıyan Property'ler sayılır; **Tab ve Separator satırları sayılmaz**)
    - # of Records (bu Card'a bağlı gerçek Item sayısı)
    - Edit (düzenleme butonu)
    - Del (silme butonu — kaydı fiziksel silmez, `is_disabled` alanıyla devre dışı bırakır/gizler)
    - Purge/Recover (yalnızca disable edilmiş Card'lar için görünür: Purge = kalıcı silme, Recover = tekrar aktif etme; **Card'a bağlı en az bir Item varsa Purge butonu devre dışıdır** ve ipucu gösterir: "This Card has Items and cannot be purged. Delete its Items first.")
16. Del butonuna basıldığında, işlem gerçekleşmeden önce bir **onay modalı** gösterilmeli (İngilizce — ör. "Are you sure you want to disable this Card?").
17. Purge butonuna basıldığında da bir **onay modalı** gösterilmeli (ör. "This action cannot be undone. This Card will be permanently deleted. Nodes linked to this Card will become Folders."). Tek adımlı modal onayı yeterlidir.
18. Bir Card disable edildiğinde: Card'a ait Item verileri **değişmez/silinmez**. Card'a bağlı düğümler Item-Tree'de kalır, Items tablosu gizlenmez; ancak "New" butonu gösterilmez ve satırların düzenleme (kalem) ikonu devre dışı olur (silme ve satır sıralama çalışır). Disable edilmiş Card'lar düğüm Type dropdown'ında listelenmez. Disable edilmiş Card'ın **Edit butonu aktiftir** (Card düzenlenebilir). Recover edilince her şey normale döner.
19. Card'a bağlı Item yoksa purge mümkündür ve Card kalıcı silinir (Property'ler ve EnumValue'lar cascade ile silinir). Purge edilen Card'a bağlı düğümler silinmez, otomatik olarak **"Folder"** tipine döner. Item'ı olan Card purge edilemez.
20. Disable edilmiş Card'lar tablodan kaldırılmaz; listede **soluk (opacity: 0.5)** görünümle kalırlar. Tablo üzerinde bir **"Hide disabled cards"** filtre anahtarı bulunmalı; varsayılan kapalıdır (disabled Card'lar soluk gösterilir), açıldığında tamamen gizlenir.
21. Tablo boşsa veya filtre sonucunda hiç kayıt kalmıyorsa uygun bir **empty-state** mesajı gösterilmeli (İngilizce, ör. "No Cards yet" / "No Cards match your filter").
22. Veri veritabanından yüklenirken uygun bir **loading-state** gösterilmeli.

### 3.5 New Card / Edit Card Modalı
23. "New Card" butonu bir modal açar; bu modal aşağıdaki alanları içerir:
    - **Card Name** (zorunlu, Text)
    - **Title** (zorunlu, Text) — Kullanıcı bu alanı boş bırakıp Card Name'i girip başka bir alana geçerse, Title alanına **otomatik olarak Card Name'deki değer kopyalanır** (yalnızca Title hâlâ boşsa).
    - **Description** (Text area, opsiyonel)
    - **Properties** — Card'a ait alt-kayıt listesi (bkz. 3.6)
    - **Template** — en altta, Text area, opsiyonel. Şimdilik yalnızca veri modelinde yer tutucu olarak bulunur.
24. Edit Card akışında aynı modal, mevcut Card verileriyle önceden doldurulmuş olarak açılır; Properties listesi **düzenlenebilir bir tablo** olarak gösterilmelidir. Tablodaki her satırda **yukarı / aşağı taşıma butonları** bulunur; taşıma sonrası tüm Property'lerin `sort_order` değerleri ardışık olarak, tek SQL transaction'ı içinde yeniden yazılır (Tab yerleşimi bu sıraya bağlı olduğu için sıra kalıcıdır). İlk satırda "yukarı", son satırda "aşağı" butonu devre dışıdır. Edit modunda Title'ın Card Name'den otomatik kopyalanması davranışı tekrar tetiklenmez.
25. **Card Name benzersizlik kontrolü:** Aynı veritabanı dosyası içindeki Card'lar arasında (**disable edilmiş Card'lar dahil**) aynı isim (Card Name) kullanılamaz; Title alanında tekrar serbesttir. Kullanıcı Card Name alanına yazarken **canlı doğrulama** yapılmalı: çakışma varsa input **kırmızı**, uygunsa **koyu yeşil** olmalı. Kaydet butonu çakışma varken devre dışı bırakılmalı.
    - **Karşılaştırma kuralı:** Hem yeni isim hem mevcut isimler üzerinde tüm boşluklar kaldırılır ve case-insensitive karşılaştırılır. Örnek: "First CarD" ile "F i rst  Card" aynı isim sayılır. Veritabanına kaydedilen `name` değeri yalnızca trim uygulanmış orijinal metindir; aradaki boşluklar korunur. Bu kural hem client-side canlı doğrulamada hem veritabanı seviyesinde (bkz. Bölüm 6) uygulanmalıdır.

### 3.6 Property Tanımı
26. Her Card'ın altında sınırsız sayıda Property tanımlanabilir; "New Property" butonu ile ekleme yapılır.
27. Property tipleri:
    - **Icon**: Uygulamaya **gömülü (embedded), harici CDN/font gerektirmeyen bir SVG ikon alt kümesi** (ör. Tabler/Lucide'dan seçilmiş ~150–200 ikonluk, kategorilere ayrılmış bir alt küme, tek renk/outline). Seçici, ikonları kategorilere göre gruplu göstermelidir. Veritabanında yalnızca ikonun adı/anahtarı saklanır.
    - **Text**: Serbest metin girişi. Maksimum uzunluk Card tanımı ekranında belirlenir (bkz. madde 29).
    - **Number**: Sayısal değer girişi. Sayı türü (tam sayı / ondalık) Card tanımı ekranında seçilir (bkz. madde 29).
    - **Hex**: Genel amaçlı onaltılık (hexadecimal) veri girişi (renk kodu değil; ör. `0x1a2f`, bayt dizisi, kimlik değeri). Giriş alanı her zaman **`0x` ön ekiyle dolu** gelir; ön ek **zorunludur** ve kullanıcı tarafından silinememeli/değiştirilememelidir. Minimum/maksimum uzunluk Card tanımı ekranında belirlenir (bkz. madde 29).
    - **Enum**: Item kaydı sırasında type-ahead ile değer girilen alan. Değer listesi tamamen dinamiktir — kullanıcı yeni değer yazarsa önerilere otomatik eklenir (Ekran Görüntüsü 4'e görsel referans). **Enum değer listesi her zaman ilgili Property'ye özeldir; Card'lar/property'ler arası çapraz kullanılmaz.**
    - **Date**: Tarih seçici. Saat bilgisinin de istenip istenmeyeceği Card tanımı ekranında seçilir (bkz. madde 29).
    - **Tab**: Statik seçenek; değer taşımaz. Items formunda yeni bir sekme başlatır (bkz. madde 30).
    - **Separator**: Statik seçenek; değer taşımaz. Items formuna alt başlık ve altına yatay çizgi ekler (ör. "Example________").
    - **Percent (%)**: Min ile Max arasında **tam sayı** olarak yüzde değeri alır. Arayüzde etiketi "%" olarak görünür.
    - **CheckBox**: Boolean (True/False) değer alır; veritabanında `0`/`1` saklanır.
28. Her Property için ek ayarlar:
    - **Label**: Property'nin soru/etiket metni. **Zorunlu**, maksimum 255 karakter, trim uygulanır.
    - **Key**: Checkbox. Şimdilik yalnızca veri modelinde tutulur; davranışı Map fazında tanımlanacaktır.
    - **Visibility**: Show / Hide / Private. Şimdilik yalnızca veri modelinde tutulur; "Private" davranışı Map fazında tanımlanacaktır.
    - **Tab ve Separator** tiplerinde Key ve Visibility ayarları arayüzde gösterilmez; veritabanında `is_key = 0`, `visibility = 'show'` sabit olarak saklanır.
29. **Tipe özel ayarlar:** Property'nin tipi seçildiğinde, Property tanım satırında/modalında o tipe özel aşağıdaki ek ayarlar gösterilmelidir (ilgili tip seçili değilken gizlenir). Bu ayarlar Card tanımı ekranında girilir ve saklanır; Item formunda uygulanır (bkz. 3.7 ve Bölüm 8.1).
    - **Text** → "Max length": tam sayı, **1–4000** aralığında, varsayılan **255**. Aralık dışı veya tam sayı olmayan değerde Property kaydedilemez ve İngilizce hata mesajı gösterilir. `config`'te yoksa (eski Property'ler) **255** varsayılır; varsayım okuma anında uygulanır, kayıtlar değiştirilmek zorunda değildir.
    - **Number** → "Number type": `Integer` | `Decimal` (zorunlu seçim, varsayılan: `Integer`). Ondalık ayırıcı nokta (`.`) olmalıdır.
    - **Hex** → "Min length" ve "Max length": `0x` ön eki HARİÇ onaltılık hane sayısı cinsinden, opsiyonel tam sayı alanları (boş bırakılırsa o yönde sınır yoktur). Doğrulama: değerler pozitif tam sayı olmalı; ikisi de girilmişse Min ≤ Max olmalı, aksi halde Property kaydedilemez ve İngilizce bir hata mesajı gösterilmelidir.
    - **Date** → "Include time" onay kutusu (varsayılan: kapalı). Kapalıysa yalnızca tarih (`YYYY-MM-DD`), açıksa tarih+saat (`YYYY-MM-DDTHH:mm:ss`, ISO 8601) tutulur. Okurken saniyesiz (`THH:mm`) bir değerle karşılaşılırsa saniye `00` kabul edilir.
    - **Percent** → "Min" ve "Max": tam sayı, varsayılanlar 0 ve 100. Doğrulama: ikisi de 0–100 aralığında tam sayı olmalı ve Min ≤ Max olmalıdır; aksi halde Property kaydedilemez ve İngilizce hata mesajı gösterilir. Items formundaki giriş alanı: sayı kutusu ve yanında "%" işareti; kaydırıcı (slider) kullanılmaz.
    - **CheckBox** → "Default value": `Unchecked` (varsayılan) | `Checked`.
    - Icon, Enum, Tab ve Separator tiplerinde ek ayar yoktur (Tab ve Separator için yalnızca zorunlu Label).
30. **Tab / Separator kuralları:**
    - Tab ve Separator'ın Label'ı zorunludur (max 255 karakter, trim). Tab'de Label sekmenin adı, Separator'da alt başlık metnidir.
    - **Tab yerleşimi** (Property'ler `sort_order` sırasına göre işlenir): İlk Tab'den önce kalan tüm Property'ler otomatik olarak **"Main"** adlı sekmeye girer. "Main" adı sabittir, kullanıcı değiştiremez ve Card'da bir Tab Property'si olarak tutulmaz. Her Tab Property'si, kendisinden sonraki Property'leri bir sonraki Tab'e veya listenin sonuna kadar kendi sekmesine alır. Örnek: [text, hex, tab "data", hex2] → "Main" sekmesinde text ve hex; "data" sekmesinde hex2. Hiç Tab yoksa tek sekme (Main) vardır ve bu durumda sekme çubuğu gösterilmez (bkz. 3.7.5).
    - Tab ve Separator'ın Items formundaki render'ı için bkz. 3.7.5.

### 3.7 Items Sayfası
**3.7.1 Genel yerleşim**
- Items tabı File Explorer benzeri iki panelli bir ekrandır: solda **"Item-Tree"** (~%30 genişlik), sağda **"Items"** (~%70 genişlik). Bu ekranda **sayfanın kendisi dikey kaymaz** (header ve tab çubuğu hariç kalan yüksekliği kaplar); sol ağaç ve sağ tablo **kendi içinde dikey kayar**. Tablo çok kolonluysa kendi içinde yatay da kayar.
- Harici kütüphane kullanılmaz; sürükle-bırak için tarayıcının yerleşik HTML5 Drag & Drop API'si kullanılır.

**3.7.2 Sol panel — Item-Tree**
- Üst alan: panel başlığı "Item-Tree" ve aynı satırda **"New Project"** butonu; altında **"Project:"** etiketi ve yanında bir dropdown. Dropdown, hiçbir yere bağlı olmayan **kök düğümleri (Project)** listeler (`sort_order` sırasına göre); seçim değişince ağaç yenilenir. "New Project", düğüm formunu açar ve `parent_id = NULL` olan yeni bir kök düğüm yaratır; oluşunca otomatik seçili Project olur.
- Ağaç, seçili Project'in kök düğümüyle başlar; kök düğümde de diğer düğümler gibi ikonlar bulunur. Her düğüm satırında: aç/kapa oku (alt düğümü varsa), tip ikonu (Folder veya Card; gömülü SVG), düğüm adı, Card-tipliyse bağlı Card'ın adı (soluk metin) ve sağda her zaman görünür üç ikon: **+** (alt düğüm ekle), **kalem** (düzenle), **çöp kutusu** (sil).
- **Sürükle-bırak:** Bir düğümü başka bir düğümün **üstüne** bırakmak onu o düğümün son alt düğümü yapar; iki düğüm **arasına** bırakmak sıralamayı değiştirir (bırakma konumu görsel göstergeyle belirtilir). Düğüm kendisine veya kendi alt ağacına bırakılamaz (adapter'da da engellenir). Kök (Project) düğümleri sürüklenemez. Yeni konum/sıra kalıcıdır (`parent_id` ve `sort_order`, tek transaction).
- Aç/kapa durumu, seçili düğüm ve seçili Project **localStorage**'da (veritabanı dosya adına göre anahtarlanarak) saklanır ve açılışta geri yüklenir; kayıtlı kimlik artık yoksa ilk Project'e ve boş seçime düşülür. localStorage'dan okunan değerler güvenilmez girdi sayılır ve doğrulanır.
- Düğüm adları **tekil olmak zorunda değildir** (aynı üst düğüm altında dahil; A → A → A geçerlidir); benzersizlik doğrulaması yoktur.

**3.7.3 Düğüm formu (New Project / + / Kalem) — modal**
- Alanlar: **Name** (zorunlu, max 255, trim) ve **Type** dropdown'ı. Varsayılan **"Folder"**; diğer seçenekler aktif (disable edilmemiş) Card'lardır. Yeni düğüm eklenirken tip her zaman sorulur. Disable edilmiş Card'lar listelenmez (yalnızca düğüm hâlihazırda disable edilmiş bir Card'a bağlıysa düzenleme formunda mevcut değer olarak görünür).
- **Düzenleme kuralı:** Düğümde en az bir Item varsa Type dropdown'ı kilitlidir (açıklama: "This node has Items, so its type cannot be changed."). Item'ı olmayan düğümde tip serbestçe değişir; alt düğümlerin olması tip değişimini engellemez. Card tipli düğümün de alt düğümü olabilir; "Folder" düğümü Item tutamaz.
- **Düğüm silme:** Çöp ikonu → onay modalı; düğüm adı, alt düğüm sayısı ve toplam Item sayısı gösterilir (ör. "This will permanently delete 'X', 3 sub-nodes and 12 Items. This action cannot be undone."). Onayda düğüm, tüm alt ağacı ve bu düğümlerdeki tüm Item/ItemValue kayıtları tek transaction'da kalıcı silinir. Kök silinirse ilk kalan Project seçilir; yoksa "No projects yet" durumu gösterilir.

**3.7.4 Sağ panel — Items tablosu**
- Başlık "Items". Tablo, seçili düğümün **yalnızca doğrudan** Item'larını gösterir (alt düğümlerinkini göstermez).
- Üst çubuk: solda görünen değerler üzerinde çalışan **arama/filtre kutusu**, sağda **"New"** butonu. Düğüm Card tipliyse ve Card disable edilmemişse "New" görünür ve Item formunu (3.7.5) modal olarak açar; düğüm "Folder" ise veya Card disable edilmişse "New" gösterilmez.
- **Kolonlar:** Seçili düğümün Card'ına ait değer taşıyan Property'ler `sort_order` sırasıyla kolon olur (Label başlık). **Tab ve Separator dahil edilmez; Visibility = Hide olanlar gizlenir** (Private bu fazda Show gibi davranır). Gösterim: Icon → ikon; CheckBox → işaret/boş kutu; Percent → değer + "%"; Hex → saklanan hâliyle; diğerleri metin.
- **Date gösterimi:** saatsiz Date kolonu `YYYY-MM-DD`; saatli Date kolonu `YYYY-MM-DD hh:mm:ss` (ISO değerdeki `T` yerine boşluk). Arama, ekranda görünen bu biçimler üzerinde çalışır; sıralama ISO değerle yapılır.
- **Çok satırlı Text:** satır sonları boşluk gibi gösterilerek tek satıra indirgenir, taşan kısım `…` ile kısaltılır; tam değer hücre üzerine gelince ipucunda (tooltip) görünür.
- Satır başında sürükleme tutacağı, en sağda **kalem** ve **çöp kutusu** ikonları bulunur. Satırlar sürükle-bırak ile **yalnızca aynı düğüm içinde** yeniden sıralanır (`items.sort_order`, kalıcı, tek transaction); Item'ı başka düğüme/klasöre taşımak **mümkün değildir** (ileride kurallarıyla tanımlanacak). Kalem, Item formunu mevcut değerlerle açar; Card disable edilmişse kalem devre dışıdır (ipucu: "This Card is disabled"). Çöp kutusu onay modalı ("This Item will be permanently deleted. This action cannot be undone.") sonrası Item ve ItemValue'ları kalıcı siler.
- **Kolon sıralama:** Kolon başlığına tıklamak sırasıyla **artan → azalan → sıralamasız (manuel sıra)** döngüsü yapar; başlıkta yön oku gösterilir. Karşılaştırma tipe göredir: Text/Enum metin sırasıyla (yerel dil duyarlı), Number/Percent sayısal, Hex sayısal değere göre, Date/Date-Time ISO değerle, CheckBox işaretsiz < işaretli, Icon anahtar metniyle. **Boş değerler her iki yönde de sonda** yer alır. Aynı anda tek kolonda sıralama yapılır; sıralama durumu yalnızca bellekte tutulur ve düğüm değişince sıfırlanır; arama filtresiyle birlikte çalışır. **Sıralama aktifken satır sürükleme devre dışıdır** (tutacak soluk, ipucu: "Clear column sorting to reorder rows"); manuel sıra (`sort_order`) sıralamayla hiçbir zaman değişmez.
- **Satır tıklama:** Satırın herhangi bir yerine tıklamak, kalem ikonuyla aynı şekilde Item formunu (Edit) açar; sürükleme tutacağı, kalem ve çöp kutusu ikonlarına tıklamak bunu tetiklemez. Bağlı Card disable edilmişse satır tıklanabilir değildir. Satır üzerine gelince vurgu ve el imleci gösterilir.
- **Boş durumlar (İngilizce):** hiç Project yoksa "No projects yet" + New Project çağrısı; düğüm seçili değilse "Select a node from the Item-Tree"; Folder seçiliyse "This node is a Folder. Select a Card node to see its Items."; Item yoksa "No Items yet"; arama sonucu boşsa "No Items match your search". Yüklenirken loading-state gösterilir.

**3.7.5 Item formu (New / Edit) — modal**
- Form, düğümün Card'ının Property'lerini `sort_order` sırasıyla gösterir. **Tab** Property'leri sekmeleri oluşturur: ilk Tab'den önce kalan alanlar otomatik **"Main"** sekmesine, her Tab'in sonrasındaki alanlar o Tab'in Label'ıyla adlanan sekmeye girer (Tab yoksa tek sekme; sekme çubuğu gösterilmez). **Separator** Label'ı alt başlık olarak ve altına yatay çizgiyle gösterilir; değer taşımaz. Visibility = Hide olan Property'ler tabloda gizlenir ama formda **gösterilir**; Key ve Private bu fazda işlevsizdir.
- **Yerleşim:** Her alan tek satırdadır: solda **Label** (~%30–35 genişlik; uzun Label kendi sütununda alt satıra kırılır), sağda **giriş alanı**. Separator ve sekme çubuğu tam genişliktedir. Yüksek denetimler (Enum type-ahead açılır listesi, Icon seçici, çok satırlı Text) sağ sütunda açılır. Dar ekranlarda (≈500px altı) Label giriş alanının üstüne yığılır.
- **Giriş denetimleri:**
  - **Text:** Property'nin `max_length` değerine göre: **≤ 100** → tek satırlı `input type="text"` (satır sonları boşluğa çevrilir); **> 100** → çok satırlı `textarea` (başlangıçta 3 satır, dikey yeniden boyutlandırılabilir; satır sonu ve tab korunur, diğer kontrol karakterleri temizlenir). Mevcut değer satır sonu içeriyorsa alan `textarea` açılır. Alanın altında/yanında **"12/255"** biçiminde canlı karakter sayacı bulunur (satır sonları dahil sayılır); limite ulaşılınca yazma durur ve sayaç kırmızıya döner; yapıştırılan metin limitte kesilir. `max_length` sonradan düşürülürse mevcut değerler silinmez/kesilmez; düzenlerken sayaç kırmızı görünür ve değer limite indirilmeden kayıt yapılamaz.
  - **Number:** Card ayarına göre Integer/Decimal. **Percent:** sayı kutusu + yanında "%" işareti (slider yok). **CheckBox:** onay kutusu, yeni Item'da başlangıç değeri Card'daki "Default value". **Icon:** gömülü ikon seçici (kategorili). **Hex:** giriş alanı `0x` ön ekiyle dolu gelir, ön ek silinemez/değiştirilemez; kayıtta küçük harfe normalize edilir.
  - **Date:** tarayıcının yerleşik seçicisi yerine **maskeli metin kutusu** + yanında takvim butonu; takvimden seçim aynı biçimde kutuya yazılır. "Include time" kapalıysa `DD-MM-YYYY`, açıksa `DD-MM-YYYY hh:mm:ss` (24 saat). Saklama ISO 8601 olarak kalır (`YYYY-MM-DD` / `YYYY-MM-DDTHH:mm:ss`).
  - **Enum:** Type-ahead (Ekran Görüntüsü 4 referansı): arama kutusu + öneri listesi (kullanım sayısıyla). Öneriler **yalnızca o Property'ye özeldir**. Listede olmayan yeni değer kaydedilirse o Property'nin önerilerine otomatik eklenir (case-insensitive tekilleştirme, ilk girilen biçim saklanır); seçilen/girilen her değer için `usage_count` artar.
- Tüm alanlar **opsiyoneldir**: boş alan için ItemValue kaydı oluşturulmaz (yalnızca `0x` içeren Hex boş sayılır).
- Doğrulama hem arayüzde (alan bazlı, anlık, İngilizce) hem adapter'da (kayıt öncesi) uygulanır; geçersiz alan varken kayıt yapılamaz (kurallar için bkz. 8.1). "Save"/"Cancel" butonları bulunur; kayıt sonrası dosya diske otomatik yazılır ve tablo güncellenir.

### 3.8 Ortak Arayüz Kuralları
- **Kaydedilmemiş değişiklik uyarısı:** Düzenleme formu içeren tüm modallarda (New/Edit Card, Item formu, düğüm formu) başlangıç değerlerinden bir fark varsa (dirty) kapatma girişimlerinde (X, Cancel, Esc, arka plana tıklama) onay modalı gösterilir: "You have unsaved changes. Discard them?" — **"Keep editing"** / **"Discard"**. Değişiklik yoksa modal doğrudan kapanır. Card modalındaki Property ekleme/silme/sıralama/düzenleme de değişiklik sayılır. Silme/purge onay modalları bu kuralın dışındadır.

## 4. Kullanıcı Akışları

**Akış A — İlk kullanım ve veritabanı oluşturma/açma**
1. Kullanıcı uygulamayı açar; henüz bir veritabanı seçilmediği için karşılama ekranında "Open Database" / "New Database" seçenekleri görünür.
2. "New Database" seçerse: varsayılan ad `CMAP.sqlite` önerilir, dosya konumu seçilir, boş şema ile dosya oluşturulur ve diske yazılır. "Open Database" seçerse: mevcut bir `.sqlite` dosyası seçilir ve sql.js ile yüklenir.
3. Veritabanı hazır olduğunda header'da dosya adı görünür, Map/Items/Cards tabları belirir; varsayılan sekme 3.3'teki kurala göre seçilir (Card kaydı varsa Items, yoksa Cards).

**Akış B — Card oluşturma**
1. "New Card" butonuna tıklar, modal açılır.
2. Card Name girer; canlı benzersizlik kontrolü çalışır. Title boşsa Card Name'den otomatik kopyalanır; kullanıcı dilerse değiştirir. Description doldurulur (opsiyonel).
3. "New Property" ile Property'ler ekler; her biri için tip, Label, tipe özel ayarlar, Key, Visibility ayarlanır.
4. Template alanına (opsiyonel) serbest metin girilebilir.
5. Kaydeder; işlem adapter fonksiyonu üzerinden çalışır, ardından dosya diske otomatik yazılır; Card, Cards tablosunda listelenir.

**Akış C — Card düzenleme**
1. Kullanıcı Cards tablosunda bir satırın "Edit" butonuna tıklar.
2. Modal, mevcut verilerle açılır; Properties bir tablo olarak düzenlenebilir; satırlar yukarı/aşağı butonlarıyla yeniden sıralanabilir.
3. Değişiklikler kaydedilir, dosya otomatik yazılır, tablo güncellenir.

**Akış D — Card devre dışı bırakma / kalıcı silme**
1. "Del" butonu → onay modalı → onaylanırsa Card `disabled` olur, tabloda soluk görünür; Items sayfasında bağlı düğümlerde "New" gizlenir ve Item düzenleme devre dışı kalır.
2. Disable edilmiş Card için "Purge" veya "Recover" seçenekleri görünür.
3. "Purge" (yalnızca Card'a bağlı Item yoksa aktiftir) → onay modalı → onaylanırsa Card kalıcı silinir; bağlı düğümler "Folder" tipine döner.
4. "Recover" → Card tekrar aktif olur, soluk görünüm kalkar.

**Akış E — Items sayfasında Item oluşturma**
1. Kullanıcı Items tabına geçer; "New Project" ile bir Project (kök düğüm) oluşturur ve Project dropdown'ında seçer.
2. Ağaçta + ikonuyla alt düğümler ekler; her düğümde Type'ı sorulur (Folder veya bir Card).
3. Card tipli bir düğümü seçer; sağda Items tablosu açılır ve "New" butonu görünür.
4. "New" ile Card'ın formunu modalda doldurur ("Save"); doğrulama geçerse Item tabloya eklenir ve dosya diske yazılır.
5. Satırları sürükle-bırak ile sıralar; kalem ile düzenler, çöp kutusu ile (onaylı) kalıcı siler.

**Akış F — Veritabanı değiştirme**
1. Kullanıcı header'daki "Switch Database" butonuna tıklar.
2. Bekleyen değişiklikler hemen diske yazılır (yazma başarısız olursa kullanıcı uyarılır ve geçiş iptal edilebilir), ardından karşılama ekranına (Akış A, adım 1) dönülür.

## 5. Teknik Tercihler / Kısıtlar
- **Veritabanı motoru:** [sql.js](https://sql.js.org) (SQLite'ın WASM derlemesi), `cdnjs.cloudflare.com/ajax/libs/sql.js` üzerinden dahil edilir. Kararlı bir sürüm **tam sürüm numarasıyla sabitlenir** (`latest`/aralık yok); `<script>` etiketine bu dosya için **SRI hash** (`integrity="sha384-…"`, `crossorigin="anonymous"`) eklenir. `.wasm` dosyası aynı sürümden `locateFile` ile yüklenir. Sürüm numarası ve hash, tek bir yorumlu blokta belgelenir. Bundler, build sistemi veya npm bağımlılığı **kullanılmaz**.
- **Dosya yapısı:** Uygulama **tek bir `index.html` dosyası** içinde, tüm CSS ve JS inline olacak şekilde yapılandırılır. Ana giriş dosyası **repo kökünde `index.html`** olmalıdır (GitHub Pages için gereklidir).
- **Veri erişim katmanı (adapter) — mimari soyutlama:** Dosya tek olduğundan ayrı bir `db-adapter.js` dosyası oluşturulmaz; bunun yerine `index.html` içindeki `<script>` bloğunda, açıkça yorumla sınırlandırılmış (ör. `// ---- DB ADAPTER START/END ----`) tek bir bölümde tüm CRUD fonksiyonları (`createCard`, `readCards`, `updateCard`, `disableCard`, `purgeCard`, `createProperty`, vb.) toplanır. UI kodu **doğrudan `db.run()` / `db.exec()` çağırmaz**, yalnızca bu adapter fonksiyonlarını çağırır. Bu sayede ileride adapter bloğunun içi Supabase client çağrılarıyla değiştirilebilir; UI katmanına dokunulmaz.
- **Kalıcılık:** File System Access API (`showOpenFilePicker` / `showSaveFilePicker`, `createWritable`), yalnızca Chromium tabanlı tarayıcılarda; desteklenmiyorsa Blob + `<a download>` / `<input type="file">` fallback'i (bkz. 3.2/10).
- **SQL lehçesi — Postgres uyumluluğu için kısıtlar:** SQLite'a özgü sentakstan kaçınılır: `AUTOINCREMENT` kullanılmaz (bkz. ID stratejisi), `STRICT` tablo modu **kullanılmaz** ve şema (`CREATE TABLE`) bloğunda `PRAGMA` kullanılmaz. **Tek istisna:** veritabanı her açıldığında/oluşturulduğunda, şema bloğunun dışında, bağlantı kurulur kurulmaz `PRAGMA foreign_keys = ON` çalıştırılır (SQLite'ta FK ve `ON DELETE CASCADE/SET NULL` yalnızca bu etkinken çalışır); değer okunarak doğrulanır, etkin değilse hata gösterilir ve veritabanı açılmaz. Adapter ayrıca kritik silmeleri (Card purge, düğüm silme) FK cascade'ine **ek olarak** açık işlemlerle de yapar (tek transaction): Card purge'de önce `nodes.card_id` `NULL` yapılır, sonra Card silinir; düğüm silmede alt ağacın Item ve ItemValue kayıtları açıkça silinir, sonra düğümler silinir.
- **ID stratejisi:** `INTEGER PRIMARY KEY AUTOINCREMENT` yerine her tabloda `id TEXT PRIMARY KEY`, uygulama tarafında `crypto.randomUUID()` ile üretilir. Bu, Postgres'teki `uuid` tipiyle doğrudan eşleşir, migration script'i gerekmez.
- **Timestamp'ler:** `created_at` / `updated_at` kolonları ISO 8601 string olarak tutulur (`new Date().toISOString()`).
- **Şema dosyası:** Tüm `CREATE TABLE` ifadeleri, `index.html` içinde tek bir, açıkça yorumla ayrılmış `schema.sql` bloğunda toplanır; bu blok ileride neredeyse değişiklik yapılmadan Supabase migration'ına kopyalanabilecek şekilde yazılmalıdır.
- **Versiyon kontrol:** Tüm kod GitHub'da, repo **public**. Repo kökünde `README.md`, şimdilik yalnızca `index.html`'e link içerir.
- **Hosting:** GitHub Pages (statik hosting).
- **Çalışma ortamı:** Uygulama HTTPS veya localhost üzerinde çalışmalıdır (File System Access API güvenli bağlam ister; `file://` desteklenmez). `file://` ile açılırsa anlaşılır bir İngilizce uyarı gösterilir.
- **Dil:** Arayüz ve tüm mesajlar tamamen İngilizce.
- **Versiyonlama:** Versiyon numarası header'daki rozette gösterilir ve kodda tek bir sabitte tutulur; başlangıç değeri ve sonraki numaralar geliştirme tarafında belirlenir (bu prompt'ta numara atanmaz). Mantık: her güncellemede RC numarası bir artırılır; ilk kararlı sürüm yayınlandığında adlandırma `v1.0` olur.
- **Kapsam dışı (şimdilik):** Map sekmesi, Authentication, RLS (row-level security), gerçek zamanlı senkronizasyon, çoklu kullanıcı/CoWork paylaşımı — bunlar sonraki fazlara/Supabase fazına ertelenir (bkz. `ToDo.md`).

## 6. Veri Modeli

> Aşağıdaki `CREATE TABLE` ifadeleri `index.html` içinde tek bir yorumla sınırlandırılmış şema bloğunda toplanmalıdır.
>
> **Tablo adları sabittir:** `cards`, `properties`, `enum_values`, `nodes`, `items`, `item_values`, `meta`. Farklı adlar kullanılıyorsa idempotent yükseltme adımıyla bu adlara dönüştürülür; mevcut veri korunur.

**Meta** (`meta`)
- `key` TEXT PRIMARY KEY, `value` TEXT NOT NULL. Zorunlu anahtarlar: `db_id` (UUID, `crypto.randomUUID()`) ve `schema_version` (tam sayı metni). Bkz. 3.2.1.

**Card**
- `id` TEXT PRIMARY KEY (uygulama tarafında `crypto.randomUUID()` ile üretilir)
- `user_id` TEXT NOT NULL DEFAULT `'sqlite'` — *gelecekte Supabase'e geçişte `auth.uid()` değeriyle değiştirilecek placeholder kolon; şimdilik her zaman sabit `'sqlite'` değerini taşır.*
- `name` TEXT NOT NULL (Card Name, max 255 char, trim)
- `name_normalized` TEXT NOT NULL — `name`'den tüm boşluklar kaldırılmış + lowercase hâli; uygulama tarafında hesaplanıp yazılır (INSERT/UPDATE sırasında JS tarafında)
- `title` TEXT NOT NULL (max 255 char, trim; boşsa Card Name'den otomatik doldurulur)
- `description` TEXT (opsiyonel, trim)
- `template` TEXT (opsiyonel, serbest)
- `is_disabled` INTEGER NOT NULL DEFAULT 0 (boolean yerine 0/1)
- `created_at` TEXT NOT NULL, `updated_at` TEXT NOT NULL (ISO 8601)
- **UNIQUE** (`user_id`, `name_normalized`) — şimdilik `user_id` sabit olduğu için pratikte dosya genelinde benzersizliği sağlar; Supabase'e geçişte aynı kısıt gerçek `user_id` ile birebir çalışmaya devam eder.

**Property**
- `id` TEXT PRIMARY KEY
- `card_id` TEXT NOT NULL (FK → Card.id, `ON DELETE CASCADE`)
- `type` TEXT NOT NULL (`icon` | `text` | `number` | `hex` | `enum` | `date` | `tab` | `separator` | `percent` | `checkbox`; şemada `type` üzerinde CHECK kısıtı kullanılacaksa tüm bu değerler listelenmelidir)
- `label` TEXT NOT NULL (max 255 char, trim)
- `config` TEXT (opsiyonel, JSON string; tipe özel ayarları taşır; Postgres'e geçişte `jsonb`'ye dönüştürülebilir). İçerik tipe göre:
  - number: `{"kind":"integer"}` veya `{"kind":"decimal"}`
  - hex: `{"min_length":<int|null>,"max_length":<int|null>}`
  - date: `{"include_time":true|false}`
  - percent: `{"min":<int>,"max":<int>}`
  - checkbox: `{"default":true|false}`
  - text: `{"max_length":<int 1-4000>}` (yoksa 255 varsayılır)
  - icon / enum / tab / separator: `null`
- `is_key` INTEGER NOT NULL DEFAULT 0 (şimdilik işlevsiz, yalnızca saklanır; tab/separator için her zaman 0)
- `visibility` TEXT NOT NULL DEFAULT `'show'` (`show` | `hide` | `private`; şimdilik işlevsiz; tab/separator için her zaman `'show'`)
- `sort_order` INTEGER NOT NULL (`order` SQL'de ayrılmış kelime olduğu için bu isim kullanılır)
- `created_at` TEXT NOT NULL, `updated_at` TEXT NOT NULL

**EnumValue**
- `id` TEXT PRIMARY KEY
- `property_id` TEXT NOT NULL (FK → Property.id, `ON DELETE CASCADE`)
- `value` TEXT NOT NULL (max 255 char, trim, orijinal biçimiyle saklanır)
- `value_normalized` TEXT NOT NULL (lowercase hâli, uygulama tarafında hesaplanır)
- `usage_count` INTEGER DEFAULT 0 (opsiyonel)
- `created_at` TEXT NOT NULL
- **UNIQUE** (`property_id`, `value_normalized`)

**Node** (`nodes` — Item-Tree düğümleri / Project'ler)
- `id` TEXT PRIMARY KEY (`crypto.randomUUID()`)
- `user_id` TEXT NOT NULL DEFAULT `'sqlite'`
- `parent_id` TEXT NULL (FK → nodes.id, `ON DELETE CASCADE`; NULL = kök/Project)
- `name` TEXT NOT NULL (max 255, trim; tekil olmak zorunda değil)
- `card_id` TEXT NULL (FK → cards.id, `ON DELETE SET NULL`; NULL = "Folder" tipi, dolu = o Card'a bağlı)
- `sort_order` INTEGER NOT NULL (aynı `parent_id` altındaki kardeşler arasında; kökler için `parent_id IS NULL` grubu)
- `created_at`, `updated_at` TEXT NOT NULL (ISO 8601)

**Item**
- `id` TEXT PRIMARY KEY, `card_id` TEXT NOT NULL (FK → cards.id, `ON DELETE CASCADE`), `node_id` TEXT NOT NULL (FK → nodes.id, `ON DELETE CASCADE`), `user_id` TEXT NOT NULL DEFAULT `'sqlite'`, `sort_order` INTEGER NOT NULL (aynı düğüm içindeki sıra), `created_at` TEXT NOT NULL, `updated_at` TEXT NOT NULL
- **Tutarlılık kuralı:** Bir Item'ın `card_id` değeri, bağlı olduğu düğümün `card_id` değerine **eşit olmalıdır**; adapter Item oluşturma/güncellemede bunu doğrular ve ihlalde kaydı reddeder.

**ItemValue**
- `id` TEXT PRIMARY KEY, `item_id` TEXT NOT NULL (FK, `ON DELETE CASCADE`), `property_id` TEXT NOT NULL (FK), `value` TEXT
- `value` kolonu tüm tipler için TEXT olarak kalır (Tab ve Separator değer taşımadığı için ItemValue kaydı oluşturmaz; CheckBox `0`/`1`, Percent tam sayı metni olarak saklanır); Hex değerleri `0x` ön ekiyle ve küçük harfe normalize edilmiş olarak saklanır (ör. `0x1a2f`).

**Silme davranışları:**
- **Disable** (Del): `UPDATE cards SET is_disabled = 1 WHERE id = ?`. Cascade yoktur.
- **Purge (Card):** Card kaydının gerçek `DELETE`'idir ve **yalnızca Card'a bağlı Item yoksa** yapılabilir (UI'da devre dışı; adapter'da da engellenir). Property ve EnumValue kayıtları `ON DELETE CASCADE` ile silinir; Card'a bağlı düğümlerin `card_id`'si `NULL` olur (düğümler "Folder"a döner).
- **Düğüm silme:** Düğüm, tüm alt ağacı ve bu düğümlerdeki Item/ItemValue kayıtları (`ON DELETE CASCADE`) tek transaction'da kalıcı silinir.
- **Item silme:** Onay sonrası Item ve ItemValue'ları kalıcı silinir (disable kavramı yoktur).
- **Geriye dönük uyumluluk:** Önceki sürümlerle oluşturulmuş `.sqlite` dosyaları açılırken şema `meta.schema_version`'a göre idempotent biçimde yükseltilir (`CREATE TABLE IF NOT EXISTS`; `meta` yoksa oluşturulur; eski dosyalarda boş olan `items` tablosu yeni kolonlarla yeniden oluşturulabilir). Dosya açılışı hata vermemelidir (bkz. 3.2.1).

## 7. Arayüz / UX Notları
- Header ve tab menüsü tasarımı, Ekler bölümündeki ekran görüntülerinin genel yerleşim mantığına (logo + isim + versiyon solda, altında tab menüsü) görsel referans olarak uygulanmalıdır.
- Enum property'lerin type-ahead arama kutusu, Ekran Görüntüsü 4'teki gibi olmalı.
- Card Name alanı: kullanıcı yazarken canlı doğrulama (kırmızı/yeşil, boşluksuz + case-insensitive karşılaştırma).
- Title alanı: New Card modalında Card Name'den otomatik doldurma davranışı.
- Tema desteği (dark/light), header'daki bağımsız ikon buton ile toggle edilebilir; seçim kalıcı olmalı (localStorage).
- Header'da dosya adı rozetinin yanında kayıt durumu göstergesi bulunur ("Saved / Saving… / Unsaved changes / Save failed", bkz. 3.2.1).
- Items sayfası: iki panelli File Explorer görünümü (bkz. 3.7); sayfa kaymaz, paneller kendi içinde kayar. Item formunda Label solda / giriş sağda tek satır yerleşimi kullanılır.
- **Marka Kimliği:**
  - **Logo:** "Stacked cards" konsepti — üç adet üst üste, diyagonal kaydırılmış yuvarlak köşeli dikdörtgen; sırasıyla arkadan öne **indigo (#4F46E5)**, **teal (#0D9488)**, **coral (#F97316)** renkleriyle doldurulur. Gömülü SVG olarak header'da wordmark'ın solunda kullanılır (referans: `CMAP-Logo.svg`).
  - **Slogan:** "Your records, unlimited relationships." — wordmark'ın altında/yanında, ikincil (muted) metin rengiyle gösterilir.

## 8. Güvenlik ve Gizlilik Gereksinimleri
- Şimdilik kimlik doğrulama veya sunucu tarafı erişim kontrolü **yoktur**; veri erişimi, kullanıcının kendi diskindeki `.sqlite` dosyasıyla sınırlıdır. Dosya sistemi izinleri (File System Access API izin diyalogları) tamamen tarayıcı tarafından yönetilir; uygulama bu izinleri varsayamaz, her zaman izin reddi senaryosunu ele almalıdır (bkz. Bölüm 9).
- **Gelecek faz notu:** Supabase'e geçişte RLS politikaları `user_id = auth.uid()` temelinde uygulanacak ve CoWork paylaşım mekanizması (`card_access` tablosu) eklenecektir; bu, Bölüm 6'daki `user_id` placeholder kolonu ve adapter soyutlaması sayesinde UI katmanına dokunmadan yapılabilmelidir. Şimdilik bu uygulanmaz.
- **Girdi temizleme / injection koruması:** Tüm metin girdileri (Card Name, Title, Description, Template, Property Label, Enum değerleri, düğüm adları, Item değerleri, filtre/arama alanları) zararlı kod enjeksiyonuna karşı temizlenmelidir:
  - Tüm SQL sorguları **parametreli** (prepared statement / bind parametresi) olmalı; kullanıcı girdisinin SQL string'ine birleştirilmesi yasaktır (filtre/arama alanındaki `LIKE` sorguları dahil).
  - Kullanıcıdan gelen veya veritabanından okunan hiçbir metin `innerHTML` ile ekrana basılmamalı; `textContent` kullanılmalı veya çıktı HTML-escape edilmelidir.
  - Kaydetmeden önce metinlerden **kontrol karakterleri** temizlenmelidir (Text area alanlarında satır sonu/tab korunur).
  - Tab (sekme adı) ve Separator (alt başlık) Label'ları hiçbir koşulda HTML olarak yorumlanmamalıdır.
  - Kullanıcı girdisi hiçbir koşulda SVG/HTML/script olarak yorumlanmamalıdır; ikonlar yalnızca gömülü sabit set içinden, anahtar adıyla çözülür.
  - `db_id`, IndexedDB'den okunan handle/dosya adı ve localStorage değerleri güvenilmez girdi sayılır ve doğrulanır; dosya adı, durum ve hata mesajları `textContent`/HTML-escape ile gösterilir.
  - "Open Database" ile açılan `.sqlite` dosyasının içeriği de güvenilmez girdi sayılır; okunan tüm metinler ekrana basılırken aynı escape kurallarına tabidir.

**Girdi doğrulama kuralları (alan bazında):**
| Alan | Zorunlu | Max Karakter | Diğer Kurallar |
|---|---|---|---|
| Card Name | Evet | 255 | Harf/rakam/boşluk + özel karakterlere izin verilir; trim; dosya içinde benzersiz (boşluksuz + case-insensitive) |
| Card Title | Evet | 255 | Trim; boşsa Card Name'den otomatik doldurulur |
| Card Description | Hayır | Sınırsız | Trim |
| Template | Hayır | Sınırsız | Serbest metin |
| Property Label | Evet | 255 | Trim |
| Enum değeri | Evet (Item girişinde) | 255 | Trim; case-insensitive tekilleştirme (property'ye özel) |
| Düğüm adı (Item-Tree) | Evet | 255 | Trim; tekil olmak zorunda değil |
| Text Item değeri | Hayır | Property'nin `max_length` değeri (1–4000, varsayılan 255) | Trim; ≤ 100 ise satır sonu yok |

### 8.1 Item Formu Doğrulama Kuralları
Item formu, her Property tipi için değeri kaydetmeden önce şu kurallarla doğrulamalı ve hata durumunda İngilizce, alan bazlı mesaj göstermelidir:

| Tip | Geçerlilik kuralı |
|---|---|
| Text | Trim uygulanır, kontrol karakterleri temizlenir (çok satırlı alanlarda satır sonu ve tab korunur); uzunluk Property'nin `max_length` değerini (varsayılan 255) aşamaz; `max_length` ≤ 100 ise satır sonu içeremez. |
| Number | Config'teki türe uyar: `integer` için yalnızca tam sayı; `decimal` için nokta ayraçlı ondalık sayı. Sayısal olmayan girdi reddedilir. |
| Hex | Giriş alanı `0x` ön ekiyle dolu gelir. Ön ek zorunludur; ondan sonra yalnızca `0-9`, `a-f`, `A-F` karakterleri ve en az 1 hane olmalıdır. Hane sayısı (ön ek hariç) Config'teki min/max sınırları içinde olmalıdır. Kayıtta ön ek `0x` ve tüm harfler **küçük harfe** normalize edilir. |
| Date | Giriş `DD-MM-YYYY` veya (Include time açıksa) `DD-MM-YYYY hh:mm:ss`; gerçek takvim tarihi (ör. 31-02-2025 geçersiz) ve geçerli saat (00–23, 00–59, 00–59); ISO 8601 olarak saklanır (`YYYY-MM-DD` / `YYYY-MM-DDTHH:mm:ss`). |
| Enum | Boş olamaz, max 255 karakter, trim; Property'ye özel type-ahead kuralları geçerlidir. |
| Icon | Değer, gömülü ikon setindeki geçerli bir anahtar olmalıdır. |
| Tab | Değer taşımaz; doğrulama yok. Label yalnızca görüntülenir. |
| Separator | Değer taşımaz; doğrulama yok. Label yalnızca görüntülenir. |
| Percent | Tam sayı olmalı; ondalık veya sayısal olmayan girdi reddedilir; Config'teki min/max aralığında olmalıdır. |
| CheckBox | Yalnızca true/false (`1`/`0`) kabul edilir; yeni Item'da başlangıç değeri Config'teki `default`'tur. |

Doğrulama hem arayüzde (anlık geri bildirim) hem adapter katmanında (kayıt öncesi, son savunma) uygulanmalıdır.

## 9. Hata Yönetimi ve Kenar Durumlar
- **Dosya izni reddi (`NotAllowedError`)** veya **handle kaybı** (ör. dosya taşınmış/silinmiş): kullanıcıya açık bir hata mesajı gösterilmeli ve elle "Save"/export seçeneği her zaman UI'da bulunmalı.
- File System Access API desteklenmeyen tarayıcılarda otomatik kaydetme yapılamayacağı kullanıcıya açıkça belirtilmeli; Blob/download + file-input fallback akışı devreye girmeli.
- Bir Card silinirken/purge edilirken işlem yarıda kesilirse veri tutarsızlığı oluşturmayacak şekilde (tek SQL transaction'ı içinde) uygulanmalı.
- Purge onayı tek adımlı bir modal ile alınır; ek bir ikincil onay adımına gerek yoktur.
- `PRAGMA foreign_keys = ON` etkinleştirilemezse veritabanı açılmaz (bkz. Bölüm 5). Yabancı/bozuk/daha yeni şemalı dosya, yazma hatası ve dış değişiklik senaryoları için bkz. 3.2.1.
- Çok adımlı Items işlemleri (düğüm taşıma, alt ağaç silme, sıralama, Item + ItemValue + EnumValue yazımı) tek SQL transaction'ı içinde yapılır; her işlemden sonra dosya diske yazılır.
- Properties listesi boş bir Card kaydedilebilir; ayrıca bir uyarı mesajına gerek yoktur.
- Enum type-ahead önerileri boşken ayrıca bir empty-state mesajına gerek yoktur.

## 10. Kabul Kriterleri (Definition of Done)
- [ ] Uygulama açıldığında veritabanı seçilmemişken "Open Database" / "New Database" seçenekleri doğru gösteriliyor; bu durumda Map/Items/Cards tabları görünmüyor.
- [ ] "New Database" ile `CMAP.sqlite` varsayılan adıyla yeni dosya oluşturulabiliyor; "Open Database" ile mevcut dosya açılabiliyor.
- [ ] Veritabanı açıldıktan sonra header'da dosya adı, kayıt durumu göstergesi, "Save" ve "Switch Database" butonları doğru gösteriliyor; veritabanı açılışında Card kaydı varsa Items, yoksa Cards sekmesi seçili geliyor.
- [ ] Her CRUD işleminden sonra dosya otomatik olarak diske yazılıyor (debounce ile); manuel "Save" her zaman çalışıyor.
- [ ] File System Access API desteklenmeyen tarayıcıda Blob/download + file-input fallback akışı çalışıyor.
- [ ] Header logo, wordmark, slogan ve versiyon rozeti (kodda tanımlı sabitten okunarak) doğru görünüyor; tema toggle header'da bağımsız çalışıyor.
- [ ] Tüm arayüz metinleri ve sistem mesajları İngilizce.
- [ ] Cards tabında: filtre, "Hide disabled cards" anahtarı, "New Card" butonu ve 7 kolonlu tablo doğru çalışıyor.
- [ ] Disable edilmiş Card'lar tabloda soluk görünüyor; filtre anahtarı açıkken tamamen gizleniyor.
- [ ] New/Edit Card modalı tüm alanları doğru kaydediyor; Card Name canlı benzersizlik doğrulaması (kırmızı/yeşil) + DB unique constraint (`name_normalized`) birlikte çalışıyor.
- [ ] New Card'da Title alanı boşken Card Name'den otomatik dolduruluyor.
- [ ] Property tipleri (Icon — gömülü SVG set, Text, Number, **Hex**, Enum, Date, **Tab, Separator, %, CheckBox**) ve ayarları (Label, Key, Visibility) tanımlanabiliyor; Label zorunlu alan kontrolü çalışıyor.
- [ ] Tab ve Separator seçildiğinde Key/Visibility alanları gizleniyor; Percent'te Min/Max (varsayılan 0/100, 0–100 aralığı, Min ≤ Max) doğrulaması ve CheckBox'ta "Default value" seçimi (varsayılan Unchecked) çalışıyor ve `config` alanında saklanıyor.
- [ ] Properties tablosunda yukarı/aşağı taşıma butonları çalışıyor, `sort_order` kalıcı ve ardışık güncelleniyor; uç satırlarda ilgili buton devre dışı.
- [ ] Cards tablosundaki "# of Properties" Tab ve Separator satırlarını saymıyor.
- [ ] Number/Hex/Date tipleri seçildiğinde tipe özel ayarlar (Integer/Decimal, Min/Max length, Include time) Card tanımı ekranında görünüyor ve `config` alanında saklanıyor; Hex Min/Max doğrulaması (pozitif tam sayı, Min ≤ Max) çalışıyor.
- [ ] Enum type-ahead öneri sistemi Property'ye özel ve case-insensitive tekilleştirme ile çalışıyor.
- [ ] Del → onay modalı → disable (cascade yok); Purge → onay modalı → gerçek DELETE (yalnızca Item'ı olmayan Card; Property/EnumValue cascade ile silinir, bağlı düğümler Folder'a döner); Item'ı olan Card'da Purge devre dışı + ipucu; Recover çalışıyor.
- [ ] Tüm CRUD işlemleri UI'dan doğrudan değil, tek bir adapter bloğu üzerinden yapılıyor (kod incelemesiyle doğrulanabilir).
- [ ] Şema (`CREATE TABLE` ifadeleri) tek bir yorumla ayrılmış blokta toplanmış; `AUTOINCREMENT`/`PRAGMA` kullanılmamış; tüm ID'ler `TEXT` UUID, tüm timestamp'ler ISO 8601 string.
- [ ] Tüm metin girdileri (düğüm adları, Item değerleri, tooltip metinleri dahil) parametreli sorgularla yazılıyor, ekrana `textContent`/escape ile basılıyor, kontrol karakterleri temizleniyor; açılan `.sqlite` dosyasındaki içerik de güvenli render ediliyor.
- [ ] Items tabı iki panel gösteriyor (~%30 "Item-Tree", ~%70 "Items"); sayfa kaymıyor, paneller kendi içinde dikey kayıyor; Map tabı "Coming soon".
- [ ] "New Project", "Project:" dropdown'ı, kökte dahil +/kalem/çöp ikonları, düğüm formu (Name zorunlu max 255; Type varsayılan Folder, disable edilmiş Card'lar listelenmiyor; Item'ı olan düğümün tipi kilitli), düğüm adlarının tekil olmak zorunda olmaması çalışıyor.
- [ ] Ağaçta sürükle-bırak (alt düğüm yapma, sıralama, kendi alt ağacına bırakmanın engellenmesi, kök düğümlerin sürüklenememesi, kalıcılık) ve düğüm silme (alt düğüm/Item sayısı gösteren onay, tüm alt ağacın kalıcı silinmesi) çalışıyor; aç/kapa, seçili düğüm ve Project localStorage'dan geri yükleniyor.
- [ ] Items tablosu seçili düğümün doğrudan Item'larını Card Property'leriyle (Tab/Separator ve Hide hariç) gösteriyor; arama çalışıyor; "New" yalnızca aktif Card'a bağlı düğümde görünüyor; disable edilmiş Card'da tablo görünüyor, kalem devre dışı; satırlar aynı düğüm içinde sürüklenerek sıralanıyor ve sıra kalıcı; Item silme onaylı ve kalıcı.
- [ ] Item formu: Tab'ler sekme, Separator alt başlık+çizgi olarak render ediliyor; Label solda / giriş sağda tek satır yerleşimi (dar ekranda üstte); tüm tip denetimleri (Hex `0x` ön eki, Percent "%", CheckBox varsayılanı, Icon seçici, Enum type-ahead) çalışıyor.
- [ ] Text: "Max length" ayarı (1–4000, varsayılan 255; eski Property'lerde 255 varsayımı); `max_length` ≤ 100 tek satırlı input, > 100 textarea; "n/max" sayacı canlı, limitte yazma duruyor ve sayaç kırmızı, yapıştırma limitte kesiliyor; adapter limit aşımını reddediyor; limiti sonradan düşürülmüş Card'ın Item'ları değişmiyor ama düzenlerken limit aşıldıkça kayıt yapılamıyor.
- [ ] Date: maskeli metin kutusu + takvim butonu (`DD-MM-YYYY` / `DD-MM-YYYY hh:mm:ss`), geçersiz tarih/saat reddediliyor, ISO 8601 saklanıyor; tabloda saatsiz Date `YYYY-MM-DD`, saatli `YYYY-MM-DD hh:mm:ss` görünüyor.
- [ ] Bölüm 8.1'deki doğrulama kuralları Item formunda hem arayüzde hem adapter'da uygulanıyor.
- [ ] Cards tablosundaki "# of Records" gerçek Item sayısını gösteriyor.
- [ ] Önceki sürümlerle oluşturulmuş `.sqlite` dosyaları sorunsuz açılıyor (şema idempotent yükseltilir).
- [ ] Veritabanı açılışında `PRAGMA foreign_keys = ON` uygulanıyor ve doğrulanıyor; şema bloğunda PRAGMA ve STRICT yok; Card purge ve düğüm silme hem FK hem açık işlemlerle doğru çalışıyor.
- [ ] `meta` tablosu (`db_id`, `schema_version`) yeni dosyalarda oluşuyor; eski dosyalar açılırken yükseltilip `db_id` alıyor; localStorage anahtarları `db_id` ile tutuluyor; tablo adları sabit; `items.card_id` = `nodes.card_id` tutarlılığı adapter'da zorlanıyor.
- [ ] CMAP olmayan, boş, bozuk veya daha yeni şemalı dosya açılınca doğru İngilizce hata gösteriliyor ve uygulama karşılama ekranında kalıyor.
- [ ] "Reopen last: <ad>" butonu çalışıyor (izin isteme, geçersiz handle temizliği, Switch sonrası görünüm).
- [ ] Autosave 1500 ms debounce ile çalışıyor; durum göstergesi doğru; yazma hatasında veri korunuyor ve yeniden deneniyor; kaydedilmemiş değişiklikle kapatmada `beforeunload` uyarısı çıkıyor; fallback modunda uyarı geçerli.
- [ ] Dosya dışarıdan değiştirildiğinde yazma duruyor ve üç seçenekli modal (Overwrite file / Reload from disk / Save a copy…) çalışıyor.
- [ ] Değişiklik yapılmış modallarda kapatma girişimi "Keep editing / Discard" uyarısı gösteriyor; değişiklik yoksa uyarı yok.
- [ ] Items tablosunda kolon başlığı sıralaması (artan/azalan/sıralamasız, tipe uygun karşılaştırma, boşlar sonda) çalışıyor; sıralama aktifken satır sürükleme devre dışı ve ipucu görünüyor.
- [ ] Satıra tıklamak Edit formunu açıyor; ikonlara ve tutacağa tıklamak açmıyor; disable edilmiş Card'da satır tıklanamıyor.
- [ ] Disable edilmiş Card'ın Edit butonu aktif; Card Name benzersizliği disabled Card'ları da kapsıyor.
- [ ] sql.js sürümü sabit, SRI hash'li; `.wasm` aynı sürümden yükleniyor; `file://` açılışında uyarı veriliyor.
- [ ] Bölüm 8'deki tüm girdi doğrulama kuralları uygulanmış.
- [ ] Empty-state ve loading-state'ler uygulanmış.
- [ ] Uygulama GitHub Pages üzerinde deploy edilebiliyor; repo public, tek dosya `index.html` (sql.js CDN script dahil, inline CSS/JS), kökte `README.md`.
- [ ] Tema (dark/light) toggle çalışıyor ve tercih kalıcı.

## 11. Ekler
⚠️ **AI kodlama ajanına not:** Aşağıdaki dosyalar bu prompt ile aynı klasörde (`Prompt/`) bulunur. **Ekran Görüntüsü 1 ve 2 (Google/Microsoft Sign-In ve profil dropdown) bu mimaride geçersizdir ve uygulanmamalıdır** — yerine Bölüm 3.2'de tanımlanan "Open Database / New Database" akışı kullanılacaktır. Ekran Görüntüsü 3 ve 4 hâlâ geçerlidir; kodlamaya başlamadan önce bu görsellere eriş, erişemiyorsan kullanıcıdan paylaşmasını iste.
- ~~`CMAP-Ref-1-signin-dropdown.png`~~ — *geçersiz, arşiv amaçlı tutulur; kullanılmayacak.*
- ~~`CMAP-Ref-2-profile-dropdown.png`~~ — *geçersiz, arşiv amaçlı tutulur; kullanılmayacak.*
- `CMAP-Ref-3-header-and-tabs.png` — CMAP header düzeni (logo + isim + versiyon rozeti) ve altındaki kırmızı alt çizgili tab menüsü; genel yerleşim referansı olarak geçerli. (Not: görseldeki ikon başka bir projeye aittir; CMAP logosu için `CMAP-Logo.svg` kullanılacaktır.)
- `CMAP-Ref-4-enum-typeahead.png` — Enum property için type-ahead arama kutusu ve dinamik öneri listesi tasarımı; geçerli.
- `CMAP-Logo.svg` — Üç renkli "stacked cards" logo (Bölüm 7).
