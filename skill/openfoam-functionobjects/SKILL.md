---
name: openfoam-functionobjects
description: Write, choose or check OpenFOAM (ESI/openfoam.com) function object settings in controlDict functions{} — e.g. "output y+ on walls", "monitor drag coefficient", "flow rate through a section", "add probes". OpenFOAM（ESI版）の functionObject の設定を書く・選ぶ・確認するときに使う。「壁面のyPlusを出したい」「抗力係数をモニタしたい」「断面の流量を出力したい」「controlDictにprobesを追加して」のように、後処理やモニタリングの目的を自然文で受け取り、ソースのヘッダとtutorialsに基づいて正しい functions{} の記述を作る。
---

# OpenFOAM functionObject 設定支援

`foamFO` は、OpenFOAM のソースのヘッダ（Description/Usage/パラメータ表）、tutorials、`#includeFunc` テンプレートから作ったデータベースを検索するツールです。
このスキルでは、**記述をすべて foamFO が返す内容（＝ソースの記述）に基づいて書く**ことを原則とします。記憶や推測でキーワード名や libs 名を書かないでください。

ツールの場所:
- `foamFO` が PATH にあればそれを使います（`command -v foamFO` で確認）。
- PATH に無い場合は、このスキルのベースディレクトリから見て `../../foamFO` を使います（リポジトリ同梱）。

データベース:
- 同梱の DB は OpenFOAM-v2606 から作ったものです。
- ユーザーの OpenFOAM の版が違い、「bundled database is for ...」という警告が出た場合は、OpenFOAM 環境を source した状態で `foamFO build` を実行します（数秒）。作られた DB はその版で自動的に使われます。
- `example` コマンドで tutorials の本文を表示するには、OpenFOAM 環境（`$WM_PROJECT_DIR`）が必要です。環境が無い場合は `--src <OpenFOAMのルート>` を指定します。

回答はユーザーの言語で行います。

## 手順

1. **目的を functionObject に対応付ける**
   - `foamFO search <キーワード...>` で候補を探します（キーワードは英語。例: `wall shear`, `force coeff`, `flow rate`, `probe`, `average`, `min max`）。
   - 候補が複数ある場合は、各 summary を比較して選びます。目的に合うものが無い場合は `foamFO list -g <group>` も見ます。
   - よく使う対応:
     - 壁面 y+ → `yPlus`
     - 壁面せん断応力 → `wallShearStress`
     - 力・モーメント → `forces`
     - 揚力・抗力係数 → `forceCoeffs`
     - 面の流量・平均・積分 → `surfaceFieldValue`（例: `operation sum` + `fields (phi)` で流量）
     - 体積平均 → `volFieldValue`
     - 点のサンプリング → `probes`
     - 線上のサンプリング → `sets`
     - 断面などの面出力 → `surfaces`
     - 時間平均 → `fieldAverage`
     - 最大・最小 → `fieldMinMax`
     - 残差 → `solverInfo`
     - Courant数 → `CourantNo`
     - 渦の可視化 → `Q`、`vorticity`、`Lambda2`
     - VTK出力 → `vtkWrite`
     - 停止条件 → `runTimeControl`
     - 任意の式 → `exprField` / `coded`
   - この対応は手がかりにすぎません。必ず次の手順で裏付けを取ってください。

2. **正しい記述を確認する**
   - `foamFO show <name>` で記述例、パラメータ表（Reqd=yes が必須）、libs、Note を確認します。
   - writeControl などの共通項目が必要なときは、`foamFO show <name> --inherited` で確認します。
   - 実際の使い方を見るには `foamFO example <name> -m 3`（tutorials から抜粋）を使います。
   - foamFO に情報が無い、または曖昧な場合は、`show` に表示されるヘッダパスのファイルを直接 Read して確認します。

3. **設定を作る**
   - `#includeFunc` テンプレートがあり、それで目的を満たす場合は、`#includeFunc name(arg=value, ...)` の1行で済む方法を優先して提示します（`foamFO show` の "#includeFunc templates" 欄で確認できます）。
   - それ以外の場合は、`foamFO add <name> -case <case> -n <entryName> [--full] --stdout` で雛形を作ります。`<word>` などのプレースホルダは、ユーザーの条件（パッチ名、場の名前、座標など）で置き換えます。
   - パッチ名や場の名前は、ケースの `constant/polyMesh/boundary` や `0/` を読んで実在するものを使います。分からない場合はユーザーに確認します。
   - 圧縮性か非圧縮性かで変わる項目（`forces` の `rho rhoInf; rhoInf ...` など）に注意します。

4. **ケースに書き込む**（ユーザーが書き込みを求めた場合のみ）
   - `foamFO add <name> -case <case> -n <entryName> --insert` を使います。controlDict の `functions{}` に `#include` が追加され、`controlDict.bak` が作られます。
   - 生成したファイル内のプレースホルダを Edit で埋めます。foamFO が "Edit these placeholders" と警告したものは必ず埋めてください。
   - OpenFOAM 環境がある場合は、`foamDictionary system/controlDict -expand` で構文を確認します（`#includeFunc` を含む場合は使えないため、`<solver> -postProcess -latestTime` で確認します）。
   - `#includeFunc` の引数にはスペースを入れません。名前付きでない引数は fields になります。例: `#includeFunc patchAverage(name=outlet,p,U)`

## 出力のしかた
- ユーザーの言語で説明し、最終的な `functions { ... }` ブロック（またはファイル内容）を示します。
- 各必須パラメータの意味と、主要なオプションのデフォルト値をパラメータ表に基づいて添えます。
- 出力先（`postProcessing/<entryName>/<time>/...`）と、`<solver> -postProcess -func ...` や `postProcess -func` で後から実行できるかどうかを一言添えます。
