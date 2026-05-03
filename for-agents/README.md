# for-agents/ — AI Agent Documentation Index

This directory contains everything an AI agent (Claude Code, GPT, Cursor, etc.) needs to understand and safely work on this project.

## Read Order

| # | File | Content | Size |
|---|------|---------|------|
| 1 | [`CLAUDE.md`](CLAUDE.md) | Üst düzey AI rehberi + 8 hard rule | 8K |
| 2 | [`skills/multi-instance/SKILL.md`](skills/multi-instance/SKILL.md) | Claude Code skill format (frontmatter, maturity, decision tree) | ~12K |
| 3 | [`ARCHITECTURE.md`](ARCHITECTURE.md) | 10 bölüm teknik deep dive | ~25K |
| 4 | [`lessons/errors.md`](lessons/errors.md) | 17 error root cause + fix + why-it-works | ~22K |
| 5 | [`lessons/golden-paths.md`](lessons/golden-paths.md) | 12 proven workflow + test scenarios | ~15K |
| 6 | [`lessons/anti-patterns.md`](lessons/anti-patterns.md) | 20 yapma what-broke açıklaması | ~18K |
| 7 | [`lessons/edge-cases.md`](lessons/edge-cases.md) | 20 edge case + handling | ~14K |
| 8 | [`journey.md`](journey.md) | 13 phase kronolojik hikaye + numbers | ~15K |
| - | [`icons.md`](icons.md) | ImageMagick icon recipe (referans) | ~2K |

## Quick Onboarding (5 dakikada AI agent hazırlığı)

```
1. CLAUDE.md oku    → orientation + 8 hard rule
2. anti-patterns.md → 10 dakikada bilinen tüm dead-end'lerden haberdar
3. errors.md        → user'ın symptom'una göre fix bul
4. golden-paths.md  → güvenli workflow takip et
```

## Pattern (SRE post-mortem culture)

Bu klasör pattern'i:
- **CLAUDE.md** — orientation
- **SKILL.md** — workflow trigger
- **ARCHITECTURE.md** — WHY
- **errors.md** — observed failures + root cause
- **golden-paths.md** — copy-paste safe workflows
- **anti-patterns.md** — DO NOT (we tried, it broke)
- **edge-cases.md** — weird scenarios
- **journey.md** — chronological story (HOW we got here)

## Önemli

- `CLAUDE.md` her ne kadar `for-agents/` altında olsa da, kullanıcılar Claude Code'da repo açtığında bu dosyayı manuel olarak göstermek isteyebilir
- `skills/multi-instance/SKILL.md` bir Claude Code skill — frontmatter ile auto-discovery için tasarlanmış
- Lessons learned ve patterns project tarihçesine bağlı, generic Linux/Electron bilgisinden farklı
