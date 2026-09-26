# Uzay Ada — Space Sitting

Uzayda, bir galaksinin içinde, kırık bir uydunun tablasında bir masada oturduğun
tek dosyalık Three.js sahnesi. **Çok oyunculu:** oda kur, arkadaşlarınla aynı odaya
gir, boş sandalyeye otur, birbirinizi gör.

Klasörü çift tıkla, `index.html` tarayıcıda açılır. Build yok, sunucu yok, paket
yöneticisi yok — tek istisna: three.js CDN'den.

## Çalıştırma

`index.html` dosyasına çift tıklayın. İnternet bağlantısı gerekir.

- **Oyuncular:** oda adı ve şifre belirle, `ODA OLUŞTUR`. Arkadaşın aynı ad ve
  şifreyle `ODAYA GİR` der. Boş sandalyeye oturursunuz. Yanlış şifreyle girersen
  kimse bağlanamaz ve "şifre yanlış" uyarısı verirsin.
- **TEK KİŞİ:** ağ olmadan ilk sandalyeye doğrudan oturur (ağ kitaplığı
  yüklenemezse de bu yol çalışır).
- **Oturduktan sonra:** tıkla → fare kilitlenir → hareket ettir (sağa/sola ±80°,
  dikey −58°…+72°). `R` başı sıfırlar, `L` odadan çıkar, `ESC` kilidi bırakır.
- **Görünenler:** sandalyenin üstünde kendi emojin, diğerlerinin emojileri ve
  baktıkları yön (ışın çizgisi). Sağ üstte oda listesi, sağ altta oda adı ve
  çıkış düğmesi.
- **Sol üstte:** FPS, frame süresi, draw call, üçgen sayısı.

Aynı anda 4 kişi (4 sandalye). Ağ katmanı sunucusuzdur: tarayıcılar halka açık
röle üzerinden eşlerini bulur, veriyi doğrudan birbirine (WebRTC) gönderir.
Hesap, anahtar veya sunucu gerekmez.

## Yapı

Tek dosya, dört bölümden oluşur:

1. **Ayar** — `CFG` (kamera, ışık, ölçek, doku boyutları, saçılım tohumu, `multi`
   ağ/avatar ayarları) ve `SURF` (prosedürel yüzey renkleri).
2. **Üretim** — `woodMaps / metalMaps / compositeMaps / regolithMaps / cloudMap /
   planetMap / galaxyMap / nebulaMap / flareMap / envEquirect`. Bütün dokular
   kodla üretilir; dış görsel dosyası yoktur. Yükseklik alanından normal haritası
   `normalTex()` ile türetilir. `mulberry32(seed)` + `fbm3()` ile deterministik.
3. **Sahne** — `buildMoon / buildDeck / buildTable / buildChair / buildPlanets /
   buildBelt / buildSky / buildAvatars / buildLights`.
4. **Döngü** — `applyHead / look / tweenCameraTo / updateAvatars / resize / loop`
   + post-process zinciri (bloom → ton eşleme → hafif kenar distorsiyonu →
   vignette → dither).

Durum makinesi: `lobby` (galaksiye bakan kamera, masa gizli) → `seated`
(2.6 sn'lik kamera geçişiyle sandalyeye) → `lobby`.

Tüm parçalar `window.APP.PARTS` altında adlandırılmış; konsoldan `APP.appState`,
`APP.Net` gibi erişip canlı değiştirebilirsiniz.

## Teknik notlar

- three.js **r152.2**, UMD build (`unpkg.com/three@0.152.2/build/three.min.js`).
  UMD seçildi çünkü ES module `import` `file://` üzerinde CORS'a takılıyor.
- Gerçek zamanlı katman **Trystero 0.20** (`esm.sh`, dinamik `import()`), Nostr
  stratejisi. `file://` üzerinde de çalışır; yüklenemezse oyun tek kişilik devam
  eder. Oda anahtarı `roomHash(oda adı + şifre)`.
- `fov: 100` dikey → 16:9'da yatay ~118°, insan görüşüne yakın. three'te
  `fov > 180` verilirse projeksiyon matrisinin işaretleri ters dönüp görüntü 180°
  döner; `wideProjection()` bunu düzeltiyor. Ayrıca `fov > 180` için three'in
  frustum düzlem çıkarımı bozulduğu (`_frustum`) sahne `frustumCulled = false`.
- `renderer.info.autoReset = false` + kare başına manuel `reset()` — aksi halde
  overlay her `render()` çağrısında sıfırlanıp yalnızca son passı gösteriyor.
- Sandalyeye oturma açısı `a = i · 2π / 4`; kameranın yaw'ı `a + head.yaw`. Sabit
  bir `baseYaw` yoktur — her sandalye kendi yönüne bakar.
- Gölge tek yönlü (ufuktaki tek yıldız), 2048 harita, sadece platform alanı.
- Ortam ışığı (IBL): `envEquirect()` gezegenleri ve yıldızı HDR float equirect
  olarak çizer, `scene.environment` olur; metal yüzeyler bunu yansıtır.

## Adımlar

| Dosya | Aşama |
|---|---|
| `asamalar/01-whitebox.html` | Kompozisyon ve ölçek (gri clay) |
| `asamalar/02-materyal.html` | Malzemeler, atmosfer, ışık, yıldızlar, galaksi |

Sırada: göktaş saçılımı (`PARTS.groundRocks` boş grup olarak duruyor, tohumlu
`mulberry32` hazır), geçen meteorlar, zemin efektleri, gerçek avatar modelleri.

## Bilinen sınırlar

- Işık ve gölge tek yönlü — gökyüzü için ikinci bir yön yok.
- Metaller, ahşap ve regolit fizik tabanlı değil; koda gömülü roughness/metalness
  değerleriyle ayarlanmış.
- Oda şifresi bir isim uzayı anahtarıdır, gerçek kimlik doğrulaması değildir:
  anahtar halka açık röle üzerinden görülebilir. Zayıf/kişisel şifre kullanma.
- Ağ tek bir arayüzün (`Net`) arkasında; sunucuya taşımak sahne kodunu değiştirmez.
