# Error Log: Student Housing Supply Analysis (London Airbnb)

AI tool used: Claude (to draft SQL queries and explain errors).
Every query was run in PostgreSQL by me, and key results were checked against Excel.

| # | What went wrong | How I caught it | Fix | Lesson |
|---|-----------------|-----------------|-----|--------|
| 1 | Price outliers: max listing price was 75,000 per night, median was 175 | Percentile query showed the 99th percentile was about 1,272 | Capped prices at 1,500 (just above p99); this removed 436 listings (0.5%) | Check the distribution before trusting averages |
| 2 | The AI's first suggested price cap was 1,000, which was too low for central London | Compared it with my own p99 result | Used 1,500 based on the data | Don't accept a number from AI without checking it against the data |
| 3 | 32,049 of 91,954 listings (34.9%) had no price | Compared raw rows with clean rows, then counted missing, zero and above-cap rows separately | Excluded them and documented the limitation | Results describe listings that show a price, not all listings |
| 4 | `ERROR: date/time field value out of range: "9/14/2026"` when building the clean table | PostgreSQL error message | Dates were month/day/year text; converted with an explicit format (`to_date(..., 'MM/DD/YYYY')`) instead of relying on the default datestyle | Never rely on default date parsing |
| 5 | `invalid input syntax for type bigint: "1.71258E+18"` | PostgreSQL error message | The file had been damaged by Excel (long IDs turned into scientific notation); kept IDs as text and did not use them in any query | Don't open CSVs in Excel before loading them |
| 6 | The AI's `DISTINCT ON (id)` step would have deleted real listings, because damaged IDs collide | Noticed while fixing error 5 | Removed the de-duplication step, since IDs were unreliable | Check an AI-suggested cleaning step against the data it will run on |
| 7 | Month-over-month `LAG()` query looked like fast growth in listings (1,278 to 14,191) | Questioned why the trend rose so sharply near the scrape date | `last_review` is each listing's latest review, so dates pile up near the scrape date by construction; reported it as "listings by month of latest review", not demand growth | A query can be correct and still mislead |
| 8 | Query 5 (`AVG ... PARTITION BY`) gave area averages far lower than Query 1 | Compared the two outputs | The `WHERE` clause filtered to long-stay private rooms before the window function ran, so it describes that subset only | A `WHERE` clause runs before window functions |
| 9 | Excel check formulas were copied down the whole table, producing many shifted copies | Saw different numbers in rows below the first result | Kept only the first row (correct); used absolute references (`$T$2:$T$91955`) | Use `$` references and convert the table to a range |

## Verification
Results were checked against an independent Excel calculation (all matched):
raw rows 91,954; missing price 32,049; above cap 436; clean rows 59,469;
Westminster average 344.22; Sutton average 117.31; long-stay listings 1,466.
