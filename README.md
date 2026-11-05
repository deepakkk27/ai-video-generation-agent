# ai-video-generation-agent
AI-powered agent for generating educational videos from documents using RAG, LLMs, TTS, and automated video generation.

# Local Course Video Generator

Turns learning material (PDF, PPTX, DOCX, TXT, MD, SRT, VTT) into a narrated educational MP4 with
animated scenes and burned-in captions. Everything runs locally: Ollama for the LLM, Kokoro or Piper for
speech, Pillow + FFmpeg for video. No paid APIs, no keys, no cloud calls at runtime.

## 1. Prerequisites (Windows 10/11, 64-bit)

1. **Python 3.11** (3.10–3.12 recommended): https://www.python.org/downloads/ — tick "Add to PATH".
2. **Ollama**: https://ollama.com/download, then download the model:
   ```
   ollama run qwen3.5:4b
   ```
   Smaller fallbacks if RAM is tight: `ollama pull qwen3.5:2b` or `ollama pull qwen3.5:0.8b`
   (then set `OLLAMA_MODEL` in `.env` or the app's System readiness panel).
3. **FFmpeg** (full build): `winget install Gyan.FFmpeg`, restart the terminal, then verify:
   ```
   ffmpeg -version
   ffprobe -version
   ```
4. **Tesseract (optional)** — only needed for scanned PDFs: install the UB-Mannheim build
   (https://github.com/UB-Mannheim/tesseract/wiki) and add it to PATH. Text-based PDFs work without it.
5. **A TTS backend** (at least one):
   - **Kokoro (preferred):** `pip install kokoro soundfile` inside the venv, and install **espeak-ng** for
     Windows (https://github.com/espeak-ng/espeak-ng/releases). Test it with *Run voice self-test* in the app.
   - **Piper (fallback):** `pip install piper-tts`, then place a voice (`name.onnx` + `name.onnx.json`)
     from the Piper voices collection into the `voices/` folder and set `TTS_PROVIDER=piper`.
   The app never switches backend silently and never uses an online TTS.

## 2. Install and run

```
powershell -ExecutionPolicy Bypass -File setup_windows.ps1
run.bat
```
Or manually:
```
python -m venv .venv
.venv\Scripts\activate
pip install -r requirements.txt
copy .env.example .env
streamlit run app.py
```
`setup_windows.ps1` only checks for tools, creates `.venv`, and installs `requirements.txt`. It downloads nothing else.

## 3. Using it

1. Drop a file, click **Analyze source** (extraction summary, warnings, coverage, extracted-text download).
2. Set duration, content mode, voice, theme; review the confirmation line; click **Generate video**.
3. Watch stage-by-stage progress. On success: preview, measured duration vs target, QA report, downloads
   (MP4, SRT, script, outline, coverage report).

If a stage fails, completed work is kept; click **Generate video** again to resume (analysis chunks, outline,
script and finished audio clips are reused).

## 4. Where files go

`outputs/<job_id>/` holds: `original_*`, `source_records.json`, `chunks.json`, `analysis/`, `outline.json|md`,
`script.json|md`, `audio/` (WAVs), `frames/` (PNGs), `clips/`, `captions.srt`, `course_video.mp4`,
`video_metadata.json`, `qa_report.json|md`, `job.json`, `job.log`. Set `KEEP_INTERMEDIATES=false` to remove
`audio/`, `frames/`, `clips/` after a successful export. Use *Saved jobs → Delete job and files* to remove a job.

## 5. How it works (and honest limits)

- **Duration:** the script is planned at ~130 words/min, then real WAV durations are measured. If the result is
  too short, up to two bounded expansions run (only new segments are synthesized). If it is too long the app does
  **not** cut material or speed up speech. If the source has too little content, the video is shorter and the UI
  says so. The measured duration is always displayed; it is never claimed equal to the target unless it is within tolerance.
- **Captions:** the narration text is the caption source; timing comes from measured clip lengths. This is
  segment-level sync, not word-level or phoneme-level. Captions are drawn into the frames and also exported as SRT.
- **Animation:** per-narration-unit frames with progressive reveals and a short fade-in at each segment start.
  It is deliberately simple, not cinematic.
- **Accuracy:** source-grounded mode flags factual segments with no source ID. Expanded mode labels illustrative
  content and flags claims that need outside confirmation; nothing is "verified". Automated QA does not prove
  educational correctness, so review the script before publishing.
- Images/diagrams inside PDFs and slides are not understood (OCR only reads text in scanned pages).

## 6. Troubleshooting

| Problem | Fix |
|---|---|
| Ollama unavailable | Start the Ollama app or run `ollama serve`; check `OLLAMA_BASE_URL`. |
| Model missing | `ollama pull qwen3.5:4b` (or the fallback tag you configured). |
| Out of memory / slow | Use `qwen3.5:2b`, lower `OLLAMA_NUM_CTX` to 4096, tick 1280×720 in the app. |
| TTS missing | Follow section 1.5; the Generate button stays disabled until a backend is ready. |
| FFmpeg unavailable | `winget install Gyan.FFmpeg`, reopen the terminal, verify both commands. |
| OCR unavailable | Install Tesseract (section 1.4). Pages with no text are marked failed and coverage shows incomplete. |
| Extraction gaps | See the coverage line after Analyze; image-only slides/pages are reported, not guessed. |
| Duration mismatch | Source too short → add material or use Expanded mode; too long → lower the target or split the source. |

## 7. Performance expectations

On an i5-1334U / 16 GB, expect the 4B model to be the bottleneck: analysis plus scripting for a 30–60 minute
video can take well over an hour, and TTS plus FFmpeg add more. Total time is often several times the video length.
Prefer 720p for long courses.

## 8. Privacy

After the one-time downloads of software and model weights, processing is local. The app calls only
`localhost` (Ollama) and runs FFmpeg/TTS on your machine. Uploaded content is never sent to an external service.

## 9. License checklist (do this before distributing or commercial use)

Check the license/terms of each of these as you use them: the Ollama model you pulled (e.g. Qwen), the Kokoro
model and voice, any Piper voice (voices carry individual licenses), your FFmpeg build (GPL vs LGPL, x264),
the fonts used for rendering (system fonts such as Segoe UI/Arial have their own terms), and every Python
dependency (`pip-licenses` helps). Nothing here should be assumed free for commercial use.

## 10. Tests

```
pip install -r requirements.txt
python -m pytest -q
```
Tests mock Ollama and TTS and work offline; FFmpeg tests skip if FFmpeg is missing.
