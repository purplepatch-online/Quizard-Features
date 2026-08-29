# Quizard-Features

**Quizard Features Storybook: Master Technical Guide & Domain Storytelling**  
*Confidential — Digital For Good Engineering Documentation*

---

## 📖 Overview

This repository contains the complete LaTeX source files and the compiled 43-page master technical volume (**`quizard_features_storybook.pdf`**) documenting the end-to-end architecture, domain workflows, and competitive gaming mechanics of the **Quizard** platform.

---

## 🏛️ Repository Structure

```
Quizard-Features/
├── quizard_features_storybook.tex               # Master LaTeX entry point
├── quizard_features_storybook.pdf               # Compiled 49-page master PDF volume
├── .gitignore                                   # Ignore LaTeX build artifacts (*.aux, *.log, etc.)
├── README.md                                    # Repository documentation
├── RATING_SYSTEM_DOTNET_IMPLEMENTATION_GUIDE.md # .NET 10 Clean Architecture Implementation Guide
└── chapters/
    ├── chapter01_authentication_identity_lifecycle_logout.tex # 1. Authentication, MSISDN Normalization & Logout
    ├── chapter02_rules_engine_governance.tex     # 2. Rules Engine, 180s Timers & 2-Round Cap
    ├── chapter03_gameplay_question_engine.tex    # 3. Dynamic 60-Question Sampling & Weighting
    ├── chapter04_security_anticheat_scoring.tex  # 4. AES-256-CBC Security & AntiCheatService
    ├── chapter05_leaderboards_tournaments_winners.tex # 5. Live Leaderboards, 4:30 AM & 5:30 AM Crons
    ├── chapter06_rating_algorithm_score_bands.tex     # 6. 6 Integer Rating Bands (0 to 60 Scale)
    ├── chapter07_xp_ledger_daily_streaks_badges.tex   # 7. Daily Streaks (+20 XP/day) & Day 15 Surge (+500 XP)
    ├── chapter08_user_personalization_rank_velocity.tex # 8. Personalization Nudges & Rank Velocity Arrows
    ├── chapter09_multimodal_quiz_formats.tex     # 9. Multi-Modal Formats (Image, Audio, Video, Tile)
    ├── chapter10_adaptive_difficulty_retention.tex # 10. Adaptive Match Governance & Grandmaster Blitz
    ├── chapter11_websockets_realtime_architecture.tex # 11. Real-Time WebSockets & Live 1v1 Duels
    ├── chapter12_ai_voice_qa_rag_architecture.tex # 12. AI Voice Q&A, RAG & Small vs Large LLMs
    ├── chapter13_cms_fastapi_telemetry_service.tex # 13. FastAPI Telemetry Service (cms.quizard.live)
    ├── chapter14_cms_admin_multigame_ecosystem.tex # 14. CMS Ops, Multi-Game Engines & Telco DOB
    ├── chapter15_reporting_analytics_payouts.tex # 15. Financial Reporting, bKash B2C & BI
    └── chapter16_api_architecture_dotnet_migration.tex # 16. .NET 10 Clean Architecture & PostgreSQL
```

---

## 🛠️ Building the Master PDF

To recompile the document locally:

```bash
pdflatex quizard_features_storybook.tex
pdflatex quizard_features_storybook.tex
```

---

*Engineered by Digital For Good Engineering Architecture Team.*
