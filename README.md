# AI Weekly Business Review

An AI-written weekly sales review for a 45-store retail chain. Every week it produces a short, plain-English brief that covers what happened, whether it matters, and what to keep an eye on next week. It's built in Python on public Walmart sales data, with the Claude API writing the commentary.

**Built with:** Python (pandas, matplotlib) · Claude API (`claude-haiku-5-5`) · Google Colab · Hugging Face

---

## Why I built this

Most retail leadership teams start the week with a business review. An analyst pulls the numbers, compares them with last week and last year, and writes a few paragraphs on what happened. The maths is easy. The hard part is judgement: telling a real change apart from the normal noise of seasons, holidays and calendar quirks.

I wanted to see whether an AI model could write that commentary reliably if it was given the right comparisons and clear rules, and how you'd check that it was getting it right.

## What a brief looks like

Two real examples from the output.

**A routine week (week ending 10 Aug 2012)**
> 🟢 **Sales of $47.4M, up 2.5% (+$1.2M) on last year, in line with trend.**
> Growth matched the recent 2.5% year-over-year trend, and sales stayed inside the 13-week range of $44.1M to $51.3M.
> *Drivers:* Store 14 was the largest drag, down $0.3M (-14.5%) vs last year.
> *Watch next week:* Whether Store 14 stays well below last year after its -14.5% drop.

**Thanksgiving (week ending 25 Nov 2011)**
> 🟢 **Holiday-week sales of $66.6M, up 1.2% (+$0.8M) on last year.**
> Holiday-week sales were up 1.2% on last year, within 2 points of the recent 3.1% trend, and the 43% jump from last week reflects the holiday peak.
> *Drivers:* Dept 72 was the largest drag, down $0.4M (-4.2%) vs last year.
> *Watch next week:* Whether Dept 72 returns to or above last year's level in the post-holiday week.

Thanksgiving sales were 45% above the recent average, which would look alarming on most dashboards. The brief calls the week routine because it was almost exactly in line with last Thanksgiving, and that's the call a good analyst would make.

## Results

I tested it on two quarters. **Oct–Dec 2011** was used to develop and refine the prompt. **Jul–Sep 2012** was held back and only run once the prompt was final, so it's the fair test.

| | Oct–Dec 2011 (development) | Jul–Sep 2012 (held-out test) |
|---|---|---|
| Briefs generated and read cleanly | 13 / 13 | 13 / 13 |
| Summaries within their length limit | 13 / 13 | 13 / 13 |
| Figures that matched the source data | 218 / 219 | 170 / 171 |
| Routine / notable / needs attention | 3 / 8 / 2 | 9 / 4 / 0 |

What stood out:

- **Holidays were read correctly.** Thanksgiving 2011 and Labor Day 2012 were both called routine, and the drop in the week after each was put down to the holiday unwinding, not a slump.
- **It caught the Christmas Eve shift a week early.** In 2010, Christmas Eve fell in the week ending 24 Dec; in 2011 it fell a week later. The brief for the week ending 16 Dec 2011 warned that next week's comparison would look weaker for that reason. The next week came in 4.9% below last year and was explained as calendar timing, not a decline.
- **The quiet quarter looked quiet.** In the held-out summer quarter, 9 of 13 weeks were routine, which matches a steady trading period.
- **The numbers held up.** 388 of 390 figures quoted in the briefs matched the data. Of the two that didn't, one was rounding ("about 20%" for -20.9%). The other was a genuine slip: on 21 Oct 2011 the brief said the top three departments added $1.35M, but they added $1.39M, because it summed the wrong three.

## The data

[Walmart store sales](https://huggingface.co/datasets/large-traversaal/Walmart-sales) (originally from a Kaggle forecasting competition), loaded from Hugging Face.

- Weekly sales for 45 US Walmart stores, broken down by department, from Feb 2010 to Oct 2012 (143 weeks)
- Each store has a type (A = large, B = mid-size, C = small) and a size in square feet
- Walmart flagged four holiday weeks a year: Super Bowl, Labor Day, Thanksgiving and Christmas
- Departments are numbers only (e.g. "Dept 72"), with no names

Only `train.csv` and `stores.csv` are used. The dataset's `test.csv` has no sales figures.

Each date in the data is the Friday that ends the week (Saturday to Friday). The dataset doesn't say this anywhere, so the notebook checks it against the holiday flags before relying on it.

## How it works

### 1. Compare each week against four yardsticks

| Yardstick | Question it answers |
|---|---|
| vs last week | Did something just change? |
| vs the average and range of the previous 13 weeks | Is this normal for recent trading? |
| vs the same week last year | Is this normal for this time of year? |
| vs the recent year-over-year trend | Is this week's growth normal for how the business has been running? |

The last one does most of the work. If the business has been running about 5% ahead of last year for months, another +5% week isn't news. A -1% week is, even though it looks flat: it's a 6-point slowdown.

### 2. Track key shopping days

Holidays tied to a weekday, like Thanksgiving, land in the same week every year. Fixed-date days don't. Christmas Eve can make one week look 5% down and the next 14% up purely because of where it falls. The notebook tracks **July 4, Halloween, Christmas Eve, Christmas Day and New Year's Eve**. It tells Claude when one has moved compared with last year, including for *next* week, so a brief can warn in advance. Next week's key days are calendar facts, not future sales, so nothing is leaked.

### 3. Break the week down

For each week, the notebook builds tables by store type, plus the top 5 stores and top 5 departments that moved most against last year. Each table shows dollars, % change and share of sales. The % change matters because large stores always contribute the most dollars simply because they're the biggest. A change only counts as "concentrated" if one group's *percentage* change is out of line with the rest.

### 4. Claude writes the brief

All of that goes to Claude with a system prompt that sets out how to judge a week. Claude labels the week first, then writes to the length that label allows:

| Label | When | Length |
|---|---|---|
| 🟢 Routine | In line with trend, or explained by a holiday or calendar shift | 1 sentence |
| 🟡 Notable | Clearly ahead of or behind trend with no holiday explanation, or one group moving very differently | 2–3 sentences |
| 🔴 Needs attention | Far off trend with no explanation, or a broad decline | 3–4 sentences |

Ground rules in the prompt: use only the numbers provided, never invent causes (weather, promotions, competitors, the economy), never guess what a numbered department sells, and use the dollar changes given rather than doing its own arithmetic.

### 5. Check the output automatically

- **Length check:** does each summary stay within the limit for its label?
- **Grounding check:** every $ figure, % figure and "N of 45 stores" count in each brief is compared with the numbers Claude was given, allowing for rounding. Anything that doesn't match is listed for a person to check.

A match confirms a number exists in the data. It doesn't prove the number is attached to the right store or department, or points in the right direction. That still needs a quick manual spot-check, which the drivers file makes easy.

## The output files

### `weekly_business_review_*.csv`: one row per week

| Field | Meaning |
|---|---|
| `week_ending` | The Friday the week ends on |
| `is_holiday` | `True` if Walmart flagged it as a holiday week |
| `key_days` / `key_days_ly` | Key shopping days in this week / in the same week last year |
| `revenue_$M` | Total sales this week ($ millions) |
| `prev_week_$M` | Last week's sales |
| `avg_13wk_$M` | Average weekly sales over the previous 13 weeks |
| `range_low_$M` / `range_high_$M` | Lowest and highest week in the previous 13 weeks |
| `last_year_$M` | Sales in the same week last year |
| `chg_vs_ly_$M` | Dollar change vs last year |
| `wow_pct` | % change vs last week |
| `vs_13wk_pct` | % change vs the 13-week average |
| `yoy_pct` | % change vs the same week last year |
| `yoy_trend_13wk` | Recent trend: the average year-over-year % across the previous 13 weeks |
| `vs_trend_pts` | How far this week's year-over-year % is above or below that trend, in percentage points |
| `range_position` | Whether sales were within, above or below the recent 13-week range |
| `stores_up_vs_ly` / `store_count` | Number of stores up on last year / total stores |
| `attention_level` | Claude's label: routine, notable or needs attention |
| `attention_reason` | Why it chose that label |
| `headline` | One-line summary |
| `summary` | The commentary |
| `key_drivers` | The stores, store types or departments behind the result |
| `watch_next_week` | One thing to keep an eye on |
| `parse_status` | Whether Claude's reply was read cleanly (`ok`) |
| `summary_sentences`, `length_ok` | Added by the length check (Step 12 of the notebook) |
| `figures_cited`, `figures_matched`, `unmatched_figures` | Added by the grounding check (Step 12 of the notebook) |

### `weekly_drivers_*.csv`: the detail behind each brief

| Field | Meaning |
|---|---|
| `week_ending` | The week |
| `breakdown` | Store type, Store or Department |
| `item` | e.g. Type A, Store 14, Dept 72 |
| `this_week_$K` | Sales this week ($ thousands) |
| `avg_13wk_$K` | Average weekly sales over the previous 13 weeks |
| `last_year_$K` | Sales in the same week last year |
| `chg_vs_13wk_$K` / `chg_vs_ly_$K` | Dollar change vs the 13-week average / vs last year |
| `pct_vs_ly` | % change vs last year |
| `share_of_sales_%` | This item's share of the week's total sales |

Store and department rows are the top 5 movers against last year each week, not every store and department.

## Running it yourself

1. **Open the notebook in Google Colab:** *File → Open notebook → GitHub*, then paste this repo's URL. Or download the notebook and upload it.
2. **Add your Anthropic API key** as a Colab secret: click the 🔑 icon in the left sidebar, add a secret named `ANTHROPIC_API_KEY`, and turn on notebook access.
3. **Run everything with `DRY_RUN = True`** (the default, set in Step 7). This makes no API calls and costs nothing. It shows the exact prompts Claude would receive, so you can check the numbers first.
4. **Set `DRY_RUN = False`** and re-run from Step 7. One quarter is about 16 API calls.
5. **Pick the quarter** with `REPORT_END` in Step 9: `2011-12-30` for Oct–Dec 2011, or `2012-09-28` for Jul–Sep 2012.

## What's in this repo

| File | What it is |
|---|---|
| `ai_weekly_business_review.ipynb` | The notebook: data, comparisons, prompts, briefs and checks |
| `weekly_business_review_dev.csv` | Weekly briefs alongside their numbers, Oct–Dec 2011 (development quarter) |
| `weekly_business_review_test.csv` | Weekly briefs alongside their numbers, Jul–Sep 2012 (held-out test quarter) |
| `weekly_drivers_dev.csv` / `weekly_drivers_test.csv` | The store and department detail behind each brief, for each quarter |

When you run the notebook yourself, it saves these as `weekly_business_review.csv` and `weekly_drivers.csv`. The field descriptions below apply to both quarters.

## Limitations

- **Departments have no names**, so a brief can say "Dept 72 fell 18%" but not what Dept 72 sells.
- **Calendar effects are likely explanations, not measured ones.** The data is weekly, so it can't show how much Christmas Eve is worth on its own.
- **Each week is written on its own.** Store 14 came up as a drag in 12 of the 13 summer 2012 briefs, each time as if it were new. A person would say "Store 14 has been running 10–29% below last year all quarter."
- **The labels don't separate good news from bad.** Strong, unexplained growth (+6.4% on 21 Oct 2011) is labelled "needs attention" in the same way a slump would be.
- **It's historical data from one retailer** (2010–2012), so it shows the method rather than a live tool.

## What's next

- **Give it a memory of recent weeks.** Work out streaks in code (e.g. "down vs last year 4 weeks running") and pass them in, along with last week's "watch next week" item so the brief can follow up on it.
- **Add a forecast**, so the review also says what the coming weeks are likely to look like.
- **Deliver it automatically** every Monday by email or Slack.

## License

MIT. See `LICENSE`.
