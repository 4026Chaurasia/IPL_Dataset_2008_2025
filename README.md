# IPL_Dataset_2008_2025
This project dives into Indian Premier League (IPL) finals data, leveraging SQL for deep analysis and Tableau for impactful visual storytelling. The goal is to uncover patterns, venue biases, player performance, and match dynamics that influence outcomes on cricket’s biggest domestic stage.

Dataset
-ipl_with_total_runs.csv — a cleaned and curated ball-by-ball dataset for IPL finals, containing:
-Match ID, teams, season, venue
-Over-level total runs
-Batter, bowler, innings info (if available)
-Match outcomes and metadata

Key Analyses (SQL + Tableau)

-Titles Won – Championship count per team over seasons
-Top Batsmen – Leading scorers in finals history
-Toss Impact – How toss outcomes affect win probabilities by venue
-Venue Bias – Bowling or batting-friendly finals grounds
-Run Trends – 1st vs 2nd innings run patterns across venues
-Winning Margins – Trends in close vs dominant final wins
-Player Impact – High-pressure performance insights
-Finals Evolution – Are finals becoming more high-scoring over time?

Tools Used
-SQL (BigQuery) – Data modeling, aggregation, and analysis
-Tableau – Interactive dashboards and visual storytelling

Business Use Cases
-Strategic insights for broadcasters, teams, and analysts
-Enhanced pre-match commentary and predictions
-Fan engagement via storytelling dashboards


-- IPL FINALS INSIGHTS – BUSINESS OVERVIEW

-- BUSINESS QUESTIONS & INSIGHTS
-- This workbook answers key championship-level questions from IPL Finals (2008–2025)
-- using only SQL queries on the IPL ball-by-ball data with match metadata.

-- Q1: Titles Won by Each Team
--   Input: Final match winners
--   Output: Total IPL titles per team
--   Use: Brand value, consistency, fan engagement

-- Q2: Top Batsmen in Finals
--   Input: Batter-level run data from finals
--   Output: Top 10 highest run-scorers in finals
--   Use: Identify big-match players and performance pressure handlers

-- Q3: Most Wickets in Finals
--   Input: Bowler wicket data
--   Output: Bowlers with most wickets in finals
--   Use: Highlight top death/powerplay performers in high-stake games

-- Q4: Toss Winner vs Match Winner
--   Input: Toss decision and match result
--   Output: Correlation between toss and winning
--   Use: Toss bias analysis and strategy optimization

-- Q5: Death Overs Run Rate (Overs 16–20)
--   Input: Over-wise runs
--   Output: Run rate in death overs by team
--   Use: Finishing strength analysis

-- Q6: Powerplay Wickets vs Match Outcome
--   Input: Powerplay wickets lost
--   Output: Impact of early collapse on final match outcome
--   Use: Risk control in first 6 overs

-- Q7: Finals Wins by Venue
--   Input: Match venue and winner
--   Output: Finals hosted and wins by venue
--   Use: Venue bias, fan base leverage, BCCI planning

-- Q8: Total Runs Scored by Venue
--   Input: Runs scored in finals per venue
--   Output: Aggregate and average scoring trends
--   Use: Pitch behavior, fan excitement, broadcast planning

-- Q9: Venue Popularity Over Years
--   Input: Year-wise venue selection for finals
--   Output: Hosting trends, repeated venues, COVID impact
--   Use: BCCI rotation policy, regional balance, infrastructure use


-- IPL Finals Insights
-- Dataset: `ipl_dataset.ipl_ball_by_ball`

--Q 0: Create Synthetic Match ID and Filter Final Matches

-- Get the final match date per season
WITH final_dates AS (
  SELECT season, MAX(date) AS final_date
  FROM `ipl-data-analysis-465918.ipl_dataset.ipl_dataset_ball_by_ball`
  GROUP BY season
),
-- Join to get only final match rows
final_matches AS (
  SELECT b.*
  FROM `ipl-data-analysis-465918.ipl_dataset.ipl_dataset_ball_by_ball` b
  JOIN final_dates f
    ON b.season = f.season AND b.date = f.final_date
),
-- Add synthetic match_id
finals_with_match_id AS (
  SELECT
    CONCAT(CAST(season AS STRING), '_', date, '_', team1, '_vs_', team2) AS match_id,
    *
  FROM final_matches
)
Final result: show final match data
SELECT *
FROM finals_with_match_id;

-- Q1: Titles Won by Each Team
WITH final_dates AS (
  SELECT season, MAX(date) AS final_date
  FROM `ipl_dataset.ipl_ball_by_ball`
  GROUP BY season
),
final_matches AS (
  SELECT b.*
  FROM `ipl_dataset.ipl_ball_by_ball` b
  JOIN final_dates f ON b.season = f.season AND b.date = f.final_date
),
finals_with_match_id AS (
  SELECT
    CONCAT(CAST(season AS STRING), '_', date, '_', team1, '_vs_', team2) AS match_id,
    *
  FROM final_matches
)
SELECT
  winner AS team,
  COUNT(DISTINCT match_id) AS titles_won
FROM finals_with_match_id
WHERE winner IS NOT NULL
GROUP BY winner
ORDER BY titles_won DESC;

--Q 2: Top Batsmen in Finals (by Total Runs)

-- Step 1: Identify final match date for each season
WITH final_dates AS (
  SELECT season, MAX(date) AS final_date
  FROM `ipl-data-analysis-465918.ipl_dataset.ipl_dataset_ball_by_ball`
  GROUP BY season
),

-- Step 2: Join to get final match rows
final_matches AS (
  SELECT b.*
  FROM `ipl-data-analysis-465918.ipl_dataset.ipl_dataset_ball_by_ball` b
  JOIN final_dates f
    ON b.season = f.season AND b.date = f.final_date
),

-- Step 3: Create synthetic match_id
finals_with_match_id AS (
  SELECT
    CONCAT(CAST(season AS STRING), '_', date, '_', team1, '_vs_', team2) AS match_id,
    *,
    SAFE_CAST(JSON_EXTRACT_SCALAR(runs, '$.batter') AS INT64) AS batter_runs
  FROM final_matches
)

-- Step 4: Top batsmen by runs scored in finals
SELECT
  batter,
  SUM(batter_runs) AS total_runs
FROM finals_with_match_id
GROUP BY batter
ORDER BY total_runs DESC;

--Q3 - Top Bowlers in Finals (by Wickets Taken)

-- Step 1: Identify final match date per season
WITH final_dates AS (
  SELECT season, MAX(date) AS final_date
  FROM `ipl-data-analysis-465918.ipl_dataset.ipl_dataset_ball_by_ball`
  GROUP BY season
),

-- Step 2: Join to get only final matches
final_matches AS (
  SELECT b.*
  FROM `ipl-data-analysis-465918.ipl_dataset.ipl_dataset_ball_by_ball` b
  JOIN final_dates f
    ON b.season = f.season AND b.date = f.final_date
),

-- Step 3: Create synthetic match_id and flag wickets
finals_with_match_id AS (
  SELECT
    CONCAT(CAST(season AS STRING), '_', date, '_', team1, '_vs_', team2) AS match_id,
    *,
    CASE 
      WHEN wickets IS NOT NULL AND TRIM(wickets) != '' THEN 1 
      ELSE 0 
    END AS wicket_flag
  FROM final_matches
)

-- Step 4: Aggregate wickets per bowler
SELECT
  bowler,
  SUM(wicket_flag) AS wickets
FROM finals_with_match_id
GROUP BY bowler
ORDER BY wickets DESC;

--Q4 Toss Impact by Venue: Does winning toss help win match?

WITH ranked_deliveries AS (
  SELECT
    match_id,
    venue,
    toss_winner,
    winner AS winning_team,
    ROW_NUMBER() OVER (PARTITION BY match_id ORDER BY CAST(overs AS FLOAT64) DESC) AS row_num
  FROM `ipl-data-analysis-465918.ipl_dataset.ipl_dataset_ball_by_ball`
),
last_balls AS (
  SELECT *
  FROM ranked_deliveries
  WHERE row_num = 1
)
SELECT
  venue,
  COUNT(*) AS total_matches,
  SUM(CASE WHEN toss_winner = winning_team THEN 1 ELSE 0 END) AS toss_win_match_win,
  ROUND(SUM(CASE WHEN toss_winner = winning_team THEN 1 ELSE 0 END) * 100.0 / COUNT(*), 2) AS toss_win_bias_pct
FROM last_balls
GROUP BY venue
ORDER BY toss_win_bias_pct DESC;

-- Q5: Run Rate in Death Overs (16–20)
WITH final_dates AS (
  SELECT season, MAX(date) AS final_date
  FROM `ipl-data-analysis-465918.ipl_dataset.ipl_dataset_ball_by_ball`
  GROUP BY season
),
final_matches AS (
  SELECT b.*
  FROM `ipl-data-analysis-465918.ipl_dataset.ipl_dataset_ball_by_ball` b
  JOIN final_dates f ON b.season = f.season AND b.date = f.final_date
),
finals_with_match_id AS (
  SELECT
    *,
    SAFE_CAST(JSON_EXTRACT_SCALAR(runs, '$.total') AS INT64) AS total_runs
  FROM final_matches
),
death_overs AS (
  SELECT *
  FROM finals_with_match_id
  WHERE overs BETWEEN 16 AND 20
)
SELECT
  batting_team,
  ROUND(SUM(total_runs) / COUNT(DISTINCT overs), 2) AS death_over_run_rate
FROM death_overs
GROUP BY batting_team
ORDER BY death_over_run_rate DESC;

-- Q6: Early wickets and match result
WITH final_dates AS (
  SELECT season, MAX(date) AS final_date
  FROM `ipl-data-analysis-465918.ipl_dataset.ipl_dataset_ball_by_ball`
  GROUP BY season
),
final_matches AS (
  SELECT b.*
  FROM `ipl-data-analysis-465918.ipl_dataset.ipl_dataset_ball_by_ball` b
  JOIN final_dates f ON b.season = f.season AND b.date = f.final_date
),
finals_with_match_id AS (
  SELECT
    CONCAT(CAST(season AS STRING), '_', date, '_', team1, '_vs_', team2) AS match_id,
    *,
    CASE 
      WHEN wickets IS NOT NULL AND TRIM(wickets) != '' THEN 1 
      ELSE 0 
    END AS wicket_flag
  FROM final_matches
),
powerplay_wickets AS (
  SELECT
    match_id,
    season,
    batting_team,
    SUM(wicket_flag) AS wickets_in_powerplay
  FROM finals_with_match_id
  WHERE overs BETWEEN 1 AND 6
  GROUP BY match_id, season, batting_team
),
match_winners AS (
  SELECT DISTINCT match_id, winner
  FROM finals_with_match_id
)
SELECT
  p.batting_team,
  p.wickets_in_powerplay,
  m.winner,
  COUNT(*) AS match_count
FROM powerplay_wickets p
JOIN match_winners m ON p.match_id = m.match_id
GROUP BY p.batting_team, p.wickets_in_powerplay, m.winner
ORDER BY p.wickets_in_powerplay ASC;

--Q7: Wins by Venue in Finals

WITH final_dates AS (
  SELECT season, MAX(date) AS final_date
  FROM `ipl-data-analysis-465918.ipl_dataset.ipl_dataset_ball_by_ball`
  GROUP BY season
),
final_matches AS (
  SELECT b.*
  FROM `ipl-data-analysis-465918.ipl_dataset.ipl_dataset_ball_by_ball` b
  JOIN final_dates f ON b.season = f.season AND b.date = f.final_date
),
finals_with_match_id AS (
  SELECT
    CONCAT(CAST(season AS STRING), '_', date, '_', team1, '_vs_', team2) AS match_id,
    *
  FROM final_matches
)
SELECT
  venue,
  winner,
  COUNT(DISTINCT match_id) AS finals_won
FROM finals_with_match_id
GROUP BY venue, winner
ORDER BY venue, finals_won DESC;

-- Q8: Venue Popularity Trend Over Time

WITH final_dates AS (
  SELECT season, MAX(date) AS final_date
  FROM `ipl-data-analysis-465918.ipl_dataset.ipl_dataset_ball_by_ball`
  GROUP BY season
),
final_matches AS (
  SELECT b.*
  FROM `ipl-data-analysis-465918.ipl_dataset.ipl_dataset_ball_by_ball` b
  JOIN final_dates f ON b.season = f.season AND b.date = f.final_date
),
finals_with_match_id AS (
  SELECT
    CONCAT(CAST(season AS STRING), '_', date, '_', team1, '_vs_', team2) AS match_id,
    *
  FROM final_matches
)
SELECT
  season,
  venue,
  COUNT(DISTINCT match_id) AS finals_hosted
FROM finals_with_match_id
GROUP BY season, venue
ORDER BY season, finals_hosted DESC;

-- Q9: Total Runs Scored by Venue in Finals

WITH final_dates AS (
  SELECT season, MAX(date) AS final_date
  FROM `ipl-data-analysis-465918.ipl_dataset.ipl_dataset_ball_by_ball`
  GROUP BY season
),
final_matches AS (
  SELECT b.*
    FROM `ipl-data-analysis-465918.ipl_dataset.ipl_dataset_ball_by_ball`
 b
  JOIN final_dates f ON b.season = f.season AND b.date = f.final_date
),
finals_with_runs AS (
  SELECT
    venue,
    SAFE_CAST(JSON_EXTRACT_SCALAR(runs, '$.total') AS INT64) AS total_runs
  FROM final_matches
  WHERE runs IS NOT NULL
)
SELECT
  venue,
  SUM(total_runs) AS total_runs_scored,
  COUNT(*) AS balls_counted,
  ROUND(SUM(total_runs) / COUNT(*), 2) AS avg_runs_per_ball
FROM finals_with_runs
GROUP BY venue
ORDER BY total_runs_scored DESC;

-- Q10: Venue Popularity Trend Over Time

WITH final_dates AS (
  SELECT season, MAX(date) AS final_date
  FROM `ipl-data-analysis-465918.ipl_dataset.ipl_dataset_ball_by_ball`
  GROUP BY season
),
final_matches AS (
  SELECT b.*
    FROM `ipl-data-analysis-465918.ipl_dataset.ipl_dataset_ball_by_ball`
 b
  JOIN final_dates f ON b.season = f.season AND b.date = f.final_date
),
finals_with_runs AS (
  SELECT
    venue,
    SAFE_CAST(JSON_EXTRACT_SCALAR(runs, '$.total') AS INT64) AS total_runs
  FROM final_matches
  WHERE runs IS NOT NULL
)
SELECT
  venue,
  SUM(total_runs) AS total_runs_scored,
  COUNT(*) AS balls_counted,
  ROUND(SUM(total_runs) / COUNT(*), 2) AS avg_runs_per_ball
FROM finals_with_runs
GROUP BY venue
ORDER BY total_runs_scored DESC;

-- Venue Win Bias: Bat First vs Chase

WITH match_with_last_ball AS (
  SELECT
    *,
    ROW_NUMBER() OVER (PARTITION BY match_id ORDER BY CAST(overs AS FLOAT64) DESC) AS row_num
  FROM `ipl-data-analysis-465918.ipl_dataset.ipl_dataset_ball_by_ball`
),
last_ball_matches AS (
  SELECT
    match_id,
    venue,
    team1 AS first_batting_team,
    team2 AS second_batting_team,
    winner AS winning_team
  FROM match_with_last_ball
  WHERE row_num = 1
),
venue_win_bias AS (
  SELECT
    venue,
    COUNT(*) AS total_matches,
    SUM(CASE WHEN winning_team = first_batting_team THEN 1 ELSE 0 END) AS bat_first_wins,
    SUM(CASE WHEN winning_team = second_batting_team THEN 1 ELSE 0 END) AS chase_wins
  FROM last_ball_matches
  GROUP BY venue
)
SELECT
  venue,
  total_matches,
  bat_first_wins,
  chase_wins,
  ROUND(bat_first_wins * 100.0 / total_matches, 2) AS bat_first_win_pct,
  ROUND(chase_wins * 100.0 / total_matches, 2) AS chase_win_pct
FROM venue_win_bias
ORDER BY chase_win_pct DESC;
