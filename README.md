# Personal Habit & Productivity Analytics Dashboard
6-Month Self-Improvement & Productivity Tracker built in Power BI Desktop with DAX measures and interactive visualizations.

## STEP 1 — Load Data

1. Open Power BI Desktop → **Get Data → Text/CSV**
2. Load `habit_tracker_6months.csv`
3. Go to **Power Query Editor** → Verify columns:
   - `Date` → change type to **Date**
   - `GymCompleted`, `SleepGoalMet`, `StudyGoalMet`, `WaterGoalMet`, `ScreenGoalMet`, `HabitsMet`, `WinStreak` → **Whole Number**
   - `SleepHours`, `StudyHours`, `WaterLitres`, `ScreenHours` → **Decimal Number**
   - `MoodScore` → **Whole Number**
4. **Close & Apply**

---

## STEP 2 — Create a Date Table (Required for time intelligence)

In **Modeling tab → New Table**, paste:

```dax
DateTable = 
CALENDAR(
    MIN(habit_tracker_6months[Date]),
    MAX(habit_tracker_6months[Date])
)
```

Then add these columns to DateTable:

```dax
Month = FORMAT(DateTable[Date], "MMMM")
MonthNum = MONTH(DateTable[Date])
WeekDay = FORMAT(DateTable[Date], "dddd")
WeekNum = WEEKNUM(DateTable[Date])
IsWeekend = IF(WEEKDAY(DateTable[Date], 2) >= 6, "Weekend", "Weekday")
```

Go to **Model View** → draw relationship: `DateTable[Date]` → `habit_tracker_6months[Date]` (Many to One, Single direction)

---

## STEP 3 — All DAX Measures

Create a dedicated **Measures** table: Home → Enter Data → name it `_Measures` → Load.

Paste each measure via **New Measure**:

```dax
-- ── CORE METRICS ──────────────────────────────────────────────

Total Days Tracked = COUNTROWS(habit_tracker_6months)

Avg Sleep Hours = 
AVERAGEX(habit_tracker_6months, habit_tracker_6months[SleepHours])

Avg Study Hours = 
AVERAGEX(habit_tracker_6months, habit_tracker_6months[StudyHours])

Avg Water Litres = 
AVERAGEX(habit_tracker_6months, habit_tracker_6months[WaterLitres])

Avg Screen Hours = 
AVERAGEX(habit_tracker_6months, habit_tracker_6months[ScreenHours])

Gym Days = 
CALCULATE(
    COUNTROWS(habit_tracker_6months),
    habit_tracker_6months[GymCompleted] = 1
)

Gym Consistency % = 
DIVIDE([Gym Days], [Total Days Tracked]) * 100

-- ── GOAL ACHIEVEMENT ──────────────────────────────────────────

Sleep Goal Rate = 
DIVIDE(
    CALCULATE(COUNTROWS(habit_tracker_6months), habit_tracker_6months[SleepGoalMet] = 1),
    [Total Days Tracked]
) * 100

Study Goal Rate = 
DIVIDE(
    CALCULATE(COUNTROWS(habit_tracker_6months), habit_tracker_6months[StudyGoalMet] = 1),
    [Total Days Tracked]
) * 100

Water Goal Rate = 
DIVIDE(
    CALCULATE(COUNTROWS(habit_tracker_6months), habit_tracker_6months[WaterGoalMet] = 1),
    [Total Days Tracked]
) * 100

Screen Goal Rate = 
DIVIDE(
    CALCULATE(COUNTROWS(habit_tracker_6months), habit_tracker_6months[ScreenGoalMet] = 1),
    [Total Days Tracked]
) * 100

Overall Habit Score % = 
AVERAGEX(
    habit_tracker_6months,
    DIVIDE(habit_tracker_6months[HabitsMet], 5) * 100
)

-- ── STREAK & MOMENTUM ─────────────────────────────────────────

Max Win Streak = MAX(habit_tracker_6months[WinStreak])

Current Streak = 
CALCULATE(
    LASTNONBLANK(habit_tracker_6months[WinStreak], 1),
    FILTER(habit_tracker_6months, habit_tracker_6months[Date] = MAX(habit_tracker_6months[Date]))
)

-- ── TREND ANALYSIS ────────────────────────────────────────────

MoM Sleep Change = 
VAR CurrentMonth = MAX(DateTable[MonthNum])
VAR AvgCurrent = CALCULATE([Avg Sleep Hours], DateTable[MonthNum] = CurrentMonth)
VAR AvgPrev = CALCULATE([Avg Sleep Hours], DateTable[MonthNum] = CurrentMonth - 1)
RETURN DIVIDE(AvgCurrent - AvgPrev, AvgPrev) * 100

Best Day of Week (Study) = 
TOPN(
    1,
    SUMMARIZE(habit_tracker_6months, habit_tracker_6months[DayOfWeek], "AvgStudy", [Avg Study Hours]),
    [AvgStudy], DESC
)

-- ── MOOD CORRELATION ──────────────────────────────────────────

Avg Mood = AVERAGE(habit_tracker_6months[MoodScore])

High Habit Mood = 
CALCULATE(
    AVERAGE(habit_tracker_6months[MoodScore]),
    habit_tracker_6months[HabitsMet] >= 4
)

Low Habit Mood = 
CALCULATE(
    AVERAGE(habit_tracker_6months[MoodScore]),
    habit_tracker_6months[HabitsMet] <= 2
)

Mood Lift from Habits = [High Habit Mood] - [Low Habit Mood]
```

---

## STEP 4 — Pages & Visuals

### PAGE 1: "Overview" (Summary KPIs)

**Background color:** `#0F0F1A` (near-black, dark theme)
**Accent color:** `#7C3AED` (violet/purple — personal brand color)

**Layout (top row — 4 KPI Cards):**

| Card | Measure | Icon |
|------|---------|------|
| Overall Habit Score | `[Overall Habit Score %]` | ⭐ |
| Avg Sleep | `[Avg Sleep Hours]` | 🌙 |
| Gym Days | `[Gym Days]` | 💪 |
| Max Streak | `[Max Win Streak]` | 🔥 |

→ Use **New Card visual** (2024 style) with callout value + subtitle
→ Set card background: `#1A1A2E`, border: `#7C3AED` (thin, 1px)

**Middle section:**

1. **Line Chart** (full width)
   - X-axis: `Date`
   - Values: `SleepHours`, `StudyHours`, `WaterLitres`
   - Title: "Daily Habit Trends — Jan to Jun 2024"
   - Colors: Sleep=`#818CF8`, Study=`#34D399`, Water=`#60A5FA`

2. **Clustered Bar Chart** (half width)
   - Axis: `Month`
   - Values: `[Sleep Goal Rate]`, `[Study Goal Rate]`, `[Gym Consistency %]`
   - Title: "Monthly Goal Achievement %"

3. **Donut Chart** (half width)
   - Values: Gym Days vs Non-Gym Days
   - Colors: `#7C3AED` vs `#374151`
   - Title: "Gym Consistency"

**Bottom row:**

4. **Gauge** — `[Overall Habit Score %]` | Min: 0, Max: 100, Target: 80
5. **KPI Card** — `[Mood Lift from Habits]` with subtitle "Avg mood on high-habit days vs low-habit days"

---

### PAGE 2: "Deep Dive" (Patterns & Correlations)

1. **Heatmap (Matrix visual)**
   - Rows: `DayOfWeek`
   - Columns: `Month`
   - Values: `[Overall Habit Score %]`
   - Conditional formatting: White-to-purple gradient
   - Title: "Habit Score by Day & Month"

2. **Scatter Chart**
   - X-axis: `SleepHours`
   - Y-axis: `StudyHours`
   - Size: `MoodScore`
   - Legend: `IsWeekend`
   - Title: "Sleep vs Study vs Mood"
   - This is your most impressive visual — shows multi-variable correlation

3. **Area Chart**
   - X-axis: `Date`
   - Values: `ScreenHours` (area, `#EF4444`) vs `StudyHours` (line, `#34D399`)
   - Title: "Screen Time vs Study Time — The Tradeoff"

4. **Bar Chart (horizontal)**
   - Axis: `DayOfWeek`
   - Values: `Avg Study Hours`
   - Sort: Descending
   - Title: "Which Day Do I Study Most?"

---

### PAGE 3: "Streak & Momentum"

1. **Line Chart**
   - X-axis: `Date`
   - Y-axis: `WinStreak`
   - Reference line at 7 (milestone)
   - Title: "Habit Streak Over 6 Months"
   - Fill below line with `#7C3AED` at 30% opacity

2. **Waterfall Chart**
   - Category: `Month`
   - Y-axis: Average `HabitsMet`
   - Shows improvement month-over-month

3. **Card row:** Current Streak / Max Streak / Gym Days / Total Perfect Days (HabitsMet = 5)

---

## STEP 5 — Filters & Slicers

Add to every page (top right):
- **Date Range Slicer** → Between style → `DateTable[Date]`
- **Month Slicer** → Dropdown → `DateTable[Month]`
- **Weekday/Weekend Toggle** → `DateTable[IsWeekend]`

---

## STEP 6 — Design Polish

1. **Theme:** Download "Midnight Blue" theme from community gallery OR use the JSON below:
2. Font: Segoe UI everywhere
3. Enable **Grid lines** in Format → Gridlines → Color: `#1F2937`
4. Add a text box at top of each page with the page title in `#7C3AED`, size 24, bold
5. Add logo box top-left: Your name + "| Personal Analytics" in white

---
