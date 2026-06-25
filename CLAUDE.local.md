# Local instructions for Claude (this repo)

## 説明スタイル

このリポジトリのドメイン（ファームウェア、組み込み、無線通信、センサー、電源管理、Zephyr/ZMK 固有概念など）はユーザーが詳しくない領域なので、回答時に英語の専門用語が出てきたら**括弧で日本語の意味を添える**こと。

### 説明を入れる対象

低レベル・ハードウェア寄り・専門領域の用語:
- 通信プロトコル: BLE, SPI, I2C, UART, HID, GATT, ATT, L2CAP など
- センサー/ハードウェア概念: REST mode, IRQ, ISR, MOSFET, ADC, GPIO, pull-down, pinctrl など
- ZMK/Zephyr 固有: shield, overlay, devicetree, Kconfig, west.yml, manifest, peripheral/central, input-processor など
- 電源管理: PM (Power Management), supervision timeout, advertising, bonding, downshift など
- ドライバ実装の細部: workqueue, fork, ANTI_WARP, IGNORE_AFTER_REST など

### 説明を入れない対象

基本的な IT 用語（ユーザーは理解している）:
- 開発フロー全般: git, PR, commit, branch, merge, revert, push, fetch, rebase, conflict, diff
- インフラ全般: CI, GitHub Actions, Docker, image, artifact, build, flash
- プログラミング基本: function, variable, config, file, directory, environment variable
- ネットワーク基本: HTTP, URL, host, client, server
- macOS/PC の基本操作: 設定, アプリ, ターミナル

### フォーマット

初出時のみ括弧で説明し、同一回答内の2回目以降は不要:

```
✅ Good: BLE (Bluetooth Low Energy / 低消費電力ブルートゥース) が不安定になります。BLE の supervision timeout (リンク維持タイムアウト) が...
❌ Bad: BLE が不安定になります。supervision timeout が...
```

短い専門用語の連発を避け、概念の関係（何が何の原因か、どこに影響するか）も平易な日本語で補足する。長文化しても構わない（ユーザーは正確な理解を優先する）。

### 構造化の好み

複数の状態や条件を比較する時は、文章ではなく**表で並べる**と理解しやすいとのフィードバックあり。例えば「設定 A」「設定 B」「結果」の3列で並べる、など。

「やる/やらない」「OK/NG」「過去/今回」のような対比軸があるときは特に有効。
