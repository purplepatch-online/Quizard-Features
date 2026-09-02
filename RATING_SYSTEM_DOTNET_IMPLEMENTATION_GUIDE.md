# Implementing the 0–60 Skill Rating Architecture in .NET 10 Clean Architecture

**Author:** Engineering Architecture Team  
**Branding:** Digital For Good Engineering Documentation  
**Target Framework:** .NET 10 (C# 13) with PostgreSQL & EF Core  
**Reading Tool:** Use `glow RATING_SYSTEM_DOTNET_IMPLEMENTATION_GUIDE.md` or `grip` to view this document.

---

## 📑 Table of Contents
1. [Overview & Core Concepts](#1-overview--core-concepts)
2. [The 6 Integer Performance Bands (0 to 60 Scale)](#2-the-6-integer-performance-bands-0-to-60-scale)
3. [The Rating Formula & Mathematical Logic](#3-the-formula--mathematical-logic)
4. [Step-by-Step Clean Architecture Implementation](#4-step-by-step-clean-architecture-implementation)
   - [4.1 Domain Layer (`Quizard.Domain`)](#41-domain-layer-quizarddomain)
   - [4.2 Application Layer (`Quizard.Application`)](#42-application-layer-quizardapplication)
   - [4.3 Infrastructure Layer (`Quizard.Infrastructure`)](#43-infrastructure-layer-quizardinfrastructure)
   - [4.4 API Presentation Layer (`Quizard.Api`)](#44-api-presentation-layer-quizardapi)
5. [Database Migration & PostgreSQL Schema](#5-database-migration--postgresql-schema)
6. [Unit Testing Suite (`Quizard.Application.Tests`)](#6-unit-testing-suite-quizardapplicationtests)
7. [How to Test and Read on Your Machine](#7-how-to-test-and-read-on-your-machine)

---

## 1. Overview & Core Concepts

In the Quizard tournament engine, every match consists of **60 questions** with a **180-second** time limit.

Rather than using arbitrary 4-digit numbers (like 1200 or 2000), Quizard uses an **intuitive 0 to 60 integer scale**:
* A player's rating ($R$) is a whole integer: $0 \le R \le 60$.
* A player's round score ($S$) is the number of correct answers: $0 \le S \le 60$.
* Every completed round calculates an integer delta ($\Delta R$) that pulls the rating up or down based on performance.

---

## 2. The 6 Integer Performance Bands (0 to 60 Scale)

Every integer rating $R \in [0, 60]$ maps directly to one of six named competitive rank tiers:

| Tier | Rank Band Title | Integer Rating Range ($R$) | Color Theme | Competitive Meaning |
| :---: | :--- | :---: | :--- | :--- |
| **1** | **Novice** | **0 – 19** | Slate Gray | Casual participant / frequent drop-outs |
| **2** | **Apprentice** | **20 – 29** | Emerald Green | Developing player / building foundation |
| **3** | **Specialist** | **30 – 39** | Cyan Blue | Consistent mid-tier competitor |
| **4** | **Expert** | **40 – 49** | Royal Blue | High-level tournament contender |
| **5** | **Master** | **50 – 59** | Deep Violet | Tournament elite ($\ge 83\%$ accuracy) |
| **6** | **Grandmaster** | **60 (Max)** | Imperial Crimson | Flawless perfection ($100\%$ accuracy) |

---

## 3. The Formula & Mathematical Logic ($\Delta S \times 0.6$)

When a player with current rating $R$ scores $S$ in a match:

1. **Calculate the Score Gap:**
   $$\Delta S = S - R$$

2. **Calculate the Integer Delta ($\Delta R$):**
   Rating delta is calculated simply on **$\Delta S \times 0.6$**:
   $$\Delta R = \text{round}\big(0.60 \cdot \Delta S\big) = \text{round}\big(0.60 \cdot (S - R)\big)$$

3. **Calculate the New Rating ($R_{\text{new}}$):**
   $$R_{\text{new}} = \text{Math.Clamp}(R + \Delta R, \, 0, \, 60)$$

### 🔍 Concrete Example: A Player at Rating 34 (Specialist)
* **Scores 32:** $\Delta S = -2 \implies \Delta R = \text{round}(0.6 \times -2) = -1 \implies R_{\text{new}} = \mathbf{33}$ *(Remains Specialist)*
* **Scores 36:** $\Delta S = +2 \implies \Delta R = \text{round}(0.6 \times +2) = +1 \implies R_{\text{new}} = \mathbf{35}$ *(Climbs within Specialist)*
* **Scores 38:** $\Delta S = +4 \implies \Delta R = \text{round}(0.6 \times +4) = +2 \implies R_{\text{new}} = \mathbf{36}$ *(Climbs toward Expert)*
* **Scores 52 (Master Breakout):** $\Delta S = +18 \implies \Delta R = \text{round}(0.6 \times 18) = +11 \implies R_{\text{new}} = \mathbf{45}$ *(Promoted directly to Expert!)*
* **Scores 12 (Severe Drop):** $\Delta S = -22 \implies \Delta R = \text{round}(0.6 \times -22) = -13 \implies R_{\text{new}} = \mathbf{21}$ *(Demoted to Apprentice!)*

---

## 4. Step-by-Step Clean Architecture Implementation

### 4.1 Domain Layer (`Quizard.Domain`)

The Domain layer contains pure business logic and entities with **zero third-party dependencies**.

#### File 1: `src/Quizard.Domain/Enums/RatingBand.cs`
```csharp
namespace Quizard.Domain.Enums;

public enum RatingBand
{
    Novice = 1,      // 0 - 19
    Apprentice = 2,  // 20 - 29
    Specialist = 3,  // 30 - 39
    Expert = 4,      // 40 - 49
    Master = 5,      // 50 - 59
    Grandmaster = 6  // 60
}
```

#### File 2: `src/Quizard.Domain/Entities/ParticipantRatingHistory.cs`
```csharp
using Quizard.Domain.Common;
using Quizard.Domain.Enums;

namespace Quizard.Domain.Entities;

public class ParticipantRatingHistory : BaseEntity
{
    public long Id { get; set; }
    public string Msisdn { get; set; } = string.Empty;
    public int PortalId { get; set; } = 15;
    public int EventId { get; set; }
    public int RoundScore { get; set; }
    public int RatingBefore { get; set; }
    public int RatingDelta { get; set; }
    public int RatingAfter { get; set; }
    public RatingBand Band { get; set; }
    public DateTime CreatedAt { get; set; } = DateTime.UtcNow;
}
```

#### File 3: `src/Quizard.Domain/Services/RatingCalculator.cs`
```csharp
using Quizard.Domain.Enums;

namespace Quizard.Domain.Services;

public static class RatingCalculator
{
    public const int MinRating = 0;
    public const int MaxRating = 60;

    public const double Sensitivity = 0.60;

    public static int CalculateDelta(int currentRating, int roundScore)
    {
        currentRating = Math.Clamp(currentRating, MinRating, MaxRating);
        roundScore = Math.Clamp(roundScore, MinRating, MaxRating);

        int gap = roundScore - currentRating;
        return (int)Math.Round(Sensitivity * gap, MidpointRounding.AwayFromZero);
    }

    public static int ApplyDelta(int currentRating, int delta)
    {
        return Math.Clamp(currentRating + delta, MinRating, MaxRating);
    }

    public static RatingBand GetBand(int rating) => rating switch
    {
        <= 19 => RatingBand.Novice,
        <= 29 => RatingBand.Apprentice,
        <= 39 => RatingBand.Specialist,
        <= 49 => RatingBand.Expert,
        <= 59 => RatingBand.Master,
        _     => RatingBand.Grandmaster
    };
}
```

---

### 4.2 Application Layer (`Quizard.Application`)

The Application layer defines use cases, interfaces, and business workflows.

#### File 1: `src/Quizard.Application/Rating/DTOs/RatingDtos.cs`
```csharp
using Quizard.Domain.Enums;

namespace Quizard.Application.Rating.DTOs;

public record RatingHistoryItemDto(
    long Id,
    int EventId,
    int RoundScore,
    int RatingBefore,
    int RatingDelta,
    int RatingAfter,
    string BandTitle,
    DateTime CreatedAt
);

public record RatingSummaryDto(
    string Msisdn,
    int CurrentRating,
    string CurrentBandTitle,
    int HighestRating,
    int TotalRoundsPlayed,
    List<RatingHistoryItemDto> RecentHistory
);
```

#### File 2: `src/Quizard.Application/Rating/Interfaces/IRatingRepository.cs`
```csharp
using Quizard.Domain.Entities;

namespace Quizard.Application.Rating.Interfaces;

public interface IRatingRepository
{
    Task<int> GetLatestRatingAsync(string msisdn, int portalId, CancellationToken ct = default);
    Task AddRatingHistoryAsync(ParticipantRatingHistory history, CancellationToken ct = default);
    Task<List<ParticipantRatingHistory>> GetHistoryAsync(string msisdn, int portalId, int limit = 20, CancellationToken ct = default);
    Task<int> GetHighestRatingAsync(string msisdn, int portalId, CancellationToken ct = default);
    Task<int> GetTotalRoundsCountAsync(string msisdn, int portalId, CancellationToken ct = default);
}
```

#### File 3: `src/Quizard.Application/Rating/Interfaces/IRatingService.cs`
```csharp
using Quizard.Application.Rating.DTOs;

namespace Quizard.Application.Rating.Interfaces;

public interface IRatingService
{
    Task<RatingHistoryItemDto> ProcessRoundRatingAsync(
        string msisdn,
        int portalId,
        int eventId,
        int roundScore,
        CancellationToken ct = default);

    Task<RatingSummaryDto> GetRatingSummaryAsync(
        string msisdn,
        int portalId,
        CancellationToken ct = default);
}
```

#### File 4: `src/Quizard.Application/Rating/Services/RatingService.cs`
```csharp
using Microsoft.Extensions.Logging;
using Quizard.Application.Rating.DTOs;
using Quizard.Application.Rating.Interfaces;
using Quizard.Domain.Entities;
using Quizard.Domain.Services;

namespace Quizard.Application.Rating.Services;

public class RatingService : IRatingService
{
    private readonly IRatingRepository _ratingRepo;
    private readonly ILogger<RatingService> _logger;

    public RatingService(IRatingRepository ratingRepo, ILogger<RatingService> logger)
    {
        _ratingRepo = ratingRepo;
        _logger = logger;
    }

    public async Task<RatingHistoryItemDto> ProcessRoundRatingAsync(
        string msisdn,
        int portalId,
        int eventId,
        int roundScore,
        CancellationToken ct = default)
    {
        // 1. Fetch current integer rating (defaults to 20 Apprentice for fresh players)
        int currentRating = await _ratingRepo.GetLatestRatingAsync(msisdn, portalId, ct);
        if (currentRating == 0)
        {
            currentRating = 20; // Default onboarding baseline
        }

        // 2. Compute integer delta and new rating
        int delta = RatingCalculator.CalculateDelta(currentRating, roundScore);
        int newRating = RatingCalculator.ApplyDelta(currentRating, delta);
        var newBand = RatingCalculator.GetBand(newRating);

        // 3. Persist transaction record
        var historyRecord = new ParticipantRatingHistory
        {
            Msisdn = msisdn,
            PortalId = portalId,
            EventId = eventId,
            RoundScore = roundScore,
            RatingBefore = currentRating,
            RatingDelta = delta,
            RatingAfter = newRating,
            Band = newBand,
            CreatedAt = DateTime.UtcNow
        };

        await _ratingRepo.AddRatingHistoryAsync(historyRecord, ct);

        _logger.LogInformation(
            "[Rating Engine] MSISDN {Msisdn} scored {Score}/60: Rating {Before} -> {After} (Delta: {Delta}, Band: {Band})",
            msisdn, roundScore, currentRating, newRating, delta, newBand);

        return new RatingHistoryItemDto(
            historyRecord.Id,
            eventId,
            roundScore,
            currentRating,
            delta,
            newRating,
            newBand.ToString(),
            historyRecord.CreatedAt
        );
    }

    public async Task<RatingSummaryDto> GetRatingSummaryAsync(
        string msisdn,
        int portalId,
        CancellationToken ct = default)
    {
        int currentRating = await _ratingRepo.GetLatestRatingAsync(msisdn, portalId, ct);
        if (currentRating == 0) currentRating = 20;

        int highestRating = await _ratingRepo.GetHighestRatingAsync(msisdn, portalId, ct);
        if (highestRating == 0) highestRating = currentRating;

        int totalRounds = await _ratingRepo.GetTotalRoundsCountAsync(msisdn, portalId, ct);
        var history = await _ratingRepo.GetHistoryAsync(msisdn, portalId, 20, ct);

        var historyDtos = history.Select(h => new RatingHistoryItemDto(
            h.Id,
            h.EventId,
            h.RoundScore,
            h.RatingBefore,
            h.RatingDelta,
            h.RatingAfter,
            h.Band.ToString(),
            h.CreatedAt
        )).ToList();

        return new RatingSummaryDto(
            msisdn,
            currentRating,
            RatingCalculator.GetBand(currentRating).ToString(),
            highestRating,
            totalRounds,
            historyDtos
        );
    }
}
```

---

### 4.3 Infrastructure Layer (`Quizard.Infrastructure`)

The Infrastructure layer implements database persistence using EF Core and PostgreSQL.

#### File 1: `src/Quizard.Infrastructure/Persistence/Configurations/ParticipantRatingHistoryConfiguration.cs`
```csharp
using Microsoft.EntityFrameworkCore;
using Microsoft.EntityFrameworkCore.Metadata.Builders;
using Quizard.Domain.Entities;

namespace Quizard.Infrastructure.Persistence.Configurations;

public class ParticipantRatingHistoryConfiguration : IEntityTypeConfiguration<ParticipantRatingHistory>
{
    public void Configure(EntityTypeBuilder<ParticipantRatingHistory> builder)
    {
        builder.ToTable("participant_rating_history");

        builder.HasKey(e => e.Id);
        builder.Property(e => e.Id).HasColumnName("id");

        builder.Property(e => e.Msisdn).HasColumnName("msisdn").HasMaxLength(15).IsRequired();
        builder.Property(e => e.PortalId).HasColumnName("portal_id").HasDefaultValue(15);
        builder.Property(e => e.EventId).HasColumnName("event_id").IsRequired();
        builder.Property(e => e.RoundScore).HasColumnName("round_score").IsRequired();
        builder.Property(e => e.RatingBefore).HasColumnName("rating_before").IsRequired();
        builder.Property(e => e.RatingDelta).HasColumnName("rating_delta").IsRequired();
        builder.Property(e => e.RatingAfter).HasColumnName("rating_after").IsRequired();
        builder.Property(e => e.Band).HasColumnName("band").HasConversion<string>().HasMaxLength(30).IsRequired();
        builder.Property(e => e.CreatedAt).HasColumnName("created_at").HasDefaultValueSql("CURRENT_TIMESTAMP");

        builder.HasIndex(e => new { e.Msisdn, e.PortalId, e.CreatedAt })
               .HasDatabaseName("idx_rating_history_lookup");
    }
}
```

#### File 2: `src/Quizard.Infrastructure/Repositories/RatingRepository.cs`
```csharp
using Microsoft.EntityFrameworkCore;
using Quizard.Application.Rating.Interfaces;
using Quizard.Domain.Entities;
using Quizard.Infrastructure.Persistence;

namespace Quizard.Infrastructure.Repositories;

public class RatingRepository : IRatingRepository
{
    private readonly QuizardDbContext _dbContext;

    public RatingRepository(QuizardDbContext dbContext)
    {
        _dbContext = dbContext;
    }

    public async Task<int> GetLatestRatingAsync(string msisdn, int portalId, CancellationToken ct = default)
    {
        var latest = await _dbContext.ParticipantRatingHistories
            .Where(h => h.Msisdn == msisdn && h.PortalId == portalId)
            .OrderByDescending(h => h.CreatedAt)
            .Select(h => (int?)h.RatingAfter)
            .FirstOrDefaultAsync(ct);

        return latest ?? 0;
    }

    public async Task AddRatingHistoryAsync(ParticipantRatingHistory history, CancellationToken ct = default)
    {
        await _dbContext.ParticipantRatingHistories.AddAsync(history, ct);
        await _dbContext.SaveChangesAsync(ct);
    }

    public async Task<List<ParticipantRatingHistory>> GetHistoryAsync(string msisdn, int portalId, int limit = 20, CancellationToken ct = default)
    {
        return await _dbContext.ParticipantRatingHistories
            .Where(h => h.Msisdn == msisdn && h.PortalId == portalId)
            .OrderByDescending(h => h.CreatedAt)
            .Take(limit)
            .ToListAsync(ct);
    }

    public async Task<int> GetHighestRatingAsync(string msisdn, int portalId, CancellationToken ct = default)
    {
        var max = await _dbContext.ParticipantRatingHistories
            .Where(h => h.Msisdn == msisdn && h.PortalId == portalId)
            .MaxAsync(h => (int?)h.RatingAfter, ct);

        return max ?? 0;
    }

    public async Task<int> GetTotalRoundsCountAsync(string msisdn, int portalId, CancellationToken ct = default)
    {
        return await _dbContext.ParticipantRatingHistories
            .CountAsync(h => h.Msisdn == msisdn && h.PortalId == portalId, ct);
    }
}
```

---

### 4.4 API Presentation Layer (`Quizard.Api`)

#### File: `src/Quizard.Api/Controllers/ParticipantProfileController.cs`
```csharp
using Microsoft.AspNetCore.Authorization;
using Microsoft.AspNetCore.Mvc;
using Quizard.Application.Rating.DTOs;
using Quizard.Application.Rating.Interfaces;

namespace Quizard.Api.Controllers;

[ApiController]
[Route("api/[controller]")]
[Authorize]
public class ParticipantProfileController : ControllerBase
{
    private readonly IRatingService _ratingService;

    public ParticipantProfileController(IRatingService ratingService)
    {
        _ratingService = ratingService;
    }

    [HttpGet("rating_history")]
    public async Task<ActionResult<RatingSummaryDto>> GetRatingHistory(
        [FromQuery] int portal = 15,
        CancellationToken ct = default)
    {
        var msisdn = User.Identity?.Name ?? User.FindFirst("username")?.Value;
        if (string.IsNullOrEmpty(msisdn))
        {
            return Unauthorized(new { error = "Invalid token credentials." });
        }

        var summary = await _ratingService.GetRatingSummaryAsync(msisdn, portal, ct);
        return Ok(summary);
    }
}
```

---

## 5. Database Migration & PostgreSQL Schema

Execute this SQL migration in PostgreSQL to create the table and query indexes:

```sql
-- Historical Rating Snapshots Table
CREATE TABLE IF NOT EXISTS participant_rating_history (
    id BIGSERIAL PRIMARY KEY,
    msisdn VARCHAR(15) NOT NULL,
    portal_id INT NOT NULL DEFAULT 15,
    event_id INT NOT NULL,
    round_score INT NOT NULL CHECK (round_score BETWEEN 0 AND 60),
    rating_before INT NOT NULL CHECK (rating_before BETWEEN 0 AND 60),
    rating_delta INT NOT NULL,
    rating_after INT NOT NULL CHECK (rating_after BETWEEN 0 AND 60),
    band VARCHAR(30) NOT NULL,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

-- Index for instant profile time-series graph loading (<2ms)
CREATE INDEX IF NOT EXISTS idx_rating_history_lookup 
ON participant_rating_history (msisdn, portal_id, created_at DESC);
```

---

## 6. Unit Testing Suite (`Quizard.Application.Tests`)

Create comprehensive xUnit test cases in `tests/Quizard.Application.Tests/RatingCalculatorTests.cs`:

```csharp
using Quizard.Domain.Enums;
using Quizard.Domain.Services;
using Xunit;

namespace Quizard.Application.Tests;

public class RatingCalculatorTests
{
    [Theory]
    [InlineData(34, 34, 0)]    // Identical score: 0 delta
    [InlineData(34, 36, 1)]    // +2 gap: round(0.60 * 2) = +1
    [InlineData(34, 38, 2)]    // +4 gap: round(0.60 * 4) = +2
    [InlineData(34, 52, 11)]   // +18 gap (Master breakout): round(0.60 * 18) = +11
    [InlineData(34, 32, -1)]   // -2 gap: round(0.60 * -2) = -1
    [InlineData(34, 12, -13)]  // -22 gap (Severe quit): round(0.60 * -22) = -13
    public void CalculateDelta_ReturnsExpectedIntegerDelta(int currentRating, int roundScore, int expectedDelta)
    {
        int actualDelta = RatingCalculator.CalculateDelta(currentRating, roundScore);
        Assert.Equal(expectedDelta, actualDelta);
    }

    [Theory]
    [InlineData(10, RatingBand.Novice)]
    [InlineData(19, RatingBand.Novice)]
    [InlineData(20, RatingBand.Apprentice)]
    [InlineData(29, RatingBand.Apprentice)]
    [InlineData(30, RatingBand.Specialist)]
    [InlineData(39, RatingBand.Specialist)]
    [InlineData(40, RatingBand.Expert)]
    [InlineData(49, RatingBand.Expert)]
    [InlineData(50, RatingBand.Master)]
    [InlineData(59, RatingBand.Master)]
    [InlineData(60, RatingBand.Grandmaster)]
    public void GetBand_ReturnsCorrectRankTier(int rating, RatingBand expectedBand)
    {
        var actualBand = RatingCalculator.GetBand(rating);
        Assert.Equal(expectedBand, actualBand);
    }

    [Fact]
    public void ApplyDelta_ClampsStrictlyBetweenZeroAndSixty()
    {
        Assert.Equal(60, RatingCalculator.ApplyDelta(58, 10)); // Capped at 60
        Assert.Equal(0, RatingCalculator.ApplyDelta(5, -15));  // Floored at 0
    }
}
```

---

## 7. How to Test and Read on Your Machine

### 📖 Reading This Document
We have installed **`glow`** on your machine. You can read this document in your terminal with:

```bash
# Interactive scrolling viewer with syntax highlighting
glow -p RATING_SYSTEM_DOTNET_IMPLEMENTATION_GUIDE.md

# Or render in a local browser with GitHub styling
grip RATING_SYSTEM_DOTNET_IMPLEMENTATION_GUIDE.md
```

### 🧪 Running the .NET Tests
To verify all unit tests in the .NET backend:

```bash
cd /home/eagle/projects/Quizard-Backend-DotNet
dotnet test
```

---

*Digital For Good — Engineering Architecture & Best Practices Guide*
