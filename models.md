# models.md — Bu projeyi değiştirecek yapay zekâ için

Bu dosya, `index.html`'i açıp düzenleyecek bir modelin **nereye bakması gerektiğini**,
**neyi bozmaması gerektiğini** ve **nasıl doğrulaması gerektiğini** anlatır.
Kod okunabilir ama bazı kararlar koddan anlaşılmıyor; burada yazılıdır.

Proje: `index.html` — tek dosya, 1717 satır, harici bağımlılık yok.
three.js r152.2, UMD build, unpkg CDN. Build yok, sunucu yok, paket yöneticisi yok.

> **Proje kısıtı:** Tek dosya kalmalı. Modüle bölme, npm'e geçirme, Vite/webpack
> ekleme. Kullanıcı çift tıklayıp açabilmeli.

---

## 1. Hızlı başlangıç

| Yapılacak | Nerede |
|---|---|
| Kompozisyonu değiştir | `CFG` bloğu (satır ~80-145) |
| Malzeme değiştir | `buildMaterials()` + üretici fonksiyonlar |
| Sahne parçası ekle | `build*()` fonksiyonları (~satır 800-1300) |
| Render/ışık zinciri | `buildLights()`, `loop()` (~satır 1300-1700) |
| Bir şeyin nerede olduğunu bul | `window.APP.PARTS` (konsoldan erişilebilir) |

Çalıştırma: `index.html`'e çift tıkla. Kontroller: tıkla → fare kilitlenir → hareket
et. `R` başı sıfırlar, `ESC` kilidi bırakır.

---

## 2. Dosyanın 4 bloğu

1. **Ayar** — `CFG` (tüm sayılar), `SURF` (prosedürel yüzey renkleri), `PARTS` (sahne
   parçası kayıt defteri).
2. **Üretim** — `mulberry32 / hash3 / vnoise3 / fbm3` (gürültü) →
   `dataTex / normalTex / mixRGB` (doku altyapısı) → `woodMaps, metalMaps,
   compositeMaps, regolithMaps, cloudMap, planetMap, galaxyMap, nebulaMap,
   flareMap, envEquirect` (yüzeyler).
3. **Sahne** — `buildMoon, buildDeck, buildTable, buildChair, buildPlanets,
   buildBelt, buildSky, buildLights`.
4. **Döngü** — `applyHead / look / resize / loop` + post-process zinciri.

---

## 3. SERT DEĞİŞMEZLER

Bunlar kodu okumadan anlaşılmıyor, hepsi bir hata sonucu bulundu. İhlal edilirse
sahne **sessizce** bozulur (hata vermez, sadece yanlış görünür).

### 3.1 `fov > 180` three'de iki yeri birden bozar

`PerspectiveCamera.updateProjectionMatrix()` içinde `tan(fov/2)` negatife düşer,
`m[0]` ve `m[5]` negatif olur ve:

- **Görüntü 180° döner** (yanlış yön, ters elipsler).
- **`Frustum.setFromProjectionMatrix` bozulur.** Üst/alt düzlemler `(0,±1,0)`
  hâline gelir; kameranın üstünde veya altında **kendi bounding yarıçapından daha
  uzakta olan her nesne yanlışlıkla cull edilir.** Gezegenler ve sandalyeler
  görünmez olur, draw call sayısı düşer.

İki savunma var ve **ikisi de şart**:

```js
function wideProjection(cam) {         // işaret düzeltmesi
  cam.updateProjectionMatrix();
  var e = cam.projectionMatrix.elements;
  if (e[0] < 0) e[0] = -e[0];
  if (e[5] < 0) e[5] = -e[5];
  cam.projectionMatrixInverse.copy(cam.projectionMatrix).invert();
  return cam;
}
scene.traverse(function (o) { o.frustumCulled = false; });   // cull düzeltmesi
```

`wideProjection()` **her** `updateProjectionMatrix()` çağrısından sonra
yeniden uygulanmalı — `resize()` içinde zaten yapılıyor, yeni bir kamera
eklenirse de yapılmalı. `frustumCulled = false` çağrısı sahne kurulduktan
**sonra** yapılmalı.

Şu an `fov: 100` olduğu için ikisi de atıl kalmış durumda. **FOV'u 180 üstüne
çıkarırsanız bu iki mekanizma olmadan sahne çalışmaz.**

### 3.2 Renk hattı elle kuruldu — `toneMapped` ve `NoToneMapping` birlikte

`renderer.toneMapping = THREE.NoToneMapping`. Ton eşleme ve sRGB dönüşümü
post-process shader'ında elle yapılır (ACES + sRGB transfer).

- `renderer.toneMapping`'i bir şeye çevirme. Çevirirsen gökyüzü çift tone-map
  olur ve `toneMapped: false` taşıyan malzemelerle birleşince tutarsızlaşır.
- `glowMat()` ve gökyüzü `MeshBasicMaterial`/`ShaderMaterial`'ları
  `toneMapped: false` taşır. Yeni temel malzeme eklerken de koru.
- Ana render target `HalfFloatType`: 1'in üstü değerler bloom'a kadar yaşar.
  `UnsignedByteType`'a çevirme.

### 3.3 `DataTexture` 8 bittir — 1'in üstü değer yazma

`dataTex()` içinde `Uint8Array`'e yazılıyor. 1.6 gibi bir değer `1.6*255 = 408`
olur ve **sarmalanır** → 408 mod 256 = 152. Sonuç: galaksi dokusu gökkuşağı
bantlarına döner (bu yaşandı).

- Prosedürel renkleri `[0,1]` aralığında yaz, clamp et.
- HDR ihtiyacı varsa malzemenin `color` çarpanını kullan ya da `FloatType`
  doku üret (`envEquirect()` böyle yapıyor, `DataTexture` + `FloatType`).
- `envEquirect()` dışında HDR doku kullanma.

### 3.4 Doku renk uzayı

| Doku | Seçenek |
|---|---|
| Albedo (ahşap, metal, panel, regolit, gezegen, bulut, galaksi, nebula, flare) | `{ srgb: true }` |
| Roughness, normal, metalness, alpha | varsayılan (`NoColorSpace` = doğrudan veri) |

Roughness/normal haritalarını sRGB yapmak ışığı kırar (çok parlak/karanlık).

### 3.5 `renderer.info` kare başına elle sıfırlanır

`renderer.info.autoReset = false` ve `loop()` içinde `renderer.info.reset()`.
Üç `render()` çağrısı (gökyüzü, sahne, post) yapılıyor; otomatik sıfırlamada
overlay **sadece son çağrının** (1 draw call, 2 üçgen) sayısını gösterirdi.
`info.reset()` kare başına tam **bir** kez, render'lardan önce çağrılmalı.

### 3.6 Koordinat ve yön sözleşmesi

- `dirFrom(az, el)`: **az > 0 = ekranın sağı**, `el` = yükseklik derecesi.
  Formül: `(-cos(el)·sin(az), sin(el), cos(el)·cos(az))`. X işaretinin eksi
  olması kasıtlı — kamera +Z'ye bakarken ekranın sağı −X'tir.
- Kameranın `head.baseYaw = 180` derece. Yani `yaw = 0` iken baktığı yön **+Z**.
  `dirFrom`'un sıfır yönü de +Z. **İkisinden birini değiştirip diğerini
  bırakma**, yoksa tüm kompozisyon 180° döner.
- Fare yönü: `head.yaw -= dx * sensitivity`. `movementX > 0` (fare sağa) bakışı
  **sağa** çevirmeli. `rotation.y = 180 + yaw` yukarıdan bakışta saat yönünün
  tersine döndüğü için işaret negatiftir. Bu işaret bir kez ters döndü, düzeltildi.
- Dikey: `head.pitch -= dy`, negatif pitch = aşağı bakmak.

### 3.7 Dünya yükseklikleri birbirine bağlı

```
zemin (plato)      y = 0
platform üstü      y = CFG.deck.thickness   (0.07)
sandalye tabanı    y = CFG.deck.thickness
göz                y = CFG.deck.thickness + CFG.head.height  (0.07 + 1.12 = 1.19)
```

- `camera.position` oturduğun sandalyeden hesaplanır
  (`PARTS.chairs[CFG.props.chair.seatIndex]`). Kamera o sandalyenin **içinde**
  olmalı, `head.forward` kadar öne eğik.
- `CFG.moon.plateau` (8.5 m) ile göz yüksekliği birbirine **bağlıdır**: 8.5 m
  yarıçaplı bir düzlükte ufuk tam olarak 8.4 m'de düşer, yani plato kenarı
  ufukla çakışır ve zemin bitince uzay başlar. Biri değişirse kompozisyonun
  yarısı değişir.

### 3.8 Uydunun plato matematiği

`buildMoon()` içinde küre merkezi fiilen `y = -H`'ye kaydırılıyor
(`H = sqrt(R² − P²)`), tepe kapağı `y = 0`'a düzleştiriliyor:

```js
var yThresh = Math.sqrt(Math.max(0.25, r * r - Pe * Pe));   // Pe = θ'ye göre plato yarıçapı
var t = smoothstep(yThresh - C.lip, yThresh + C.lip, v.y);
var wy = (v.y - H) * (1 - t);
```

- `Pe(θ) = P · (1 + Σ aᵢ·sin(kᵢθ + φᵢ))` — harmoniklerle dalgalanan kenar,
  `mulberry32(seed)` ile üretilir. Silüeti kıran şey budur.
- `micro` rölyef `microStart` (2.6 m) radyal mesafeden önce **yok**: mobilya
  altındaki zemin düz kalmalı, yoksa sandalyeler havada durur.
- `+ H` yazmayı unutma. Bir ara `pos.y = y + H` yazılıydı; plato 28.77 m
  yukarıda kalmış, kamera kürenin içine düşmüş ve her şey gölgede kaybolmuştu.

### 3.9 Determinizm — `Math.random` yasak

Her şey `mulberry32(seed)` veya `fbm3(..., seed)` üzerinden. Aynı tohum → aynı
uydu, aynı göktaşlar, aynı yıldızlar. `asamalar/` anlık görüntülerinin anlamlı
olması buna bağlı.

Yeni saçılım/kararsızlık eklerken `Math.random()` kullanma, `CFG.scatter.seed`
türet.

### 3.10 Render sırası

```
1. rt'ye: skyScene  (tüm malzemeler depthTest:false — derinlik yazmaz)
2. clearDepth()
3. rt'ye: scene     (normal depth test)
4. brightMat  rt → bloomA    (1/2 çözünürlük, eşik+knee)
5. blurMat    bloomA → bloomB (yatay)
6. blurMat    bloomB → bloomA (dikey)
7. lensMaterial (rt + bloomA) → ekran
```

- `skyCamera`, `camera`'nın konumunu ve quaternion'ını kopyalar; `near/far`
  farklıdır (10 / 400000) çünkü yıldız küreleri 90000 birimde.
- 3 render target var: `rt` (tam çözünürlük, HalfFloat, WebGL2'de MSAA 4×),
  `bloomA`/`bloomB` (yarım çözünürlük).
- `blit(material, target)` tam ekran quad çizer; `rtScene`/`rtCamera` paylaşılan.
- `resize()` bloom hedeflerini de yeniden boyutlandırmak zorunda.

### 3.11 Işık bütçesi

Şu an 7 ışık: 3 directional (yıldız, gezegen parıltısı, gaz devi dolgusu),
1 hemisphere, 3 point (LED'ler).

- **Her yeni ışık tüm malzemelerin shader'ını yeniden derler.** Sayılı tut.
- Sadece yıldız gölge atar: 2048 harita, extent ±9 (yalnızca platform + mobilya).
  Platform 2.05 m yarıçap olduğu için ±9 yeterli; büyütürsen çözünürlük düşer.
- Point ışıklar gölge atmaz, geometrinin içinden geçer. Masa altı LED'i
  y=0.44'te duruyor ve masanın içinden de aydınlatıyor — kasıtlı.

---

## 4. YAPMA listesi

Bunlar fiilen yapıldı, geri alındı ve tekrar yapılırsa aynı sonucu verir:

| Yapma | Sonuç | Doğrusu |
|---|---|---|
| `fov: 200` + `wideProjection`/`frustumCulled` olmadan | Görüntü ters, gezegenler/sandalyeler kaybolur | §3.1 |
| `applyHead()`'i atlayıp `camera.rotation` doğrudan yazmak | Kamera masadan uzağa bakar | `baseYaw`'ı koru |
| `head.yaw += dx` | Fare sağa gidince sola bakar | `head.yaw -= dx` |
| `pos.setXYZ(i, x, y + H, z)` | Plato 28.77 m yukarıda, kamera küre içinde | §3.8 |
| 8-bit dokuya >1 yazmak | Gökkuşağı renk bozulması | §3.3 |
| Roughness/normal haritayı sRGB yapmak | Işık patlar | §3.4 |
| `renderer.toneMapping = ACESFilmic` | Çift tone-map | §3.2 |
| `updateProjectionMatrix()` çağırıp `wideProjection`'ı atlamak | FOV 180+ ise görüntü ters | §3.1 |
| Masa altına ışığı `topY - 0.09`'a koymak | Zeminde düzenli "pırlama" halkası | `y = 0.44` |
| Kabuk sweep'inde twist eklemeden bırakmak | Sırtlık düz plaka gibi görünür | `twistFn` ≈ 0.44 rad |
| `Math.random()` | `asamalar/` anlık görüntüleri tutmaz | §3.9 |
| Galaksiye `lookAt` uygulayıp `rotation` ile tilt vermek | `lookAt` quaternion'ı ezdiği için tilt kaybolur | tilt'i iç grupta ver |

---

## 5. Uzantı tarifleri

### 5.1 Zemin göktaşları (sıradaki adım için hazır)

Zaten yer duruyor: `PARTS.groundRocks` boş bir grup, tohumlu RNG hazır,
`T.rock` malzemesi hazır.

```js
function buildGroundRocks() {
  var group = new THREE.Group();
  group.name = 'ground-rocks';
  var r = mulberry32(CFG.scatter.seed + 501);
  var geo = new THREE.IcosahedronGeometry(1, 0);
  var mesh = new THREE.InstancedMesh(geo, PARTS.tex.rock, 90);
  var m = new THREE.Matrix4(), q = new THREE.Quaternion();
  var e = new THREE.Euler(), p = new THREE.Vector3(), s = new THREE.Vector3();
  for (var i = 0; i < 90; i++) {
    var a = r() * Math.PI * 2;
    var rad = 2.3 + r() * 5.6;                       // platform dışı, kenara yakın
    p.set(Math.sin(a) * rad, 0, Math.cos(a) * rad);
    e.set(r() * 6.28, r() * 6.28, r() * 6.28);
    q.setFromEuler(e);
    var sc = 0.06 + Math.pow(r(), 2) * 0.42;         // 6–48 cm
    s.set(sc, sc * (0.6 + 0.5 * r()), sc);
    m.compose(p, q, s);
    mesh.setMatrixAt(i, m);
  }
  mesh.instanceMatrix.needsUpdate = true;
  mesh.castShadow = true; mesh.receiveShadow = true;
  group.add(mesh);
  PARTS.groundRocks = group;
  return group;
}
```

Yerleştirme kuralları: radyal mesafe 2.3 m'den küçük olmasın (masa + platform
alanı), zemine gömülmesin (`y = 0`, plakanın üstü değil), tek `InstancedMesh`
kullan (1 draw call), `frustumCulled = false` unutma.

### 5.2 Yeni malzeme

1. Yüzey üreticisi yaz (`dataTex` + gerekiyorsa `normalTex`).
2. `buildMaterials()` içinde `T.yeni = pbr({...})` tanımla.
3. Kullan. `pbr()` roughness/normal/roughnessMap/metalnessMap/emissive/normalScale
   destekler; `repeatTex(tex, x, y)` ile tekrar ayarla.

### 5.3 Yeni gök cismi

1. `CFG.planets` içine `{ azimuth, elevation, distance, radius, seed, wobble, atmo?, atmoColor?, atmoPower? }`.
2. `buildPlanets()` içinde `build*()` + `placeAt()` + `group.rotation.y`.
3. Malzeme: `planetMap('home'|'arid'|'giant', seed)` veya `T.planetRock`.
4. Atmosfer isteğe bağlı: `TX.atmosphere(renk, güç)` → `SphereGeometry(radius * atmo)`.
5. Işıkta kullanılacaksa `buildLights()` içinden yönü `dirFrom` ile al.

Gökyüzüne yeni cisim: `buildSky()` içine `placeAt(..., 80000+)` ile koy.
Gökyüzündeki her şeyin `depthTest:false` olmalı (gökyüzü sahnesi önce çizilir).

### 5.4 Geçen meteor / parçacık

Gökyüzü sahnesine, `PARTS.anim` dizisine `{ obj, spin }` gibi bir giriş ekleyip
`loop()` içinde ilerlet; ya da `loop()` içine kendi güncellemeni yaz. Sahne
`skyScene` ise derinlik yazmaz, önce çizilir.

---

## 6. Ayar haritası — neyi değiştirmek istiyorsan

| Anahtar | Etki |
|---|---|
| `CFG.renderer.fov` | 100 = dikey, 16:9'da yatay ~118°. 180+ ise §3.1 |
| `CFG.renderer.exposure` | 0.72. Genel parlaklık; ACES ton eşleme öncesi |
| `CFG.renderer.pixelRatio` | 1.25 tavan. Retina'da 2 → 1.25'e düşer |
| `CFG.lens.distortion` | 0.07. Kenar kıvrımı; arttıkça kenar sıkışır |
| `CFG.lens.bloom` / `bloomThreshold` | LED ve yıldız parlaması |
| `CFG.lens.vignette*` | Köşe karartma |
| `CFG.head.yaw` | ±80 kelepçe (sabit kural) |
| `CFG.head.pitchMin/Max` | −58° … +72° |
| `CFG.head.height` | Göz yüksekliği → §3.7 |
| `CFG.moon.plateau` | Plato yarıçapı → §3.7 ile eşleşmeli |
| `CFG.moon.rimWobble` | Plato kenarının dalgalılığı (silüet) |
| `CFG.moon.uvScale` | Regolit dokusunun tekrarlanma sayısı (5.5) |
| `CFG.deck.*` | Platform yarıçapı/kalınlığı, sütun sayısı |
| `CFG.props.table.radius` / `chair.radius` | Masa yarıçapı / sandalye halkası |
| `CFG.props.chair.seatIndex` | Hangi sandalyede oturuyoruz (0–3) |
| `CFG.sun / fill / rim / ambient` | Işık renkleri ve şiddetleri |
| `CFG.led.*` | LED rengi ve nokta ışığı şiddetleri |
| `CFG.env.*` | Ortam haritasındaki gezegen/yıldız parıltısı |
| `CFG.scatter.seed` | Tüm prosedürel üretimin tohumu |
| `CFG.scatter.belt.*` | Göktaş kuşağı |
| `CFG.planets.*` | Gök cisimleri (az/el = ekran konumu) |
| `CFG.sky.galaxy` | Galaksinin konumu, boyutu, eğimi |
| `CFG.sky.nebula[]` | Bulutumalar (dizi) |
| `CFG.sky.flares` | Parlak yıldız sayısı |
| `SURF.*` | Prosedürel yüzey renkleri (albedo aralıkları) |

Gök cisimlerinin **ekranda nereye düştüğünü** bilmek için: `fov` ve `startPitch`
biliniyorsa `az`/`el` değişikliği ndc'yi doğrudan değiştirir; pratikte
`window.APP` üzerinden ekran görüntüsü alıp bakmak daha hızlıdır.

---

## 7. Doğrulama

1. `index.html`'i aç, konsolu kontrol et (hata olmamalı — CDN yüklenemezse
   `#fatal` ekranı çıkar ve açıklama verir).
2. Sol üstteki overlay: fps, frame ms, draw calls, triangles.
   **Draw call veya üçgen sayısı düştüyse bir şey cull ediliyordur** (§3.1).
3. Bakışı şu açılarda kontrol et: düz (`resetHead`), sol-yukarı, sağ-aşağı,
   tam yukarı. Gök cisimleri bu dört görünümün en az birinde kadraj dışında
   kalmamalı.
4. Yakın plan: masa/sandalye dokuları, platform kenarı, zemin kraterleri.

Headless doğrulama: geliştirme sırasında Chrome DevTools protokolüyle
(`--headless=new --remote-debugging-port` → `Page.captureScreenshot`,
`Runtime.evaluate`) ekran görüntüsü alındı; bu araç repo içinde **değil**.
Headless SwiftShader ile ölçülen fps **anlamsızdır** (yazılım rasterizasyonu) —
donanım hızlandırıcısında 188 draw call / 233k üçgen hafif bir yüktür.

Değişiklikten sonra en az bir kez `asamalar/` içine yeni bir anlık görüntü
kaydetmeyi ve `git commit` atmayı unutma.

---

## 8. Performans bütçesi

Şu an: **188 draw call**, **233k üçgen**, 33 doku, 23 shader programı.

- Ana maliyet draw call'lar (gölge geçişi çağrıları da sayılıyor — gölge atan
  her mesh ek bir çağrı).
- Dokular `mulberry32` ile üretildiği için **önbelleklenebilir** ama şu an her
  yüklemede yeniden üretiliyor (~200–400 ms yükleme süresi). Bu kabul edilebilir.
- Ayarı bozma: `dpr ≤ 1.25`, MSAA yalnızca ana RT'de, bloom yarım çözünürlükte,
  gökyüzü tek `Points`/`InstancedMesh` çağrılarıyla, parıltılar tek `Points`.

---

## 9. Durum ve yol haritası

**Yapıldı:** whitebox kompozisyonu ve ölçeği → prosedürel malzemeler,
atmosfer, ışık, yıldızlar, galaksi, bloom, sandalye tasarımı.
`asamalar/01-whitebox.html`, `asamalar/02-materyal.html`.

**Sırada:** zemin göktaşları (§5.1 için altyapı hazır), geçen meteorlar,
özel zemin efektleri.

**Bilinen sınırlar (bilerek):**
- Işık ve gölge tek yönlü — gökyüzü için ikinci yön yok.
- Metaller/ahşap/regolit fizik tabanlı tanımlı değil; koda gömülü
  roughness/metalness değerleriyle ayarlanmış.
- Uydunun UV'si küreye düz projekte edilmiş, `uvScale` bir uzlaşma değeri —
  kutuplarda doku sıkışır.
- 4 sandalye aynı geometri nesnesini paylaşıyor; sandalye başına varyasyon
  istenirse geometri klonlanmalı.
- Uzay boşluğunda atmosfer yok (yalnız gezegenlerde Fresnel kabuk).
