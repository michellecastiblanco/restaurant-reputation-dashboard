# Reputation Intelligence Dashboard
## Michelin-Star Restaurant — Hong Kong

A self-initiated business intelligence project built to demonstrate end-to-end data skills: from database engineering and sentiment analysis to star schema modeling and interactive Power BI reporting.

> Built independently as a portfolio project. The goal: transform unstructured multilingual guest review data into actionable insights for management — using tools typically reserved for enterprise analytics teams.

---

## Project Highlights

- **954 guest reviews** collected from Google Maps and OpenRice
- **Multilingual sentiment analysis** — English, Traditional Chinese, Simplified Chinese
- **Medallion architecture** (bronze/silver) on Supabase/PostgreSQL
- **Star schema** data model in Power BI with 20+ DAX measures
- **5-page interactive dashboard**: Welcome · Overview · Reviews · Insights · Review Detail

---

## Tech Stack

| Layer | Tool |
|---|---|
| Cloud Database | Supabase (PostgreSQL) |
| Data Pipeline | Python · pandas · Helsinki-NLP |
| Data Modeling | Power BI star schema |
| Transformation | Power Query (M language) |
| Reporting | Power BI Desktop |

---

## Architecture

```
Raw Reviews (Google Maps + OpenRice)
        │
        ▼
public.bronze_reviews     ← raw scraped data, untouched
        │
        ▼
public.silver_reviews     ← cleaned + enriched with sentiment scores
        │
        ▼
Power BI Star Schema
├── Fact_Reviews
├── Dim_Date
├── Dim_Platform
├── Dim_Location
├── Dim_Sentiment
├── Dim_Topics          ← calculated table (DAX)
└── _Measures
```

---

## Sentiment Analysis Pipeline

Reviews arrive in three languages. The pipeline:

1. **Language detection** via `detected_language` column in the silver layer
2. **Translation** using Helsinki-NLP/opus-mt-zh-en — a local model (no API cost)
3. **Keyword scoring** across 5 dimensions: Food · Service · Ambiance · Price · Location
4. **Classification**: Positive / Neutral / Negative per dimension, stored in `silver_reviews`

---

## DAX Measures

All measures live in the `_Measures` table, organized in display folders.

### Overview Page — KPI Cards

```dax
-- Average star rating (guest-facing score, 1-5 stars)
Avg Star Score = AVERAGE(Fact_Reviews[Score])

-- Total reviews in selected period
Total Reviews = COUNTROWS(Fact_Reviews)

-- % reviews with 4 or 5 stars
Positive Review Rate =
DIVIDE(
    COUNTROWS(FILTER(Fact_Reviews, Fact_Reviews[Score] >= 4)),
    COUNTROWS(FILTER(Fact_Reviews, NOT ISBLANK(Fact_Reviews[Score])))
)

-- % reviews with 1 or 2 stars
Negative Review Rate =
DIVIDE(
    COUNTROWS(FILTER(Fact_Reviews, Fact_Reviews[Score] <= 2)),
    COUNTROWS(FILTER(Fact_Reviews, NOT ISBLANK(Fact_Reviews[Score])))
)
```

### Overview Page — Alert Cards (dynamic text)

```dax
-- Identifies the strongest performing topic and returns a label with mention count
Top Strength Alert =
VAR _foodPct  = DIVIDE(COUNTROWS(FILTER(Fact_Reviews, Fact_Reviews[food_sentiment]     = "Positive")), [Total Reviews])
VAR _ambPct   = DIVIDE(COUNTROWS(FILTER(Fact_Reviews, Fact_Reviews[ambiance_sentiment] = "Positive")), [Total Reviews])
VAR _svcPct   = DIVIDE(COUNTROWS(FILTER(Fact_Reviews, Fact_Reviews[service_sentiment]  = "Positive")), [Total Reviews])
VAR _foodN    = COUNTROWS(FILTER(Fact_Reviews, NOT ISBLANK(Fact_Reviews[food_sentiment])))
VAR _ambN     = COUNTROWS(FILTER(Fact_Reviews, NOT ISBLANK(Fact_Reviews[ambiance_sentiment])))
VAR _svcN     = COUNTROWS(FILTER(Fact_Reviews, NOT ISBLANK(Fact_Reviews[service_sentiment])))
VAR _best     = MAXX({_foodPct, _ambPct, _svcPct}, [Value])
RETURN
SWITCH(TRUE(),
    _best = _foodPct,  "Food quality — "     & FORMAT(_foodPct, "0%") & " positive across " & _foodN & " mentions",
    _best = _ambPct,   "Ambiance — "         & FORMAT(_ambPct,  "0%") & " positive across " & _ambN  & " mentions",
    _best = _svcPct,   "Service quality — "  & FORMAT(_svcPct,  "0%") & " positive across " & _svcN  & " mentions"
)

-- Identifies the most problematic topic
Top Issue Alert =
VAR _pricePct = DIVIDE(COUNTROWS(FILTER(Fact_Reviews, Fact_Reviews[price_sentiment]   = "Negative")), [Total Reviews])
VAR _svcPct   = DIVIDE(COUNTROWS(FILTER(Fact_Reviews, Fact_Reviews[service_sentiment] = "Negative")), [Total Reviews])
VAR _foodPct  = DIVIDE(COUNTROWS(FILTER(Fact_Reviews, Fact_Reviews[food_sentiment]    = "Negative")), [Total Reviews])
VAR _priceN   = COUNTROWS(FILTER(Fact_Reviews, NOT ISBLANK(Fact_Reviews[price_sentiment])))
VAR _svcN     = COUNTROWS(FILTER(Fact_Reviews, NOT ISBLANK(Fact_Reviews[service_sentiment])))
VAR _foodN    = COUNTROWS(FILTER(Fact_Reviews, NOT ISBLANK(Fact_Reviews[food_sentiment])))
VAR _worst    = MAXX({_pricePct, _svcPct, _foodPct}, [Value])
RETURN
SWITCH(TRUE(),
    _worst = _pricePct, "Pricing — "         & FORMAT(_pricePct, "0%") & " negative across " & _priceN & " mentions",
    _worst = _svcPct,   "Service — "         & FORMAT(_svcPct,   "0%") & " negative across " & _svcN   & " mentions",
    _worst = _foodPct,  "Food quality — "    & FORMAT(_foodPct,  "0%") & " negative across " & _foodN  & " mentions"
)
```

### Insights Page — Topic Sentiment (SWITCH pattern)

```dax
-- Disconnected dimension table — no relationship to Fact_Reviews
Dim_Topics =
DATATABLE(
    "TopicID", INTEGER,
    "Topic",   STRING,
    {
        {1, "Food"},
        {2, "Ambiance"},
        {3, "Service"},
        {4, "Price"},
        {5, "Location"}
    }
)

-- Routes calculation based on slicer selection
Topic Positive % =
VAR _topic = SELECTEDVALUE(Dim_Topics[Topic])
RETURN
SWITCH(_topic,
    "Food",     DIVIDE(COUNTROWS(FILTER(Fact_Reviews, Fact_Reviews[food_sentiment]     = "Positive")), [Total Reviews]),
    "Ambiance", DIVIDE(COUNTROWS(FILTER(Fact_Reviews, Fact_Reviews[ambiance_sentiment] = "Positive")), [Total Reviews]),
    "Service",  DIVIDE(COUNTROWS(FILTER(Fact_Reviews, Fact_Reviews[service_sentiment]  = "Positive")), [Total Reviews]),
    "Price",    DIVIDE(COUNTROWS(FILTER(Fact_Reviews, Fact_Reviews[price_sentiment]    = "Positive")), [Total Reviews]),
    "Location", DIVIDE(COUNTROWS(FILTER(Fact_Reviews, Fact_Reviews[location_sentiment] = "Positive")), [Total Reviews])
)

Topic Neutral % =
VAR _topic = SELECTEDVALUE(Dim_Topics[Topic])
RETURN
SWITCH(_topic,
    "Food",     DIVIDE(COUNTROWS(FILTER(Fact_Reviews, Fact_Reviews[food_sentiment]     = "Neutral")), [Total Reviews]),
    "Ambiance", DIVIDE(COUNTROWS(FILTER(Fact_Reviews, Fact_Reviews[ambiance_sentiment] = "Neutral")), [Total Reviews]),
    "Service",  DIVIDE(COUNTROWS(FILTER(Fact_Reviews, Fact_Reviews[service_sentiment]  = "Neutral")), [Total Reviews]),
    "Price",    DIVIDE(COUNTROWS(FILTER(Fact_Reviews, Fact_Reviews[price_sentiment]    = "Neutral")), [Total Reviews]),
    "Location", DIVIDE(COUNTROWS(FILTER(Fact_Reviews, Fact_Reviews[location_sentiment] = "Neutral")), [Total Reviews])
)

Topic Negative % =
VAR _topic = SELECTEDVALUE(Dim_Topics[Topic])
RETURN
SWITCH(_topic,
    "Food",     DIVIDE(COUNTROWS(FILTER(Fact_Reviews, Fact_Reviews[food_sentiment]     = "Negative")), [Total Reviews]),
    "Ambiance", DIVIDE(COUNTROWS(FILTER(Fact_Reviews, Fact_Reviews[ambiance_sentiment] = "Negative")), [Total Reviews]),
    "Service",  DIVIDE(COUNTROWS(FILTER(Fact_Reviews, Fact_Reviews[service_sentiment]  = "Negative")), [Total Reviews]),
    "Price",    DIVIDE(COUNTROWS(FILTER(Fact_Reviews, Fact_Reviews[price_sentiment]    = "Negative")), [Total Reviews]),
    "Location", DIVIDE(COUNTROWS(FILTER(Fact_Reviews, Fact_Reviews[location_sentiment] = "Negative")), [Total Reviews])
)
```

### Insights Page — Executive Recommendation

```dax
Executive Recommendation =
VAR _avgScore = [Avg Star Score]
VAR _topIssue = [Top Issue Alert]
VAR _topStrength = [Top Strength Alert]
RETURN
IF(
    _avgScore >= 4.5,
    "Outstanding guest satisfaction. Maintain current standards and leverage strengths in marketing: " & _topStrength,
    IF(
        _avgScore >= 4.0,
        "Strong reputation with clear growth opportunity. Key strength: " & _topStrength & ". Priority area: " & _topIssue,
        IF(
            _avgScore >= 3.5,
            "Moderate performance. Prioritize consistency and address: " & _topIssue,
            "Reputation risk. Immediate action needed on: " & _topIssue
        )
    )
)
```

### Reviews Page — Star Rating SVG

```dax
Star Rating SVG =
VAR _score = Fact_Reviews[Score]
VAR _gold = "#C8A84B"
VAR _empty = "#444444"
VAR _star = UNICHAR(9733)
RETURN
"<svg xmlns='http://www.w3.org/2000/svg' width='110' height='20'>" &
    "<text x='0'  y='15' fill='" & IF(_score >= 1, _gold, _empty) & "' font-size='18'>" & _star & "</text>" &
    "<text x='22' y='15' fill='" & IF(_score >= 2, _gold, _empty) & "' font-size='18'>" & _star & "</text>" &
    "<text x='44' y='15' fill='" & IF(_score >= 3, _gold, _empty) & "' font-size='18'>" & _star & "</text>" &
    "<text x='66' y='15' fill='" & IF(_score >= 4, _gold, _empty) & "' font-size='18'>" & _star & "</text>" &
    "<text x='88' y='15' fill='" & IF(_score >= 5, _gold, _empty) & "' font-size='18'>" & _star & "</text>" &
"</svg>"
```

---

## Key Technical Decisions

**Bronze/Silver medallion architecture**
Raw data in `bronze_reviews` is never modified. All enrichment (sentiment scores, translated text, topic classification) lives exclusively in `silver_reviews`. This makes the pipeline reproducible and auditable.

**The Google Dates problem**
Google Maps displays relative dates ("9 years ago") instead of exact timestamps. The raw scraper JSON contains an `iso_date` field with the exact UTC timestamp. Solution: a standalone Power Query table `Google_Dates` that reads directly from Supabase and parses `iso_date` via `Json.Document()` — recovering 838 exact review dates without touching any source table.

**SWITCH pattern for disconnected dimensions**
`Dim_Topics` is a disconnected calculated table (no relationship to `Fact_Reviews`). All topic measures use SWITCH on `SELECTEDVALUE()` to route calculations based on slicer selection — avoiding many-to-many relationships while keeping the model clean.

**Local translation model**
Helsinki-NLP/opus-mt-zh-en runs locally — no API costs, data stays private. Used to translate ~300 Chinese reviews before keyword scoring.

---

## What the Data Revealed

- **Most mentioned dish:** Dim Sum — 95 reviews (63% positive)
- **Top guest concern:** Pricing — 56 mentions
- **Hidden dates:** 838 exact ISO timestamps recovered from raw JSON
- **Review corpus:** 954 total vs 838 publicly visible on Google Maps

---

## About

Built by **Joanne Michelle Castiblanco** — Economist and Analytics Strategist based in Hong Kong. Currently part of the GCI World 2026 program at the University of Tokyo (Matsuo Lab).

Focused on building BI infrastructures that connect raw data to strategic decisions — integrating Power BI, cloud databases, and Generative AI to eliminate the friction between data and leadership.

Also building a parallel star schema system for a **non-profit organization in Colombia** focused on women's digital empowerment (2,216 participant records, 8-dimension model, PostgreSQL → Power BI pipeline).

🔗 [LinkedIn](https://www.linkedin.com/in/joannemichellecastiblancofernandez)
