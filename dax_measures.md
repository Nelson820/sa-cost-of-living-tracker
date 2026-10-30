# DAX measures

Measures and calculated tables used in `dashboard.pbix`. Last updated 30 October 2026.

## Date table

### DateTable

Calculated table, marked as the date table and related to the CPI table.

```
DateTable =
ADDCOLUMNS(
    CALENDAR(DATE(2008, 1, 1), DATE(2026, 12, 31)),
    "Year", YEAR([Date])
)
```

### MonthStart

Calculated column on `DateTable`. Used as the x-axis of the prime rate vs inflation chart.

```
MonthStart = DATE(YEAR(DateTable[Date]), MONTH(DateTable[Date]), 1)
```

## What-if parameters (Household impact page)

Each parameter is a numeric range table with a selected-value measure.

```
Loan Amount = GENERATESERIES(500000, 5000000, 50000)
```

```
Loan Term = GENERATESERIES(5, 30, 1)
```

```
Loan Term Value = SELECTEDVALUE('Loan Term'[Loan Term], 20)
```

```
Years Ago = GENERATESERIES(1, 8, 1)
```

```
Years Ago Value = SELECTEDVALUE('Years Ago'[Years Ago], 2)
```

`Loan Amount Value` is the matching `SELECTEDVALUE` measure for the loan amount slicer (default 1,500,000).

## Prime rate measures (Rates table)

### Current Prime Rate

Prime rate from the most recent rate change in the table.

```
Current Prime Rate =
VAR LatestChange = CALCULATE(MAX(Rates[effective_date]), ALL(Rates))
RETURN
DIVIDE(
    CALCULATE(MAX(Rates[prime_rate_pct]), ALL(Rates), Rates[effective_date] = LatestChange),
    100
)
```

### Prime Rate in Effect

Prime rate in force on the date in the current filter context. Drives the prime rate line on the chart.

```
Prime Rate in Effect =
VAR d = MAX(DateTable[Date])
VAR ChangeDate =
    CALCULATE(MAX(Rates[effective_date]), ALL(Rates), Rates[effective_date] <= d)
RETURN
DIVIDE(
    CALCULATE(MAX(Rates[prime_rate_pct]), ALL(Rates), Rates[effective_date] = ChangeDate),
    100
)
```

### Prime Rate Years Ago

Prime rate in force on today's date, minus the number of years chosen on the Years Ago slider.

```
Prime Rate Years Ago =
VAR CompareDate = EDATE(TODAY(), -12 * [Years Ago Value])
VAR ChangeDate =
    CALCULATE(MAX(Rates[effective_date]), ALL(Rates), Rates[effective_date] <= CompareDate)
RETURN
DIVIDE(
    CALCULATE(MAX(Rates[prime_rate_pct]), ALL(Rates), Rates[effective_date] = ChangeDate),
    100
)
```

## Repayment measures (Household impact page)

These three are wrapped in `FORMAT()` so the cards show full rand values instead of "K". The result is text, so they are for display only. For charts or further calculations, use the `PMT` expression without `FORMAT()`.

### Monthly Repayment Now

```
Monthly Repayment Now =
FORMAT(
    -PMT([Current Prime Rate] / 12, [Loan Term Value] * 12, [Loan Amount Value]),
    "R #,##0"
)
```

### Monthly Repayment Then

```
Monthly Repayment Then =
FORMAT(
    -PMT([Prime Rate Years Ago] / 12, [Loan Term Value] * 12, [Loan Amount Value]),
    "R #,##0"
)
```

### Repayment Change per Month

Now minus then. A negative value means the repayment is lower than it was.

```
Repayment Change per Month =
FORMAT(
    ( -PMT([Current Prime Rate] / 12, [Loan Term Value] * 12, [Loan Amount Value]) )
    - ( -PMT([Prime Rate Years Ago] / 12, [Loan Term Value] * 12, [Loan Amount Value]) ),
    "R #,##0;-R #,##0"
)
```

## Cost of living page (CPI table)

### YoY Inflation %

Year-on-year change in the average CPI index compared with the same period a year earlier. Returns blank if either period has no data.

```
YoY Inflation % =
VAR CurrentIndex = AVERAGE(CPI[CPI Index])
VAR PriorIndex =
    CALCULATE(
        AVERAGE(CPI[CPI Index]),
        SAMEPERIODLASTYEAR(DateTable[Date])
    )
RETURN
    IF(
        ISBLANK(CurrentIndex) || ISBLANK(PriorIndex),
        BLANK(),
        DIVIDE(CurrentIndex - PriorIndex, PriorIndex)
    )
```

### Latest Headline Inflation %

Headline year-on-year inflation for the most recent month in the data. Feeds the 4,4% card.

```
Latest Headline Inflation % =
VAR LatestDate =
    CALCULATE(
        MAX(CPI[Date]),
        REMOVEFILTERS(CPI),
        REMOVEFILTERS(DateTable)
    )
RETURN
CALCULATE(
    [YoY Inflation %],
    REMOVEFILTERS(DateTable),
    REMOVEFILTERS(CPI),
    CPI[Category] = "CPI Headline",
    DateTable[Date] = LatestDate
)
```

### Latest CPI Index

Headline CPI index (December 2024 = 100) for the most recent month. Feeds the 107,7 card.

```
Latest CPI Index =
VAR LatestDate =
    CALCULATE(
        MAX(CPI[Date]),
        REMOVEFILTERS(CPI),
        REMOVEFILTERS(DateTable)
    )
RETURN
CALCULATE(
    AVERAGE(CPI[CPI Index]),
    REMOVEFILTERS(DateTable),
    REMOVEFILTERS(CPI),
    CPI[Category] = "CPI Headline",
    DateTable[Date] = LatestDate
)
```

## Notes

- "Years ago" counts back from today's date, so the Household impact figures change over time.
- The repayment calculation assumes the bond rate equals prime, a fixed term, and no fees or insurance.
