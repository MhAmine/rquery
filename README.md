rquery
================
2017-12-11

<!-- README.md is generated from README.Rmd. Please edit that file -->
[`rquery`](https://johnmount.github.io/rquery/) is an experiment/demonstration of a simplified sequenced query language based on [Codd's relational algebra](https://en.wikipedia.org/wiki/Relational_algebra) and not currently recommended for non-experimental (i.e., production) use. The goal of this experiment is to see if `SQL` would be more fun if it had a sequential data-flow or pipe notation.

[`rquery`](https://github.com/JohnMount/rquery) can be an excellent advanced `SQL` training tool (it shows how some very deep `SQL` by composing `rquery` operators). Currently `rquery` is biased towards `PostgeSQL` `SQL`.

There are many prior relational algebra inspired specialized query languages. Just a few include:

-   [Alpha](https://en.wikipedia.org/wiki/Alpha_(programming_language))
-   [QUEL](https://en.wikipedia.org/wiki/QUEL_query_languages)
-   [Tutorial D](https://en.wikipedia.org/wiki/D_(data_language_specification)#Tutorial_D)
-   [`LINQ`](https://msdn.microsoft.com/en-us/library/bb308959.aspx)
-   [`SQL`](https://en.wikipedia.org/wiki/SQL)
-   [`dplyr`](http://dplyr.tidyverse.org)

`rquery` itself is a thin translation to `SQL` layer, but we are trying to put the Codd relational operators front and center (using the original naming, and back-porting `SQL` progress such as window functions to the appropriate relational operator). `rquery` differs from `dplyr` in that `rquery` is trying to stay near the Codd relational operators (in particular grouping is a transient state inside the `rquery::extend()` operator, not a durable user visible annotation as with `dplyr::group_by()`).

The primary relational operators are:

-   [`extend()`](https://johnmount.github.io/rquery/reference/extend_nse.html). Extend adds derived columns to a relation table. With a sufficiently powerful `SQL` provider this includes ordered and partitioned window functions. This operator also includes built-in [`seplyr`](https://winvector.github.io/seplyr/)-style [assignment partitioning](https://winvector.github.io/seplyr/articles/MutatePartitioner.html).
-   [`project()`](https://johnmount.github.io/rquery/reference/project_nse.html). Project is usually portrayed as the equivalent to column selection. In our opinion the original relational nature of the operator is best captured by moving `SQL`'s "`GROUP BY`" aggregation functionality to this operator.
-   [`natural_join()`](https://johnmount.github.io/rquery/reference/natural_join.html). This a specialized relational join operator, using all common columns as the equi-join condition. The next operator to add would definitely be `theta-join` as that adds a lot more expressiveness to the grammar.
-   [`theta_join()`](https://johnmount.github.io/rquery/reference/theta_join_nse.html). This is the relational join operator, insisting on distinct columns but allowing an arbitrary matching condition. The next operator to add would definitely be `theta-join` as that adds a lot more expressiveness to the grammar.
-   [`select_rows()`](https://johnmount.github.io/rquery/reference/theta_join_nse.html). This is Codd's relational row selection. Obviously `select` alone is an over-used and now ambiguous term (it is the "doit" verb in `SQL` and the *column* selector in `dplyr`).
-   [`rename_columns()`](https://johnmount.github.io/rquery/reference/rename_columns.html). This operator renames sets of columns.

The primary non-relational (traditional `SQL`) operators are:

-   [`select_columns()`](https://johnmount.github.io/rquery/reference/select_columns.html). This allows choice of columns (central to `SQL`), but is not a relational operator as it can damage row-uniqueness.
-   [`order_by()`](https://johnmount.github.io/rquery/reference/order_by.html). This is actually relational in the sense that it does not ruin a table that is a relation (has unique rows). However it is only a useful intermediate step with used with its `limit=` option. Row order is not well-defined in the relational algebra (and also not in most `SQL` implementations). If used it should be used last in a query (so it is not undone by later operations).

The primary missing relational operators are:

-   Union.
-   Direct set difference, anti-join.
-   Division.

Primary useful missing operators:

-   Deselect columns.

A great benefit of Codd's relational algebra is it decomposes data transformations into a sequence of operators. `SQL` loses a lot of the original invariants, and over-specifies how operations are strung together and insisting on a nesting function notation. `SQL` also realizes some of the Codd concepts as operators, some as expressions, and some as predicates (obscuring the uniformity of the original theory).

A lot of the grace of the Codd theory can be recovered through the usual trick changing function composition notation from `g(f(x))` as `x . f() . g()`. This is the other inspiration for this experiment: "what if `SQL` were piped?" (wrote composition as a left to right flow, instead of right to left nesting).

The `rquery` operators are passive. They don't do anything other than collect a specification of the desired calculation. This data structure can then be printed in a friendly fashion, used to generate `SQL`, and (in principle) be the representational layer for a higher-order optimizer.

As an acid test we generate a query equivalent to the non-trivial `dplyr` pipeline demonstrated in [Let’s Have Some Sympathy For The Part-time R User](http://www.win-vector.com/blog/2017/08/lets-have-some-sympathy-for-the-part-time-r-user/).

First we set up the database and the original example data:

``` r
library("rquery")
```

    ## Loading required package: wrapr

``` r
use_spark <- TRUE

if(use_spark) {
  my_db <- sparklyr::spark_connect(version='2.2.0', 
                                   master = "local")
} else {
  library('RPostgreSQL')
  my_db <- DBI::dbConnect(dbDriver("PostgreSQL"),
                          host = 'localhost',
                          port = 5432,
                          user = 'postgres',
                          password = 'pg')
}


d <- dbi_copy_to(my_db, 'd',
                 data.frame(
                   subjectID = c(1,                   
                                 1,
                                 2,                   
                                 2),
                   surveyCategory = c(
                     'withdrawal behavior',
                     'positive re-framing',
                     'withdrawal behavior',
                     'positive re-framing'
                   ),
                   assessmentTotal = c(5,                 
                                       2,
                                       3,                  
                                       4),
                   irrelevantCol1 = "irrel1",
                   irrelevantCol2 = "irrel2",
                   stringsAsFactors = FALSE),
                 temporary = TRUE, 
                 overwrite = !use_spark)

print(d)
```

    ## [1] "table('d')"

``` r
d %.>%
  to_sql(.) %.>%
  DBI::dbGetQuery(my_db, .) %.>%
  knitr::kable(.)
```

|  subjectID| surveyCategory      |  assessmentTotal| irrelevantCol1 | irrelevantCol2 |
|----------:|:--------------------|----------------:|:---------------|:---------------|
|          1| withdrawal behavior |                5| irrel1         | irrel2         |
|          1| positive re-framing |                2| irrel1         | irrel2         |
|          2| withdrawal behavior |                3| irrel1         | irrel2         |
|          2| positive re-framing |                4| irrel1         | irrel2         |

Now we write the calculation in terms of our operators.

``` r
scale <- 0.237

dq <- d %.>%
  extend_nse(.,
             probability :=
               exp(assessmentTotal * scale)/
               sum(exp(assessmentTotal * scale)),
             count := count(1),
             partitionby = 'subjectID') %.>%
  extend_nse(.,
             rank := rank(),
             partitionby = 'subjectID',
             orderby = 'probability')  %.>%
  extend_nse(.,
             isdiagnosis := rank == count,
             diagnosis := surveyCategory) %.>%
  select_rows_nse(., isdiagnosis) %.>%
  select_columns(., c("subjectID", 
                      "diagnosis", 
                      "probability")) %.>%
  order_by(., 'subjectID')
```

The above compares well to [the original `dplyr` pipeline](http://www.win-vector.com/blog/2017/08/lets-have-some-sympathy-for-the-part-time-r-user/).

We then generate our result:

``` r
dq %.>%
  to_sql(.) %.>%
  DBI::dbGetQuery(my_db, .) %.>%
  knitr::kable(.)
```

|  subjectID| diagnosis           |  probability|
|----------:|:--------------------|------------:|
|          1| withdrawal behavior |    0.6706221|
|          2| positive re-framing |    0.5589742|

We see we reproduced the result purely in terms of these database operators.

The actual `SQL` query that produces the result is quite involved:

``` r
cat(to_sql(dq))
```

    SELECT * FROM (
     SELECT
      `subjectID`,
      `diagnosis`,
      `probability`
     FROM (
      SELECT * FROM (
       SELECT
        `subjectID`,
        `surveyCategory`,
        `probability`,
        `count`,
        `rank`,
        `rank` = `count`  AS `isdiagnosis`,
        `surveyCategory`  AS `diagnosis`
       FROM (
        SELECT
         `subjectID`,
         `surveyCategory`,
         `probability`,
         `count`,
         rank() OVER (  PARTITION BY `subjectID` ORDER BY `probability` ) AS `rank`
        FROM (
         SELECT
          `subjectID`,
          `surveyCategory`,
          `assessmentTotal`,
          exp(`assessmentTotal` * 0.237) / sum(exp(`assessmentTotal` * 0.237)) OVER (  PARTITION BY `subjectID` ) AS `probability`,
          count(1) OVER (  PARTITION BY `subjectID` ) AS `count`
         FROM (
          SELECT
           `d`.`subjectID`,
           `d`.`surveyCategory`,
           `d`.`assessmentTotal`
          FROM
           `d`
          ) tsql_0000
         ) tsql_0001
        ) tsql_0002
      ) tsql_0003
      WHERE `isdiagnosis`
     ) tsql_0004
    ) tsql_0005 ORDER BY `subjectID`

The query is large, but due to its regular structure it should be very amenable to database query optimizer.

Notice the query was automatically restricted to columns actually needed from the source table. This is possible because the `rquery` representation is an intelligible network of nodes, so we can interrogate it for facts about the query. For example:

``` r
tables_used(dq)
```

    ## [1] "d"

``` r
cu <- columns_used(dq)
print(cu)
```

    ## [1] "`d`.`subjectID`"       "`d`.`surveyCategory`"  "`d`.`assessmentTotal`"

Part of the plan is: the additional record-keeping in the operator nodes would let a potentially powerful query optimizer work over the flow before it gets translated to `SQL` (perhaps an extension of or successor to [`seplyr`](https://winvector.github.io/seplyr/), which re-plans over `dplyr::mutate()` expressions). At the very least restricting to columns later used and folding selects together would be achievable. One should have a good chance at optimization as the representation is fairly high-level, and many of the operators are relational (meaning there are known legal transforms a query optimizer can use). The flow itself is represented as follows:

``` r
cat(format(dq))
```

    table('d') %.>%
     extend(.,
      probability := exp(assessmentTotal * scale) / sum(exp(assessmentTotal * scale)),
      count := count(1),
      p= subjectID) %.>%
     extend(.,
      rank := rank(),
      p= subjectID,
      o= probability) %.>%
     extend(.,
      isdiagnosis := rank == count,
      diagnosis := surveyCategory) %.>%
     select_rows(., isdiagnosis) %.>%
     select_columns(., subjectID, diagnosis, probability) %.>%
     order_by(., subjectID)

We also can [stand this system up on non-`DBI` sources such as `SparkR`](https://github.com/JohnMount/rquery/blob/master/extras/SparkRExample.md).

And that is our experiment.

We are looking for funding and partners to take this system further (including: finishing functionality, documentation, training materials, test materials, acceptance procedures, porting to more back-ends).

``` r
if(use_spark) {
  sparklyr::spark_disconnect(my_db)
} else {
  DBI::dbDisconnect(my_db)
}
```
