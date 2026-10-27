# AI Weekly Business Review

An AI analyst that writes the weekly sales review (what happened, whether it matters, and what to watch next week), built on [Walmart's public store sales dataset](https://huggingface.co/datasets/large-traversaal/Walmart-sales): 45 stores, Feb 2010 – Oct 2012, originally released for a Kaggle forecasting competition. It's a portfolio project on public data, not client work. Built in Python with the Claude API.

> 🦃 **Week ending 25 Nov 2011:** Holiday-week sales of $66.6M, up 1.2% (+$0.8M) on last year. Holiday-week sales were up 1.2% on last year, within 2 points of the recent 3.1% trend, and the 43% jump from last week reflects the holiday peak.

Sales were 45% above the recent average that week. The brief calls it routine because it matched last Thanksgiving, which is the judgement call this project is about.

## How it works

1. **Roll up** weekly sales across 45 stores (Feb 2010 – Oct 2012).
2. **Compare** each week against last week, the previous 13 weeks, the same week last year, and the recent year-over-year trend.
3. **Track key shopping days** (July 4, Halloween, Christmas Eve, Christmas Day, New Year's Eve) that land in a different week each year and distort comparisons.
4. **Break down** each week by store type, top stores and top departments.
5. **Claude writes the brief**, labelling the week 🟢 routine, 🟡 notable or 🔴 needs attention, with a length limit for each label. It may only use the numbers it's given.
6. **Check automatically:** every $ figure, % and store count in each brief is compared against the source data.

## Results

Tested on Oct–Dec 2011 (used to develop the prompt) and Jul–Sep 2012 (held back, run once at the end).

| | Oct–Dec 2011 | Jul–Sep 2012 (held-out) |
|---|---|---|
| Figures matching source data | 218 / 219 | 170 / 171 |
| Summaries within length limit | 13 / 13 | 13 / 13 |
| Routine / notable / needs attention | 3 / 8 / 2 | 9 / 4 / 0 |

The one real error: a brief added up the wrong three departments ($1.35M instead of $1.39M). The automatic check flagged it.

## Run it

1. Open `ai_weekly_business_review.ipynb` in Google Colab.
2. Add your Anthropic API key as a Colab secret named `ANTHROPIC_API_KEY`.
3. Run with `DRY_RUN = True` (Step 7) to check the prompts at no cost, then set it to `False`. That's about 16 API calls per quarter.
4. Choose the quarter with `REPORT_END` in Step 9.

## Files

| File | Contents |
|---|---|
| `ai_weekly_business_review.ipynb` | The notebook |
| `weekly_business_review_dev.csv` / `_test.csv` | Weekly briefs alongside their numbers, one row per week |
| `weekly_drivers_dev.csv` / `_test.csv` | The store and department detail behind each brief |

<details>
<summary><b>Field definitions</b></summary>

**weekly_business_review**

| Field | Meaning |
|---|---|
| `week_ending` | Friday the week ends on (weeks run Sat–Fri) |
| `is_holiday` | Walmart-flagged holiday week |
| `key_days` / `key_days_ly` | Key shopping days this week / same week last year |
| `revenue_$M`, `prev_week_$M`, `last_year_$M` | Sales this week, last week, same week last year |
| `avg_13wk_$M`, `range_low_$M`, `range_high_$M` | Average, lowest and highest week of the previous 13 |
| `chg_vs_ly_$M` | Dollar change vs last year |
| `wow_pct`, `vs_13wk_pct`, `yoy_pct` | % change vs last week, 13-week average, last year |
| `yoy_trend_13wk` | Average year-over-year % over the previous 13 weeks |
| `vs_trend_pts` | This week's year-over-year % minus that trend, in points |
| `range_position` | Within, above or below the 13-week range |
| `stores_up_vs_ly`, `store_count` | Stores up on last year, out of total |
| `attention_level`, `attention_reason` | Claude's label and why |
| `headline`, `summary`, `key_drivers`, `watch_next_week` | The brief |
| `parse_status` | Whether the reply was read cleanly |
| `summary_sentences`, `length_ok` | Length check (Step 12) |
| `figures_cited`, `figures_matched`, `unmatched_figures` | Number check (Step 12) |

**weekly_drivers**

| Field | Meaning |
|---|---|
| `breakdown`, `item` | Store type / Store / Department, and which one |
| `this_week_$K`, `avg_13wk_$K`, `last_year_$K` | Sales in $ thousands |
| `chg_vs_13wk_$K`, `chg_vs_ly_$K` | Dollar changes |
| `pct_vs_ly`, `share_of_sales_%` | % change vs last year; share of the week's sales |

Store and department rows are the top 5 movers vs last year each week.
</details>

## Limitations and next steps

- **Each week is written on its own,** so a recurring issue (Store 14 was down in 12 of 13 summer weeks) reads as new every time. Next: pass in recent streaks and last week's watch item.
- **Departments have no names,** and calendar effects are likely explanations, not measured ones.
- **The labels don't separate good news from bad.**
- **Next:** add a short-term forecast and automatic Monday delivery.

## License

MIT
