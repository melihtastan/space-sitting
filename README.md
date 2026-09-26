# Uzay Ada — Space Sitting

Uzayda, bir galaksinin içinde, kırık bir uydunun tablasında bir masada oturduğun
tek dosyalık Three.js sahnesi. Klasörü çift tıkla, `index.html` tarayıcıda açılır.
Build yok, sunucu yok, paket yöneticisi yok — tek istisna: three.js CDN'den.

## Çalıştırma

`index.html` dosyasına çift tıklayın. İnternet bağlantısı gerekir (three.js CDN'den
gelir). Başka hiçbir şey gerekmez.

- **Kontroller:** Tıklayın → fare kilitlenir → fare hareketi kafayı çevirir.
  Sağa/sola ±80°, dikey −58°…+72°. `R` başı sıfırlar, `ESC` kilidi bırakır.
  Fare kilidi desteklenmeyen ortamlarda sürükle-bak moduna düşer.
- **Sol üstte:** FPS, frame süresi, draw call, üçgen sayısı.

## Yapı

Tek dosya, üç bölümden oluşur:

1. **Ayar** — `CFG` (kamera, ışık, ölçek, doku boyutları, saçılım tohumu) ve
   `SURF` (prosedürel yüzey renkleri). Kompozisyonu değiştirmek için tek durak.
2. **Üretim** — `woodMaps / metalMaps / compositeMaps / regolithMaps / cloudMap /
   planetMap / galaxyMap / nebulaMap / flareMap / envEquirect`. Bütün dokular
   kodla üretilir; dış görsel dosyası yoktur. Yükseklik alanından normal haritası
   `normalTex()` ile türetilir. `mulberry32(seed)` + `fbm3()` sayesinde her şey
   deterministik.
3. **Sahne** — `buildMoon / buildDeck / buildTable / buildChair / buildPlanets /
   buildBelt / buildSky / buildLights` ve post-process zinciri
   (bloom → ton eşleme → hafif kenar distorsiyonu → vignette → dither).

Tüm parçalar `window.APP.PARTS` altında adlandırılmış; konsoldan
`APP.PARTS.groundRocks` gibi erişip canlı değiştirebilirsiniz.

## Teknik notlar

- three.js **r152.2**, UMD build (`unpkg.com/three@0.152.2/build/three.min.js`).
  UMD seçildi çünkü ES module `import` `file://` üzerinde CORS'a takılıyor.
- `fov: 100` dikey → 16:9'da yatay ~118°, insan görüşüne yakın. Üçte `fov > 180`
  verilirse three projeksiyon matrisinin işaretlerini ters çevirip görüntüyü 180°
  döndürüyor; `wideProjection()` bunu düzeltiyor. Ayrıca `fov > 180` için three'in
  frustum düzlem çıkarımı bozulduğu (`_frustum`) sahne `frustumCulled = false`.
- `renderer.info.autoReset = false` + kare başına manuel `reset()` — aksi halde
  overlay her `render()` çağrısında sıfırlanıp yalnızca son passı gösteriyor.
- Gölge tek yönlü (ufuktaki tek yıldız), 2048 harita, sadece platform alanı.
- Ortam ışığı (IBL): `envEquirect()` gezegenleri ve yıldızı HDR float equirect
  olarak çizer, `scene.environment` olur; metal yüzeyler bunu yansıtır.

## Adımlar

| Dosya | Aşama |
|---|---|
| `asamalar/01-whitebox.html` | Kompozisyon ve ölçek (gri clay) |
| `asamalar/02-materyal.html` | Malzemeler, atmosfer, ışık, yıldızlar, galaksi |

Sırada: göktaş saçılımı (`PARTS.groundRocks` boş grup olarak duruyor, tohumlu
`mulberry32` hazır), geçen meteorlar ve zemin efektleri.

## Bilinen sınırlar

- Işık ve gölge tek yönlü — gökyüzü için ikinci bir yön yok.
- Metaller, ahşap ve regolit fizik tabanlı tanımlı değil; koda gömülü
  roughness/metalness değerleriyle ayarlanmıştır.
