# ACE COMBAT 8: WINGS OF THEVE — 游戏内 OST 提取技术报告

**简体中文 | [English](README.en.md)**

> **TL;DR**：AC8 的音乐数据在 AES-256 加密的 UE5 IoStore 容器里。本项目记录了从拿到密钥到按曲名导出全部 75 首 Music Player 曲目的完整链路。

---

## 目录

- [1. 背景](#1-背景)
- [2. 存储结构](#2-存储结构)
- [3. 加密分析](#3-加密分析)
- [4. 密钥提取](#4-密钥提取)
- [5. 解包管线](#5-解包管线)
- [6. 音频解码](#6-音频解码)
- [7. 曲名还原](#7-曲名还原)
- [8. 结果](#8-结果)
- [9. 声明](#9-声明)
- [附录](#附录)
- [曲目对照表](#曲目对照表)

---

## 1. 背景

官方发售的数字原声带（31 首正曲 + 7 首 off-vocal）并不包含游戏内的界面/系统音乐。

## 2. 存储结构

| 层 | 文件 | 大小 | 状态 |
|---|---|---|---|
| 传统 pak 层 | `pakchunk0/10/20-Windows.pak` | 共 ~42GB | 文件头**明文**，数据基本未加密 |
| IoStore 层 | `pakchunk*.utoc/.ucas` | 共 ~100GB | **AES-256 全盘加密**，音乐数据保存在此处 |
| 主程序 | `Game\Binaries\Win64\AceCombat8.exe` (555MB) | — | Denuvo 壳（`.ecode`/`.didata` 等节区） |
| 附赠内容 | `ACZero\*.arc` (3.4GB) | — | 自有加密格式，捆绑 AC0 |

- 音频中间件：**Wwise**（`Engine\Plugins\WwiseSoundEngine`），与 AC7 相同
- 游戏内项目代号：`Live`（资源路径为 `Live/Content/...`）
- 反作弊：EasyAntiCheat（EAC）

## 3. 加密分析

### 3.1 IoStore 容器头

- Magic：`-==--==--==--==-`，TocHeaderSize = 0x90，压缩块大小 0x10000
- `ContainerFlags = 0x0F` = Compressed | Encrypted | Signed | Indexed
- 目录索引无明文路径（加密）；连 ucas 里**未压缩**的 64KB 块读出来也是高熵数据 → 全层加密实锤

### 3.2 密钥验证器

`pakchunk0optional-Windows.pak` 偏移 112 与 128 处有两个相同的 16 字节密文块（ECB 加密全零块的特征）：

```
f80e4c3c4233d8ca9442a76d815b94a0
```

任意候选 key 用它验证，一行 openssl 即可：

```bash
echo -n "f80e4c3c4233d8ca9442a76d815b94a0" | xxd -r -p | openssl enc -d -aes-256-ecb -K <HEXKEY> -nopad | xxd
# 输出全 0 = key 正确
```

### 3.3 失败的猜测

AC7 的 key 是官网 URL 前 32 字符（`https://ace7.acecombat.jp/specia`）。AC8 官网 `https://enso-order.acecombat.jp/` 正好也是 32 字符，但连同 14 个变体（http/去协议头/补零/UTF-16/哈希等）全部失败，这代是真随机密钥。且 exe 有 Denuvo，密钥在磁盘上不以明文存在，AESDumpster 类静态工具无效。

## 4. 密钥提取

### 4.1 关掉 EAC

使用 Nexus Mods 的 [No Eac Offline Mod](https://www.nexusmods.com/acecombat8wingsoftheve/mods/3)。

### 4.2 内存扫描

EAC 关闭后，游戏进程内存可被**只读**访问（无需注入/调试器，Denuvo 不影响）。key 在内存中只有两种形态：

1. **AES-256 密钥扩展表**（240 字节）：相邻轮密钥字有严格代数关系，向量化筛选后逐个全表复算验证
2. **TMap 原始 key**：UE 的 `FEncryptionKey` 结构 = 16 字节 GUID（主密钥为全零）+ 32 字节 key，用 §3.2 的明密文对逐个验证

### 4.3 结果

```
0890bb893e755d8f1c5fa30c938ea4be31b7bc5b4e2607c9809420b1be8c20c9
```

该 key 与 3dmgame 论坛用户 Geminize 于 2026-09-29 [发布的帖子](https://bbs.3dmgame.com/thread-6705905-1-1.html) 中的密钥一致。

## 5. 解包管线

工具：**repak 0.2.3**（支持 IoStore + AES key）。

```bash
repak.exe --aes-key 0x<KEY> list   pakchunk0-Windows.pak
repak.exe --aes-key 0x<KEY> unpack pakchunk0-Windows.pak -o out -i "Live/Content/WwiseAudio/..."
```

### 音频分布

- `Live/Content/WwiseAudio/Media/*.wem`：流式音频（数字 ID 命名），含语音与全部音乐
- `AKB_BGM_Campaign.bnk`等：音乐事件库（名称仅哈希存储）
- `AKB_Cinema.bnk`：内嵌 19 段过场音乐（可直接按 RIFF 头雕出）
- 全部 wem = RIFF/WAVE 容器 + Wwise Vorbis（fmt tag `0xFFFF`），48kHz

## 6. 音频解码

Wwise Vorbis 不是标准 Vorbis（打包头被压缩），直接用 **vgmstream r2117**：

```bash
vgmstream-cli.exe input.wem -o output.wav    # 输出 16bit/48kHz WAV
```

## 7. 曲名还原

1. **存档里有明文事件名**：`Campaign.sav` 的 `MusicPlayer` 段带全部 75 个播放器事件的明文字符串，形如 `AKE_Bgm_MUSICPLAYER_BGM_BRIEFING03_Play`（共 75 个：TITLE / MAINMENU / BRIEFING01–04 / HANGAR01–03 / DEBRIEFING01–03 / MISSIONFAILED / M01A–M30D1 / 若干特别曲）
2. **Wwise 事件名 → FNV-1 32 位哈希**（小写化），与 bnk 内事件 ID 碰撞：75/75 全部命中 `AKB_BGM_Campaign.bnk`
3. **wwiser** 导出 bnk 的 HIRC 结构 XML → 沿 `Event → Play Action(idExt) → MusicSegment → MusicTrack → AkTrackSrcInfo.sourceID` 图遍历，事件解析到具体 wem：75/75 成功，且一一对应

## 8. 结果

- **75 首 Music Player 曲目**，按游戏内顺序编号命名，16bit/48kHz WAV，共 3.4GB（曲名对照表见文末）
- 另有约 1,200 个未入选播放器的 wem（任务动态分层/氛围床位等）

## 9. 声明

本项目仅用于研究游戏的本地文件格式；提取的音频版权归 Bandai Namco 所有；本仓库不包含任何游戏资产。

## 附录

| 项目 | 值 |
|---|---|
| AES key | `0890bb893e755d8f1c5fa30c938ea4be31b7bc5b4e2607c9809420b1be8c20c9` |
| Key oracle（ECB 全零块密文） | `f80e4c3c4233d8ca9442a76d815b94a0` |
| repak | 0.2.3（trumank/repak） |
| oo2core_9_win64.dll | OodleUE `Sdk/2.9.10`，sha256 `6f5d41a7892ea6b2db420f2458dad2f84a63901c9a93ce9497337b16c195f457` |
| vgmstream | r2117 |
| wwiser | v20260808（bnnm/wwiser） |

### 测试版本哈希

本报告基于 Steam 首发版（2026-09-29）测试，`AceCombat8.exe` 校验值：

| 算法 | 值 |
|---|---|
| MD5 | `942DA6B4C7D3FB589202E3981CC7E45D` |
| SHA-1 | `784DC2BCDAB21AEE3DC5B95473D81D2E96548E4D` |
| SHA-256 | `51510E2A520565DBE81FB0D569E95CD4393077ACAAA859371489B80B8128829F` |
| SHA-512 | `26281ACCAE683322AAFBD497BE18FEB099B56A355291728138A8B8D74F89349D25EBA8B6541D9537D7A6C6A0E1ED9E0F47D1E434934ED99E00146AC025B80D8E` |

## 曲目对照表

| # | 曲名 | 事件名 | wem ID | 时长 |
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
