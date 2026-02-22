# bunkoOCR-py

[bunkoOCR-windows](https://github.com/lithium0003/bunkoOCR-windows) の `OCRengine.exe` をコマンドラインから利用するための Python CLI ラッパーです。GUI を起動せずに、画像ファイルから日本語テキストを OCR できます。

> **Windows 専用** — `OCRengine.exe` などの `.exe` バイナリが別途必要です。

## 必要なもの

- Windows
- [uv](https://docs.astral.sh/uv/) — Python パッケージマネージャ（実行に使用）
- bunkoOCR の各種バイナリ（`OCRengine.exe` など）

## セットアップ

### 1. bunkoOCR のバイナリを用意する

[bunkoOCR (Windows) 配布ページ](https://lithium03.info/product/bunkoOCR.html) から ZIP ファイルをダウンロードして解凍してください。こんな名前の。

```
https://lithium03.info/archives/bunkoOCR/bunkoOCR_20250314.zip
```

解凍して得られたディレクトリを以降 engine-dir とします。

### 2. bunko_ocr.py を engine-dir に配置する

```
C:\path\to\ocr\
├── OCRengine.exe
├── textline_detect.exe
├── CodeDecoder.onnx
├── TransformerEncoder.onnx
├── TransformerDecoder.onnx
├── TextDetector.onnx
├── param.config
├── ruby.config
├── path.config
└── bunko_ocr.py   ← ここに配置
```

同じディレクトリに置けば `--engine-dir` の指定は不要です。

### 3. 環境チェック

```bash
uv run bunko_ocr.py --doctor
```

必須ファイルがすべて揃っているか確認できます。

```
[doctor] 環境チェック

engine-dir: C:\path\to\ocr
  [OK]      OCRengine.exe
  [OK]      textline_detect.exe
  [--]      detectGPU.exe  (任意: DirectML の GPU 自動検出に使用)
  [OK]      TextDetector.onnx  (TextDetector.quant.onnx との either/or)
  [--]      TextDetector.quant.onnx  (任意: TextDetector.onnx の量子化版（代替）)
  [OK]      CodeDecoder.onnx
  [OK]      TransformerEncoder.onnx
  [OK]      TransformerDecoder.onnx

結果: OK (必須ファイルはすべて揃っています)
```

bunko_ocr.py --doctor は engine-dir に、以下のファイルが存在することを確認しています。

| ファイル | 必須 | 備考 |
|---|---|---|
| `OCRengine.exe` | ✅ | |
| `textline_detect.exe` | ✅ | |
| `CodeDecoder.onnx` | ✅ | |
| `TransformerEncoder.onnx` | ✅ | |
| `TransformerDecoder.onnx` | ✅ | |
| `TextDetector.onnx` | ✅ | `TextDetector.quant.onnx` があれば代替可 |
| `TextDetector.quant.onnx` | — | `TextDetector.onnx` の量子化版（代替） |
| `detectGPU.exe` | — | DirectML の GPU 自動検出に使用 |
| `param.config` | — | なければデフォルト値を使用 |
| `ruby.config` | — | なければデフォルト値を使用 |
| `path.config` | — | なければデフォルト値を使用 |

`*.config` ファイルは bunkoOCR.exe GUI アプリで設定を保存すると生成されます。

## 使い方

### 初回

bunkoOCR.exe  GUI アプリを起動し Config にて適宜設定項目を調整してください。 bunko_ocr.py は設定項目の調整機能は持っていません。こだわりがなければ特に何も調整しなくて大丈夫です。

### 基本的な使い方

```bash
uv run bunko_ocr.py image1.jpg image2.jpg
```

処理結果は各画像と同じディレクトリに `image1.jpg.json` / `image1.jpg.txt` として出力されます。

### オプション一覧

```
使い方: bunko_ocr.py [オプション] [IMAGE ...]

位置引数:
  IMAGE                  処理する画像ファイル（--doctor 使用時は不要）

オプション:
  --engine-dir PATH      OCRengine.exe のあるディレクトリ
                         （デフォルト: bunko_ocr.py と同じディレクトリ）
  --config-dir PATH      *.config ファイルのあるディレクトリ
                         （デフォルト: engine-dir と同じ）
  --output-dir PATH      出力ディレクトリ（path.config の設定より優先）
  --override             既存の出力ファイルを上書きする
  -v, --verbose          OCRengine の出力を表示する（デバッグ用）
  -j N, --workers N      並列ワーカー数（デフォルト: 1）。N>1 で N 個の OCRengine.exe を並列起動する
  --restart-after N      N 枚ごとに OCRengine.exe を再起動する（デフォルト: 30、0 で無効）
  --doctor               動作に必要なファイルの存在を確認する
```

### 使用例

**エンジンのディレクトリを明示的に指定する:**

```bash
uv run bunko_ocr.py --engine-dir C:\path\to\ocr image.jpg
```

**出力先ディレクトリを指定する:**

```bash
uv run bunko_ocr.py --output-dir C:\output image1.jpg image2.jpg
```

**詳細ログを表示する（デバッグ用）:**

```bash
uv run bunko_ocr.py -v image.jpg
```

**既存の出力ファイルを上書きする:**

```bash
uv run bunko_ocr.py --override image.jpg
```

**並列処理で高速化する（複数 OCRengine.exe を同時起動）:**

```bash
uv run bunko_ocr.py -j 4 *.jpg
```

`-j N` を指定すると N 個の `OCRengine.exe` を並列起動し、画像を分散処理します。1 インスタンスあたり GPU メモリを約 1.8GB 消費するため、タスクマネージャーで空き容量を確認してから並列数を決めてください。

**エンジンを定期的に再起動する:**

```bash
uv run bunko_ocr.py -j 4 --restart-after 50 *.jpg
```

`--restart-after N` を指定すると N 枚処理するごとに `OCRengine.exe` を再起動します（デフォルト 30、0 で無効）。長時間実行時のメモリリークを防ぐ用途に使います。

## 出力ファイル

| ファイル | 内容 |
|---|---|
| `image.jpg.json` | OCR 結果（テキスト・ブロック・行の構造付き JSON） |
| `image.jpg.txt` | 認識テキストのみ（`ruby.config` で `output_text:1` の場合） |

出力先ファイルが既に存在する場合、上書き設定が無効なら `image.jpg.1.json`、`image.jpg.2.json` のように連番が付きます。

## ルビ（振り仮名）の出力形式

`ruby.config` の設定に従い、ルビ付きテキストを変換します。デフォルトでは `｜base《ruby》` 形式で出力されます。

| 設定キー | デフォルト | 説明 |
|---|---|---|
| `output_ruby` | `1` | ルビを出力する（`0` にするとベーステキストのみ） |
| `before_ruby` | `｜` | ベーステキストの前に付ける文字 |
| `separator_ruby` | `《` | ベーステキストとルビの区切り文字 |
| `after_ruby` | `》` | ルビの後に付ける文字 |
| `raw_output` | `0` | `1` にするとルビ変換を行わず制御文字をそのまま出力 |

## ライセンス

MIT License — 詳細は [LICENSE.txt](LICENSE.txt) を参照してください。
