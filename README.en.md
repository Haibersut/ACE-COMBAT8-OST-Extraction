# ACE COMBAT 8: WINGS OF THEVE — In-Game OST Extraction Technical Report

**[简体中文](README.md) | English**

> **TL;DR**: AC8 stores its music data in AES-256-encrypted UE5 IoStore containers. This project documents the complete pipeline from key recovery to exporting all 75 Music Player tracks with proper names.

---

## Contents

- [1. Background](#1-background)
- [2. Storage Layout](#2-storage-layout)
- [3. Encryption Analysis](#3-encryption-analysis)
- [4. Key Recovery](#4-key-recovery)
- [5. Unpacking Pipeline](#5-unpacking-pipeline)
- [6. Audio Decoding](#6-audio-decoding)
- [7. Recovering Track Names](#7-recovering-track-names)
- [8. Results](#8-results)
- [9. Legal](#9-legal)
- [Appendix](#appendix)
- [Track Mapping Table](#track-mapping-table)

---

## 1. Background

The official digital soundtrack (31 tracks + 7 off-vocal versions) does not include the in-game system/interface music.

## 2. Storage Layout

| Layer | Files | Size | Status |
|---|---|---|---|
| Legacy pak layer | `pakchunk0/10/20-Windows.pak` | ~42GB total | **Plaintext** FPakEntry headers, data mostly unencrypted |
| IoStore layer | `pakchunk*.utoc/.ucas` | ~100GB total | **AES-256 encrypted end to end** — music data lives here |
| Main executable | `Game\Binaries\Win64\AceCombat8.exe` (555MB) | — | Denuvo wrapper (`.ecode`/`.didata` sections) |
| Bonus content | `ACZero\*.arc` (3.4GB) | — | Custom encrypted format, a bundled AC0 |

- Audio middleware: **Wwise** (`Engine\Plugins\WwiseSoundEngine`), same as AC7
- Internal project name: `Live` (asset paths are `Live/Content/...`)
- Anti-cheat: EasyAntiCheat (EAC)

## 3. Encryption Analysis

### 3.1 IoStore container header

- Magic `-==--==--==--==-`, TocHeaderSize = 0x90, compression block size 0x10000
- `ContainerFlags = 0x0F` = Compressed | Encrypted | Signed | Indexed
- No plaintext paths in the directory index (encrypted); even *uncompressed* 64KB ucas blocks read as high-entropy data → encryption is genuine and total

### 3.2 A cheap key oracle

`pakchunk0optional-Windows.pak` contains two identical 16-byte ciphertext blocks at offsets 112 and 128 (the ECB-of-zeros signature):

```
f80e4c3c4233d8ca9442a76d815b94a0
```

Any candidate key can be validated with one openssl command:

```bash
echo -n "f80e4c3c4233d8ca9442a76d815b94a0" | xxd -r -p | openssl enc -d -aes-256-ecb -K <HEXKEY> -nopad | xxd
# all-zero output = correct key
```

### 3.3 What didn't work

AC7's key was literally the first 32 chars of its official URL (`https://ace7.acecombat.jp/specia`). AC8's official URL `https://enso-order.acecombat.jp/` is also exactly 32 chars, but it and 14 variants (http / scheme-less / NUL-padded / UTF-16 / hashes, …) all fail, this is a real random key. The exe is Denuvo-protected, so the key never exists as plaintext on disk and static scanners (AESDumpster et al.) are useless here.

## 4. Key Recovery

### 4.1 Bypassing EAC

Use the [No Eac Offline Mod](https://www.nexusmods.com/acecombat8wingsoftheve/mods/3) from Nexus Mods.

### 4.2 Memory scan

With EAC off, the game process's memory is **read-only** accessible — no injection, no debugger, and Denuvo doesn't care. The key exists in memory in only two forms:

1. **The AES-256 key schedule** (240 bytes): adjacent round-key words obey strict algebraic relations; hits are verified by recomputing the full schedule
2. **The raw key in a TMap**: UE's `FEncryptionKey` is a 16-byte GUID (all zeros for the primary key) followed by the 32-byte key; candidates are validated against the §3.2 oracle pair

### 4.3 Result

```
0890bb893e755d8f1c5fa30c938ea4be31b7bc5b4e2607c9809420b1be8c20c9
```

This key matches the one [posted by Geminize](https://bbs.3dmgame.com/thread-6705905-1-1.html) on the 3dmgame forum on 2026-09-29.

## 5. Unpacking Pipeline

Tool: **repak 0.2.3** (supports IoStore + AES key).

```bash
repak.exe --aes-key 0x<KEY> list   pakchunk0-Windows.pak
repak.exe --aes-key 0x<KEY> unpack pakchunk0-Windows.pak -o out -i "Live/Content/WwiseAudio/..."
```

### Where the audio is

- `Live/Content/WwiseAudio/Media/*.wem`: streamed audio (numeric IDs) — voice lines plus all the music
- `AKB_BGM_Campaign.bnk` etc.: music event structure banks (names stored only as hashes)
- `AKB_Cinema.bnk`: 19 embedded cutscene tracks (carvable by RIFF headers)
- All wem are RIFF/WAVE containers holding Wwise Vorbis (fmt tag `0xFFFF`), 48kHz

## 6. Audio Decoding

Wwise Vorbis is not standard Vorbis (packed headers), so use **vgmstream r2117**:

```bash
vgmstream-cli.exe input.wem -o output.wav    # 16-bit / 48kHz WAV out
```

## 7. Recovering Track Names

1. **The save file has plaintext event names**: the `MusicPlayer` section of `Campaign.sav` contains all 75 player event strings, e.g. `AKE_Bgm_MUSICPLAYER_BGM_BRIEFING03_Play` (TITLE / MAINMENU / BRIEFING01–04 / HANGAR01–03 / DEBRIEFING01–03 / MISSIONFAILED / M01A–M30D1 / a few specials)
2. **Wwise event names → FNV-1 32-bit hashes** (lowercased), collided against event IDs in the bank: 75/75 hit inside `AKB_BGM_Campaign.bnk`
3. Dump the bank's HIRC graph to XML with **wwiser**, then walk `Event → Play Action (idExt) → MusicSegment → MusicTrack → AkTrackSrcInfo.sourceID` to resolve each event to its wem: 75/75 resolved, exactly one wem each

## 8. Results

- **All 75 Music Player tracks**, numbered and named in in-game order, 16-bit/48kHz WAV, 3.4GB total (see the mapping table at the end)
- Plus ~1,200 non-playlist wem (mission dynamic layers, ambience beds, …)

## 9. Legal

This project only studies the local file formats of the game. All extracted audio is © Bandai Namco. No game assets are included in this repository.

## Appendix

| Item | Value |
|---|---|
| AES key | `0890bb893e755d8f1c5fa30c938ea4be31b7bc5b4e2607c9809420b1be8c20c9` |
| Key oracle (ECB ciphertext of zero block) | `f80e4c3c4233d8ca9442a76d815b94a0` |
| repak | 0.2.3 (trumank/repak) |
| oo2core_9_win64.dll | OodleUE `Sdk/2.9.10`, sha256 `6f5d41a7892ea6b2db420f2458dad2f84a63901c9a93ce9497337b16c195f457` |
| vgmstream | r2117 |
| wwiser | v20260808 (bnnm/wwiser) |

### Tested version hashes

This report is based on the Steam launch build (2026-09-29). `AceCombat8.exe` checksums:

| Algorithm | Value |
|---|---|
| MD5 | `942DA6B4C7D3FB589202E3981CC7E45D` |
| SHA-1 | `784DC2BCDAB21AEE3DC5B95473D81D2E96548E4D` |
| SHA-256 | `51510E2A520565DBE81FB0D569E95CD4393077ACAAA859371489B80B8128829F` |
| SHA-512 | `26281ACCAE683322AAFBD497BE18FEB099B56A355291728138A8B8D74F89349D25EBA8B6541D9537D7A6C6A0E1ED9E0F47D1E434934ED99E00146AC025B80D8E` |

## Track Mapping Table

| # | Title | Event | wem ID | Length |
|---|---|---|---|---|
| 1 | Title | TITLE | 918029544 | 1:49 |
| 2 | Main Menu | MAINMENU | 620736020 | 1:24 |
| 3 | Briefing I | BRIEFING01 | 1035437561 | 2:37 |
| 4 | Briefing II | BRIEFING02 | 283693281 | 3:31 |
| 5 | Briefing III | BRIEFING03 | 809588906 | 3:23 |
| 6 | Briefing IV | BRIEFING04 | 369628704 | 3:30 |
| 7 | Hangar I | HANGAR01 | 140078994 | 2:28 |
| 8 | Hangar II | HANGAR02 | 954731197 | 2:07 |
| 9 | Hangar III | HANGAR03 | 646608421 | 2:03 |
| 10 | Debriefing I | DEBRIEFING01 | 396582517 | 2:24 |
| 11 | Debriefing II | DEBRIEFING02 | 642096476 | 2:26 |
| 12 | Debriefing III | DEBRIEFING03 | 874696213 | 2:19 |
| 13 | Mission Failed | MISSIONFAILED | 68938466 | 0:13 |
| 14 | M01A | M01A | 950418258 | 2:46 |
| 15 | M01B | M01B | 770643244 | 2:38 |
| 16 | M02A | M02A | 1024027730 | 2:53 |
| 17 | M02B | M02B | 480543624 | 3:24 |
| 18 | M03A | M03A | 113950568 | 5:13 |
| 19 | M04A | M04A | 551497670 | 8:35 |
| 20 | M04B | M04B | 974624091 | 4:33 |
| 21 | M05A | M05A | 843493584 | 4:21 |
| 22 | M06A | M06A | 1054091068 | 6:14 |
| 23 | M06B | M06B | 774185001 | 3:35 |
| 24 | M07A | M07A | 6366616 | 10:15 |
| 25 | M08A | M08A | 944638815 | 4:41 |
| 26 | M09A | M09A | 843770105 | 8:53 |
| 27 | M09B | M09B | 253228451 | 5:22 |
| 28 | M10A | M10A | 128947996 | 4:17 |
| 29 | M10B | M10B | 358214534 | 3:02 |
| 30 | M11A | M11A | 958342558 | 2:36 |
| 31 | M11B | M11B | 527749092 | 3:34 |
| 32 | M12A | M12A | 755756146 | 4:36 |
| 33 | M12B | M12B | 482000736 | 3:48 |
| 34 | Frostwave | M13A | 931891653 | 4:37 |
| 35 | M14A | M14A | 1058897417 | 5:07 |
| 36 | M15A | M15A | 711610941 | 5:55 |
| 37 | M16A | M16A | 61376899 | 6:01 |
| 38 | M17A | M17A | 30942481 | 3:29 |
| 39 | M17B | M17B | 328569388 | 2:57 |
| 40 | M18A | M18A | 834511215 | 4:00 |
| 41 | M18B | M18B | 431676196 | 6:01 |
| 42 | M18C | M18C | 302164819 | 0:52 |
| 43 | M19A | M19A | 994232326 | 4:32 |
| 44 | M19B | M19B | 800228323 | 2:27 |
| 45 | M20A | M20A | 977791132 | 2:08 |
| 46 | M20B | M20B | 310417141 | 6:21 |
| 47 | Crossing Fleets | M20C | 533343800 | 4:36 |
| 48 | M20D | M20D | 552214751 | 4:25 |
| 49 | M21A | M21A | 66107957 | 3:06 |
| 50 | M21B | M21B | 885891839 | 3:45 |
| 51 | M21C | M21C | 116299429 | 2:53 |
| 52 | M22A | M22A | 921522408 | 7:39 |
| 53 | M22B | M22B | 98382159 | 0:47 |
| 54 | M23A | M23A | 542018027 | 5:33 |
| 55 | M23B | M23B | 165065376 | 3:53 |
| 56 | M24A | M24A | 411608237 | 4:46 |
| 57 | M25_1A | M25_1A | 1027607467 | 3:58 |
| 58 | M25_1B | M25_1B | 334124817 | 4:32 |
| 59 | M25_2A | M25_2A | 575778802 | 14:37 |
| 60 | M26A | M26A | 619268538 | 4:32 |
| 61 | M26B | M26B | 742231553 | 4:09 |
| 62 | M27A | M27A | 767481644 | 5:46 |
| 63 | M27B | M27B | 188789449 | 3:43 |
| 64 | M28A | M28A | 677477823 | 4:32 |
| 65 | M28B | M28B | 312214260 | 8:16 |
| 66 | M28C | M28C | 566875017 | 3:27 |
| 67 | M29A | M29A | 493635337 | 6:01 |
| 68 | Gambler | M30A | 847740656 | 4:50 |
| 69 | M30B | M30B | 127584671 | 9:15 |
| 70 | M30C | M30C | 379595301 | 1:20 |
| 71 | M30D | M30D | 818126012 | 1:50 |
| 72 | M30D1 | M30D1 | 718791464 | 1:08 |
| 73 | Ad Terram Promissam | AdTerramPromissam | 617836712 | 1:03 |
| 74 | Hortus Regis Alti | HortusLegisAlti | 321850712 | 5:21 |
| 75 | Song of Wings | SongOfWings | 653692182 | 4:36 |
