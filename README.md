# Árnyékkánon — Az elveszett morál kutatása

Egyfájlos, mobil-első 3D böngészős játék a kánonképződésről. A játékos tizenegy
történelmi töréspontra lép be, ahol egy közösség eldöntötte, mit emel közös
örökséggé és mit hagy a peremen. A végén a **Tükör** nem a történelmet mutatja
meg, hanem a játékos saját, bizonyított szelekciós profilját.

Kulcstétel: **a figyelem is kánonképzés.**

**Élő:** https://egedi.art/kanon/

---

## Mi van a repóban

| Útvonal | Mi ez |
|---|---|
| `index.html` | A teljes játék: adat, 3D, UI, hang — egyetlen `<script id="game">` blokkban (~1480 sor) |
| `assets/three.min.js` | Three.js r128, helyben (nincs CDN-függés) |
| `img/cover-mobil.jpg`, `cover-desktop.jpg` | A festett borító = címképernyő, álló és fekvő változatban |
| `img/sky.webp` | Csillagköd-kupola a Könyvtár (hub) fölött |
| `img/ground.webp` | Kőspirál padló-medalion a hub közepén |
| `img/steles-off.webp`, `steles-on.webp` | 11 sztélé alfás atlaszban, alap és izzó állapot |
| `HANDOFF-kanon-v2.md` | Belső architektúra-leírás és játékszabály-invariánsok. **Szerkesztés előtt olvasd el.** |
| `PROMPTS-sztele-render.md` | Render-promptok a 11 kapu tereihez (kép + videó) és a sztélé-sprite-sheethez |
| `deploy.conf`, `.github/workflows/deploy.yml` | Az automatikus feltöltés konfigja |

A `HANDOFF` és a `PROMPTS` a repóban marad, de a szerverre **nem** megy ki
(`EXCLUDES` a `deploy.conf`-ban).

## Futtatás helyben

A textúrákat a Three.js `TextureLoader`-e tölti, amit a `file://` protokoll
CORS-ból blokkol — **HTTP-szerver kell**, különben az ég, a padló és a sztélék
nem jelennek meg:

```bash
python3 -m http.server 8899
```

Aztán nyisd meg: `http://127.0.0.1:8899/index.html`

## Deploy

Push a `main`-re, és 1-2 percen belül fent van. A GitHub Actions futtatója
végzi, tehát a desktop állapota közömbös — telefonról is működik.

```
push → GitHub Actions → rsync SSH → Kinsta → egedi.art/kanon/
```

A deployer a [0longrun/egedi-ci](https://github.com/0longrun/egedi-ci) repóban
él, minden titok egyetlen `EGEDI_DEPLOY_BUNDLE` secretben. Három védelem
működik: a célmappa nem lehet a dokumentumgyökér, léteznie kell (kivéve
szándékos `CREATE_SUBDIR=1`), és sok törlésnél megáll.

Kézi indítás vagy próba: Actions fül → `deploy` → *Run workflow* (`dry_run`
kapcsolóval megnézhető, mi változna).

## Architektúra dióhéjban

Egyetlen `<script id="game">` blokk, sorrendben: adat (`KAPUK[11]`, `KAPU12[16]`)
→ állapot (`S`) → procedurális festőműhely (canvas-textúrák) → 3D mag →
`BUILD{}` a 11 állomásépítővel → hub és Tükör → UI és kapufolyam → hang → boot.

Részletes térkép: `HANDOFF-kanon-v2.md` §4.

### Grafika: procedurális + festett

- **A 11 kapu tere procedurális** — minden kapunak saját `pal` palettája és
  `arch` architektúra-típusa van. Ezekhez a repó **nem** tartalmaz képet.
- **A Könyvtár (hub) festett** — ég, padló és a 11 sztélé képfájlból jön.

Ez a határ szándékos. Ha a kapukat is festett grafikára cserélnéd, az külön
munka, kapunként külön képekkel.

### Sztélé-atlasz

A `steles-off/on.webp` 11 cellája balról jobbra a `KAPUK` sorrendjét követi. A
két lap **pixelre fedésben** van (közös befoglaló keretből vágva), ezért az
izzás-váltás nem ugrik. Cellánként textúra-klón (`steleTex`), közös képpel.

Érintésre a sztélé felizzik és lassan visszahalványul; bejárt kapu állandóan
izzik.

### Hang

Procedurális WebAudio, külső fájl nélkül: 110–220 Hz-es szőnyeg lowpass mögött
(a korábbi 55 Hz mobilhangszórón csak zörgött), plusz öt borítékolt effekt —
`sfxShard`, `sfxGate`, `sfxTick`, `sfxHarp`, `sfxMirror`. Minden gain- és
frekvenciaváltás anchorolt ramp, hogy ne kattanjon.

## Amit ne törj el

A teljes lista a `HANDOFF-kanon-v2.md` §5-ben. A leggyakrabban véletlenül
elrontott pontok:

1. **Tengely-visszajelzés játék közben SOHA.** A tengelyek először a Tükörben
   jelennek meg.
2. **Nincs perzisztencia** — se `localStorage`. Az újratöltés új játszma.
3. **Jiankang (V.) szándékosan VILÁGOS** tusfestmény-jelenet. Ne „javítsd ki”
   sötétre.
4. Égboltkupola és sziluettrétegek: `material.fog=false`.
5. **12 archívumkulcs** az egész játékra, kapunként max 2 tanú.
6. **XII. kapu** csak 11/11 után, az obeliszk első érintésekor
   (`S.gate12===null` őrfeltétel).
7. A kódex-bejegyzések „KÁNON:” címkéi **köznévi** értelműek (maga a kánon
   fogalma), nem a játék neve — átnevezéskor ne cseréld őket.

## Tesztelés

A játék `window.__dbg` alatt teszthorgokat ad:

```js
__dbg.S              // teljes állapot
__dbg.KAPUK          // a 11 kapu adata
__dbg.enterStation('javne')
__dbg.previewGate('alex')   // kapuválasztás + sztélé-izzás
__dbg.collectShard(0)
__dbg.renderMirror()
__dbg.getSteles()    // a hub sztéléi (izzás-opacitás mérhető)
__dbg.scene, __dbg.camera, __dbg.renderer, __dbg.ART
```

Szintaxis-ellenőrzés deploy előtt (a beágyazott script kivágásával):

```bash
python3 -c "import re;h=open('index.html',encoding='utf-8').read();open('/tmp/g.js','w',encoding='utf-8').write(re.search(r'<script id=\"game\">(.*?)</script>',h,re.S).group(1))" && node --check /tmp/g.js
```

## Buktatók, amikbe már belefutottunk

- **Böngészős „mentés weboldalként” TILOS.** Nem a szűz fájlt menti, hanem a
  futó játék DOM-pillanatképét: a címképernyő `hidden`-nel, halott gombokkal, a
  canvas és a generált háttér a markupba égetve. A játék így bootkor
  megbénul. Ha artifactból mentesz, a letöltés/export gombot használd.
- **`file://` alatt nem tölt textúrát** (CORS). Mindig HTTP-szerverrel tesztelj.
- **Átlátszó, majdnem egysíkú padló eltűnik.** A festett medalion `transparent:
  true`-val némán kiesett a renderből. Ezért a pereme a képbe van festve, a lap
  átlátszatlan, és a külső talaj gyűrű, nem korong.
- **A hub alapnézete −0.25 pitch.** Pontosan vízszintesen, a kör közepéből a
  felülnézeti padlókép perspektívába nyomódik és sávosnak látszik.
- **Körbenézés panoráma-konvenció szerint** vízszintesen (a látvány követi az
  ujjat); függőlegesen szándékosan a másik irány.

## Eredet

Vallástudományi kutatásból épült: kánonképződés és szövegkirekesztés tizenegy
hagyományban. A kódexszövegek valós kutatási álláspontokat idéznek, a
bizonyosság-címkékkel együtt (`dok` jól dokumentált · `val` valószínű ·
`vit` vitatott · `spek` spekulatív).
