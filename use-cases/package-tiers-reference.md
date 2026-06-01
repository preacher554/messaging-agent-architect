# Paket & Tier — Referensi Provider

> Referensi package scope Provider berdasarkan playbook v1.0.
> **Bukan referensi competitor** — ini adalah definisi produk yang disediakan oleh Provider sebagai provider.

---

## Positioning

| Paket | Positioning | Tujuan Utama |
|---|---|---|
| **Basic** | AI Receptionist | Jawab chat, FAQ, info layanan, arahkan ke admin |
| **Pro** | AI Sales Receptionist | Semua Basic + gali kebutuhan, qualify lead, siapkan admin untuk closing |
| **Custom/Add-on** | Custom Workflow / Integrasi | Payment, catalog, ads, marketplace, voice, custom workflow |

---

## Perbandingan Kemampuan

| Capability | Basic | Pro | Custom |
|---|---|---|---|
| Auto-reply 24/7 | ✅ | ✅ | - |
| Time-aware greeting | ✅ | ✅ | - |
| FAQ answering | ✅ | ✅ | - |
| Info layanan & harga approved | ✅ | ✅ | - |
| Tanya nama & kebutuhan sederhana | ✅ | ✅ | - |
| Media (gambar/video/caption/button/list) | ✅ | ✅ | - |
| Session context labeling | ✅ | ✅ | - |
| Human handoff safety | ✅ | ✅ | - |
| Same-number takeover protection | ✅ | ✅ | - |
| Admin notification | Sederhana | Lengkap | - |
| Manual / timeout resume | ✅ | ✅ | - |
| Sales discovery ringan | ❌ | ✅ | - |
| Pain point / urgency / budget capture | ❌ | ✅ | - |
| Lead classification (hot/warm/cold) | ❌ | ✅ | - |
| Objection handling (script approved) | ❌ | ✅ | - |
| Full lead summary | ❌ | ✅ | - |
| Recommended next action | ❌ | ✅ | - |
| Light follow-up | ❌ | Terbatas | Advanced |
| Payment gateway (QRIS/VA/E-Wallet) | ❌ | ❌ | ✅ |
| Katalog / stock real-time | ❌ | ❌ | ✅ |
| Appointment scheduling | ❌ | ❌ | ✅ |
| messaging channel broadcast | ❌ | ❌ | ✅ |
| Ads integration (Meta/TikTok/Google) | ❌ | ❌ | ✅ |
| Marketplace (Shopee/Tokped) | ❌ | ❌ | ✅ |
| CRM / ERP | ❌ | ❌ | ✅ |
| Voice calls | ❌ | ❌ | ✅ |
| Open API / MCP | ❌ | ❌ | ✅ |
| Custom workflow / multi-agent | ❌ | ❌ | ✅ |

---

## Flow Comparison

### Basic Flow
```
Greeting → Identifikasi kebutuhan → Jawab dari FAQ/business profile
→ Tanya 1 pertanyaan jika perlu
→ Jika di luar data / butuh manusia → Handoff → AI diam sampai release
```

### Pro Flow
```
Greeting → Understand need → Discovery question satu per satu
→ Capture pain point / urgency / budget (jika natural)
→ Map ke service yang cocok → Jawab objection (script approved)
→ Classify lead (hot/warm/cold)
→ Jika siap closing / butuh authority → Handoff dengan full lead summary
→ Light follow-up (jika aktif)
```

---

## Cara Messaging Agent Merekomendasikan Paket

| Kalau user butuh... | Rekomendasikan |
|---|---|
| Cuma balas chat, FAQ, info jam buka/harga | **Messaging Agent Basic** |
| Closing lead, follow-up, gali kebutuhan, qualify | **Messaging Agent Pro** |
| Payment, katalog, ads integration, marketplace | **Custom/Add-on** → arahkan ke tim |

**Aturan:**
- Jangan sebut harga di awal
- Discovery dulu, baru rekomendasi paket
- Fitur di luar Basic/Pro → arahkan ke tim untuk assessment
- Untuk detail pricing → selalu arahkan ke tim Provider

---

## Custom/Add-on Response Snippet

> "Bisa Kak, tapi itu masuk kebutuhan add-on/custom karena di luar setup standar Basic dan Pro. Untuk tahap awal, kami sarankan mulai dari flow utama dulu supaya cepat jalan dan manfaatnya terlihat."

---

## Referensi Lengkap

Untuk detail flow, handoff format, QA checklist, tenant YAML template, dan intake form → lihat `use-cases/package-tiers-reference.md`.
