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
    ├── chapter1_rules_engine_governance.tex     # 1. Rules Engine, 180s Timers & 2-Round Cap
    ├── chapter2_gameplay_question_engine.tex    # 2. Dynamic 60-Question Sampling & Weighting
    ├── chapter3_security_anticheat_scoring.tex  # 3. AES-256-CBC Security & AntiCheatService
    ├── chapter4_leaderboards_tournaments_winners.tex # 4. Live Leaderboards, 4:30 AM & 5:30 AM Crons
    ├── chapter5_api_architecture_dotnet_migration.tex # 5. .NET 10 Clean Architecture & PostgreSQL
    ├── chapter6_cms_fastapi_telemetry_service.tex    # 6. FastAPI Microservice (cms.quizard.live)
    ├── chapter7_cms_admin_multigame_ecosystem.tex    # 7. CMS Ops, Question Approval & Telco DOB
    ├── chapter8_reporting_analytics_payouts.tex      # 8. Financial Reporting, bKash B2C & BI
    ├── chapter9_rating_algorithm_score_bands.tex     # 9. 6 Integer Rating Bands (0 to 60 Scale)
    ├── chapter10_xp_ledger_linear_streaks_badges.tex # 10. Linear Streaks (+20 XP*n) & Day 15 Giant Bump
    ├── chapter11_rank_velocity_personalization_adaptive.tex # 11. Rank Velocity (↑/↓), Personalization & 10s Timers
    └── chapter12_authentication_identity_lifecycle_logout.tex # 12. Authentication, MSISDN Normalization & Logout
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
