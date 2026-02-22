# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

`bunkoOCR-py` is a single-file Python CLI tool (`bunko_ocr.py`) that wraps `OCRengine.exe` from the bunkoOCR C# WinForms application to perform Japanese text OCR from the command line without a GUI. This is a **Windows-only** tool — it depends on `.exe` binaries that must exist separately.

## Running the tool

The script uses `uv` via its inline script metadata (no `requirements.txt` or `pyproject.toml`):

```bash
uv run bunko_ocr.py image1.jpg image2.jpg
uv run bunko_ocr.py --engine-dir C:\path\to\ocr image.jpg
uv run bunko_ocr.py --doctor          # check environment / required files
uv run bunko_ocr.py -v image.jpg      # verbose: show OCRengine stdout
```

There are no tests, no linting config, and no build step.

## Architecture

The entire implementation lives in `bunko_ocr.py`. The key data flow:

1. **Config loading** — reads `param.config`, `ruby.config`, `path.config` from `config_dir` (defaults to same directory as `OCRengine.exe`). These files are created by the GUI app; the CLI reads them but doesn't create them.

2. **Engine arg construction** — selects GPU backend flags (`TensorRT`, `CUDA`, `OpenVINO`, or DirectML with GPU id) from `param.config`. If `DML_GPU_id=-1`, runs `detectGPU.exe` to auto-detect.

3. **OCR pipeline (`run_ocr`)** — launches `OCRengine.exe` as a subprocess:
   - Sends config params to stdin immediately (before the engine signals readiness)
   - Waits for `"ready"` on stdout
   - Sends image-path pairs (`input_path\r\noutput_path\r\n`) for each image
   - Reads `done: <path>` / `error: <path>` responses from a background reader thread
   - Stdout is read in a daemon thread using a `queue.Queue` to avoid blocking

4. **Postprocessing (`postprocess`)** — reads the output JSON, converts ruby annotation control chars (U+FFF9 IAT, U+FFFA IAS, U+FFFB IAT) to configured bracket format (e.g., `｜base《ruby》`), overwrites the JSON, and optionally writes a `.txt` file with just the plain text.

5. **Output path resolution** — determines where JSON output goes based on `output_dir` and `override` settings, adding `.1`, `.2`, ... suffixes to avoid overwriting existing files when override is disabled.

## Required external files (engine-dir)

- `OCRengine.exe` — required
- `textline_detect.exe` — required
- `CodeDecoder.onnx` — required
- `TransformerEncoder.onnx` — required
- `TransformerDecoder.onnx` — required
- `TextDetector.onnx` or `TextDetector.quant.onnx` — at least one required
- `detectGPU.exe` — optional (DirectML GPU auto-detection)
- `param.config`, `ruby.config`, `path.config` — optional (defaults apply if absent)

Run `uv run bunko_ocr.py --doctor` to check which files are present.
