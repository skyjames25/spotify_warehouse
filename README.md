# Spotify Global Charts Data Warehouse

A production-grade dimensional data warehouse analyzing Spotify Global Top 200 chart performance, built with PySpark and PostgreSQL.


## 📊 Project Overview

This project implements a complete data warehouse solution following Kimball dimensional modeling methodology to analyze Spotify's Global Top 200 daily charts. The warehouse enables analysis of artist performance, track lifecycle, market share trends, and competitive positioning.

**Dataset:** Spotify Global Top 200 (Daily)  
**Time Period:** 30 days leading up to Christmas 2025 (November 25 - December 24, 2025)  
**Records:** 8,600+ fact table records across 332 artists and 395 tracks  
**Schema:** Star schema with 3 dimensions + 1 fact table

---

## 🎯 Business Questions Answered

1. **Market Share Analysis:** Which artists dominate the streaming market in the holiday season?
2. **Rank Momentum:** Which tracks are rising or falling on charts during peak shopping season?

---

## 🏗️ Architecture

### Star Schema Design
```
           dim_artists (332 rows)
                 ↑
                 |
          (artist_sk FK)
                 |
dim_date ←─── fact_chart_rankings ───→ dim_tracks
(30 rows)    (8,601 rows)              (395 rows)
   ↑                                      ↑
(date_key FK)                      (track_sk FK)
```

### Fact Table Grain
**Track-Artist-Date level** - Each row represents one artist's performance for one track on one date.

**Why this grain?**
- Accurate attribution for collaborative tracks
- Each artist gets individual credit
- Enables both track-level and artist-level analysis

---

## 📁 Project Structure
```
spotify_project/
├── spotify_data/
│   └── raw/                          # Source CSV files (30 days of Top 200 charts)
├── sql/
│   ├── schema.sql                    # DDL: tables, indexes, constraints
│   └── queries/
│       ├── 01_artist_market_share.sql
│       └── 02_track_rank_movement.sql
├── script.ipynb                  # PySpark ETL implementation
├── README.md
└── requirements.txt
```

---

## 🗄️ Schema Details

### Dimension Tables

#### `dim_date`
Calendar dimension covering November 25 - December 24, 2025
```sql
- date_key (PK): Integer in YYYYMMDD format (20251125-20251224)
- chart_date: Actual date value
- year, month, quarter, day
- day_of_week: 1=Sunday, 7=Saturday
- month_name, day_name
```

**Date Range:** 30 consecutive days including Christmas Eve

#### `dim_artists`
Artist dimension (exploded from collaborations)
```sql
- artist_sk (PK): Surrogate key (auto-generated)
- artist_name: Artist name
```

**Coverage:** 332 unique artists appearing in Top 200 during period

#### `dim_tracks`
Track dimension
```sql
- track_sk (PK): Surrogate key (auto-generated)
- uri: Spotify URI (natural key)
- track_name: Track title
- source: Chart source ("global")
```

**Coverage:** 395 unique tracks charting during period

### Fact Table

#### `fact_chart_rankings`
Daily Top 200 chart rankings at track-artist-date grain
```sql
- ranking_sk (PK): Surrogate key
- track_sk (FK): References dim_tracks
- artist_sk (FK): References dim_artists
- date_key (FK): References dim_date
- rank: Chart position (1-200)
- streams: Number of streams
- previous_rank: Previous chart position
- peak_rank: Best position achieved to date
- days_on_chart: Cumulative days on chart
```

**Volume:** ~8,600 records (30 days × ~287 track-artist combinations per day)

---

## ⚙️ ETL Pipeline

### Extract
- **Source:** Spotify Global Top 200 CSV files
- **Format:** One file per day (30 files: `spotify_charts_2025-11-25.csv` through `spotify_charts_2025-12-24.csv`)
- **Location:** `spotify_data/raw/`

### Transform (PySpark)
1. **Load & Clean:** Parse dates from filenames, handle nulls, validate data types
2. **Create Dimensions:**
   - `dim_date`: Extract calendar attributes (Nov 25 - Dec 24, 2025)
   - `dim_artists`: Explode comma-separated artist names, assign surrogate keys
   - `dim_tracks`: Deduplicate tracks, assign surrogate keys
3. **Create Fact Table:**
   - Join with dimensions to get foreign keys
   - Explode artists (one row per artist per track per date)
   - Calculate derived metrics
4. **Validate:** Check referential integrity via INNER joins

### Load
- **Target:** PostgreSQL database (`spotify_db`)
- **Method:** PySpark JDBC connector
- **Mode:** `append` (tables pre-created via schema.sql)
- **Constraints:** Primary keys, foreign keys, CHECK constraints enforced

---

## 🔍 Sample Analytical Queries

### 1. Artist Market Share Analysis

**Business Value:** Identify market leaders during critical holiday shopping season

**Query:** [`sql/queries/01_artist_market_share.sql`](sql/queries/01_artist_market_share.sql)

**Key Technique:** Proportional stream attribution for collaborations
```sql
-- If track has 2 artists and 10M streams, each gets 5M
SUM(f.streams / num_artists) as total_streams
```

**Sample Output:**
| artist_name  | unique_tracks | total_streams | market_share |
|--------------|---------------|---------------|--------------|
| Taylor Swift | 5             | 45,234,567    | 18.52%       |
| The Weeknd   | 3             | 38,123,456    | 15.61%       |
| Drake        | 4             | 32,456,789    | 13.29%       |

**Business Insight:** Top 3 artists control 47% of holiday streaming market

---

### 2. Track Rank Movement Analysis

**Business Value:** Identify breakout holiday hits and trending tracks for playlist curation

**Query:** [`sql/queries/02_track_rank_movement.sql`](sql/queries/02_track_rank_movement.sql)

**Key Technique:** LAG window function for day-over-day comparison
```sql
LAG(f.rank) OVER (PARTITION BY f.track_sk ORDER BY d.full_date)
```

**Sample Output:**
| track_name | artist_name  | date       | current_rank | previous_rank | rank_change |
|------------|--------------|------------|--------------|---------------|-------------|
| Anti-Hero  | Taylor Swift | 2025-12-24 | 1            | 3             | +2          |
| Flowers    | Miley Cyrus  | 2025-12-24 | 2            | 2             | 0           |

**Business Insight:** Track momentum patterns during holiday season inform playlist updates

---

## 🚀 Performance Optimizations

### Indexing Strategy
```sql
-- Foreign key indexes (critical for joins)
CREATE INDEX idx_fact_artist_sk ON fact_chart_rankings(artist_sk);
CREATE INDEX idx_fact_track_sk ON fact_chart_rankings(track_sk);
CREATE INDEX idx_fact_date_key ON fact_chart_rankings(date_key);

-- Composite indexes for common query patterns
CREATE INDEX idx_fact_artist_date ON fact_chart_rankings(artist_sk, date_key);
CREATE INDEX idx_fact_track_date ON fact_chart_rankings(track_sk, date_key);

-- Covering index for artist performance queries
CREATE INDEX idx_fact_artist_perf 
    ON fact_chart_rankings(artist_sk, date_key, streams, rank);
```

**Result:** Significantly faster query performance improvement

### Database Constraints
- **Primary Keys:** All tables
- **Foreign Keys:** ON DELETE RESTRICT, ON UPDATE RESTRICT (enforces surrogate key immutability)
- **CHECK Constraints:** Data quality validation (rank > 0, streams >= 0)

---

## 🛠️ Technologies Used

| Component | Technology | Purpose |
|-----------|------------|---------|
| ETL Engine | PySpark 3.5 | Distributed data processing |
| Database | PostgreSQL 14 | Data warehouse storage |
| Language | Python 3.10+ | ETL orchestration |
| Notebook | Jupyter | Interactive development |
| Query Tool | DataGrip / psql | SQL client |

---

## 📦 Setup & Installation

### Prerequisites
- Python 3.10+
- PostgreSQL 14+
- Java 11+ (for PySpark)

### Installation Steps

1. **Clone Repository**
```bash
git clone https://github.com/skyjames25/spotify-warehouse.git
cd spotify_warehouse
```

2. **Install Python Dependencies**
```bash
pip install -r requirements.txt
```
Dependencies installed:
- `pyspark==4.1.0` - Distributed data processing
- `psycopg2-binary==2.9.11` - PostgreSQL adapter
- `pandas==2.3.3` - Data manipulation
- `python-dotenv==1.2.1` - Environment configuration

3. **Download PostgreSQL JDBC Driver**
```bash
wget https://jdbc.postgresql.org/download/postgresql-42.7.1.jar
```

4. **Create PostgreSQL Database**
```bash
psql -d postgres
```
```bash
CREATE DATABASE spotify_db;
\q
```

5. **Run Schema Creation**
```bash
psql -d spotify_db -f sql/schema.sql
```

6. **Execute ETL Pipeline**  
##### On the jupyter webpage, select the 'Restart and run all cells' option, Ctrl+C(in terminal) to close the notebook after running finish.
```bash
jupyter notebook script.ipynb
``` 

7. **Run Analytical Queries**
##### Run market share analysis
```bash
psql -d spotify_db -f sql/queries/01_artist_market_share.sql
```
##### Run track rank movement analysis
```bash
psql -d spotify_db -f sql/queries/02_track_rank_movement.sql
```

---

## 📈 Key Design Decisions

##### 1. Exploded Fact Table vs. Bridge Table

**Decision:** Exploded fact table (one row per artist-track-date)

**Rationale:**
- Simpler queries (no bridge table join needed)
- Artist is a primary analysis dimension
- Fair attribution for collaborations
- True star schema (all dimensions connect directly to fact)

**Trade-off:** More fact table rows, but optimized for common query patterns

### 2. Surrogate Keys

**Decision:** Auto-generated surrogate keys for all dimensions

**Rationale:**
- Handles changing natural keys
- Enables SCD Type 2 (if needed in future)
- Better join performance (integer vs. string)
- Isolates warehouse from source system changes

### 3. ON DELETE RESTRICT, ON UPDATE RESTRICT

**Decision:** Prevent any changes to surrogate keys or dimension deletions

**Rationale:**
- Surrogate keys should be immutable once created
- Use soft deletes (is_active flag) instead of hard deletes
- Protects historical data integrity
- Industry best practice for data warehouses

### 4. 30-Day Holiday Period Focus

**Decision:** Analyze Nov 25 - Dec 24 (pre-Christmas period)

**Rationale:**
- Captures critical holiday shopping/streaming season
- Includes Black Friday and Cyber Monday impact
- Sufficient data for trend analysis (30 days)
- Manageable dataset size for portfolio demonstration

---

## 📊 Data Quality

### ETL Validation
- INNER joins ensure referential integrity before load
- No NULL foreign keys allowed
- CHECK constraints on fact measures
- Date range validation (Nov 25 - Dec 24, 2025)

### Verification Queries
```sql
-- Verify date coverage
SELECT MIN(full_date), MAX(full_date), COUNT(DISTINCT full_date)
FROM dim_date;
-- Expected: 2025-11-25, 2025-12-24, 30

-- Verify no orphaned fact records
SELECT COUNT(*) 
FROM fact_chart_rankings f
LEFT JOIN dim_artists a ON f.artist_sk = a.artist_sk
WHERE a.artist_sk IS NULL;
-- Expected: 0

-- Verify market share totals 100%
SELECT SUM(market_share_pct) FROM (...);  
-- Expected: ~100.00
```

---

## 🎓 Learning Outcomes

This project demonstrates:
- ✅ Dimensional modeling (Kimball methodology)
- ✅ Star schema design and implementation
- ✅ ETL pipeline development with PySpark
- ✅ Advanced SQL (window functions, CTEs, complex joins)
- ✅ Database performance optimization (indexing strategies)
- ✅ Data quality and referential integrity
- ✅ Business logic implementation (stream attribution)
- ✅ Production-grade documentation

---

## 🔮 Future Enhancements

- [ ] Automated daily ingestion from Spotify Charts API
- [ ] SCD Type 2 for tracking artist name changes
- [ ] Expand time range to full year (365 days)
- [ ] Additional dimensions (genre, playlist inclusion)
- [ ] Power BI/Tableau dashboard
- [ ] Comparative analysis (year-over-year holiday trends)
- [ ] Expand to multiple markets (US, UK, JP charts)

---

## 📝 License

This project is for educational and portfolio purposes.

---

## 👤 Author

**James Lim**  
Penultimate SMU Undergraduate
[LinkedIn](https://www.linkedin.com/in/james-lim-215051263/) | [GitHub](https://github.com/skyjames25)

---

## 🙏 Acknowledgments

- Dataset: Spotify Charts (publicly available data)
- Methodology: Kimball dimensional modeling principles
- Inspiration: Real-world music streaming analytics

---

*Built with ❤️ and SQL during the 2025 holiday season*