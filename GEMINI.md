# hitsuki46 ZMK Configuration

このプロジェクトは、46キーの分割キーボード **hitsuki46** 用の ZMK ファームウェア設定リポジトリです。

## プロジェクト概要

hitsuki46 は、左右の両方にトラックボールを搭載可能な設計を持つ分割キーボードです。
このリポジトリには、ハードウェア定義（シールド）、キーマップ設定、およびビルド自動化のための GitHub Actions 設定が含まれています。

### 主な技術スタック
- **ファームウェア**: [ZMK Firmware](https://github.com/zmkfirmware/zmk) (Zephyr 4.1)
- **ハードウェア**:
  - 46キー分割レイアウト (左右各23キー)
  - トラックボールセンサー: PixArt PMW3610
  - ステータス表示: WS2812 RGB LED
  - 電源: ニッケル水素電池対応（カスタム電圧管理）
- **主要外部モジュール**:
  - `zmk-pmw3610-driver`: トラックボールドライバー
  - `zmk-ws2812-driver`: LED ステータス表示
  - `zmk-feature-non-lipo-battery-management`: 非 LiPo バッテリー管理
  - `zmk-feature-xy_clipper`: 斜め入力防止フィルター

## ディレクトリ構造

- `boards/shields/hitsuki46/`: hitsuki46 用のシールド定義ファイル群。
- `config/`: メインの設定ファイル（キーマップ、外部モジュール定義）。
  - `hitsuki46.keymap`: キーマップ、コンボ、マクロの定義。
  - `west.yml`: 依存関係（ZMK 本体および外部モジュール）の管理。
- `.github/workflows/`: GitHub Actions による自動ビルド設定。
- `keymap-drawer/`: キーマップの可視化用ファイル。

## ビルドと実行

### GitHub Actions (推奨)
GitHub にプッシュすると、自動的にファームウェア（`.uf2` ファイル）がビルドされ、Artifacts としてダウンロード可能になります。

### ローカルビルド
ZMK の開発環境が構築されている前提で、以下の手順でビルド可能です。

1. **初期化**:
   ```bash
   west init -l config
   west update
   ```

2. **ビルド (左手側)**:
   ```bash
   west build -s zmk/app -b xiao_ble -- -DSHIELD=hitsuki46_L -DZMK_CONFIG="$(pwd)/config"
   ```

3. **ビルド (右手側)**:
   ```bash
   west build -s zmk/app -b xiao_ble -- -DSHIELD=hitsuki46_R -DZMK_CONFIG="$(pwd)/config"
   ```

## 開発上の注意点

- **トラックボールモード**: `boards/shields/hitsuki46/hitsuki46_R.overlay` 内の `#define` で左右のトラックボールを「カーソルモード」または「スクロールモード」に切り替え可能です。
- **LED ステータス**: ステータス LED は、アクティブなレイヤー、バッテリー残量、接続状態を色と点滅で示します。
- **バッテリー管理**: ニッケル水素電池を使用するための特殊な電圧設定が `hitsuki46.dtsi` に記述されています。
- **ZMK Studio**: ZMK Studio に対応しており、GUI からのキーマップ変更が可能です。
