---
title: "AliExpressの格安GNSSアンテナでQZSS L6が本当に取れるか実測してみた（ラベルにL6なし・E6ありなら物理的に同じ帯域）"
emoji: "📡"
type: "tech"
topics: ["qzss", "gnss", "rust", "embedded", "rtk"]
published: true
---

## 要約

u-blox純正のL1〜L6マルチバンドGNSSアンテナ `ANN-MB2-00`（¥13,860）を、AliExpressの格安アンテナ2種（¥3,355と¥5,912）に置き換えて、QZSS L6帯（1278.75 MHz）が本当に受信できるかをArduSimple NEO-D9C Pluginで並走比較計測した。

- **3本すべてL6Dを `errStatus=ErrorFree` で実用受信**。平均CNOは41 dBHz台で有意差なし
- 格安側の現物ラベルには **「L6」と書かれていない**。だが「**E6**」と書かれている。L6とE6は **同一周波数（1278.75 MHz）** なので物理的に同じ帯域、自動的に取れる
- 8台量産で **アンテナコスト¥168k削減**、ヘリカル型はスマホ一体型筐体への収まりも純正パッチより良い

## 背景：純正アンテナの単価がきつい

みちびきL6帯（CLAS / QZNMA信号認証）を扱うための受信機の試作8台 + アンテナ2本 = 計16本をu-blox `ANN-MB2-00` で揃えると **¥221,760**。目標単価10万円/台のプロジェクトには重い。AliExpressには「L1/L2/L5/L6対応」を謳う¥3,000〜6,000帯のアンテナが多数並んでいるが、ラベルだけ広告で実体はL1のみ、という中華GNSSあるあるパターンが容易に予想できる。

そこで純正と格安2種を実機で並走比較し、L6帯が本当に取れるかをバイナリ判定することにした。

## 対象機材

- **A: u-blox ANN-MB2-00**（¥13,860） — パッチ型、active、bias-T内蔵、L1/L2/L5/L6/E6/B3/L-band公称。入手元: [ジオセンス Yahoo!店](https://store.shopping.yahoo.co.jp/geosense2/annmb200.html)
- **B: TAN1216Q50**（AliExpress、¥3,355） — ヘリカル、28 dBゲイン、IP67、商品タイトルは「L1 L2 L5 L6 ... E6」を謳う。[AliExpress商品ページ](https://ja.aliexpress.com/item/1005010177956181.html)
- **C: HX-901**（AliExpress、¥5,912） — ヘリカル、28 dBゲイン、IP67、DC 3〜16 V。[AliExpress商品ページ](https://ja.aliexpress.com/item/1005012273324463.html)

価格比はB = A × 0.24、C = A × 0.43。

## 計測構成

```
[アンテナ] → SMA → NEO-D9C Plugin → micro USB → Linux ホスト → rxm-monitor
                                                 (/dev/ttyACM0)
```

NEO-D9C Pluginはmicro USB経由でu-blox CDC ACMデバイスとして直接ホストに見える（過去記事の解説参照: [NEO-D9C で QZSS L6 raw を Linux から取り出すまでに踏んだ 3 つの罠](https://yasu-home.com/neo-d9c-qzss-l6-raw-linux-3-traps/)）。ホスト側は自前Rustの `rxm-monitor` でUBX-RXM-QZSSL6を解析し、L6D ch1/ch2の受信率とerrStatus、CNOを集計する。

ソースコード（GitHub）: https://github.com/Ryujiyasu/michibiki

```rust
// firmware/src/qzssl6.rs — v1 spec (UBX-21031777-R02) 対応のパーサ
pub fn parse(payload: &[u8]) -> Result<Self, QzssL6Error> {
    if payload.len() != UBX_RXM_QZSSL6_PAYLOAD_LEN { /* 264 byte */ }
    if payload[0] != QZSSL6_VERSION { /* 0x01 */ }
    let cno_dbhz = (u16::from_le_bytes([payload[2], payload[3]]) as f32) / 256.0;
    let ch_info = u16::from_le_bytes([payload[10], payload[11]]);
    let err_status_bits = (ch_info >> 12) & 0b11;
    // ...
}
```

計測条件は同一窓際位置・同一時間帯（約10分以内にA → B → C順次）・各64秒。差し替え直後はL6D再捕捉に十数秒かかるので、本計測前に15秒プローブで捕捉を確認してから計測している。

## 計測結果

|  | A: ANN-MB2-00 | B: TAN1216Q50 | C: HX-901 |
|---|---|---|---|
| 価格 | ¥13,860 | ¥3,355 (A×0.24) | ¥5,912 (A×0.43) |
| L6D総フレーム | 65 | 128 | 130 |
| 受信レート | 1.0 Hz (1ch) | 2.0 Hz (2ch) | 2.0 Hz (2ch) |
| 捕捉QZS | QZS-7 | QZS-2 / QZS-4 | QZS-4 / QZS-7 |
| errStatus | 100% ErrorFree | 100% ErrorFree | 100% ErrorFree |
| 平均CNO | 41.0 dBHz | 41.3 dBHz | 41.2 dBHz |

**3本とも合格**。「L6偽装」はゼロ。CNOも41 dBHz台で完全に同レンジ。むしろ格安側のB / Cのほうが L6D ch1/ch2 を2 ch同時に捕捉している（Aは計測時 ch1単独）。

捕捉SVが計測ごとに違う（A=7 / B=2,4 / C=4,7）のは、空がほぼ同一・差し替えごとの受信機側 L6Dチャンネル再割り当てによるもので、特定アンテナ依存の偏りではない（QZS-7はAとC、QZS-4はBとCで重複して見えている）。

## 「ラベルにL6と書いてないのにL6が取れる」の謎

B（TAN1216Q50）の現物ラベルを撮ったらこう書いてある:

```
GNSS L1/L2/L5 B1/B2/B3 E1/E5/E6
GNSS ANTENNA
```

**「L6」が無い**。商品タイトルでは「L1 L2 L5 **L6** G1 G2 B1-B3 E1 E5 E6 + L-band」と謳っていたが、現物ラベルは `E6` 止まり。一瞬「やはり広告詐欺か」と思うのだが、実際はL6がしっかり取れている。物理的根拠はこうだ:

- QZSS L6: 中心周波数 **1278.75 MHz**
- Galileo E6: 中心周波数 **1278.75 MHz**
- BeiDou B3: 中心周波数 1268.52 MHz（隣接帯域、共通アンテナで取れる）

つまりQZSS L6とGalileo E6は **同一周波数の同一帯域**。アンテナは物理的に共振帯域で動作するデバイスなので、E6に対応した瞬間にL6にも自動的に対応する。逆に言えば、アンテナ屋がラベルに「L6」「E6」を別に書くインセンティブはなく、Galileo仕様のE6だけ書いておけば十分（市場が広い欧州Galileo視点）。

:::message
この物理的同一性を理解していれば「ラベルにL6と書いてないアンテナ」も候補に入れられる。逆に **「L6と書いてあるアンテナだけ」** に絞ると、E6表記の安価アンテナ群を全部見落とす。日本人ユーザーが「L6表記」だけで検索してu-blox / Tallysmanの高価品にロックインされやすい構造的な罠である。
:::

## コスト試算とフォームファクタ

本プロジェクトは試作8台 × アンテナ2本（heading用 dual-antenna構成）= 16本必要。

| 選択肢 | 単価 | 16本合計 |
|---|---|---|
| A: ANN-MB2-00で全部揃える | ¥13,860 | ¥221,760 |
| B: TAN1216Q50で全部置換 | ¥3,355 | ¥53,680 |
| 差額 | — | **¥168,080削減** |

さらにフォームファクタ。Aのパッチ型は平面実装向きで、スマホ筐体に貼ると目立つ。B / Cのヘリカル型は細長く（Ø42 × H44 mm）、スマホ背面のカメラ突起の隣に並べる構成で違和感が少ない。製品化適性はB / Cが明らかに高い。

## 実機tips（再現用の落とし穴メモ）

- rxm-monitor出力に `tracing-subscriber` のANSIカラーコードが入ると `awk -F'band='` や `grep 'cno='` がマッチしない。`env NO_COLOR=1` を付けて実行する
- `Cargo.lock` がversion 4のため **rustc/cargo 1.78+ が必要**。Ubuntu 22.04標準のcargo 1.75だとビルドが失敗するので、rustup経由でstableを入れる
- アンテナ差し替え直後はL6D再捕捉に **十数秒** かかる。本計測の前に15秒プローブで捕捉を確認してから60秒計測に入る
- 窓際は時間帯次第でQZSの仰角が変わるので、A→B→Cは **10分以内に連続実行** する

## まとめ

- QZSS L6帯のアンテナ選定で「L6」ラベルだけを検索条件にすると、同じ物理帯域であるGalileo **E6** ラベルの安価アンテナ群を全部見落とす
- L6 = E6 = 1278.75 MHzの物理的同一性を理解し、E6対応のヘリカル等を候補に入れることで、アンテナコストを **1/4以下** に下げられる可能性がある
- 実機での並走比較で3本ともerrStatus=ErrorFree、CNO 41 dBHz台で有意差なしを確認
- 製品化視点でもヘリカル型はスマホ筐体への収まり良し

もちろん中華アンテナの個体差・耐久性・温度特性は別軸で検証が必要なので、本記事の結論は「単発のRF性能評価で実用域」までである。長期安定性や屋外環境での性能は別途データを積む必要がある。が、初動の試作機材選定としては「L6ラベルなし、E6ラベルあり」の格安アンテナを選択肢から外す理由は無い。

## 関連

- 前回記事: [NEO-D9C Plugin で QZSS L6 raw を Linux から取り出すまでに踏んだ 3 つの罠](https://yasu-home.com/neo-d9c-qzss-l6-raw-linux-3-traps/)
- ソースコード: [github.com/Ryujiyasu/michibiki](https://github.com/Ryujiyasu/michibiki)
- 計測プロトコル: [docs/poc-results/antenna-comparison-protocol.md](https://github.com/Ryujiyasu/michibiki/blob/main/docs/poc-results/antenna-comparison-protocol.md)
