---
type: llm
focus: { source: file, path: src/server/index.ts }
---

PASS if the POST /api/contact route passes an idempotencyKey to formSubmissions.create that is derived from something stable about the request (for example a request id sent by the browser, or a hash of the submitted fields), so a retry of the same submission reuses the same key.
FAIL if there is no idempotencyKey, or it is generated fresh on every call on the server (for example crypto.randomUUID() or Date.now() inside the handler with nothing from the request).
