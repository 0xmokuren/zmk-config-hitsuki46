# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 概要

46キー分割キーボード **hitsuki46**（両手トラックボール搭載）用の ZMK ファームウェア設定。ボードは両側とも `xiao_ble/nrf52840/zmk`（Seeed XIAO BLE / nRF52840）、Zephyr 4.1 上の ZMK firmware を使用する。ハードウェア定義（シールド）とキーマップのみを持つコンフィグリポジトリで、ファームウェア本体はビルド時に `west` が外部から取得する。

## ビルドとコマンド

ビルドは基本的に GitHub Actions（`.github/workflows/build.yml`、ZMK 公式の再利用ワークフロー）で行い、push すると `.uf2` が Artifacts に出力される。`build.yaml` のマトリクスが**3つのターゲット**を定義する:
- `hitsuki46_R` — 右手（セントラル）。`studio-rpc-usb-uart` snippet 付き
- `hitsuki46_L` — 左手（ペリフェラル）
- `settings_reset` — BT ペアリング等を消去するユーティリティ

ローカルビルド（ZMK 開発環境構築済みが前提）:
```bash
west init -l config
west update                       # west.yml の依存モジュールを取得
# 右手（セントラル）
west build -s zmk/app -b xiao_ble -- -DSHIELD=hitsuki46_R -DZMK_CONFIG="$(pwd)/config"
# 左手（ペリフェラル）
west build -s zmk/app -b xiao_ble -- -DSHIELD=hitsuki46_L -DZMK_CONFIG="$(pwd)/config"
```

キーマップ図（`keymap-drawer/hitsuki46.svg`）は `.github/workflows/draw.yml` を手動実行（`workflow_dispatch`）して `config/*.keymap` + `config/hitsuki46.json` から再生成する。

## アーキテクチャ

### スプリットの役割はシールドで固定

両側とも同じボード・同じ `config/hitsuki46.keymap` を共有するが、左右の役割は **`Kconfig.defconfig`** で決まる:
- `hitsuki46_R` → `ZMK_SPLIT_ROLE_CENTRAL=y`（USB 接続・キーマップ実行・両トラックボールの統合先）
- `hitsuki46_L` → ペリフェラル（キー入力と左トラックボールをセントラルへ転送）

### Devicetree の階層（複数ファイルにまたがる）

`hitsuki46.dtsi`（共有）を `hitsuki46_L.overlay` と `hitsuki46_R.overlay` が `#include` する構成。
- **`hitsuki46.dtsi`**: physical-layout、`default_transform`（matrix-transform）、`kscan0`、SPI0/SPI1 ノード、`led_power`（WS2812 用 P-ch MOSFET 電源）、`non_lipo_battery`、NFC ピンの GPIO 化など全体共通部分。
- **`*_R.overlay` / `*_L.overlay`**: `col-gpios`、`pinctrl`、トラックボール実体ノード、input-listener などの左右差分。R 側だけ `default_transform` に `col-offset = <6>` を足し、右半分のキーを列 6–11 に割り当てる。
- 設定（Kconfig）は **`*_R.conf` / `*_L.conf`** に分かれる。CPI や invert 等はデバイスツリー（overlay）、PMW3610 の電源タイミングやバッテリ閾値・LED ウィジェットは `.conf` 側、と置き場所が分かれている点に注意。

### トラックボールのデータフロー

3ファイルにまたがるので全体像を把握すること:
1. 左トラックボールは物理的にペリフェラル L 上。`*_L.overlay` の `trackball_left`（SPI0）→ `trackball_left_split`（`zmk,input-split`、`device = <&trackball_left>`）が BLE スプリット経由でセントラルへ転送。
2. セントラル R 側（`*_R.overlay`）で `trackball_left_listener` が `&trackball_left_split` を、`trackball_right_listener` がローカルの `&trackball`（R 上の SPI0）を購読する。
3. **両リスナーとモード分岐はすべて `hitsuki46_R.overlay` に集約**されている。

### レイヤー構成（`config/hitsuki46.keymap`、ノード順 = インデックス）

`0 default` / `1 MOUSE` / `2 SYMBOL` / `3 NUM_AND_FUC` / `4 ARROW` / `5 SCROLL` / `6 Bluetooth`。
Layer 1 はカーソルモード時にトラックボール操作で自動遷移する auto-mouse レイヤー、Layer 5 はトラックボールをスクロール化するレイヤー。コンボ・マクロ（`to_layer_0`）・hold-tap behavior もこのファイルに定義。

## 重要な注意点・落とし穴

- **トラックボールのモードはコンパイル時 `#define`**: `hitsuki46_R.overlay` 冒頭の `LEFT_TRACKBALL_MODE` / `RIGHT_TRACKBALL_MODE`（`0`=カーソル, `1`=スクロール）。変更には再ビルドが必要。モードで input-processors のチェーン（`xy_clipper`、auto-mouse の `zip_temp_layer 1 700`、scroll mapper/scaler 等）が切り替わる。

- **左側（ペリフェラル）の LED は firmware で無効化済み（load-bearing、触らないこと）**: SPI1 の P0.19（WS2812 DIN）へのノイズ結合で緑に誤点灯する問題があり、`*_L.overlay` の `&led_power` で P-ch MOSFET を OFF 維持＝**LED を完全に無給電**にして回避している（R 側 `ACTIVE_LOW` に対し L 側だけ `ACTIVE_HIGH`）。`*_L.conf` の WS2812 ウィジェット設定（全色 `0x000000`）は**ビルドを通すためだけ**に残してある（`SHOW_*=n` にすると `widget.c` が `LAYER_x_COLOR` を無条件参照しビルドエラー）。**重要**: 「ウィジェットが周期的に黒で上書きして自己修復する」という挙動は**存在しない**（widget は event 駆動で `k_msgq_get(K_FOREVER)` でブロック、色設定パスは peripheral では `CONFIG_ZMK_SPLIT_ROLE_CENTRAL` ガードで除外され `current_color` は `{0,0,0}` 固定）。**左 LED を正しく光らせるのは firmware だけでは不可能**で、DIN への外付け ~10k プルダウン（ハード改修）が必須。安易に「整理」せず、背景はメモリ `project_left_led_noise.md` 参照。

- **PMW3610 の電源チューニングは `.conf` 側**: REST モードの sample/downshift 時間でカーソルの引っかかり・ドリフトを調整している。`force-awake` は R 側トラックボールのみ（L 側はアイドル時ドリフト対策で外した）。背景はメモリ `project_trackball_idle_lag.md` / `project_trackball_cursor_jump.md`。

- **バッテリは Ni-MH（非 LiPo）**: `vbatt` を無効化し `non_lipo_battery`（ADC ch0）を使用。`.conf` の `CONFIG_ZMK_NON_LIPO_*_MV` は**分圧後の ADC 電圧（×0.32）**であって電池電圧そのものではない。

- **ZMK Studio は現在無効**: `hitsuki46_R.conf` の `CONFIG_ZMK_STUDIO` はコメントアウト済み（Zephyr 4.1 + Studio で両側ハングする問題のため、commit 817b5cf）。`build.yaml` には `studio-rpc-usb-uart` snippet が残るが Studio は機能しない。README.md / GEMINI.md は「Studio 対応」と記載が残っているため、**実態は `.conf` を正とする**。

- **依存モジュールは `main` 追従**: `config/west.yml` の外部5モジュール（`zmk-pmw3610-driver`/badjeff, `zmk-ws2812-driver`/gohanda11, `zmk-feature-non-lipo-battery-management`/sekigon-gonnoc, `zmk-feature-xy_clipper`/iwk7273, および zmk 本体）はすべて `revision: main`。上流の変更でビルドが壊れ得る。

- **キー安定性の調整**: kscan に debounce（`debounce-press-ms`/`release-ms`）を入れて Ctrl+Space の二重入力を防いでいる。`&mt` / `&lt` の flavor・quick-tap も入力切替の信頼性のため調整済み。

## ドキュメント

`README.md`（日本語・機能/配線/レイヤーの詳細）、`GEMINI.md`（プロジェクト概要）が既存。本ファイルと内容が食い違う場合は、ハードウェア設定については各 `.conf` / `.overlay` の実コードを優先する。
