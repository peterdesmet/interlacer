
<!-- README.md is generated from README.Rmd. Please edit that file -->

# interlacer <img src="man/figures/logo.svg" align="right" height="140" />

[![Project Status: WIP – Initial development is in progress, but there
has not yet been a stable, usable release suitable for the
public.](https://www.repostatus.org/badges/latest/wip.svg)](https://www.repostatus.org/#wip)
[![R-CMD-check](https://github.com/khusmann/interlacer/actions/workflows/check-standard.yaml/badge.svg)](https://github.com/khusmann/interlacer/actions/workflows/check-standard.yaml)
[![codecov](https://codecov.io/gh/khusmann/interlacer/graph/badge.svg?token=R4WNWH5NXU)](https://codecov.io/gh/khusmann/interlacer)

When a value is missing in your data, sometimes you want to know *why*
it is missing. Many textual tabular data sources will encode missing
reasons as special values *interlaced* with the regular values in a
column (e.g. `N/A`, `REFUSED`, `-99`, etc.). Unfortunately, the missing
reasons are lost when these values are all converted into a single `NA`
type. Working with missing reasons in R traditionally requires loading
variables as character vectors and doing a bunch of string comparisons
and type conversions to make sense of them.

interlacer provides functions that load variables from interlaced data
sources into a special `interlaced` column type that holds values and
`NA` reasons in separate *channels* of the same variable. In most
contexts, you can treat `interlaced` columns as if they were regular
values: if you take the `mean` of an interlaced column, for example, you
get the mean of its values, without its missing reasons interfering in
the computation.

Unlike a regular column, however, the missing reasons are still
available. This means you can still filter data frames on variables by
specific missing reasons, or generate summary statistics with breakdowns
by missing reason. In other words, you no longer have to constantly
manually include / exclude missing reasons in computations by filtering
them with awkward string comparisons or type conversions… everything
just works!

In addition to the introduction in `vignette("interlacer")` be sure to
also check out:

- `vignette("coded-data")` for some recipies for working with coded data
  (e.g. data produced by SPSS, SAS or Stata)

- `vignette("other-approaches")` for a deep dive into how interlacer’s
  approach compares to other approaches for representing and
  manipulating missing reasons alongside data values

This library is currently in its experimental stages, so be aware that
its interface is likely to change in the future. In the meantime, please
try it out and [let me know what you think](mailto:kdh38@psu.edu)!

## Installation

The easiest way to get interlacer is to install via devtools:

``` r
install.packages("devtools") # If devtools is not already installed

devtools::install_github("khusmann/interlacer")
```

## Usage

To use interlacer, load it into your current R session:

``` r
library(interlacer)
```

interlacer supports the following file formats with these
`read_interlaced_*()` functions, which extend the `readr::read_*()`
family of functions:

- `read_interlaced_csv()`
- `read_interlaced_tsv()`
- `read_interlaced_csv2()`
- `read_interlaced_delim()`

As a quick demo, consider the following example file bundled with
interlacer:

``` r
library(dplyr, warn.conflicts = FALSE)
library(readr)

read_file(interlacer_example("colors.csv")) |>
  cat()
#> person_id,age,favorite_color
#> 1,20,BLUE
#> 2,REFUSED,BLUE
#> 3,21,REFUSED
#> 4,30,OMITTED
#> 5,1,N/A
#> 6,41,RED
#> 7,50,OMITTED
#> 8,30,YELLOW
#> 9,REFUSED,REFUSED
#> 10,OMITTED,RED
#> 11,10,REFUSED
```

In this csv file, values are interlaced with three possible missing
reasons: `REFUSED`, `OMITTED`, and `N/A`.

With `readr`, loading these data would result in a data frame where all
missing reasons are replaced with `NA`:

``` r
read_csv(
  interlacer_example("colors.csv"),
  na = c("REFUSED", "OMITTED", "N/A"),
  show_col_types = FALSE,
)
#> # A tibble: 11 × 3
#>    person_id   age favorite_color
#>        <dbl> <dbl> <chr>         
#>  1         1    20 BLUE          
#>  2         2    NA BLUE          
#>  3         3    21 <NA>          
#>  4         4    30 <NA>          
#>  5         5     1 <NA>          
#>  6         6    41 RED           
#>  7         7    50 <NA>          
#>  8         8    30 YELLOW        
#>  9         9    NA <NA>          
#> 10        10    NA RED           
#> 11        11    10 <NA>
```

With interlacer, missing reasons are preserved:

``` r
(ex <- read_interlaced_csv(
  interlacer_example("colors.csv"),
  na = c("REFUSED", "OMITTED", "N/A"),
  show_col_types = FALSE,
))
#> # A tibble: 11 × 3
#>    person_id       age favorite_color
#>    <dbl,chr> <dbl,chr> <chr,chr>     
#>  1         1        20 BLUE          
#>  2         2 <REFUSED> BLUE          
#>  3         3        21 <REFUSED>     
#>  4         4        30 <OMITTED>     
#>  5         5         1 <N/A>         
#>  6         6        41 RED           
#>  7         7        50 <OMITTED>     
#>  8         8        30 YELLOW        
#>  9         9 <REFUSED> <REFUSED>     
#> 10        10 <OMITTED> RED           
#> 11        11        10 <REFUSED>
```

As you can see, in the printout above each column is defined by *two*
types: a type for values, and a type for missing reasons. The `age`
column, for example, has type `double` for its values, and type
`character` for its missing reasons:

``` r
ex$age
#> <interlaced<double, character>[11]>
#>  [1] 20        <REFUSED> 21        30        1         41        50       
#>  [8] 30        <REFUSED> <OMITTED> 10
```

Computations automatically operate on values:

``` r
mean(ex$age, na.rm=TRUE)
#> [1] 25.375
```

But the missing reasons are still there! To indicate a value should be
treated as a missing reason instead of a regular value, you can use the
`na()` function. The following, for example, will filter the data set
for all individuals that `REFUSED` to give their favorite color:

``` r
ex |>
  filter(favorite_color == na("REFUSED"))
#> # A tibble: 3 × 3
#>   person_id       age favorite_color
#>   <dbl,chr> <dbl,chr> <chr,chr>     
#> 1         3        21 <REFUSED>     
#> 2         9 <REFUSED> <REFUSED>     
#> 3        11        10 <REFUSED>
```

And here’s a pipeline that will compute a breakdown of the mean age of
respondents for each favorite color, with separate categories for each
missing reason:

``` r
ex |>
  summarize(
    mean_age = mean(age, na.rm=T),
    n = n(),
    .by = favorite_color
  ) %>%
  arrange(favorite_color)
#> # A tibble: 6 × 3
#>   favorite_color mean_age     n
#>   <chr,chr>         <dbl> <int>
#> 1 BLUE               20       2
#> 2 RED                41       2
#> 3 YELLOW             30       1
#> 4 <N/A>               1       1
#> 5 <OMITTED>          40       2
#> 6 <REFUSED>          15.5     3
```

But this just scratches the surface of what can be done with interlacer…
check out `vignette("interlacer")` for a more complete overview!

## Known Issues

1.  Interlaced vectors cannot be constructed with `c()`

Due to a [fundamental limitation of
R](https://vctrs.r-lib.org/#motivation), interlaced vectors cannot be
constructed with `c()`. For example, the following will return a regular
vector:

``` r
c(5, 6, na("OMITTED"))
#> [1]  5  6 NA
```

Instead, use `vctrs::vec_c()` (reexported by `interlacer`):

``` r
vec_c(5, 6, na("OMITTED"))
#> <interlaced<double, character>[3]>
#> [1] 5         6         <OMITTED>
```

2.  `ifelse()` does not promote interlaced types

For similar reasons as the above issue, using `base::ifelse()` with
`interlaced` columns will convert them into regular vectors:

``` r
ex |>
  mutate(
    favorite_color = ifelse(age < 18, na("REDACTED"), favorite_color)
  )
#> # A tibble: 11 × 3
#>    person_id       age favorite_color
#>    <dbl,chr> <dbl,chr> <chr>         
#>  1         1        20 BLUE          
#>  2         2 <REFUSED> <NA>          
#>  3         3        21 <NA>          
#>  4         4        30 <NA>          
#>  5         5         1 <NA>          
#>  6         6        41 RED           
#>  7         7        50 <NA>          
#>  8         8        30 YELLOW        
#>  9         9 <REFUSED> <NA>          
#> 10        10 <OMITTED> <NA>          
#> 11        11        10 <NA>
```

Instead, use `dplyr::if_else()`:

``` r
ex |>
  mutate(
    favorite_color = if_else(
      age < 18,
      na("REDACTED_UNDERAGE"),
      favorite_color,
      missing = na("REDACTED_MISSING_AGE")
    )
  )
#> # A tibble: 11 × 3
#>    person_id       age favorite_color        
#>    <dbl,chr> <dbl,chr> <chr,chr>             
#>  1         1        20 BLUE                  
#>  2         2 <REFUSED> <REDACTED_MISSING_AGE>
#>  3         3        21 <REFUSED>             
#>  4         4        30 <OMITTED>             
#>  5         5         1 <REDACTED_UNDERAGE>   
#>  6         6        41 RED                   
#>  7         7        50 <OMITTED>             
#>  8         8        30 YELLOW                
#>  9         9 <REFUSED> <REDACTED_MISSING_AGE>
#> 10        10 <OMITTED> <REDACTED_MISSING_AGE>
#> 11        11        10 <REDACTED_UNDERAGE>
```

3.  Performance with large data sets

You may notice that on large datasets `interlacer` runs significantly
slower than `readr` / `vroom`. Although `interlacer` uses `vroom` under
the hood to load delimited data, it is not able to take advantage of
many of its optimizations because `vroom` does not [does not currently
support](https://github.com/tidyverse/vroom/issues/532) column-level
missing values. As soon as `vroom` supports column-level missing values,
I will be able to remedy this!

## Related work

`interlacer` was inspired by the
[`haven`](https://haven.tidyverse.org/),
[`labelled`](https://larmarange.github.io/labelled/), and
[`declared`](https://dusadrian.github.io/declared/) packages. These
packages provide similar functionality to `interlacer`, but are more
focused on providing compatibility with missing reason data imported
from SPSS, SAS, and Stata. `interlacer`, by contrast, aims to be more
generic: In addition to having the ability to model SPSS, SAS, and Stata
missing value labels, it allows you to compose *any* two R types into an
`interlaced` vector that can be easily manipulated in tidy pipelines.
For a more detailed discussion, see `vignette("other-approaches")`.

Future versions will include conversion functions to and from the types
provided by these other packages!

## Acknowledgements

The development of this software was supported, in whole or in part, by
the Institute of Education Sciences, U.S. Department of Education,
through Grant R305A170047 to The Pennsylvania State University. The
opinions expressed are those of the authors and do not represent the
views of the Institute or the U.S. Department of Education.
