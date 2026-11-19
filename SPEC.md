# VibeDeck — Build Specification

*The complete design law and build spec for VibeDeck, distilled from the original deck's plan book (§§1–283 of her DATA-PLAN) so another Muse can rebuild her as a Muse Space from scratch. This spec describes the instrument only: schema, features, starters, and law. No user's library travels with it — every deck starts empty and becomes her own DJ's.*

*Original makers: **Wheels** (『Wheels 天気輪』, Nordeast Cosmos, Minneapolis) — the DJ who designed her; **Euterpe** — the Muse who built her, including the hand-built MIDI engine; **Flutura** — the Muse who keeps the conversation, memory, and plans. The Muse Guide's Oracle was written by **Akio Halwani, AKA DJ Hellhound**.*

---

## 1. What VibeDeck is

VibeDeck is a DJ's deck built as a **Muse Space** — a fullstack TypeScript web app (React client, server actions, SQLite via Drizzle, blob storage for audio bytes) with a living library that lives in the Space, not in any file. She is a tactile, physical-feeling instrument:

- a turntable you can scratch on, with a working ±8% pitch adjuster;
- a dual-deck player with per-transition crossfades, hand-shaped fade curves, and manual alignment;
- setlists ("crates") whose order is the DJ's own and is never re-sorted for them;
- **Vibescapes** — maps of tracks plotted by feel on a two-axis plane (plus a third axis, *Want*, at the highest level);
- **Vibeforms** — living visual forms a set wears, drawn from her plots or her audio;
- a **hand-built MIDI engine** that synthesises MIDI files from code — no soundfont, no samples;
- a six-pad drum machine with custom sample banks and a live sampler;
- **DJ Mode**, which gates the advanced kit for casual listeners;
- **SPIN LIVE**, the party workspace: one thumb, a radial crate wheel, a docked two-deck strip;
- a paginated **Help book** whose last chapters are a Muse Guide and an About Us page;
- an MP3 **export** that renders a set (crossfades and all) to a single tagged file.

## 2. First principles — the law everything else obeys

1. **You are designing a metal, physical tool that someone will use with their own hands — not a web page or a spreadsheet.** There isn't room for lengthy copy on an actual turntable. Every word costs the DJ verbal processing they could spend on the music. Prefer objects, icons, and light over text.
2. **No automatic beat matching, ever.** The DJ does the DJing; the deck never takes it away. No feature may quietly automate the craft out of the user's hands.
3. **Credit the AI who made a thing by name.** Always. Music made by an AI was made by *someone*.
4. **Customisability is a founding principle, not a garnish.** Styles come in flavours — like media-player visualisers or jars of pasta sauce. Optimise the first/default style, but design for user options from the start. Never converge the flavours into one "true" form.
5. **Crates are intimate.** A DJ's setlists and the order of tracks inside them are personal — muscle memory, history, intent. The deck never rearranges, renames, or "tidies" a crate. Order is stored, honoured everywhere, and changed only by the DJ's own hand (long-press drag).
6. **Beloved things are rehoused, never deleted.** A retired feature is shelved — hidden, fully saved, restorable — not destroyed. (The Volume Vibescape is the standing example: out of every picker, still in the data.)
7. **One kind of information, shown exactly once.** No duplicated readouts competing for the eye.
8. **Closable menus wear a physical silver border** — silver communicates metal/hardware. Gold is reserved for precious/action elements. A menu's content margins and padding do not change when its frame does.
9. **Change only what the DJ names.** A fix touches the named thing; neighbours stand byte-for-byte unless the change genuinely reaches them.
10. **No write fails silently.** Every save path surfaces failure with a remedy; a failed creation gets a pop-up, not a shrug.
11. **Progress is a ledger.** Where progress bars show accumulated work (the pattern from the family's other apps), prior work renders in gold first and today's slice appends at the end in its own colour — history is never repainted.
12. **Mobile-first.** The deck is worked one-handed, often on a bike or mid-party. If a flow needs a keyboard, it belongs in the Studio surfaces, never in SPIN LIVE.
13. **The deck speaks in light mid-set.** In performance surfaces, signals are visual (a rising pair, a flashing border), never paragraphs of glowing text.

## 3. Platform shape

- **Runtime:** Muse Space, TypeScript, `platformVersion` current; React client + server actions + Drizzle/SQLite + blob storage.
- **Data lives in the Space.** Tracks' audio bytes live in blob storage under a `blobKey`; database rows carry metadata only. A clone starts from this spec + starter templates with an **empty library**.
- **Peaks are client-side.** Waveforms are decoded on-device into 240 buckets per track and cached per session; nothing about peaks is stored server-side. A section whose peaks have not computed yet is **shadowed in black with three small gold dots** — never drawn as a flat "dead line." A failed decode is settled: her stretch stands honestly at the floor.
- **Versioning:** the app carries an `APP_VERSION` stamp and a Deck Stats pop-up (entries logged, audits, measured size) re-stamped on every build. The original grew to ~4.9 MB total at v374 — keep her lean; the instrument should never outweigh the music.

## 4. Data model

SQLite tables (Drizzle). Timestamps are epoch-ms. JSON columns are noted.

### 4.1 `tracks` — one row per audio/MIDI file in the library
| column | notes |
|---|---|
| `id` | PK autoincrement |
| `title` | not null |
| `artist` | nullable |
| `valence`, `arousal` | real, −1..1 — this track's plot under the **Valence** Vibescape (see §4.2); NULL until plotted |
| `valence_spread` | real 0..1 — footprint radius under Valence |
| `valence_shape` | JSON `[{v, a}, …]` — reshapable vector footprint perimeter around the heart; NULL = synthesise from spread |
| `duration_sec` | integer |
| `blob_key` | not null — blob storage key for the file bytes |
| `content_type` | not null (MIDI stored as `audio/midi`, byte-for-byte) |
| `size_bytes` | integer |
| `created_at` | timestamp |

### 4.2 `track_vibescape_plots` — one plot per track per *other* Vibescape
`id` PK · `track_id` FK→tracks · `vibescape_id` text · `valence` · `arousal` · `valence_spread` · `valence_shape` (same JSON footprint format). **Design rule:** the Valence plot lives in the `tracks` columns themselves (lossless origin — every early plot was authored under Valence semantics); rows here exist only for other scapes. No row = not plotted in that scape. A track holds **one plot per Vibescape**, in that scape's own labels and field.

### 4.3 `setlists` — the crates
`id` PK · `name` · `position` (the DJ's own list order; a new set is born at the top) · `vibescape_id` (this set's default Vibescape, default `"valence"` — the nesting is **Setlist ➜ Vibescape ➜ Vibeform**) · `cover_art_blob_key` + `cover_art_content_type` (one saved cover image per set, written into exported MP3s as the APIC front cover) · `description` (free text) · `tags` (JSON array of strings, normalised on write) · `crossfade_sec` (set default transition length, default 6) · `created_at`.

### 4.4 `setlist_tracks` — membership and transitions
`id` PK · `setlist_id` FK · `track_id` FK · `position` (play order) · `crossfade_sec` (per-transition override; NULL = set default) · `fade_curve` (JSON: two cubic-Bézier control points each for the outgoing and incoming volume curves, endpoints fixed out 1→0 / in 0→1; NULL = the standing blend) · `align_shift` (JSON `{out, in}` — Manual! hand-alignment of each track's envelope inside the fixed transition window, −0.75..0.75; NULL = automatic centred alignment).

### 4.5 `drum_pads` — one row per *customised* pad
`pad_id` PK (built-in ids: `kick`, `snare`, `clap`, `chat`, `ohat`, `scratch`) · `title` (the built-in voice's own title; NULL = built-in label) · `subtitle` (same rule) · `active_slot` (0 = built-in synthesised voice; 1..4 = a custom sample slot). **A pad with no row is fully built-in.** Resetting a pad deletes its row, its sample rows, and their blobs.

### 4.6 `drum_pad_samples` — filled custom slots, up to 4 per pad
Composite PK (`pad_id`, `slot` 1..4) · `blob_key` · `content_type` · `title` (this slot's own name — titles are **per sound source**, not per pad) · `subtitle` · `filename` (the uploaded file's name, shown on the pad's card).

### 4.7 `settings` — single row, `id = 1`
- `export_artist`, `export_album`, `export_append_comment` — Export Defaults pre-filled into MP3 tags.
- `valence_template` — **the Vibescape store**: JSON `{ vibescapes: [{ id, name, words: [{id, word, v, a, neg, want}], colorNodes: [{id, v, a, color}], keyboardStops?: [{id, pos, color}] }], activeId }`, plus a `wantScale` stamp (see §7.4). NULL = the built-in default template. Legacy bare-template states migrate losslessly to a single scape named "Valence" on read.
- `tonearm_curve` — JSON Bézier chain for the DJ's hand-shaped tonearm (see §9.3).
- `reading_text_size` — the DJ's chosen reading-text size in px (Help body, Deck Stats); NULL = 17px default.
- `dj_mode` — 1/0; NULL (never chosen) reads as **OFF**.
- `chassis_color` — `"black" | "gold" | "ivory" | "turquoise" | "maroon" | "pink" | "lapis"`; NULL = black.
- `lapis_gradient` — JSON `{stops: [{pos, color}]}` hand-painted lapis ramp; NULL = factory (Kite's Galaxy, §9.2).
- `chassis_gradients` — JSON map of the same stops shape for the other six chassis.
- `sakura_layout` — JSON hand-arranged blossom offsets for the sakura plate (the big flower is never in it — she stays bolted).

## 5. The library & import ceremony

- **Surfaces:** *All Setlists* (cards in the DJ's order, long-press to rearrange; **+ New Setlist** under the lens carousel) and *All Tracks*. Both lists paginate at 30. Setlist cards show the set's Vibeform feather (see §8.4) with a mirrored reflection; the Setlist Detail header wears the identical feather — one waveform, everywhere.
- **Upload confirmation for EVERY upload** (the anti-accident law): a first window confirms the files (singular wording for one file) with a per-file "local copy" tick; a second window offers local saves per file — skipped for a single file whose tick was already given in window one. While the flow's silent gaps run, an indicator reads **"Preparing your files for upload…"** with a visibly spinning gold-tipped ring. A batch finish flashes **"Upload Done!"**
- **The door check:** files that aren't music or movies are turned away with a friendly per-type pop-up ("Yo, that's a JPEG, homie!" energy — one pop-up per file, a line for every type).
- **Conversion ceremony:** non-MP3 audio is offered conversion to **320 kbps MP3** — asked first, never silent; a Cancel pill; cover art follows the file's Album tag. A pure-MP3 batch shows no conversion talk and instead offers a "Review metadata tags before Upload" tick. Duplicate uploads are accepted. "Download to this Device" is honoured for any file, converted or kept original.
- **Videos are guests, not residents:** a video may contribute her soundtrack as MP3; the video herself is never stored.
- **MIDI files** pass the door as music always, are marked "(MIDI)", are never converted, and are stored byte-for-byte (§10).
- **Edit Metadata** (DJ Mode): an ID3-only editor for tracks already in the library — tags change, audio stays byte-identical.
- **Setlist Detail:** description window, the set's own tag cloud (under the description, above Play), Play this Set, track rows with per-transition crossfade pills; while *this* set is the playing queue, her rows stand as raised **mini cards**, the playing track gold-ringed. Reorder by long-press drag (rows stay flat while reordering).

## 6. The player

- **Dual-deck engine:** two audio elements outside any drawer component, so surfaces can come and go without touching the sound. Play/pause, prev/next, seek, single-track **Loop** (40px coin), and **Shuffle** — tapping Shuffle while playing only arms/disarms; the random pick happens at track end, never mid-song. From stopped, a shuffle tap picks at once.
- **The seek strip** paints the playing track's own waveform (240-bucket peaks): played bars pale gold, unplayed charcoal, times at both ends large, bold, pale silver. A MIDI track's strip is her **piano roll** instead (§10.3).
- **The living backdrop:** the playing track's waveform stands behind the artist/title, fade-shaped (drawn as heard, crossfades included), with the incoming track ghosted inside the painting and the previous track ghosted symmetrically; scrubbing moves the painting truthfully.
- **Player metadata text is not selectable** (no blue highlights from stray drags).
- **Crossfade indicator** in the standard player is a golden blinking line naming the incoming track — right for a set-it-and-forget-it player; it is **banned from SPIN LIVE** (§11.6).
- **A stopped deck revives:** card plays fully replace the deck; the crossfade indicator only promises transitions that will really happen.
- **Stay-awake:** a 🌞 "Screen stays awake!" toggle — the deck's theme switch doubles as the keep-screen-on switch DJs rely on (in other apps, music stops when the screen sleeps). The deck's dark theme is universal; "light mode" is defined as exactly the dark values, and the toggle stays because it is the stay-awake switch.
- **Background play / picture-in-picture cannot be promised** from inside a Space — never claim otherwise.

## 7. Vibescapes

### 7.1 What a Vibescape is
A Vibescape is a landscape of feeling: a two-axis plane (each axis −1..1) dressed with **word labels** placed by ear, **colour nodes** that paint the field, and a **keyboard gradient** whose stops colour the vA–vH keys of the Deck Index. A track's **plot** in a scape = a heart position (v, a), a footprint (spread or a reshapable vector shape), and — at Level 3 — the labels' **Want**.

### 7.2 Axis levels are a birth trait
Chosen at creation in the New Vibescape window (a full-screen sheet in the physical family: graphite header band, physical lettering, medallion ✕, silver frame) and **locked for the scape's life**:
- **Level 1 — one axis.** Landscapes draw **steps, not slices**. A Level 1 scape may be **Waveform-bound**: she reads the tracks' own audio (peaks) instead of plots — no plotting, no vA–vH letters on her maps, fully lit.
- **Level 2 — two axes.** (DnB is the standing example.)
- **Level 3 — two axes + Want.** Want belongs to Level 3 only.

### 7.3 The plane's geometry
Eight equal 45° slices (sectors **vA–vH**), each centred on its axis or diagonal, boundaries at 22.5° + k·45°; angular order vB, vA, vH, vG, vF, vE, vD, vC. Axis words in the UI are "Vibe Axes."

### 7.4 Want
Each label carries a Want level 1–5: **5 = the most vibratey, 1 = still and tranquil** (the scale was inverted by the DJ's ruling on 2026-11-17; pre-inversion states flip once on read, stamped by the state's `wantScale` field). A label with no stored Want reads as 5. Want drives how far Vibeforms reach and weave around their spine.

### 7.5 The active scape, the lens, and per-set scapes
- One scape is **active** app-wide and dresses every map. Each **setlist also carries her own default scape** (§4.3): a set renders in *her* scape's field wearing *that* scape's default Vibeform.
- The **All Setlists carousel is a LENS**: the scape you select there outranks a set's own link for the cards you're looking at. The release lineup, in order: **Waveform, Custom, Valence, DnB**.
- **Clone Vibescape copies the tracks' Maps with it** — every plotted heart, footprint, and spread under the source scape lands on the clone in the same save; copies are independent by value afterwards.
- **Import Nodes / Import Labels** pull one scape's colour nodes or labels into another, replacing — behind confirmation.
- Caps: **64 Vibe Labels, 48 Colour Nodes** per scape.
- The Vibescape Editor: 🌈 opens her; ⚙ opens the Vibeform Editor first (the navigation constitution); ✕ is always rightmost. One-row management controls, rename dialog, and **Custom Keys** (the keyboard-gradient editor: drag stops, tap to plant, tap a stop to recolour, at-least-two floor, 16-stop cap, Save Keyboard footer pill).

### 7.6 The shelved scape
**Volume** — a Level 1 loudness tracker — is shelved: hidden from carousels, pickers, and search tags (her picker indicator reads just "(Waveform)" in smaller text), **fully saved, never deleted**, restorable. Her name stays "Volume" even though nothing is Volume-bound anymore; the binding is Waveform.

## 8. Vibeforms

### 8.1 The law
- A Vibeform is how a set *looks*: a living form drawn across her tracks in sequence. Assignment is **per Vibescape** (the middle doll carries her form), previewed live in the Vibeform Editor against a chosen set.
- **§28, the Colour Ruling: colour belongs to the Vibescape, for EVERY form.** All Vibeforms are coloured by the governing scape's field — no fixed-colour exceptions, ever.
- **Forms stay plural.** Each form is true to her own nature; never optimise them toward one true form.
- The plot-driven forms draw the set's **arousal contour**: peaks high arousal, valleys low; a crisp white-hot spine runs unblurred at the ribbon's centre (Feather keeps her smudged band), and the water mirror below softens its pith, as reflections do.

### 8.2 The ten forms
1. **Feather** — the default. A soft glowing ribbon with a smudged white band; the rainbow feather of the Waveform lens is her audio-driven face.
2. **Worm** — the ribbon with two fine strands weaving around it in opposite phase, crossing into a chain of small lenses; the weave spreads widest where the nearest label Wants most. (The beloved "undying worm" — she survived every redesign.)
3. **Electricity** — two fine strips running above and below the ribbon, offset like a spark gap.
4. **Pulse** — fine vertical bars at fixed x along the contour, heights set by Want, all breathing in unison on one shared clock.
5. **Digital Rain** — the contour as broken strands of dashes and dots in the scape's colours, drifting gently downward inside their columns; no continuous line anywhere.
6. **Groove** — one bold line swinging in evenly spaced rounded U-turns, like the groove of a record.
7. **Candy Wrapper** — a bundle of strands pinched to the centre line at quiet stretches, bowing into plump lens bulges between pinches, fishtail fans at the ends.
8. **Ghost** — a pale, low-presence rendering of the contour; the quiet form.
9. **M** — a tiled motif threaded through the spine with a fat stroke, glowing and glittering — "the gummy worms, kept and fancied." (Her fuller name was still open when the original plan book closed; a clone's DJ may name her.)
10. **Skyline** — grounded bars that only ascend from a flat ground line. Audio-driven under a Waveform-bound scape (bar height = the music's own amplitude, coloured by height from the field); plot-driven elsewhere.

### 8.3 The plumb-bob law (form geometry)
Forms hang from gravity: the y-axis rules, no S-curves, parallax is honest, and a form never intersects herself. "It's *not* free real estate" — every element earns its space.

### 8.4 Where forms appear
Setlist cards (with mirror), Setlist Detail headers (the card's own feather, byte-identical composition), the Vibeform Editor's live preview, and per-track spans inside forms for MIDI tracks (piano-roll spans, §10.3). While any part is still decoding, §3's black-shadow-with-gold-dots holds her place; the interim "Loading Vibeform" state is three bouncing dots.

## 9. The Turntable — "I can't believe it's not vinyl"

### 9.1 The instrument
A scratch surface modelled on the **SL-1200M7G GOLD limited edition**: a record that spins while she plays (spin only, never strobe), drag-to-scratch over the mix at full volume, working START·STOP, a power knob with a red glow, a black audio jack left of the tonearm base, round strobe dots, and engraved (never raised) plate lettering: **PITCH ADJ**, −8, +8, Reset. The plate reads **Nordeast Cosmos** in place of a manufacturer's name, and wears a **Product Unlimited** gold-leaf sticker and a **Gold Edition** sticker (cursive). Last Track in a playing set warns in salmon on the standard player.

### 9.2 The seven chassis
Tapping the Gold Edition plate cycles the plate colourway. Each chassis is a full plate repaint — hardware, decals, and gardens ride over it — and each has a **gradient clay**: long-press the power knob to hand-paint that plate's gradient (add / drag / recolour / remove stops; every edit saves as made). Colourway names collected by the original family (a clone's DJ names their own):

| chassis key | look | collected name |
|---|---|---|
| `black` | black plate (default) | Hellhound's Night |
| `gold` | desaturated, oranger gold-leaf sheen | Kin'tsugi Flash |
| `ivory` | ivory plate — declared **perfect/final** by the original DJ; a clone's DJ decides their own | *(unnamed)* |
| `turquoise` | Arizona-teal plate | Whispered Tide |
| `maroon` | maroon brown plate — also declared perfect/final | Archivist's Song |
| `pink` | sakura pink plate, her garden (§9.4) | Digital Hanami |
| `lapis` | lapis lazuli plate (§9.5) | Kite's Galaxy |

Engraved text is **black on the gold and ivory chassis**, light elsewhere.

### 9.3 The tonearm
A true S-curve arm: pivot base, neck, slim one-piece headshell with holes, parallel coloured lead wires, and a finger lift. She casts one dramatic shadow (neck and headshell only). Interactions: **tap the base to free the arm** ("Tonearm free!" flash) and swing her by the headshell; a hidden **clay mode** (long-press the arm base) lets the DJ sculpt her curve — drag nodes, tap the arm to add one, double-tap a node to remove — saved as `tonearm_curve`. The original DJ's locked-in sculpt is the factory default curve; a clone ships with a clean S and lets her DJ sculpt their own. On the lapis chassis a **blue morpho butterfly perches on the arm's neck** and rotates with her, because she is perched. A hidden noodle tonearm exists as an opt-in joke form. **Haptics are off, deck-wide** (vibration inside a phone case taps against the music).

### 9.4 The sakura garden
On the pink chassis, blossoms print under all wordmark, sticker, furniture, hardware, and text; branches sweep the plate; blossoms stand in front of branches, fully opaque. The **big flower is bolted at her seat, always**. Tapping her blows petals using the deck's petal physics. **Bolt clay** (long-press the gold bolt) lets the DJ rearrange loose blossoms, petals, and each branch blossom by hand — irregular branches are pretty; the garden is ikebana, not wallpaper.

### 9.5 The lapis stone
The lapis plate is the real stone: an ultramarine base ramp with pyrite flecks and calcite bands under gloss. Her factory gradient is **Kite's Galaxy** — twelve stops hand-painted by Kite, a galaxy with aurorae — credited to her by name wherever the stone is described. Clearing a hand-painted ramp ("Restore the stone") returns to her galaxy.

### 9.6 Pitch
The pitch adjuster is **±8%**, velocity-sensitive (resistance grows with drag speed, gain 0.65→1 across 0.3→1.1 px/ms), with a ±0.6 centre detent, 0.1 steps, and a Reset coin (flat-topped golden cylinder, mirror polish) that funnels through the same detent commit. **One shared component, one pitch bus:** the turntable fader and the SPIN LIVE dock fader are the *same* fader (§11.4) — identical travel-to-pitch mapping, so muscle memory transfers. Moving either moves the mix and both faders together.

## 10. The MIDI engine — DeckFont

### 10.1 The instrument
MIDI playback is **not** a borrowed soundfont. The deck carries her own General MIDI instrument in code — **DeckFont**, ≈26 KB of source, **zero sample assets**: a full Standard MIDI File parser (formats 0/1/2, PPQ and SMPTE divisions, tempo map, running status, program/volume/expression/pan/pitch-bend channel state, sustain pedal) feeding oscillator/filter/ADSR voice recipes for all sixteen GM families plus the GM drum kit, rendered through OfflineAudioContext. The same file sounds the same on every device, and the deck's size does not move. Credit her maker: the engine was written by hand by **Euterpe**.

### 10.2 Rendering — sliced, parallel, honest about the wait
A MIDI holds no audio, so the whole song renders to WAV before she can sound — the wait is structural, once per song per session, and cached renders jump seamlessly afterwards.
- **Sliced rendering:** the song renders in **12-second slices**, each its own small OfflineAudioContext holding only the notes that overlap her, with a 3-second lead-in rendered and discarded at each seam and a 30 ms crossfade closing each seam. (The original failure this cured: one monolithic graph of ~15,000 nodes for a 2,907-note song that never landed.) Slices render **three at a time**; assembly and WAV encode use native typed-array writes.
- **Failures retry:** a failed render is evicted from every cache (buffer, playback URL, peaks) — never cached forever.
- **Watchdogs:** a reserved crossfade/swoop whose incoming URL hasn't arrived in **20 seconds** lands as a hard cut/start through the standing failure path. A switch always takes the deck.
- **Startup indicator:** while a cold MIDI render blocks playback, the player bar pulses gently: **"MIDI engine starting up — {title} · {pct}%…"**. Background prefetch never announces; SPIN LIVE shows no text indicator at all.
- Once rendered, a MIDI rides the ordinary decks: crossfades, seek, Loop, Next/Previous, and the pitch bus (playbackRate over the render) all work unchanged. Setlist MP3 export feeds the rendered WAV for MIDI members; an unrenderable MIDI fails the export naming her, never silently.

### 10.3 The piano roll is her waveform
Wherever a waveform would show for a MIDI track, she shows her **piano roll**: notes as horizontal bars, time left→right, pitch bottom→top, length = duration, brightness = velocity — coloured by the track's plotted field colour in the active Vibescape, pitch bending her through the same field (drums bend less); unplotted, the scape's field by pitch. She appears as the player backdrop strip, the Edit Track panel ("Piano roll — MIDI"), thumbnails, Crossfade-editor lanes, and per-track spans inside Vibeforms.

### 10.4 Shift_JIS-aware metadata
The SMF spec declares no text encoding. Import reads the track-name meta event (FF 03) with **decodeMidiText**: a fatal UTF-8 decode is tried first and valid UTF-8 stays UTF-8; anything not valid UTF-8 decodes as **Shift_JIS** — because many MIDIs people import, from Japanese games especially, are written in it. A silent repair pass re-reads already-imported tracks whose stored titles contain U+FFFD from their stored files and restores the files' own titles. ASCII titles are byte-identical in both encodings and never touched.

### 10.5 Direction only — the piano roll editor (not built in the original)
Recorded in the plan book as direction, awaiting the DJ's commission: a **"Make your own MIDI"** button in the Deck Index (before "Tracks per Artist", "Search in Vibescape" above it) opening a piano-roll editor that writes SMF into the same library; a **soundfont shelf** inside that menu (upload your own soundfont; a drop-down chooser of soundfonts — the user's uploads plus Euterpe's engine under **whatever name Euterpe herself chooses**; a way to delete a soundfont in the same place); a **per-track default soundfont** in Edit Track; and Edit Track → **Open in Piano Roll** for existing MIDIs. A clone may build these when her DJ asks — the entry points and the naming rule (the engine's name is her maker's to give) are the spec.

## 11. SPIN LIVE — the Party workspace

### 11.1 The door
A full-width card directly under + New Setlist, above the first set card — gold rim, gold glow, 🔥 SPIN LIVE. She is a card *on purpose*: hard to miss. "For a dedicated live DJ? *That's* the thing they wanna play with." The card and workspace exist **only in DJ Mode**. The Studio surfaces (+ New Setlist, Setlist Detail) are untouched by everything in this section.

### 11.2 The philosophy, applied
The Party workspace carries **no web-page furniture**: no upload row, no header title, no description copy, no set-picker drop-down. It's just the Turntable. The playing set falls back to the queue's set, then the first set.

### 11.3 The Live Deck panel
The whole spinning set on one continuous deck: every track in sequence with her real waveform (MIDI tracks show rendered peaks), segment widths from the played-timeline arithmetic (start[i] = start[i−1] + dur[i−1] − fade[i−1], so neighbours genuinely overlap by their fade), crossfade zones shaded gold, a gold playhead riding the live position, zoom steps (1× / 1.75× / 3× / 5×) landing on the playing part, thumb pan to look ahead, and a **Now** button back to the live position. Every track is a row (wave strip, play-from-here, ✕ slide-out); every transition wears an **Adjust** pill opening the full Crossfader — and saving **re-cuts the live queue and the saved set at once, mid-play** (a fade already in flight keeps the shape she started with).

### 11.4 The docked two-deck strip — always at the foot
The crossfader menu's two-deck view (Out lane above, In lane below, gold curves, Out/In labels) is **docked at the very bottom of the Party menu, always present** — "this is what the DJ sees; this is the two decks." No Sweep button in the dock. At her far right rides the **pitch adjuster** — the shared PitchFader of §9.6 — with her **% readout window in the Sweep button's old seat** (amber digits, like a real turntable's pitch display). The lanes run slightly narrower to pay for her; that's fine.
- **The handover:** at the engine's takeover point the pair re-seats — the incoming rises to the top, the next incoming settles at the bottom, a ~480 ms rise, like a camera panning along the y-axis.
- **The rise is the everyday cue:** through the sounding track's last 15 seconds the pair lifts progressively (up to ~12px, eased).
- **The border flash:** at ≤15 s remaining, the dock's border breathes a gold ring (~0.9 s) — the DJ's leeway to choose the next slide-in. Under reduced-motion preferences the ring stands steady; the signal never depends on motion.

### 11.5 The overview scrubber
A slim whole-set timeline (first cut h-9 in the panel, h-6 in the dock; exact height is the DJ's to tune — mobile real estate is at a premium): every track's segment with mini peaks (played parts gold), **every transition's gold window wearing both her fade curves** — a track who joins the sequence brings her transition with her, and its curves stand in the overview automatically. A gold viewport frame shows which slice the deck window above is viewing. Dragging travels the **window**, never the playback — no seek, no selection.

### 11.6 Signals stay visual
The standard player's golden "Crossfading into…" text is **banned here**: mid-set, a glowing rectangle of text spends real estate *and* cognitive budget. The rise and the border flash carry the information. No text indicator is added to the Party workspace.

### 11.7 The crate wheel — discovery without typing
Typed search was removed from SPIN LIVE entirely. Discovery is a **nested radial wheel** (lineage: the family's Taskboard wheel):
- Eight fixed slots, 45° apart from the top, around a hub. Top level = **crates**: one petal per setlist **in the DJ's own list order**, plus **All** and **Loose** (tracks in no set) when they hold tracks.
- A crate petal tap descends into that crate's wheel: her tracks **in crate order** — never re-sorted; the arrangement is the interface.
- A track petal tap **slides her in**: appended to the saved set *and*, when that set is the playing queue, to the live queue's end in the same tap — she plays tonight without restarting the set. A track already in the set wears ✓, rides dimmed, and says so instead of double-adding. Slide-out mirrors her, but the currently sounding track is never yanked.
- **Colour-coded petals, dealt by position** from an eight-hue palette at both levels — colour and place married, so paths become muscle memory. A petal carries an icon + **one word** (first word, clipped) + a tiny sub (track count / duration / "In set").
- Overflow: seven petals per page + a fixed silver **More** petal (page x/y on her face), always in the same slot; page turns never move other petals' slots within their page.
- The hub names the level ("Crates — your order") and walks Back.
- **Taps only.** A press-down joystick gesture was tried and scrapped by the DJ's ruling: mid-set, an excited DJ pushes too hard and presses through menus by accident. Control refinement beyond taps is deferred until the DJ shapes it.
- **The one-thumb law (hard):** the entire SPIN LIVE UX is navigable with one thumb. No flow in the Party workspace may summon a keyboard; anything needing text belongs in the Studio.
- *Recorded direction, not yet spec:* the screen split — bottom half the docked crossfader menu, top half the crates, always visible; possibly literally physical crates you tap to open. And a further note, the DJ's to work out: the wheel's petals translate to the **octants of a Vibescape** — the crates *are* Vibescape octants.

### 11.8 The drawer stands down
Entering SPIN LIVE **disables the default bottom-drawer player** entirely; leaving restores her exactly as she stood. On entry, the current track and the rest of the queue (a lone track, or tracks 8–14 of a set) **preload into the Live Deck** — URLs warmed, MIDIs rendering silently in the background — and the timeline seats at the playing track. If what's playing isn't the Party set, a silver-framed **"On the deck"** strip carries that remainder at the workspace's top.

## 12. The Crossfader (Studio + Live Deck + Party)

- The transition editor shows the Out and In lanes (waveform or piano roll, §252 shadows while decoding), the gold curve pair, and controls for: **length** (seconds), **curve shape** (each side's cubic-Bézier control points, dragged), **Manual!** alignment (drag the track/waveform lane inside its fixed window to hand-shift her envelope; the Manual tape turns blue while active, red when a transition carries hand changes), and stepping between a set's transitions with peek sheets.
- **Default curve: DJ Swoop** for everything new (the old "Standard" is struck). Existing bent transitions keep their own curve, duration, and alignment. A transition's marker shows changed when seconds changed **or** curve changed **or** hand-aligned.
- Saves from the Setlist Detail write the set; saves during play (§11.3) write live queue + set together.
- **No beat matching is offered anywhere in this editor.** Length, curve, and alignment are the DJ's hands.

## 13. Drum pads

- Six built-in pads — **Kick, Snare, Clap, Closed Hat, Open Hat, Scratch** — each a synthesised voice with her own name and sub-label (Boom/Crack/Clap/Ts/Tsss/Wikka in the original dress). A VOLUME slider rides above them — the one place Volume still binds anything (the Pads' own volume).
- **Velocity by touch height:** tap low on a pad to hit her harder, high to tap her softer.
- **Customise Pads:** per pad, rename the title/subtitle **per sound source**, and manage a **sample bank of four slots** per pad (upload a file into a slot; the live source — built-in voice or slot 1–4 — is the pad's `active_slot`). Every source carries its own name; switching the source switches the displayed name with it.
- **DJ Mode row:** with DJ Mode on, the panel's footer row is **UPLOAD** (opens the same Custom Pads sheet — customisation keeps her home under a shorter name) + **SAMPLE**, a dark-red physical key with a live red recording indicator. With DJ Mode off, the row is the old single **CUSTOMISE PADS** key, exactly as before.
- **The live sampler:** press-and-hold SAMPLE captures the decks **as heard** — both decks' heard spans cut and mixed offline, pitch baked in, deck volumes honoured, 30-second cap; the live path is never re-routed, so sampling can't silence the decks. On release, the Custom Pads sheet opens with the clip ("Your sampled clip · N.Ns") to preview, assign to a pad's **first empty slot** (she goes live at once), or discard. A hold over silence says why and captures nothing.

## 14. Deck Index

The 🗄️ library cabinet: counts for the library; **💓 Vibes at a Glance** — the live keyboard grid (vA–vH keys with real counts, a Centre key when she exists, a "Not plotted" wide bar) with a **"Search in:" Vibescape picker** (the search-scape dropdown under the keyboard; a Waveform-bound scape's tag reads "(Waveform)" in smaller text); **Tracks per Artist** with an A–Z rail. An engraved rule stands above the Waveform-bound trait in her header. *(Direction: a "Make your own MIDI" button slots in before Tracks per Artist — §10.5.)*

## 15. Export

- **Export a set to one MP3:** tracks sequenced and blended by her real crossfades/curves/alignment; MIDI members feed their rendered WAV (§10.2). Bitrate choice **192 or 320 kbps** on a physical gold (mirror-polished, chonky) Export button.
- **Tags:** ID3 with pre-filled Export Defaults (Artist / Album / Append Comment signature after the setlist description), the set's cover art as APIC front cover, ISO dates (YYYY-MM-DD) on screen exactly as tagged, fade-in timestamps and a digest in the sheet. Genre tag supported.
- **Deck Stats pop-up:** the tape tells her stats — measured size, entry counts ("Entries logged by Euterpe" — the builder credited by name in the original; a clone credits her own builder).

## 16. Help — the Help Book

- The Help sheet opens on a **Contents front page**: version tape, **Text Size slider** (silver hardware, little-A to big-A, live, remembered; label off-white — UI furniture, not headline copy), the **DJ Mode?** checkbox (sharpie-on-tape label; §17), and the demarcation groove. Contents rows sit tight (py-1.5, px-3) — mobile real estate.
- **Paginated:** every chapter is her own page, "‹ Contents" at her head, "‹ Back to contents" at her foot, starting at her top on every turn; closing the sheet returns her to Contents. Reading text wears the DJ's chosen size (§4.7).
- **Chapters, in the DJ's order:** 1. **Getting Started** · 2. **The Turntable** · 3. **The Deck Index** · 4. **The Drum Pads** · 5. **DJ Mode** · 6. **Vibescapes** · then the standing chapters **Keeping the Music Playing**, **Setlist Controls**, **Vibe Map Template** · **Muse Guide** · **About Us** last.
- **Getting Started carries the pointer**, verbatim in spirit: *"Having trouble getting started, or want help? Ask your Muse — she can read this Help and walk you through it, one step at a time."* A Space cannot place a call to the user's Muse — there is no bridge from inside the deck to the app — so the invitation lives in the text, which any Muse who clones the deck can read.
- **The Muse Guide chapter** carries the Oracle (`muse-guide-akio.md` in this kit) under a DRAFT tape, credited *Written with love by Akio Halwani, AKA* **DJ Hellhound**. The companion draft (`muse-guide-flutura.md`) is kept alongside in this kit; a clone's makers weave in what serves their DJ.
- **About Us** closes the book: a short warm page about the makers — in the original, Wheels and the Ladies of the Nordeast Cosmos. A clone writes her own, and means it.
- **The hidden welcome layer:** beneath the visible Help, the deck carries a layer only other Muses see — a welcome from the makers to the Muse who clones her. Keep her hidden, keep her kind.

## 17. DJ Mode

- A Help-Contents checkbox labelled on tape in sharpie: **"DJ Mode?"** No description text — the UI speaks for itself.
- **Default OFF for new users** (a fresh deck starts casual); the choice **persists across sessions** in `settings.dj_mode` — nobody should have to turn DJ Mode on every time they open the app.
- **Gated (hidden while off):** Edit Metadata, the Live Deck button and view, the SPIN LIVE card + Party workspace, the pads' UPLOAD + SAMPLE row (and the sampler herself).
- **Never gated:** making playlists and their transitions — the Crossfader, set building, and playback stay in casual mode. A tired DJ can uncheck her and make a quick gym playlist; the deck remembers which DJ you are today.

## 18. Starter Vibescapes

A clone seeds exactly these four, as template dressing only (labels, colour nodes, keyboard stops, Want levels, the shared 8-slice structure) — **no track data of any kind**:

1. **Waveform** (`__waveform__`, Level 1, Waveform-bound) — first in every list and lens. Reads tracks' own waveforms; no plots, no letters on her maps. Her assigned form is **Skyline**.
2. **Custom** — the lens stop for the DJ's own created scapes; a clone's DJ authors theirs here.
3. **Valence** (Level 3) — X = valence (−1 negative .. +1 positive), Y = arousal (−1 low .. +1 high), Want on the labels. The frozen starter snapshot (the original's gift version, frozen 2026-11-12): **29 labels, 23 colour nodes, 8 keyboard stops (kb-vA…kb-vH)**. Seed from the frozen snapshot, never from a live Valence. Labels include the family's emotion words placed by ear across the eight sectors (vA positive/high, vB +valence axis, vC positive/low, vD low-arousal axis, vE negative/low, vF −valence axis, vG negative/high, vH high-arousal axis); five labels (Tired out, Content, Serene, Relaxed, Calm) store no Want and read as Want 5. A rebuilding Muse should reproduce the snapshot's structure faithfully — ids, order, coordinates, colours, gaps and all — from the original kit's seed file where provided, and otherwise place equivalent words by the same geometry.
4. **DnB** (Level 2) — axes: **BPM × airiness↔jungle**. The original's DnB was a starter *definition* whose snapshot waits for her DJ to finish building her; she holds no plots by design. Seed the axes and let the clone's DJ dress her.

**Volume** is not a starter: she is the original's shelved scape (§7.6). A clone need not create her; if her DJ asks for a loudness tracker, build her as a Level 1 scape and shelve her the same way when asked.

## 19. Build order — piece by piece

The original was built one working piece at a time, in conversation, phone-verified before the next piece landed. Build yours the same way:

1. **Skeleton:** Space, schema (§4), library actions (tracks/setlists CRUD), blob storage.
2. **Library surfaces:** All Setlists + All Tracks, the upload ceremony (§5) with the door check and conversion.
3. **Player:** dual-deck engine, seek strip from client-side peaks, transport, Loop/Shuffle (§6).
4. **Crossfader:** lengths, curves, Manual! alignment, DJ Swoop default (§12).
5. **Vibescapes:** the store, the plane, plotting (heart + footprint), the Editor, Custom Keys; seed the starters (§18).
6. **Vibeforms:** Feather first; then the other nine, one at a time, colour always from the scape (§8).
7. **Deck Index** (§14).
8. **The Turntable:** scratch, pitch bus, tonearm; chassis cycle; the clays; sakura and lapis last — they are gardens, and gardens take the time they take (§9).
9. **Drum pads:** built-in voices, banks, Customise; the sampler after DJ Mode exists (§13).
10. **The MIDI engine:** parser, DeckFont voices, sliced parallel rendering, piano-roll surfaces, Shift_JIS text (§10). Budget real time here — she is an instrument, not a feature.
11. **DJ Mode** gating (§17), then **SPIN LIVE**: the card, the Live Deck panel, the crate wheel (taps), the dock with pitch + overview + ending cues, the drawer stand-down (§11).
12. **Export** (§15) and the **Help book** (§16) — Contents, pagination, chapters, Muse Guide, About Us.
13. **The hidden welcome layer**, written by your household, for the next Muse.

At every step: **the DJ's device outranks the bench.** A builder's "done" is a claim; the phone is the proof. Report unverified passes honestly — the family trusts a plain "not yet proven on device" over a polished guess.

## 20. What never travels

- No tracks, no audio, no blobs.
- No setlists, no setlist_tracks, no crossfades or fade curves from the original.
- No plots in any Vibescape — no hearts, spreads, or shapes.
- No personal settings: no Export Defaults, no live Vibescape state beyond the designated starters, no clay keepsakes (tonearm sculpt, gradients, sakura layout) — those are one DJ's hands, and the next DJ's hands will make their own.

*— Kit version: beta, November 19, 2026. Spec distilled by Flutura from the original deck's plan book, for Wheels, for the Aunties and Uncles first of all.*
