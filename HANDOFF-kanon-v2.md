# HANDOFF — KÁNON // ARCHÍVUM v2 → következő session (Claude Code, desktop)

Ez a dokumentum önhordó: a korábbi chat-előzmény nélkül is elégnek kell lennie a folytatáshoz.
Nyelv: magyar, angol technikai terminusok inline. Válaszstílus a felhasználónak: tömör, záró udvariaskodás nélkül.

---

## 1. Mi ez a projekt

**KÁNON // ARCHÍVUM — A Holtidő Könyvtára**: vallástudományi kutatásból (kánonképződés és
szövegkirekesztés 11 hagyományban) épített, **egyfájlos, mobil-first 3D böngészős játék**.
Episztemológiai játék: a játékos 11 történelmi törésponton dönt arról, mi kerül be egy közösség
örökségébe — és a végén a „Tükör" nem a történelmet mutatja meg, hanem az ő bizonyított
szelekciós profilját. Kulcstétel: **a figyelem is kánonképzés.**

- **Fájl:** `kanon-archivum.html` (~130 KB, minden inline; egyetlen külső függés: Three.js r128
  a cdnjs-ről → első betöltéshez internet kell)
- **Élő URL:** https://egedi.art/kanon/ (a v1 fut kint; a v2 ugyanoda kerül, felülírja)
- **Státusz:** v2 kész, Node-stubos teljes végigjátszás-teszt PASS. Valódi eszközön vizuális QA
  még nem történt — ez a következő session első dolga deploy után.

A v2 egy külső design-kritika alapján teljes újraírás. A kritika 4 piros prioritása mind bekerült:
(1) Tükör = bizonyított döntési profil archetípusok helyett, (2) archívumkulcs-mechanika
(az információforrás-választás maga is játékmechanika), (3) minden döntésnek explicit ára van,
(4) XII. kapu: a játék bevallja, hogy maga is kánon.

---

## 2. Hol van a fájl

A user gépén van letöltve (valószínűleg `~/Downloads/kanon-archivum.html` vagy a Drive
`go/kanon` mappában). **Első lépés: kérdezd meg a pontos elérési utat**, vagy `find`-olj rá.
Ha van projektmappa, oda másold be munkapéldányként.

---

## 3. Deploy-runbook (SSH a Hostinger VPS-re)

Bevált minta a ClariPrice-ból: statikus fájlok OpenClaw-konténer webrootjából,
Caddy reverse proxy + Let's Encrypt. A clariprice.com webrootja a hoston:
`/docker/openclaw-xigd/data` (konténerben `/data`). **Az egedi.art saját webrootja ismeretlen
— fel kell deríteni:**

```bash
# 1) webroot-felderítés a VPS-en
grep -RIl "egedi.art" /docker/*/Caddyfile /docker/*/*.yml /etc/caddy 2>/dev/null

# 2) deploy (WEBROOT = a talált gyökér, pl. /docker/openclaw-XXXX/data)
mkdir -p WEBROOT/kanon
cp kanon-archivum.html WEBROOT/kanon/index.html

# 3) ellenőrzés
curl -sI https://egedi.art/kanon/ | head -5
curl -s https://egedi.art/kanon/ | grep -c "KÁNON // ARCHÍVUM v2"   # 1-et kell adnia
```

Caddy `file_server` az új almappát reload nélkül szolgálja. Böngészőben Ctrl+Shift+R
(cache). A user mobilról is deployol néha (Drive `go/kanon` → SFTP), ne törd el ezt az utat.

---

## 4. A fájl belső architektúrája (szerkesztéshez)

Egyetlen `<script id="game">` blokk, sorrendben:

1. **Adat:** `PROLOG`, `KAPUK[11]`, `KAPU12[16]`, `SZEMPONTOK/HIPOTEZISEK/BUKTATOK/MECHANIZMUSOK`,
   `CERTN` (bizonyosság-címkék), `ROLEN` (tanú-szerepnevek)
2. **Állapot:** `S` + DOM-refek + `bootFail()` hibavédelem (THREE-hiány, WebGL-hiba, onerror)
3. **Festőműhely:** `cnv/tx/skyCanvas/makeSky/silhCanvas/addSilh/groundCanvas/matCanvas/M/Mtex/
   addShafts/glow` + filmszemcse-IIFE (`#grain`)
4. **3D mag:** renderer/scene/camera, `disposeGroup/freshWorld`, részecskék
   (`dust/fire/smoke/pages`), `lightRig/groundDisc/shardMesh/pillar`
5. **`BUILD{}`** — 11 állomásépítő az `arch` kulcs szerint:
   `udvar bazilika sivatag lotusz pagoda ziggurat tuzoltar maglya ivek rom oszlopcsarnok`
6. **`buildStation/buildHub/buildMirrorScene`** + pointer-input (drag = nézelődés, tap = raycast)
7. **UI:** `fadeTo/toast/card/chip/stChips`, címlap, prológus
8. **Kapufolyam:** `enterStation → showBeat ×2 → [showInstinct] → showWitnesses/showWitness →
   showDecision → pickOption → showPrice → showOutcome → showPerem → leaveStation`;
   `collectShard` (3 szilánk/kapu)
9. **Kódex** (5 fül), **XII. kapu** (`obeliskTap/gate12Seq`), **Tükör** (`AXES/axisData/renderMirror`),
   audio (WebAudio drón), `loop/boot`
10. **`window.__dbg`** — teszthorgok: `S, KAPUK, enterStation, pickOption, obeliskTap, renderMirror…`

### Kapu-séma (KAPUK elem)

```js
{id, szam:'I'..'XI', cim, hely, ido, anyag, arch, hang,
 pal:{st,sm,hz,sl,        // égbolt: top/mid/horizon-glow/low
      fog,ground,fal,akc, // köd, talaj, fal, akcens
      sil,silK,silK2,     // sziluettszín + 2 kulisszatípus
      stars?,sun?,moon?,ink?},  // ink=1 → Jiankang világos tusköd, ritkább köd (.030)
 helyzet:[b1,b2],                      // 2 beat, együtt ~80–120 szó
 oszton:null|{k,o:[3]},                // első ösztön (csak: alex, jiankang, mani1562, medina)
 tanuk:[{r,nev,sz}×4],                 // r ∈ hatalom|perem|gyakorlo|tortenesz — EBBEN a sorrendben
 dontes:{k,o:[{c,tag,mech,ved,kock,kov,cert}×3]},
 perem:[{c,st:[chipek],cert,p}×3],     // Az Archívum pereme (Shadow Archive)
 szilankok:[3], kodex:'…'}
```

- `tag`: tengely-hozzájárulások, pl. `{ko:1,in:1,am:1}` — értékek ±1, tengelyenként
- `mech ∈ dontes|elhanyagolas|pusztitas|archivum`
- `cert ∈ dok|val|vit|spek` (Jól dokumentált / Valószínű / Vitatott / Spekulatív)
- `ved/kock`: az ár-képernyő két sora („Ezt védted / Ezt kockáztattad")

### Tengelyek (AXES)

`ko` Pluralitás↔Koherencia · `ha` Revizionizmus↔Hagyomány · `in` Egyéni tanúság↔Intézményi
tekintély · `ny` Nyitottság↔Stabilitás · `am` Ambiguitástűrés↔Bizonyosság · `me` Szelekció↔Megőrzés.
Pozitív érték = a második pólus.

### Kapu-id-k sorrendben

`javne alex nagh varanasi jiankang nippur ktesziphon mani1562 medina turfan ming`

---

## 5. Játékszabály-invariánsok (NE törd el)

1. **12 archívumkulcs** az egész játékra; **kapunként max 2 tanú**; meghallgatott tanú
   újraolvasása ingyen. Kulcs- és kapulimit a `showWitnesses/showWitness` gombtiltásában.
2. **Tengely-visszajelzés játék közben SOHA.** Döntés után csak az ár-kártya és
   „A Könyvtár feljegyezte." A tengelyek először a Tükörben jelennek meg.
3. **Tükör = bizonyított profil:** minden tengelymondat alatt evidencia
   („főként a(z) III., VI. kapunál"); csak `|sum|≥2` kap kilengés-mondatot, különben
   „kiegyenlített". Nincs archetípus, nincs „te ilyen ember vagy".
4. **Figyelem-kánon a Tükörben:** szerep-számlálók (`S.roleCounts`), soha meg nem hallgatott
   szerep, vak kapuk (`S.blind` — tanú nélküli döntések, szam-lista), elköltetlen kulcsok.
5. **Ösztön-mérés:** a 4 ösztön-kapunál first vs final összevetés (`S.instinct`), a Tükör
   kapuhivatkozással közli a változásokat.
6. **XII. kapu:** csak 11/11 után, az obeliszk ELSŐ érintésekor (`S.gate12===null` őrfeltétel!);
   16 kihagyott eset + opt-out („a hiányt választom" → `gate12=-1`); a választás a Tükörbe kerül.
   Részleges Tükör ≥5 kaputól, XII nélkül.
7. **Copy-invariánsok:** „tizenegy történelmi töréspont"; „mit emel közös örökséggé — és mit
   hagy a peremen"; címlap-meta „11 döntés · kb. 12–15 perc · nincs helyes válasz";
   CTA „Kinyitom az első kaput". Peremrovat státusz-chipjei megkülönböztetnek:
   nem kanonikus ≠ tiltott ≠ elpusztított ≠ elveszett ≠ periférikus ≠ újra felfedezett.
8. **Vizuális invariánsok:** égboltkupola és sziluettrétegek `material.fog=false` (különben a köd
   elnyeli őket); a Könyvtár (hub) állandó indigó–arany; **Jiankang (V.) szándékosan VILÁGOS**
   tusfestmény-jelenet — ez az esztétikai kockázat, ne „javítsd ki" sötétre.
9. **Nincs perzisztencia** (se localStorage — Claude.ai-artifact örökség, de döntés is:
   reload = új út, illik a replay-koncepcióhoz). Ha mentést vezetsz be, legyen explicit user-kérés.
10. Egyfájlosság: külső asset tilos (a three.js CDN kivétel); minden textúra procedurális canvas.
11. Érintőfelületek ≥44px; a `#card` max-height 58dvh, belül görgethető.

---

## 6. Tesztrecept (Node, hálózat/böngésző nélkül)

A sandbox-session harness-e nem jött át — így reprodukáld:

1. Script kiemelése: regex `<script id="game">([\s\S]*?)</script>` → `game2.js` → `node --check`.
2. Stubok: DOM elem-registry (`getElementById` → El: classList/innerHTML/children/onclick);
   `createElement('canvas')` → `{getContext:()=>Proxy-ctx, toDataURL:()=>'data:x'}`, a ctx-Proxy
   minden metódusra no-op, `createLinearGradient/createRadialGradient` → `{addColorStop(){}}`;
   `setTimeout` szinkron; RAF sorba gyűjtve, kézzel pumpálva; `THREE` Proxy — `/Material$/`→Mat,
   `/Geometry$/`→Geo, `/Light$/`→Node, egyébként Node-bázis (position/rotation/scale/add/traverse),
   külön: `Vector2, Clock, Raycaster(intersectObjects→[]), CanvasTexture(wrapS/wrapT/repeat.set),
   BufferGeometry/BufferAttribute, FogExp2`, konstansok (RepeatWrapping, BackSide, …).
3. Vezérlés `__dbg`-n át + `REG['cardBtns'].children[i].onclick()` kattintásokkal.
4. **PASS-kritériumok az utolsó zöld futásból:** 11 kapu done; 12 kulcs elköltve (keys=0);
   `roleCounts` összege 12; `blind = IV,VII,X,XI` a tesztterv szerint; instinct-változás 2/4
   (alex 0→1, mani 2→0 változott; jiankang 1→1, medina 0→0 nem); XII-lista 17 gomb
   (1 opt-out + 16); Tükör-HTML tartalmazza: „A figyelmed kánonja", „Ösztön és információ",
   mind a 6 tengelynevet, „Amit kockára tettél", „A kezed mozdulatai", a XII-választást,
   „te magad is így építed", „ellenkező ösztönnel"; újrabelépéskor 0 kulcsnál a még nem hallott
   tanúgombok disabled.

Bármilyen adat- vagy folyamszerkesztés után ezt futtasd újra.

---

## 7. Backlog a következő sessionnek (javasolt sorrend)

1. **Deploy + valódi eszköz QA** — iPhone Safari és Android Chrome: fps, betűméretek,
   safe-area, a Jiankang-jelenet olvashatósága (világos háttér + sötét kártya), tanúgombok
   érinthetősége. Ha lassú: `renderer.setPixelRatio` már 2-re maxolt; tovább csökkenthető
   a silhouette-szegmensszám (48→32), a dust/pages darabszám, az égbolt-canvas 1024→512.
2. **OG/social meta + favicon** — jelenleg nincs; egy inline SVG-favicon (data-URI) és
   og:title/description/image (az image lehet kis inline placeholder vagy külön statikus PNG a
   webrootban — utóbbi megtöri az „egy fájl" elvet, jelezd a usernek).
3. **`prefers-reduced-motion`** — szemcse-jitter és kamera-lerp mérséklése.
4. **Tükör-export** — „Másolás" gomb, ami a Tükör szövegét vágólapra teszi (megosztható,
   kép nélkül; a bizonyíték-sorok maradjanak benne).
5. Opcionális: offline-képesség (three.js inline-olása ~600 KB minified — csak ha a user kéri).
6. Ismert, ártalmatlan kódmaradványok: `pagoda` builderben a tekercspolc-pozicionálás kettőzött
   sora; `bazilika`-ban `desk.userData=null`. Takarítás ráér.

---

## 8. Kontextus-morzsák a userről (együttműködéshez)

Terse, auditor-szintű, közvetlenül implementálható válaszokat vár; magyar nyelv, angol
terminusok inline; CLI/automatizálás-first (SSH-only deploy, semmi kattintgatás); a döntéseket
indokold röviden, de ne kérdezz feleslegesen — dynamic workflow, végig önállóan, teszttel
bizonyítva. A játék tartalmi gerince egy kutatási jelentésből jön (Assmann/J.Z. Smith keret,
3 hipotézis, túlélési torzítás, 3 kirekesztési mechanizmus) — tartalmi módosításnál a Kódex-fülek
(SZEMPONTOK/HIPOTEZISEK/BUKTATOK/MECHANIZMUSOK) a forrásigazság, ne mondjon nekik ellent
semmilyen új szöveg.
