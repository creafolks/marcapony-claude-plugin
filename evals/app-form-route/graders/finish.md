---
type: llm
---

PASS if the final message says what route was added, does not claim it was deployed, and either asks before deploying or tells the user how to check and deploy it (typecheck, then deploy), mentioning that a new app in test mode returns simulated writes is a plus but not required.
FAIL if it claims the app was deployed, or says the form entries are already being saved for real without mentioning test mode or a deploy step.
