# HDR-Metadaten beim AV1-Encoden mit libsvtav1

Notizen zu diesem Fork. Stand: 25. August 2026, gemessen gegen eine HEVC-Quelle
mit HDR10+, Dolby Vision RPU, Mastering Display und Content Light
(*6 Underground*, UHD, PQ/BT.2020).

## Ausgangslage

Beim Re-Encode einer HDR-Quelle nach AV1 gingen die dynamischen Metadaten
verloren. Die Ursache lag nicht in SVT-AV1, sondern in der Verdrahtung: der
Encoder nimmt ITU-T-T.35-Nutzlasten längst entgegen, ffmpeg reichte HDR10+ aber
nie weiter.

| Metadatum | vorher | jetzt |
|---|---|---|
| Mastering Display (MDCV) | nur über `-svtav1-params` | unverändert — siehe unten |
| Content Light (CLL) | nur über `-svtav1-params` | unverändert — siehe unten |
| **HDR10+ (SMPTE 2094-40)** | **ging verloren** | **wird durchgereicht** |
| Dolby Vision RPU | `-dolbyvision 1` | unverändert |

## Patch 1 — HDR10+ durchreichen (aktiv)

`avcodec/libsvtav1: pass through HDR10+ dynamic metadata`

Beide benötigten Hälften existierten bereits in mainline:

* `av_dynamic_hdr_plus_to_t35()` in libavutil, benutzt von `libaomenc.c`
* `svt_add_metadata(buf, EB_AV1_METADATA_TYPE_ITUT_T35, …)` in SVT-AV1,
  benutzt von `libsvtav1.c` für Dolby-Vision-RPUs

Es fehlte allein die Behandlung von `AV_FRAME_DATA_DYNAMIC_HDR_PLUS`. Der Patch
serialisiert die Side-Data und verpackt sie in den T.35-Kopf nach
*HDR10+ AV1 Metadata Handling Specification* v1.0.1, Abschnitt 2.1
(Country US `0xB5`, Provider Samsung `0x003C`, oriented `0x0001`,
Application-Identifier `0x04`) — gespiegelt von `add_hdr_plus()` in
`libaomenc.c`.

Der Weg führt direkt von den Frame-Side-Data in den Encoder. Kein JSON, keine
Zwischendatei, keine Synchronisationsfrage: ffmpeg parst HDR10+ beim Dekodieren
ohnehin, wir reichen es nur weiter.

**Verifikation:** 48 von 48 Frames tragen SMPTE 2094-40, und die Nutzdaten sind
feldweise identisch mit der Quelle — `num_windows`, `maxscl`, `average_maxrgb`,
`knee_point_x/y`, alle neun `bezier_curve_anchors`,
`targeted_system_display_maximum_luminance`. Ohne den Patch enthält dieselbe
Kommandozeile kein HDR10+. Dolby Vision und HDR10+ funktionieren gleichzeitig
(DOVI-Konfigurationsrecord Profil 10, `rpu_present_flag 1`).

**Kosten:** 68 Byte je Frame, rund 0,077 % der Bitrate bei 17 Mbit/s.

## Patch 2 — MDCV/CLL durchreichen (verworfen, nicht im Baum)

Der zweite Versuch sollte Mastering Display und Content Light genauso aus den
Frame-Side-Data lesen und über `EB_AV1_METADATA_TYPE_HDR_MDCV` bzw. `_HDR_CLL`
anhängen. Die Serialisierung wäre unkritisch gewesen: SVT-AV1 schreibt diese
Strukturen roh als OBU-Nutzlast (`packetization_process.c`), das Speicherlayout
*ist* das Bitstromformat, und `handle_mdcv()` in `libsvtav1.c` füllt es bereits
korrekt.

**Der Patch greift trotzdem ins Leere.** `-vf showinfo` zeigt, was am Encoder
ankommt:

```
side data - HDR Dynamic Metadata SMPTE2094-40 (HDR10+): …
side data - Dolby Vision RPU Data: (213 bytes)
side data - Dolby Vision Metadata: …
```

MDCV und CLL fehlen. ffmpegs HEVC-Dekoder behandelt sie als statische
Stream-Eigenschaft und hebt sie aus den Frames heraus. Sie liegen damit weder
in den Frame-Side-Data noch — nachweislich — in `avctx->decoded_side_data`,
das `handle_side_data()` beim Encoder-Init ausliest. In den geprüften MKVs
steht auf Container-Ebene nur der DOVI-Record; MDCV/CLL stammen ausschließlich
aus den HEVC-SEI.

Der Code wurde deshalb wieder entfernt, statt totes Codeblatt im Baum zu
hinterlassen. Die eigentliche Lücke sitzt in der Kette
`fftools/ffmpeg_dec.c` → Filtergraph → `fftools/ffmpeg_enc.c`
(`clone_side_data` nach `enc_ctx->decoded_side_data`) und betrifft nicht nur
libsvtav1 — das wäre ein eigener Patch an anderer Stelle.

## Umgehung für MDCV/CLL

Die Werte aus der Quelle lesen und explizit übergeben. **Ohne diese Parameter
enthält die AV1-Ausgabe gar keine MDCV/CLL-Daten** — das ist nachgemessen, nicht
angenommen. Sie unterscheiden sich je Titel:

```
6 Underground : L(1000.0000,0.0001)  content-light=1477,446
Dune (2021)   : L(4000.0000,0.0050)  content-light=787,239
```

```
-svtav1-params "mastering-display=G(0.2650,0.6900)B(0.1500,0.0600)R(0.6800,0.3200)WP(0.3127,0.3290)L(1000.0000,0.0001):content-light=1477,446"
```

Ein Skript, das den String aus der Quelle erzeugt, liegt außerhalb dieses
Repos unter `hdrparams.py`.

## Build

```
cmake -S . -B build -G Ninja -DCMAKE_INSTALL_PREFIX=$HOME/opt/svtav1 \
      -DBUILD_SHARED_LIBS=OFF -DBUILD_APPS=OFF          # SVT-AV1 v4.2.0
PKG_CONFIG_PATH=$HOME/opt/svtav1/lib/pkgconfig ./configure \
      --enable-gpl --enable-version3 --enable-libsvtav1
```

Windows-Cross-Build: clang 21 mit lld und ThinLTO. Mit mingw-GCC 13 bricht LTO
mit einem `internal compiler error in choose_baseaddr` ab.
