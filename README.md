## Service APIs

The project exposes three stable microservice endpoints: two for speech-to-text and one for Notion integration.

Base url: https://titan-stt-bot.vercel.app/

### 1) Notion Integration service

- Method: POST
- URL: `/api/notion`
- Headers:
  - `Accept: application/json`
- Body (multipart/form-data):
  - `clientName` (optional): string, e.g. `"John Doe"`
  - `meetingDate` (optional): ISO date string, e.g. `"2024-01-15T10:30:00Z"`. Defaults to current date.
  - `transcription` (required): string, the transcribed text
  - `audioFile` (optional): audio file (MP3, WAV, etc.) to upload to Cloudinary
  - `personalNotes` (optional): string, additional notes
  - `meetingSummary` (optional): string, meeting summary
- Response (JSON):

```
{
  "success": true,
  "message": "Meeting record created in Notion",
  "pageId": "12345678-1234-1234-1234-123456789abc",
  "url": "https://notion.so/12345678123412341234123456789abc",
  "audioUrl": "https://res.cloudinary.com/.../audio.mp3",
  "used_database_id": "26f9fd15-6b5d-809c-831c-d460edd3ab0f"
}
```

- Example cURL:

```
curl -X POST http://localhost:3000/api/notion \
  -F "clientName=John Doe" \
  -F "meetingDate=2024-01-15T10:30:00Z" \
  -F "transcription=Meeting discussion about project requirements..." \
  -F "audioFile=@meeting-recording.mp3" \
  -F "personalNotes=Important decisions made" \
  -F "meetingSummary=Project kickoff meeting"
```

- Notes:
  - Requires `NOTION_INTEGRATION_SECRET` and `CLOUDINARY_URL` in environment variables
  - Audio files are uploaded to Cloudinary and linked in Notion (Notion doesn't support direct audio uploads)
  - Long transcriptions (>2000 chars) are split into multiple blocks in the Notion page
  - Creates a new page in the specified Notion database with all provided data

### 2) OpenAI Whisper service

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
  - Large files are supported by Whisper; typical upload limits depend on your server and deployment proxy (Vercel, Nginx, etc.).

### 3) Google Cloud STT service

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
