# Fundares Extraction Service — Frontend Integration Guide

How to call the AI extraction service from any frontend (web, React Native, Flutter).

Base URL: `https://<api-gateway-id>.execute-api.us-east-1.amazonaws.com`

> The exact URL is available in the CloudFormation stack output `ApiEndpoint` after deployment.

---

## Flows

### Text message (simplest)
```
POST /api/v1/extract/text  →  ExtractionResult
```

### Image or video
```
POST /api/v1/extract/presign  →  { sessionId, uploadUrl }
PUT  <uploadUrl>              →  (direct S3 upload, no Lambda hop)
POST /api/v1/extract/media    →  ExtractionResult
```

Media is uploaded **directly to S3** using a presigned URL — it never passes through the Lambda. This removes file-size limits and reduces latency.

---

## TypeScript / JavaScript

### Types

```typescript
interface PresignResponse {
  sessionId: string;
  uploadUrl: string;
  expiresIn: number;        // seconds (default 300)
}

interface Material {
  type: string;             // e.g. "cardboard", "PET plastic", "glass"
  quantity: number | null;
  unit: 'kg' | 'unit' | null;
}

interface ExtractionData {
  company: string | null;
  date: string | null;      // YYYY-MM-DD
  materials: Material[];
  notes: string | null;
}

interface ExtractionResult {
  sessionId: string;
  inputType: 'text' | 'image' | 'video';
  confidence: 'high' | 'medium' | 'low';
  extracted: ExtractionData | null;
  rejectedReasons?: string[];
}
```

### Extract from text

```typescript
const BASE = 'https://<api-id>.execute-api.us-east-1.amazonaws.com/api/v1';

async function extractFromText(message: string): Promise<ExtractionResult> {
  const res = await fetch(`${BASE}/extract/text`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ message }),
  });

  if (!res.ok && res.status !== 422) {
    throw new Error(`Extraction failed: ${await res.text()}`);
  }

  const { data } = await res.json();
  return data;
}

// Usage
const result = await extractFromText(
  'Today I collected 35 kg of cardboard and 20 PET bottles at Industrias Bisa'
);

if (result.confidence === 'low') {
  console.warn('Low confidence:', result.rejectedReasons);
} else {
  console.log(result.extracted?.materials);
}
```

### Extract from image or video

```typescript
async function presign(mimeType: string): Promise<PresignResponse> {
  const res = await fetch(`${BASE}/extract/presign`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ mimeType }),
  });
  if (!res.ok) throw new Error(`Presign failed: ${await res.text()}`);
  const { data } = await res.json();
  return data;
}

async function uploadToS3(uploadUrl: string, file: File): Promise<void> {
  const res = await fetch(uploadUrl, {
    method: 'PUT',
    headers: { 'Content-Type': file.type },
    body: file,
  });
  if (!res.ok) throw new Error(`S3 upload failed: ${res.status}`);
}

async function extractFromMedia(
  sessionId: string,
  type: 'image' | 'video',
): Promise<ExtractionResult> {
  const res = await fetch(`${BASE}/extract/media`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ sessionId, type }),
  });
  if (!res.ok && res.status !== 422) {
    throw new Error(`Media extraction failed: ${await res.text()}`);
  }
  const { data } = await res.json();
  return data;
}

// Full flow
async function extractFromFile(file: File): Promise<ExtractionResult> {
  const type = file.type.startsWith('video/') ? 'video' : 'image';
  const { sessionId, uploadUrl } = await presign(file.type);
  await uploadToS3(uploadUrl, file);
  return extractFromMedia(sessionId, type);
}

// Usage
const fileInput = document.querySelector<HTMLInputElement>('#file-input')!;
fileInput.addEventListener('change', async () => {
  const file = fileInput.files?.[0];
  if (!file) return;

  const result = await extractFromFile(file);
  console.log(result);
});
```

---

## React / React Native (Expo)

### Text message

```tsx
import { useState } from 'react';

const BASE = 'https://<api-id>.execute-api.us-east-1.amazonaws.com/api/v1';

function TextExtractor() {
  const [message, setMessage] = useState('');
  const [result, setResult] = useState<ExtractionResult | null>(null);
  const [loading, setLoading] = useState(false);

  async function handleSubmit() {
    setLoading(true);
    try {
      const res = await fetch(`${BASE}/extract/text`, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ message }),
      });
      const { data } = await res.json();
      setResult(data);
    } finally {
      setLoading(false);
    }
  }

  return (
    <>
      <textarea value={message} onChange={e => setMessage(e.target.value)} />
      <button onClick={handleSubmit} disabled={loading}>
        {loading ? 'Extracting...' : 'Extract'}
      </button>
      {result && (
        <pre>{JSON.stringify(result.extracted, null, 2)}</pre>
      )}
    </>
  );
}
```

### Image upload (Expo)

```typescript
import * as ImagePicker from 'expo-image-picker';
import * as FileSystem from 'expo-file-system';

const BASE = 'https://<api-id>.execute-api.us-east-1.amazonaws.com/api/v1';

async function pickAndExtract(): Promise<ExtractionResult> {
  const picked = await ImagePicker.launchImageLibraryAsync({
    mediaTypes: ImagePicker.MediaTypeOptions.All,  // images + videos
    quality: 0.9,
  });

  if (picked.canceled) throw new Error('Cancelled');

  const asset    = picked.assets[0];
  const mimeType = asset.mimeType ?? 'image/jpeg';
  const type     = mimeType.startsWith('video/') ? 'video' : 'image';

  // 1. Get presigned URL
  const presignRes  = await fetch(`${BASE}/extract/presign`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ mimeType }),
  });
  const { data: presign } = await presignRes.json();

  // 2. Upload directly to S3
  await FileSystem.uploadAsync(presign.uploadUrl, asset.uri, {
    httpMethod: 'PUT',
    headers: { 'Content-Type': mimeType },
    uploadType: FileSystem.FileSystemUploadType.BINARY_CONTENT,
  });

  // 3. Extract
  const extractRes = await fetch(`${BASE}/extract/media`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ sessionId: presign.sessionId, type }),
  });
  const { data } = await extractRes.json();
  return data;
}
```

---

## Flutter / Dart

```dart
import 'dart:io';
import 'dart:convert';
import 'package:http/http.dart' as http;

const _base = 'https://<api-id>.execute-api.us-east-1.amazonaws.com/api/v1';

class ExtractionClient {
  // ── Text ──────────────────────────────────────────────────────────────────
  Future<Map<String, dynamic>> extractText(String message) async {
    final res = await http.post(
      Uri.parse('$_base/extract/text'),
      headers: {'Content-Type': 'application/json'},
      body: jsonEncode({'message': message}),
    );
    _assertOk(res, [200, 422]);
    return (jsonDecode(res.body) as Map<String, dynamic>)['data'];
  }

  // ── Image / Video ─────────────────────────────────────────────────────────
  Future<Map<String, dynamic>> extractMedia(File file) async {
    final mimeType = _mimeType(file.path);
    final type     = mimeType.startsWith('video/') ? 'video' : 'image';

    // 1. Presign
    final presignRes = await http.post(
      Uri.parse('$_base/extract/presign'),
      headers: {'Content-Type': 'application/json'},
      body: jsonEncode({'mimeType': mimeType}),
    );
    _assertOk(presignRes, [200]);
    final presign = (jsonDecode(presignRes.body))['data'] as Map<String, dynamic>;

    // 2. Upload to S3
    final uploadRes = await http.put(
      Uri.parse(presign['uploadUrl'] as String),
      headers: {'Content-Type': mimeType},
      body: await file.readAsBytes(),
    );
    _assertOk(uploadRes, [200]);

    // 3. Extract
    final extractRes = await http.post(
      Uri.parse('$_base/extract/media'),
      headers: {'Content-Type': 'application/json'},
      body: jsonEncode({'sessionId': presign['sessionId'], 'type': type}),
    );
    _assertOk(extractRes, [200, 422]);
    return (jsonDecode(extractRes.body) as Map<String, dynamic>)['data'];
  }

  void _assertOk(http.Response res, List<int> allowed) {
    if (!allowed.contains(res.statusCode)) {
      throw Exception('HTTP ${res.statusCode}: ${res.body}');
    }
  }

  String _mimeType(String path) {
    if (path.endsWith('.png'))  return 'image/png';
    if (path.endsWith('.webp')) return 'image/webp';
    if (path.endsWith('.mp4'))  return 'video/mp4';
    if (path.endsWith('.mov'))  return 'video/mov';
    return 'image/jpeg';
  }
}

// Usage
void main() async {
  final client = ExtractionClient();

  // Text
  final textResult = await client.extractText(
    'Today I collected 35 kg of cardboard at Industrias Bisa',
  );
  print(textResult['extracted']);

  // Image
  final imageResult = await client.extractMedia(File('/path/to/photo.jpg'));
  print(imageResult['confidence']);
}
```

---

## cURL (manual testing)

### Text

```bash
BASE="https://<api-id>.execute-api.us-east-1.amazonaws.com/api/v1"

curl -s -X POST "$BASE/extract/text" \
  -H "Content-Type: application/json" \
  -d '{"message":"Today I collected 35 kg of cardboard at Industrias Bisa"}' | jq .
```

### Image

```bash
# 1. Get presigned URL
PRESIGN=$(curl -s -X POST "$BASE/extract/presign" \
  -H "Content-Type: application/json" \
  -d '{"mimeType":"image/jpeg"}')

SESSION_ID=$(echo $PRESIGN | jq -r '.data.sessionId')
UPLOAD_URL=$(echo $PRESIGN | jq -r '.data.uploadUrl')

# 2. Upload to S3
curl -s -X PUT "$UPLOAD_URL" \
  -H "Content-Type: image/jpeg" \
  --data-binary @/path/to/photo.jpg

# 3. Extract
curl -s -X POST "$BASE/extract/media" \
  -H "Content-Type: application/json" \
  -d "{\"sessionId\":\"$SESSION_ID\",\"type\":\"image\"}" | jq .
```

### Video

```bash
PRESIGN=$(curl -s -X POST "$BASE/extract/presign" \
  -H "Content-Type: application/json" \
  -d '{"mimeType":"video/mp4"}')

SESSION_ID=$(echo $PRESIGN | jq -r '.data.sessionId')
UPLOAD_URL=$(echo $PRESIGN | jq -r '.data.uploadUrl')

curl -s -X PUT "$UPLOAD_URL" \
  -H "Content-Type: video/mp4" \
  --data-binary @/path/to/video.mp4

curl -s -X POST "$BASE/extract/media" \
  -H "Content-Type: application/json" \
  -d "{\"sessionId\":\"$SESSION_ID\",\"type\":\"video\"}" | jq .
```

---

## Handling the result

```typescript
const result: ExtractionResult = await extractFromText(message);

switch (result.confidence) {
  case 'high':
    // All fields present and unambiguous — safe to save
    saveToDatabase(result.extracted!);
    break;

  case 'medium':
    // Partial data — show to user for review before saving
    showReviewScreen(result.extracted!);
    break;

  case 'low':
    // Not enough data — ask the collector to resend
    showRetryPrompt(result.rejectedReasons ?? []);
    break;
}
```

---

## Common mistakes

| Mistake | Fix |
|---|---|
| Sending wrong `Content-Type` to S3 | Must match the `mimeType` sent in `/presign` exactly — S3 returns `403` otherwise |
| Reusing a `sessionId` | Sessions are single-use — S3 object is deleted after `/extract/media` |
| Waiting more than 5 min to upload | Call `/presign` again to get a fresh URL |
| Treating `422` as a network error | `422` is a valid structured response — parse it the same as `200` |
| Video larger than 2 min | Nova 2 Lite rejects videos over 2 minutes — trim before uploading |
