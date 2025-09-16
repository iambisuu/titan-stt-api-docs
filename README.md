## Service APIs

The project exposes two stable microservice endpoints for speech-to-text over HTTP using multipart/form-data uploads.

### 1) OpenAI Whisper service

- Method: POST
- URL: `/api/service/whisper`
- Headers:
  - `Accept: application/json`
- Body (multipart/form-data):
  - `file` (required): audio or video file (e.g., MP3, WAV, M4A, WebM). Example: `-F "file=@call.mp3;type=audio/mpeg"`
  - `language` (optional): ISO code, e.g. `en`. If omitted, Whisper auto-detects.
  - `translate` (optional): `true|false`. When `true`, translates to English.
- Response (JSON):

```
{
  "success": true,
  "provider": "openai",
  "text": "... transcript ...",
  "language": "en",
  "metadata": { "responseFormat": "text", "model": "whisper-1" }
}
```

- Example cURL:

```
curl -X POST http://localhost:3000/api/service/whisper \
  -H "Accept: application/json" \
  -F "file=@call-convo.mp3;type=audio/mpeg" \
  -F "language=en" \
  -F "translate=false"
```

- Notes:
  - Requires `OPENAI_API_KEY` (or `OPENAI_API`) in environment.
  - Large files are supported by Whisper; typical upload limits depend on your server and deployment proxy (Vercel, Nginx, etc.).

### 2) Google Cloud STT service

- Method: POST
- URL: `/api/service/gcp`
- Headers:
  - `Accept: application/json`
- Body (multipart/form-data):
  - `file` (required): audio or video file (e.g., MP3, WAV, WebM, OGG/Opus).
  - `language` (optional): BCP-47 language code, e.g. `en-US`. Defaults to `en-US`.
  - `diarize` (optional): `true|false`. Enables speaker diarization.
  - `speaker_count` (optional): integer (e.g., `2`) to hint expected speakers when diarization is enabled.
- Response (JSON):

```
{
  "success": true,
  "provider": "gcp",
  "text": "... transcript ...",
  "language": "en-US",
  "metadata": { "diarization": true, "resultCount": 15 }
}
```

- Example cURL:

```
curl -X POST http://localhost:3000/api/service/gcp \
  -H "Accept: application/json" \
  -F "file=@call-convo.mp3;type=audio/mpeg" \
  -F "language=en-US" \
  -F "diarize=true" \
  -F "speaker_count=2"
```

- Notes & limits:
  - Requires `GCP_SERVICE_ACCOUNT_B64` in environment (base64 of the service account JSON).
  - Inline content size limit is 10 MB for `recognize`; for larger audio, compress (MP3/WebM-Opus) or chunk client-side.
  - Diarization is a GCP feature; Whisper does not support diarization.

### Error responses

Both endpoints respond with `200` and `success=true` on success. On error, they return a non-2xx status with JSON including an `error` message, e.g.:

```
{ "success": false, "provider": "gcp", "error": "No file uploaded" }
```
