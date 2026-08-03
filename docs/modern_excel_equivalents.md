# Classic Excel vs Microsoft 365

The workbook in this repository targets **Excel 2016**. That is a deliberate
constraint, not an accident: a portfolio file that opens with `#NAME?` errors on
the reviewer's machine has failed before anyone reads a formula.

This page shows the same problems solved both ways.

## Why the constraint matters

Functions introduced after Excel 2007 have to be written with an `_xlfn.` prefix
when a file is generated programmatically — `_xlfn.XLOOKUP`, `_xlfn.LET`,
`_xlfn.IFS`. Dynamic-array functions need a second prefix,
`_xlfn._xlws.FILTER`. If the target Excel does not implement the function, the
prefix survives into the saved file and every dependent cell shows `#NAME?`.

That is not a theoretical risk. Building this workbook, the first pass used
`XLOOKUP`, `UNIQUE`, `SORT`, `LET` and `TEXTJOIN`; all five returned `#NAME?`
and had to be rewritten. The classic forms below are what shipped.

## Side by side

### Distinct list of categories

```excel
365      =SORT(UNIQUE(tbl_Sales[category]))
2016     dimension table on a Lookups sheet, maintained upstream
```

The classic version is arguably the better engineering choice regardless of
version: a dimension you control beats a list derived from whatever happens to be
in the fact table this month, and it makes a missing category visible instead of
silently absent.

### Look up a value with a fallback

```excel
365      =XLOOKUP(MAX(B23:B48),B23:B48,A23:A48,"not found")
2016     =IFERROR(INDEX($A$23:$A$48,MATCH(MAX($B$23:$B$48),$B$23:$B$48,0)),"not found")
```

### Two-way lookup

```excel
365      =XLOOKUP($B$64,$A$23:$A$48,XLOOKUP($B$65,$B$22:$G$22,$B$23:$G$48))
2016     =INDEX($B$23:$G$48,MATCH($B$64,$A$23:$A$48,0),MATCH($B$65,$B$22:$G$22,0))
```

`INDEX`/`MATCH`/`MATCH` is barely longer and works in every version anyone has.

### Top 10 by value

```excel
365      =TAKE(SORT(matrix,2,-1),10)
2016     =INDEX($A$23:$A$48,MATCH(LARGE($B$23:$B$48,$A52),$B$23:$B$48,0))
```

The 365 version spills a whole block from one cell. The 2016 version needs a rank
column — which has a side benefit: the rank is visible and auditable.

### Named intermediate steps

```excel
365      =LET(rev,SUM(tbl_Sales[revenue]),
               prof,SUM(tbl_Sales[profit]),
               IF(rev=0,0,prof/rev))
2016     helper cells B5 and B6, then =IFERROR(B6/B5,0)
```

`LET` avoids recomputing `SUM` twice. Helper cells achieve the same thing and are
easier for a reviewer to check, because each step is visible on the sheet.

### Conditional aggregation

```excel
365      =SUMIFS(...)          (unchanged - SUMIFS is from 2007)
2016     =SUMIFS(...)
```

Worth stating plainly: the workhorses — `SUMIFS`, `COUNTIFS`, `AVERAGEIFS`,
`INDEX`, `MATCH`, `SUMPRODUCT`, `PERCENTILE.INC` — are not new and have not
changed. Most real analytical work in Excel needs nothing newer.

### Filter a list by a condition

```excel
365      =FILTER($A$23:$A$48,$D$23:$D$48>0.5,"none")
2016     AutoFilter on the range, or a rank/flag column plus lookup
```

This is the one place where 365 is a genuine step change rather than
convenience — spilled dynamic arrays remove a real class of drudgery.

### Join text from a list

```excel
365      =TEXTJOIN(", ",TRUE,FILTER(range,condition))
2016     helper column with =IF(condition,name&", ","") then =CONCATENATE / &
```

## What to take from this

Knowing `XLOOKUP` matters. Knowing **why the file in front of you does not have
it**, and being able to deliver the same answer anyway, matters more — because
in most companies the finance team is on whatever build the last IT rollout left
them, and the report still has to work.
