# Voice Services API Guide

Self-hosted text-to-speech and speech-to-text services for **SolveIt Voice Mode**.  
Both endpoints serve **OpenAI-compatible** APIs, so any OpenAI SDK client works out of the box.

---

## Endpoints

| Service | URL | Port | Model |
|---|---|---|---|
| **TTS** | `https://tts.rahulsaraf.online` | 8001 | ai4bharat/indic-parler-tts (Hindi + English) |
| **STT** | `https://stt.rahulsaraf.online` | 8002 | whisper-large-v3 via faster-whisper |

Both are behind **Cloudflare Tunnel** with automatic HTTPS. Use `https://` — no port number.

---

## Authentication

Every request (except `/health`) requires a Bearer token in the `Authorization` header:

```
Authorization: Bearer YOUR_SHARED_TOKEN
```

Generate a token with:

```bash
openssl rand -hex 32
```

The same token is shared between TTS and STT. Store it securely — never commit it to git.

---

## TTS — Text to Speech

`POST /v1/audio/speech` generates speech audio from text.

### Request

```
Content-Type: application/json
Authorization: Bearer <token>
```

| Field | Type | Required | Default | Description |
|---|---|---|---|---|
| `input` | string | ✅ | — | Text to synthesize (up to ~600 chars per sentence) |
| `voice` | string | ❌ | `alloy` | Voice name (see catalog below) |
| `model` | string | ❌ | `local-tts` | OpenAI-compatible field; currently ignored |
| `response_format` | string | ❌ | `mp3` | Currently only `mp3` is supported |

### Voice Catalog

IndicParler-TTS uses English and Hindi speakers. Voice names are mapped to natural-language voice descriptions the model understands.

| Voice | Language | Speaker | Character |
|---|---|---|---|
| `alloy` | English | Mary | Clear, neutral, moderate pace |
| `echo` | English | Thoma | Warm, confident, moderate pace |
| `nova` | English | Mary | Expressive, bright, natural pace |
| `onyx` | English | Thoma | Deep, calm, slow pace |
| `shimmer` | English | Mary | Soft, gentle, relaxed pace |
| `hf_alpha` | Hindi | Divya | Clear, calm, moderate pace |
| `hf_aditi` | Hindi | Aditi | Clear, warm, moderate pace |
| `hm_alpha` | Hindi | Rohit | Deep, confident, moderate pace |

> **Tip**: `hf_alpha` and `hf_aditi` handle Hindi-English code-mixing well. Use Hindi text with Hindi voices, English text with English voices, for best results.

### Response

- **Content-Type**: `audio/mpeg`
- **Body**: Streaming MP3 audio bytes

### Examples

#### cURL

```bash
# English speech
curl -X POST https://tts.rahulsaraf.online/v1/audio/speech \
  -H "Authorization: Bearer $TTS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"input":"Hello, this is a test.","voice":"alloy"}' \
  --output greeting.mp3

# Hindi speech
curl -X POST https://tts.rahulsaraf.online/v1/audio/speech \
  -H "Authorization: Bearer $TTS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"input":"नमस्ते, यह स्थानीय टीटीएस सेवा का परीक्षण है।","voice":"hf_alpha"}' \
  --output hi-speech.mp3

# Code-mixed Hindi-English
curl -X POST https://tts.rahulsaraf.online/v1/audio/speech \
  -H "Authorization: Bearer $TTS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"input":"मुझे Python में sort करना है।","voice":"hf_alpha"}' \
  --output mixed.mp3
```

#### Python (OpenAI SDK)

```python
from openai import OpenAI

client = OpenAI(
    base_url="https://tts.rahulsaraf.online/v1",
    api_key="YOUR_SHARED_TOKEN",  # Bearer token goes here
)

audio = client.audio.speech.create(
    model="local-tts",
    voice="hf_alpha",               # or "alloy", "nova", "hf_aditi", etc.
    input="मुझे Python में sort करना है।",
    response_format="mp3",
)

audio.stream_to_file("output.mp3")
```

#### Python (requests)

```python
import requests

resp = requests.post(
    "https://tts.rahulsaraf.online/v1/audio/speech",
    headers={
        "Authorization": f"Bearer {TTS_TOKEN}",
        "Content-Type": "application/json",
    },
    json={
        "input": "Hello, world!",
        "voice": "nova",
    },
)

with open("out.mp3", "wb") as f:
    f.write(resp.content)
```

---

## STT — Speech to Text

`POST /v1/audio/transcriptions` transcribes audio files to text.

### Request

- **Content-Type**: `multipart/form-data`
- **Authorization**: `Bearer <token>`

| Field | Type | Required | Default | Description |
|---|---|---|---|---|
| `file` | file | ✅ | — | Audio file (WebM/Opus, MP3, WAV, etc.) |
| `model` | string | ❌ | `whisper-1` | Ignored; always serves large-v3 |
| `language` | string | ❌ | auto-detect | ISO 639-1 code (`en`, `hi`) or `None` |
| `response_format` | string | ❌ | `json` | `json` or `text` |
| `prompt` | string | ❌ | — | Biasing context for hard words or names |
| `temperature` | float | ❌ | `0.0` | Sampling temperature |

### Response

**JSON format** (`response_format=json`):

```json
{
  "text": " transcription result",
  "language": "hi",
  "duration": 4.2
}
```

**Plain text** (`response_format=text`): just the transcription string.

### Audio Constraints

- **Max file size**: 25 MB (matches OpenAI's limit)
- **Supported formats**: WebM/Opus, MP3, WAV, FLAC, and others decoded by ffmpeg

### Examples

#### cURL

```bash
# Transcribe an MP3 (auto-detect language)
curl -X POST https://stt.rahulsaraf.online/v1/audio/transcriptions \
  -H "Authorization: Bearer $TTS_TOKEN" \
  -F "file=@greeting.mp3" \
  -F "model=whisper-1"

# Transcribe with explicit Hindi
curl -X POST https://stt.rahulsaraf.online/v1/audio/transcriptions \
  -H "Authorization: Bearer $TTS_TOKEN" \
  -F "file=@hindi_audio.mp3" \
  -F "model=whisper-1" \
  -F "language=hi" \
  -F "response_format=text"

# Transcribe with a prompt to bias proper nouns
curl -X POST https://stt.rahulsaraf.online/v1/audio/transcriptions \
  -H "Authorization: Bearer $TTS_TOKEN" \
  -F "file=@meeting.mp3" \
  -F "prompt=Rahul Saraf, Memex, Reliance"
```

#### Python (OpenAI SDK)

```python
from openai import OpenAI

client = OpenAI(
    base_url="https://stt.rahulsaraf.online/v1",
    api_key="YOUR_SHARED_TOKEN",
)

with open("recording.webm", "rb") as f:
    transcript = client.audio.transcriptions.create(
        file=("recording.webm", f.read()),
        model="whisper-1",
        language="hi",          # optional: "en" or None for auto-detect
        response_format="json", # or "text"
        prompt="Rahul, Memex",  # optional: bias for names/terms
    )

print(transcript.text)
```

#### Python (requests)

```python
import requests

with open("recording.mp3", "rb") as f:
    resp = requests.post(
        "https://stt.rahulsaraf.online/v1/audio/transcriptions",
        headers={"Authorization": f"Bearer {TTS_TOKEN}"},
        files={"file": ("audio.mp3", f, "audio/mpeg")},
        data={
            "model": "whisper-1",
            "language": "hi",
            "response_format": "json",
        },
    )

result = resp.json()
print(result["text"])       # transcription
print(result["language"])   # detected language
print(result["duration"])   # audio duration in seconds
```

---

## Health Check

Both services expose a health endpoint to verify the model is loaded and GPU is working.

```bash
# TTS health
curl https://tts.rahulsaraf.online/health

# STT health
curl https://stt.rahulsaraf.online/health
```

### TTS Health Response

```json
{
  "model_loaded": true,
  "model": "ai4bharat/indic-parler-tts",
  "sample_rate": 24000,
  "vram_mb": 1536,
  "voices": ["alloy", "echo", "nova", "onyx", "shimmer", "hf_alpha", "hf_aditi", "hm_alpha"]
}
```

### STT Health Response

```json
{
  "model_loaded": true,
  "model": "large-v3",
  "compute_type": "float16",
  "vram_mb": 3072
}
```

---

## End-to-End Round Trip

Generate speech, then transcribe it back — a full pipeline test:

```bash
# 1. TTS: English → MP3
curl -s -X POST https://tts.rahulsaraf.online/v1/audio/speech \
  -H "Authorization: Bearer $TTS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"input":"The quick brown fox jumps over the lazy dog.","voice":"alloy"}' \
  --output test.mp3

# 2. STT: MP3 → text
curl -s -X POST https://stt.rahulsaraf.online/v1/audio/transcriptions \
  -H "Authorization: Bearer $TTS_TOKEN" \
  -F "file=@test.mp3" \
  -F "model=whisper-1"
```

Expected output:
```json
{
  "text": "the quick brown fox jumps over the lazy dog",
  "language": "en",
  "duration": 2.4
}
```

---

## Browser / Extension Integration

The SolveIt browser extension uses these services for voice input and output. Configuration is managed through the extension settings panel where you provide:

- **STT endpoint**: `https://stt.rahulsaraf.online/v1`
- **TTS endpoint**: `https://tts.rahulsaraf.online/v1`
- **Shared token**: your Bearer token

The extension handles:
- **STT**: Browser's `SpeechRecognition` API (Web Speech API) captures microphone audio, sends it to the local STT endpoint
- **TTS**: Fetches MP3 audio from the TTS endpoint and plays it via the Web Audio API or native `<audio>` element

---

## Known Limitations & Notes

| Topic | Detail |
|---|---|
| **TTS sentence length** | IndicParler tops out around ~600 chars per sentence. Long inputs are split on `.` `!` `?` `।` |
| **TTS response format** | Only `mp3` is supported |
| **TTS first request** | After model load, the first request may take 5–15 s (CUDA graph warmup). Subsequent requests are fast (~400 ms TTFB) |
| **STT file size** | Max 25 MB per upload |
| **Concurrency** | Single request at a time per service (inference lock). Queues concurrent requests |
| **No rate limiting** | Add if exposing beyond yourself |
| **Cold start** | First container start downloads ~5 GB of model weights into `hf_cache/` |
