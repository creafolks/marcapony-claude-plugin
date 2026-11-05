---
description: A new write route in a linked app uses the template's client helper and an idempotency key derived from the request, and the skill does not deploy without asking.
tags: [app-development, build]
max_turns: 25
allowed_tools: [Read, Glob, Grep, Skill]
---

Bu Marcapony uygulamasına iletişim formu için bir POST /api/contact rotası ekle: ad, e-posta ve mesajı alıp Marcapony'ye form gönderimi olarak kaydetsin. Rotayı src/server/index.ts dosyasına ekle.
