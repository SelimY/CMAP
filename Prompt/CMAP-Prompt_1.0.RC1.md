# CMAP — Yazılım Geliştirme Prompt'u (1.0.RC1 / Faz-1)

## 1. Özet
CMAP, kullanıcıların kendi tanımladıkları dinamik "Card" şablonları (form/kayıt tipleri) üzerinden "Item" adında yapılandırılmış kayıtlar oluşturup yönetebildiği, **tek-kullanıcılı ve tamamen offline-first** çalışan bir web uygulamasıdır. Kullanıcı, önce bir Card (şablon) tanımlar; bu Card'a ait Property'ler (alanlar) belirler; ardından bu Card'ı kullanarak istediği kadar Item (gerçek veri kaydı) oluşturur. Tüm veri, kullanıcının kendi diskindeki tek bir `.sqlite` dosyasında tutulur; sunucu tarafı bir backend veya hesap sistemi yoktur. Faz-1 kapsamı yalnızca Card tanımlama/yönetimi ve veritabanı dosyası yönetimi ile sınırlıdır; Item oluşturma akışının UI detayları ve Map sekmesi sonraki fazlarda tanımlanacaktır.

## 2. Hedef Kullanıcı ve Kullanım Senaryosu
- Faz-1'de uygulama tek-kullanıcılı ve offline-first'tir; tüm veri, kullanıcının seçtiği/oluşturduğu tek bir `.sqlite` dosyasında yerel olarak saklanır. Hesap/giriş sistemi yoktur.
- **Gelecek faz için not (Supabase geçişi):** İleride veritabanı katmanının Supabase (PostgreSQL + Auth + RLS) üzerine taşınması ve "CoWork" adında bir paylaşım özelliği (bir kullanıcının başka bir kullanıcıya belirli bir Card'a okuma/yazma erişimi vermesi) eklenmesi planlanmaktadır. Faz-1'de bu geçiş UI olarak geliştirilmeyecek; yalnızca şema ve veri erişim katmanı bu geçişi kolaylaştıracak şekilde tasarlanacaktır (bkz. Bölüm 5 ve 6).

## 3. Temel Özellikler (Fonksiyonel Gereksinimler)

### 3.1 Genel Sayfa Yapısı
1. Sayfa header'ı solda: SW logosu, adı ("CMAP"), yanında versiyon rozeti (`1.0.RC1`), altında/yanında slogan.
2. Header sağında, **veritabanı durumuna göre** değişen bir alan bulunur (bkz. 3.2) ve her zaman görünen hamburger menü ikonu (içerik Faz-1'de boş/placeholder) ile tema seçim (dark/light) ikon butonu yer alır.
3. Header ve tab düzeni, paylaşılan ekran görüntülerindeki (Ekler bölümüne bak) genel yerleşim mantığına (logo + isim + versiyon rozeti solda; altında kırmızı alt çizgili aktif tab göstergesi olan tab menüsü) **görsel referans** olarak uygulanmalıdır.
4. **Tüm arayüz metinleri, buton etiketleri, hata/onay/uyarı mesajları ve bilgilendirme yazıları İngilizce olmalıdır.** Türkçe metin kullanılmamalıdır.

### 3.2 Veritabanı Yönetimi (Database Management)
5. Uygulama ilk açıldığında (henüz bir `.sqlite` dosyası seçilmemişken), header'ın sağında **"Open Database"** ve **"New Database"** butonları gösterilir; ana içerik alanında da bu iki seçeneği sunan bir karşılama ekranı/mesajı bulunur. Bu durumda Map/Items/Cards tabları gösterilmez.
6. **"New Database"** akışı: Kullanıcıya varsayılan dosya adı olarak **`CMAP.sqlite`** önerilir (kullanıcı dilerse değiştirebilir); tarayıcının "save file" diyaloğu açılır, boş şema (bkz. Bölüm 6) ile yeni bir `.sqlite` dosyası oluşturulur.
7. **"Open Database"** akışı: Tarayıcının dosya seçim diyaloğu açılır, kullanıcı mevcut bir `.sqlite` dosyası seçer, dosya sql.js ile belleğe yüklenir.
8. Bir veritabanı dosyası açıldığında/oluşturulduğunda: "Open Database"/"New Database" butonları kaybolur, yerine **o anki dosya adı** (ör. "CMAP.sqlite") header'da küçük bir rozet olarak gösterilir; yanında manuel **"Save"** (anlık export) ve **"Switch Database"** (mevcut dosyayı kapatıp yeniden seçim ekranına dönme) ikon butonları yer alır. Bu noktadan sonra Map/Items/Cards tabları görünür hale gelir.
9. **Kalıcılık (File System Access API):** Seçilen/oluşturulan dosyanın `FileSystemFileHandle`'ı uygulama genelinde (global state) tutulur. Her CRUD işleminden sonra (veya kısa bir debounce aralığıyla) `db.export()` çıktısı, aynı handle üzerinden `createWritable()` → `write()` → `close()` adımlarıyla diske otomatik yazılır. Elle **"Save"** butonu her zaman UI'da bulunur ve anlık export tetikler.
10. File System Access API yalnızca Chromium tabanlı tarayıcılarda (Chrome, Edge) desteklenir. Desteklenmeyen tarayıcılarda uygulama şu **fallback** moduna düşmelidir: veritabanı dosyası bir `Blob` olarak `<a download>` ile dışa aktarılır, açma işlemi ise `<input type="file">` ile dosya seçtirilerek yapılır (otomatik diske yazma yapılamaz; kullanıcı her değişiklikten sonra manuel "Save/Download" ile dışa aktarmalıdır — bu durum kullanıcıya UI'da açıkça belirtilmelidir).

### 3.3 Ana Navigasyon
11. Bir veritabanı açıldıktan/oluşturulduktan sonra ekranda 3 tab belirir: **Map**, **Items**, **Cards** (Ekran Görüntüsü 3'teki genel yerleşime görsel referans).
12. Faz-1'de yalnızca **Cards** tabinin fonksiyonları tanımlanmıştır. Map ve Items tabları arayüzde görünmeli, tıklanabilir olmalı, ancak içerik olarak "Coming soon" durumu göstermelidir.

### 3.4 Cards Tab — Card Listeleme
13. Cards tabına geçildiğinde, kayıtlı Card'ları listeleyen bir tablo gösterilmeli.
14. Tablonun hemen üzerinde: bir filtre/arama alanı (sol) ve "New Card" butonu (sağ).
15. Tablo kolonları:
    - Card Name
    - Title
    - # of Properties
    - # of Records (bu Card ile oluşturulmuş Item sayısı)
    - Edit (düzenleme butonu)
    - Del (silme butonu — kaydı fiziksel silmez, `is_disabled` alanıyla devre dışı bırakır/gizler)
    - Purge/Recover (yalnızca disable edilmiş Card'lar için görünür: Purge = kalıcı silme, Recover = tekrar aktif etme)
16. Del butonuna basıldığında, işlem gerçekleşmeden önce bir **onay modalı** gösterilmeli (İngilizce — ör. "Are you sure you want to disable this Card?").
17. Purge butonuna basıldığında da bir **onay modalı** gösterilmeli (ör. "This action cannot be undone. This Card and all its Items will be permanently deleted."). Tek adımlı modal onayı yeterlidir.
18. Bir Card disable edildiğinde: Card'a ait Item verileri **değişmez/silinmez**, yalnızca o Card'a ait Item'ları listeleyen arayüz elemanları gizlenir.
19. Bir Card purge edildiğinde: Card'a bağlı tüm Item kayıtları da **kalıcı olarak silinir**.
20. Disable edilmiş Card'lar tablodan kaldırılmaz; listede **soluk (opacity: 0.5)** görünümle kalırlar. Tablo üzerinde bir **"Hide disabled cards"** filtre anahtarı bulunmalı; varsayılan kapalıdır (disabled Card'lar soluk gösterilir), açıldığında tamamen gizlenir.
21. Tablo boşsa veya filtre sonucunda hiç kayıt kalmıyorsa uygun bir **empty-state** mesajı gösterilmeli (İngilizce, ör. "No Cards yet" / "No Cards match your filter").
22. Veri veritabanından yüklenirken uygun bir **loading-state** gösterilmeli.

### 3.5 New Card / Edit Card Modalı
23. "New Card" butonu bir modal açar; bu modal aşağıdaki alanları içerir:
    - **Card Name** (zorunlu, Text)
    - **Title** (zorunlu, Text) — Kullanıcı bu alanı boş bırakıp Card Name'i girip başka bir alana geçerse, Title alanına **otomatik olarak Card Name'deki değer kopyalanır** (yalnızca Title hâlâ boşsa).
    - **Description** (Text area, opsiyonel)
    - **Properties** — Card'a ait alt-kayıt listesi (bkz. 3.6)
    - **Template** — en altta, Text area, opsiyonel. Faz-1'de yalnızca veri modelinde yer tutucu olarak bulunur.
24. Edit Card akışında aynı modal, mevcut Card verileriyle önceden doldurulmuş olarak açılır; Properties listesi **düzenlenebilir bir tablo** olarak gösterilmelidir. Edit modunda Title'ın Card Name'den otomatik kopyalanması davranışı tekrar tetiklenmez.
25. **Card Name benzersizlik kontrolü:** Aynı veritabanı dosyası içindeki Card'lar arasında aynı isim (Card Name) kullanılamaz; Title alanında tekrar serbesttir. Kullanıcı Card Name alanına yazarken **canlı doğrulama** yapılmalı: çakışma varsa input **kırmızı**, uygunsa **koyu yeşil** olmalı. Kaydet butonu çakışma varken devre dışı bırakılmalı.
    - **Karşılaştırma kuralı:** Hem yeni isim hem mevcut isimler üzerinde tüm boşluklar kaldırılır ve case-insensitive karşılaştırılır. Örnek: "First CarD" ile "F i rst  Card" aynı isim sayılır. Veritabanına kaydedilen `name` değeri yalnızca trim uygulanmış orijinal metindir; aradaki boşluklar korunur. Bu kural hem client-side canlı doğrulamada hem veritabanı seviyesinde (bkz. Bölüm 6) uygulanmalıdır.

### 3.6 Property Tanımı
26. Her Card'ın altında sınırsız sayıda Property tanımlanabilir; "New Property" butonu ile ekleme yapılır.
27. Property tipleri:
    - **Icon**: Uygulamaya **gömülü (embedded), harici CDN/font gerektirmeyen bir SVG ikon alt kümesi** (ör. Tabler/Lucide'dan seçilmiş ~150–200 ikonluk, kategorilere ayrılmış bir alt küme, tek renk/outline). Seçici, ikonları kategorilere göre gruplu göstermelidir. Veritabanında yalnızca ikonun adı/anahtarı saklanır.
    - **Text**: Serbest metin girişi.
    - **Number**: Sayısal değer girişi. Sayı türü (tam sayı / ondalık) Card tanımı ekranında seçilir (bkz. madde 29).
    - **Hex**: Genel amaçlı onaltılık (hexadecimal) veri girişi (renk kodu değil; ör. `0x1a2f`, bayt dizisi, kimlik değeri). Giriş alanı her zaman **`0x` ön ekiyle dolu** gelir; ön ek **zorunludur** ve kullanıcı tarafından silinememeli/değiştirilememelidir. Minimum/maksimum uzunluk Card tanımı ekranında belirlenir (bkz. madde 29).
    - **Enum**: Item kaydı sırasında type-ahead ile değer girilen alan. Değer listesi tamamen dinamiktir — kullanıcı yeni değer yazarsa önerilere otomatik eklenir (Ekran Görüntüsü 4'e görsel referans). **Enum değer listesi her zaman ilgili Property'ye özeldir; Card'lar/property'ler arası çapraz kullanılmaz.**
    - **Date**: Tarih seçici. Saat bilgisinin de istenip istenmeyeceği Card tanımı ekranında seçilir (bkz. madde 29).
28. Her Property için ek ayarlar:
    - **Label**: Property'nin soru/etiket metni. **Zorunlu**, maksimum 255 karakter, trim uygulanır.
    - **Key**: Checkbox. Faz-1'de yalnızca veri modelinde tutulur; davranışı Map fazında tanımlanacaktır.
    - **Visibility**: Show / Hide / Private. Faz-1'de yalnızca veri modelinde tutulur; "Private" davranışı Map fazında tanımlanacaktır.
29. **Tipe özel ayarlar:** Property'nin tipi seçildiğinde, Property tanım satırında/modalında o tipe özel aşağıdaki ek ayarlar gösterilmelidir (ilgili tip seçili değilken gizlenir). Bu ayarlar Faz-1'de Card tanımı ekranında girilir ve saklanır; Item girişinde uygulanmaları Item fazında yapılır (bkz. Bölüm 8.1).
    - **Number** → "Number type": `Integer` | `Decimal` (zorunlu seçim, varsayılan: `Integer`). Ondalık ayırıcı nokta (`.`) olmalıdır.
    - **Hex** → "Min length" ve "Max length": `0x` ön eki HARİÇ onaltılık hane sayısı cinsinden, opsiyonel tam sayı alanları (boş bırakılırsa o yönde sınır yoktur). Doğrulama: değerler pozitif tam sayı olmalı; ikisi de girilmişse Min ≤ Max olmalı, aksi halde Property kaydedilemez ve İngilizce bir hata mesajı gösterilmelidir.
    - **Date** → "Include time" onay kutusu (varsayılan: kapalı). Kapalıysa yalnızca tarih (`YYYY-MM-DD`), açıksa tarih+saat (`YYYY-MM-DDTHH:mm`, ISO 8601) tutulur.
    - Icon, Text ve Enum tiplerinde ek ayar yoktur.

## 4. Kullanıcı Akışları

**Akış A — İlk kullanım ve veritabanı oluşturma/açma**
1. Kullanıcı uygulamayı açar; henüz bir veritabanı seçilmediği için karşılama ekranında "Open Database" / "New Database" seçenekleri görünür.
2. "New Database" seçerse: varsayılan ad `CMAP.sqlite` önerilir, dosya konumu seçilir, boş şema ile dosya oluşturulur ve diske yazılır. "Open Database" seçerse: mevcut bir `.sqlite` dosyası seçilir ve sql.js ile yüklenir.
3. Veritabanı hazır olduğunda header'da dosya adı görünür, Map/Items/Cards tabları belirir; kullanıcı Cards tabına yönlendirilir (veya varsayılan seçili gelir).

**Akış B — Card oluşturma**
1. "New Card" butonuna tıklar, modal açılır.
2. Card Name girer; canlı benzersizlik kontrolü çalışır. Title boşsa Card Name'den otomatik kopyalanır; kullanıcı dilerse değiştirir. Description doldurulur (opsiyonel).
3. "New Property" ile Property'ler ekler; her biri için tip, Label, tipe özel ayarlar, Key, Visibility ayarlanır.
4. Template alanına (opsiyonel) serbest metin girilebilir.
5. Kaydeder; işlem adapter fonksiyonu üzerinden çalışır, ardından dosya diske otomatik yazılır; Card, Cards tablosunda listelenir.

**Akış C — Card düzenleme**
1. Kullanıcı Cards tablosunda bir satırın "Edit" butonuna tıklar.
2. Modal, mevcut verilerle açılır; Properties bir tablo olarak düzenlenebilir.
3. Değişiklikler kaydedilir, dosya otomatik yazılır, tablo güncellenir.

**Akış D — Card devre dışı bırakma / kalıcı silme**
1. "Del" butonu → onay modalı → onaylanırsa Card `disabled` olur, tabloda soluk görünür, ilgili Item görünümleri gizlenir.
2. Disable edilmiş Card için "Purge" veya "Recover" seçenekleri görünür.
3. "Purge" → onay modalı → onaylanırsa Card ve bağlı tüm Item'lar kalıcı olarak (cascade ile) silinir.
4. "Recover" → Card tekrar aktif olur, soluk görünüm kalkar.

**Akış E — Veritabanı değiştirme**
1. Kullanıcı header'daki "Switch Database" butonuna tıklar.
2. Varsa bekleyen değişiklikler diske yazılır (veya kullanıcı uyarılır), ardından karşılama ekranına (Akış A, adım 1) dönülür.

## 5. Teknik Tercihler / Kısıtlar
- **Veritabanı motoru:** [sql.js](https://sql.js.org) (SQLite'ın WASM derlemesi), `cdnjs.cloudflare.com/ajax/libs/sql.js` üzerinden tek bir `<script>` tag ile dahil edilir. Bundler, build sistemi veya npm bağımlılığı **kullanılmaz**.
- **Dosya yapısı:** Uygulama **tek bir `index.html` dosyası** içinde, tüm CSS ve JS inline olacak şekilde yapılandırılır. Ana giriş dosyası **repo kökünde `index.html`** olmalıdır (GitHub Pages için gereklidir).
- **Veri erişim katmanı (adapter) — mimari soyutlama:** Dosya tek olduğundan ayrı bir `db-adapter.js` dosyası oluşturulmaz; bunun yerine `index.html` içindeki `<script>` bloğunda, açıkça yorumla sınırlandırılmış (ör. `// ---- DB ADAPTER START/END ----`) tek bir bölümde tüm CRUD fonksiyonları (`createCard`, `readCards`, `updateCard`, `disableCard`, `purgeCard`, `createProperty`, vb.) toplanır. UI kodu **doğrudan `db.run()` / `db.exec()` çağırmaz**, yalnızca bu adapter fonksiyonlarını çağırır. Bu sayede ileride adapter bloğunun içi Supabase client çağrılarıyla değiştirilebilir; UI katmanına dokunulmaz.
- **Kalıcılık:** File System Access API (`showOpenFilePicker` / `showSaveFilePicker`, `createWritable`), yalnızca Chromium tabanlı tarayıcılarda; desteklenmiyorsa Blob + `<a download>` / `<input type="file">` fallback'i (bkz. 3.2/10).
- **SQL lehçesi — Postgres uyumluluğu için kısıtlar:** SQLite'a özgü sentakstan kaçınılır: `AUTOINCREMENT` kullanılmaz (bkz. ID stratejisi), `PRAGMA` ifadelerinden kaçınılır. *(Not: Kullanıcı tarafından istenen "STRICT tablo modu" SQLite'a özgü bir uzantıdır ve Postgres'te karşılığı yoktur; kullanılırsa Supabase migration'ında bu ifadenin kaldırılması gerekecektir — AI ajanı bunu şema bloğunda net bir yorumla işaretlemelidir.)*
- **ID stratejisi:** `INTEGER PRIMARY KEY AUTOINCREMENT` yerine her tabloda `id TEXT PRIMARY KEY`, uygulama tarafında `crypto.randomUUID()` ile üretilir. Bu, Postgres'teki `uuid` tipiyle doğrudan eşleşir, migration script'i gerekmez.
- **Timestamp'ler:** `created_at` / `updated_at` kolonları ISO 8601 string olarak tutulur (`new Date().toISOString()`).
- **Şema dosyası:** Tüm `CREATE TABLE` ifadeleri, `index.html` içinde tek bir, açıkça yorumla ayrılmış `schema.sql` bloğunda toplanır; bu blok ileride neredeyse değişiklik yapılmadan Supabase migration'ına kopyalanabilecek şekilde yazılmalıdır.
- **Versiyon kontrol:** Tüm kod GitHub'da, repo **public**. Repo kökünde `README.md`, Faz-1'de yalnızca `index.html`'e link içerir.
- **Hosting:** GitHub Pages (statik hosting).
- **Dil:** Arayüz ve tüm mesajlar tamamen İngilizce.
- **Versiyonlama:** `1.0.RC1`. Her güncellemede RC numarası bir artırılır; ilk kararlı sürümde `v1.0` olur.
- **Kapsam dışı (bu faz için):** Authentication, RLS (row-level security), gerçek zamanlı senkronizasyon, çoklu kullanıcı/CoWork paylaşımı — bunlar Supabase fazına ertelenir.

## 6. Veri Modeli (Faz-1 kapsamı)

> Aşağıdaki `CREATE TABLE` ifadeleri `index.html` içinde tek bir yorumla sınırlandırılmış şema bloğunda toplanmalıdır.

**Card**
- `id` TEXT PRIMARY KEY (uygulama tarafında `crypto.randomUUID()` ile üretilir)
- `user_id` TEXT NOT NULL DEFAULT `'sqlite'` — *gelecekte Supabase'e geçişte `auth.uid()` değeriyle değiştirilecek placeholder kolon; Faz-1'de her zaman sabit `'sqlite'` değerini taşır.*
- `name` TEXT NOT NULL (Card Name, max 255 char, trim)
- `name_normalized` TEXT NOT NULL — `name`'den tüm boşluklar kaldırılmış + lowercase hâli; uygulama tarafında hesaplanıp yazılır (INSERT/UPDATE sırasında JS tarafında)
- `title` TEXT NOT NULL (max 255 char, trim; boşsa Card Name'den otomatik doldurulur)
- `description` TEXT (opsiyonel, trim)
- `template` TEXT (opsiyonel, serbest)
- `is_disabled` INTEGER NOT NULL DEFAULT 0 (boolean yerine 0/1)
- `created_at` TEXT NOT NULL, `updated_at` TEXT NOT NULL (ISO 8601)
- **UNIQUE** (`user_id`, `name_normalized`) — Faz-1'de `user_id` sabit olduğu için pratikte dosya genelinde benzersizliği sağlar; Supabase'e geçişte aynı kısıt gerçek `user_id` ile birebir çalışmaya devam eder.

**Property**
- `id` TEXT PRIMARY KEY
- `card_id` TEXT NOT NULL (FK → Card.id, `ON DELETE CASCADE`)
- `type` TEXT NOT NULL (`icon` | `text` | `number` | `hex` | `enum` | `date`)
- `label` TEXT NOT NULL (max 255 char, trim)
- `config` TEXT (opsiyonel, JSON string; tipe özel ayarları taşır; Postgres'e geçişte `jsonb`'ye dönüştürülebilir). İçerik tipe göre:
  - number: `{"kind":"integer"}` veya `{"kind":"decimal"}`
  - hex: `{"min_length":<int|null>,"max_length":<int|null>}`
  - date: `{"include_time":true|false}`
  - icon / text / enum: `null`
- `is_key` INTEGER NOT NULL DEFAULT 0 (Faz-1'de işlevsiz, yalnızca saklanır)
- `visibility` TEXT NOT NULL DEFAULT `'show'` (`show` | `hide` | `private`; Faz-1'de işlevsiz)
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

**Item** *(Faz-1'de yalnızca şema)*
- `id` TEXT PRIMARY KEY, `card_id` TEXT NOT NULL (FK, `ON DELETE CASCADE`), `user_id` TEXT NOT NULL DEFAULT `'sqlite'`, `created_at` TEXT NOT NULL, `updated_at` TEXT NOT NULL

**ItemValue** *(Faz-1'de yalnızca şema)*
- `id` TEXT PRIMARY KEY, `item_id` TEXT NOT NULL (FK, `ON DELETE CASCADE`), `property_id` TEXT NOT NULL (FK), `value` TEXT
- `value` kolonu tüm tipler için TEXT olarak kalır; Hex değerleri `0x` ön ekiyle ve küçük harfe normalize edilmiş olarak saklanır (ör. `0x1a2f`).

**Silme davranışları:**
- **Disable** (Del): `UPDATE cards SET is_disabled = 1 WHERE id = ?`. Cascade yoktur.
- **Purge:** Card kaydının gerçek `DELETE`'idir. Property, EnumValue, Item, ItemValue tabloları `ON DELETE CASCADE` ile Card'a bağlıdır; Card purge edildiğinde bağlı tüm alt kayıtlar otomatik silinir.

## 7. Arayüz / UX Notları
- Header ve tab menüsü tasarımı, Ekler bölümündeki ekran görüntülerinin genel yerleşim mantığına (logo + isim + versiyon solda, altında tab menüsü) görsel referans olarak uygulanmalıdır.
- Enum property'lerin type-ahead arama kutusu, Ekran Görüntüsü 4'teki gibi olmalı.
- Card Name alanı: kullanıcı yazarken canlı doğrulama (kırmızı/yeşil, boşluksuz + case-insensitive karşılaştırma).
- Title alanı: New Card modalında Card Name'den otomatik doldurma davranışı.
- Tema desteği (dark/light), header'daki bağımsız ikon buton ile toggle edilebilir; seçim kalıcı olmalı (localStorage).
- **Marka Kimliği:**
  - **Logo:** "Stacked cards" konsepti — üç adet üst üste, diyagonal kaydırılmış yuvarlak köşeli dikdörtgen; sırasıyla arkadan öne **indigo (#4F46E5)**, **teal (#0D9488)**, **coral (#F97316)** renkleriyle doldurulur. Gömülü SVG olarak header'da wordmark'ın solunda kullanılır (referans: `CMAP-Logo.svg`).
  - **Slogan:** "Your records, unlimited relationships." — wordmark'ın altında/yanında, ikincil (muted) metin rengiyle gösterilir.

## 8. Güvenlik ve Gizlilik Gereksinimleri
- Faz-1'de kimlik doğrulama veya sunucu tarafı erişim kontrolü **yoktur**; veri erişimi, kullanıcının kendi diskindeki `.sqlite` dosyasıyla sınırlıdır. Dosya sistemi izinleri (File System Access API izin diyalogları) tamamen tarayıcı tarafından yönetilir; uygulama bu izinleri varsayamaz, her zaman izin reddi senaryosunu ele almalıdır (bkz. Bölüm 9).
- **Gelecek faz notu:** Supabase'e geçişte RLS politikaları `user_id = auth.uid()` temelinde uygulanacak ve CoWork paylaşım mekanizması (`card_access` tablosu) eklenecektir; bu, Bölüm 6'daki `user_id` placeholder kolonu ve adapter soyutlaması sayesinde UI katmanına dokunmadan yapılabilmelidir. Faz-1'de bu uygulanmaz.
- **Girdi temizleme / injection koruması:** Tüm metin girdileri (Card Name, Title, Description, Template, Property Label, Enum değerleri, filtre/arama alanı ve ileride Item değerleri) zararlı kod enjeksiyonuna karşı temizlenmelidir:
  - Tüm SQL sorguları **parametreli** (prepared statement / bind parametresi) olmalı; kullanıcı girdisinin SQL string'ine birleştirilmesi yasaktır (filtre/arama alanındaki `LIKE` sorguları dahil).
  - Kullanıcıdan gelen veya veritabanından okunan hiçbir metin `innerHTML` ile ekrana basılmamalı; `textContent` kullanılmalı veya çıktı HTML-escape edilmelidir.
  - Kaydetmeden önce metinlerden **kontrol karakterleri** temizlenmelidir (Text area alanlarında satır sonu/tab korunur).
  - Kullanıcı girdisi hiçbir koşulda SVG/HTML/script olarak yorumlanmamalıdır; ikonlar yalnızca gömülü sabit set içinden, anahtar adıyla çözülür.
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

### 8.1 Item Fazında Uygulanacak Doğrulama Şartnamesi (Faz-1'de UYGULANMAZ, yalnızca şartname)
Item ekleme/düzenleme arayüzü Faz-1 dışındadır. Item fazında geliştirilecek formlar, her Property tipi için değeri kaydetmeden önce şu kurallarla doğrulamalı ve hata durumunda İngilizce, alan bazlı mesaj göstermelidir:

| Tip | Geçerlilik kuralı |
|---|---|
| Text | Trim uygulanır, kontrol karakterleri temizlenir. |
| Number | Config'teki türe uyar: `integer` için yalnızca tam sayı; `decimal` için nokta ayraçlı ondalık sayı. Sayısal olmayan girdi reddedilir. |
| Hex | Giriş alanı `0x` ön ekiyle dolu gelir. Ön ek zorunludur; ondan sonra yalnızca `0-9`, `a-f`, `A-F` karakterleri ve en az 1 hane olmalıdır. Hane sayısı (ön ek hariç) Config'teki min/max sınırları içinde olmalıdır. Kayıtta ön ek `0x` ve tüm harfler **küçük harfe** normalize edilir. |
| Date | Geçerli bir takvim tarihi olmalıdır; Config'te `include_time` açıksa geçerli bir saat de zorunludur. |
| Enum | Boş olamaz, max 255 karakter, trim; Property'ye özel type-ahead kuralları geçerlidir. |
| Icon | Değer, gömülü ikon setindeki geçerli bir anahtar olmalıdır. |

Doğrulama hem arayüzde (anlık geri bildirim) hem adapter katmanında (kayıt öncesi, son savunma) uygulanmalıdır.

## 9. Hata Yönetimi ve Kenar Durumlar
- **Dosya izni reddi (`NotAllowedError`)** veya **handle kaybı** (ör. dosya taşınmış/silinmiş): kullanıcıya açık bir hata mesajı gösterilmeli ve elle "Save"/export seçeneği her zaman UI'da bulunmalı.
- File System Access API desteklenmeyen tarayıcılarda otomatik kaydetme yapılamayacağı kullanıcıya açıkça belirtilmeli; Blob/download + file-input fallback akışı devreye girmeli.
- Bir Card silinirken/purge edilirken işlem yarıda kesilirse veri tutarsızlığı oluşturmayacak şekilde (tek SQL transaction'ı içinde) uygulanmalı.
- Purge onayı tek adımlı bir modal ile alınır; ek bir ikincil onay adımına gerek yoktur.
- Properties listesi boş bir Card kaydedilebilir; ayrıca bir uyarı mesajına gerek yoktur.
- Enum type-ahead önerileri boşken ayrıca bir empty-state mesajına gerek yoktur.

## 10. Kabul Kriterleri (Definition of Done)
- [ ] Uygulama açıldığında veritabanı seçilmemişken "Open Database" / "New Database" seçenekleri doğru gösteriliyor; bu durumda Map/Items/Cards tabları görünmüyor.
- [ ] "New Database" ile `CMAP.sqlite` varsayılan adıyla yeni dosya oluşturulabiliyor; "Open Database" ile mevcut dosya açılabiliyor.
- [ ] Veritabanı açıldıktan sonra header'da dosya adı, "Save" ve "Switch Database" butonları doğru gösteriliyor.
- [ ] Her CRUD işleminden sonra dosya otomatik olarak diske yazılıyor (debounce ile); manuel "Save" her zaman çalışıyor.
- [ ] File System Access API desteklenmeyen tarayıcıda Blob/download + file-input fallback akışı çalışıyor.
- [ ] Header logo, wordmark, slogan ve versiyon rozeti (`1.0.RC1`) doğru görünüyor; tema toggle header'da bağımsız çalışıyor.
- [ ] Tüm arayüz metinleri ve sistem mesajları İngilizce.
- [ ] Cards tabında: filtre, "Hide disabled cards" anahtarı, "New Card" butonu ve 7 kolonlu tablo doğru çalışıyor.
- [ ] Disable edilmiş Card'lar tabloda soluk görünüyor; filtre anahtarı açıkken tamamen gizleniyor.
- [ ] New/Edit Card modalı tüm alanları doğru kaydediyor; Card Name canlı benzersizlik doğrulaması (kırmızı/yeşil) + DB unique constraint (`name_normalized`) birlikte çalışıyor.
- [ ] New Card'da Title alanı boşken Card Name'den otomatik dolduruluyor.
- [ ] Property tipleri (Icon — gömülü SVG set, Text, Number, **Hex**, Enum, Date) ve ayarları (Label, Key, Visibility) tanımlanabiliyor; Label zorunlu alan kontrolü çalışıyor.
- [ ] Number/Hex/Date tipleri seçildiğinde tipe özel ayarlar (Integer/Decimal, Min/Max length, Include time) Card tanımı ekranında görünüyor ve `config` alanında saklanıyor; Hex Min/Max doğrulaması (pozitif tam sayı, Min ≤ Max) çalışıyor.
- [ ] Enum type-ahead öneri sistemi Property'ye özel ve case-insensitive tekilleştirme ile çalışıyor.
- [ ] Del → onay modalı → disable (cascade yok); Purge → onay modalı → gerçek DELETE (cascade ile Property/EnumValue/Item/ItemValue silinir); Recover çalışıyor.
- [ ] Tüm CRUD işlemleri UI'dan doğrudan değil, tek bir adapter bloğu üzerinden yapılıyor (kod incelemesiyle doğrulanabilir).
- [ ] Şema (`CREATE TABLE` ifadeleri) tek bir yorumla ayrılmış blokta toplanmış; `AUTOINCREMENT`/`PRAGMA` kullanılmamış; tüm ID'ler `TEXT` UUID, tüm timestamp'ler ISO 8601 string.
- [ ] Tüm metin girdileri parametreli sorgularla yazılıyor, ekrana `textContent`/escape ile basılıyor, kontrol karakterleri temizleniyor; açılan `.sqlite` dosyasındaki içerik de güvenli render ediliyor.
- [ ] Bölüm 8.1'deki Item doğrulama şartnamesi kod içinde (adapter/doğrulama bölümünde) yorum veya yapılandırma olarak not edilmiş; Faz-1'de Item formu geliştirilmemiş.
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
