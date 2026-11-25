# foamFO — OpenFOAM functionObject 検索・雛形生成ツール

[English](README.md) | 日本語

OpenFOAM（ESI 版、openfoam.com）には 130〜140 種類の functionObject があります。使うたびに「必須パラメータは何か」「libs は何か」「どう書けばよいか」をソースや tutorials から探すのは手間がかかります。
**foamFO** は、OpenFOAM のソースにある doc コメント（`Description`、`Usage`、`\verbatim` の記述例、`\table` のパラメータ表）、`tutorials`、`etc/caseDicts/postProcessing` を解析してデータベースを作ります。そのデータベースを使って、検索、表示、雛形の生成ができます。

| ファイル | 内容 |
|---|---|
| `foamFO` | CLI（Python 3.8 以上、標準ライブラリのみ） |
| `catalog.html` | 検索できるカタログ。ブラウザで開くだけで使えます（オフライン、単一ファイル） |
| `foDatabase.json` | 同梱のデータベース（OpenFOAM-v2606 から生成） |
| `catalog_template.html` | カタログの雛形 |
| `skill/openfoam-functionobjects/` | Claude Code 用スキル |
| `install.sh` | インストール補助（スキルのリンク、PATH の設定） |

## インストール

```bash
git clone https://github.com/zhang-softflow/functionObjectTool.git
cd functionObjectTool
./install.sh            # ~/.claude/skills にスキルをリンクし、PATH 設定の1行を表示
./install.sh --bashrc   # ~/.bashrc に PATH を追記する場合（バックアップあり）
./install.sh --uninstall
```

使う OpenFOAM が v2606 以外の場合は、その環境を読み込んでからデータベースを作ってください（数秒で終わります）。

```bash
source /usr/lib/openfoam/openfoam2412/etc/bashrc
foamFO build            # -> ~/.local/share/foamFO/foDatabase-openfoam2412.json
```

`$WM_PROJECT_DIR` の名前と一致するデータベースが自動的に選ばれます。

- **探索順**: `$FOAMFO_DB` → `~/.local/share/foamFO/foDatabase-<版>.json` → 同梱の `foDatabase.json`
- **版が違う場合**: 同梱のデータベースの版と環境の版が違うと警告が出ます（`FOAMFO_QUIET=1` で抑止できます）。

## 使い方

| コマンド | 説明 |
|---|---|
| `foamFO list [-g field]` | 一覧を表示します（F=#includeFunc テンプレートあり、T=tutorials に使用例あり） |
| `foamFO list --groups` | グループ別の件数を表示します |
| `foamFO search wall shear` | 名前、説明、パラメータ名、同義語を全文検索します（`drag`、`流量`、`残差` なども可） |
| `foamFO show forces [--inherited]` | 説明、記述例、パラメータ表（必須は赤）、Note、#includeFunc、tutorials の場所を表示します |
| `foamFO example forces -m 3` | tutorials 内の実際の記述を抜き出して表示します（OpenFOAM 環境または `--src` が必要） |
| `foamFO add yPlus -case .` | `system/yPlus1` に雛形を出力します（必須項目のみ） |
| `foamFO add forces --full --stdout` | オプション項目もコメントとして含めた雛形を表示します |
| `foamFO add fieldMinMax -n minMaxU --insert` | `system/minMaxU` を作り、controlDict の `functions{}` に `#include "minMaxU"` を追加します（元のファイルは `controlDict.bak`） |
| `foamFO build [--src DIR] [-o FILE]` | データベースを作ります |
| `foamFO html` | `catalog.html` を作り直します |

- 雛形内の `<word>` や `<wordList>` はプレースホルダです。残っている場合は警告が出ます。
- 雛形は、ヘッダの記述例のコメント（`// Mandatory entries`、`// Optional entries`、`// Conditional ...`）とパラメータ表の Reqd 列から作ります。

## catalog.html

ブラウザで開くだけで使えます。OpenFOAM がインストールされていなくても見られます。

- **左側**: インクリメンタル検索（`/` キーで検索欄へ）、グループの絞り込み
- **右側**: 記述例（Copy ボタン付き）、パラメータ表、継承した共通項目、tutorials での使用箇所
- **項目を直接開く**: `catalog.html#forces` のように末尾に名前を付けます

## Claude Code スキル

`install.sh` で `~/.claude/skills/openfoam-functionobjects` にリンクされます。
「壁面の y+ と抗力係数を出したい」のように頼むと、foamFO でソースの記述を確認してから `functions{}` の設定を書きます。名前で呼ぶ場合は `/openfoam-functionobjects` です。

## OpenFOAM の既存機能

### `#includeFunc`
```cpp
functions
{
    #includeFunc Q
    #includeFunc mag(U)                         // one argument -> 'field' / 'fields'
    #includeFunc patchAverage(name=outlet,p,U)  // named args override entries, others -> fields
    #includeFunc flowRateFaceZone(name=fZone1)
}
```
- **引数にスペースを入れないでください**（1つの word として読まれるため）。`patchAverage(name=outlet, fields=(p U))` のように書くとエラーになります。
- **検索順**: ケースの `system/<名前>` → `~/.OpenFOAM/<ver>/caseDicts/postProcessing` → `$WM_PROJECT_DIR/etc/caseDicts/postProcessing`。
  自作のテンプレートを `~/.OpenFOAM/<ver>/caseDicts/postProcessing/` に置けば、全ケースで使えます。

### `postProcess` / `<solver> -postProcess`
```bash
postProcess -list                          # available #includeFunc templates
postProcess -func 'mag(U)' -latestTime
postProcess -funcs '(Q vorticity)'
simpleFoam -postProcess -func wallShearStress -latestTime   # needs solver models
```

### `foamGetDict`、`foamSearch`、`foamDictionary`
```bash
foamGetDict forcesIncompressible                      # copy a template into system/
foamSearch $FOAM_TUTORIALS functions controlDict      # collect entries from tutorials
foamDictionary system/controlDict -expand             # resolves #include, NOT #includeFunc
```

## 制限と注意
- 対応しているのは **ESI 版（openfoam.com, vYYMM）** です。Foundation 版（openfoam.org）は確認していません。
- ヘッダの記述が不完全な functionObject は、例や表を取り出せません。`foamFO build -v` で一覧を確認でき、その場合は `foamFO show` に表示されるヘッダを直接確認してください。
- `lagrangian/intermediate` の CloudFunctionObject は既定で非表示です（`-a` オプションまたはカタログのチェックボックスで表示）。
- 動作確認の範囲:
  - DB の生成: v2606（ソース）と v2412（パッケージ版）
  - 雛形を使った実行: v2512（simpleFoam/pitzDaily）

## ライセンス
GPL-3.0-or-later（[LICENSE](LICENSE)）。
同梱の `foDatabase.json` と `catalog.html` には、OpenFOAM のソース（GPL-3.0-or-later、© OpenCFD Ltd. ほか）の doc コメントの文章が含まれます。

OPENFOAM® is a registered trade mark of OpenCFD Limited, producer and distributor of the OpenFOAM software via www.openfoam.com. This offering is not approved or endorsed by OpenCFD Limited.
