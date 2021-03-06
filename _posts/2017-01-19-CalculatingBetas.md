---
layout: post
title: "How to calculate betas (systematic risk) for a large number of stocks"
subtitle: "A comparison between using a loop, function by and the package dplyr"
author: "Marcelo Perlin"
output: md_document
image: /img/2017-01-15-CalculatingBetas_files/figure-markdown_strict/unnamed-chunk-3-1.png
tags: [R, stock market, beta, linear regression]
---

One of the first examples about using linear regression models in 
finance is the calculation of betas, the so called market model.
Coefficient beta is a measure of systematic risk and it is calculated by
estimating a linear model where the dependent variable is the return
vector of a stock and the explanatory variable is the return vector of a
diversified local market index, such as SP500 (US), FTSE (UK), Ibovespa
(Brazil), or any other.

From the academic side, the calculation of beta is part of a (very)
famous asset pricing model, CAPM - Capital Asset Pricing Model, that
relates expected return and systematic risk. One can reach the market
model equation by assuming several conditions such as Normal distributed
returns, rational investors and frictionless market. Summing up, the
CAPM model predicts that betas have a linear relationship to expected
returns, that is, stocks with higher betas should present, collectively,
higher average of historical returns.

In the quantitative side, we can formulate the market model as:

![](http://www.sciweavers.org/tex2img.php?eq=R_%7Bt%7D%20%3D%20%5Calpha%20%2B%5Cbeta%20R_%7BM%2Ct%7D%20%2B%20%5Cepsilon%20_t%0A&bc=White&fc=Black&im=jpg&fs=12&ff=arev&edit=0)

where *R*<sub>*t*</sub> is the return of the stock at time *t*,
*R*<sub>*M*, *t*</sub> is the return of the market index, *α* is the
constant (also called Jensen's alphas) and, finally, *β* is the measure
of systematic risk for the stock. The values of *α* and *β* are found by
minimizing the sum of squared errors of the model. So, if you have a
vector of prices for a stock and another vector of prices for the local
market index, you can easily find the stock's beta by calculating the
daily returns and estimating the market model.

The problem here is that, usually, you don't want the beta of a single
stock. You want to calculate the systematic risk for a large number of
stocks. This is where students usually have problems, as they only
learned in class how to estimate one model. In order to do the same
procedure for more than one stock, some programming is needed. This is
where R really shines in comparison to simpler programs such as Excel.

In this post I will download some data from the US market, make some
adjustments to the resulting dataframe and discuss three ways to
calculate the betas of several stocks. These are:

1.  Using a `loop`
2.  Using function `by`
3.  Using package `dplyr`

But first, lets load the data.

Loading the data and preparing it
=================================

I'm a bit biased, but I really like using package `BatchGetSymbols` to
download financial data from yahoo finance. In this example we will
download data for 10 stocks selected randomly from the SP500 index. I
will also add the ticker `^GSPC`, which belongs to the SP500 index. We
will need it to calculate the betas. In order for the code to be
reproducible, I will set `random.seed(100)`. This means that anyone that
runs the code available here will get the exact same results.

    library(BatchGetSymbols)

    ## Loading required package: rvest

    ## Loading required package: xml2

    ## 
    ## Thank you for using BatchGetSymbols! If applicable, please use the following citation in your research report. Thanks! 
    ## 
    ## APA:
    ##  Perlin, M. (2016). BatchGetSymbols: Downloads and Organizes Financial Data for Multiple Tickers. CRAN Package, Available in https://CRAN.R-project.org/package=BatchGetSymbols. 
    ## 
    ## BIBTEX:
    ##  @misc{perlin2016batchgetsymbols,
    ##   title = {BatchGetSymbols: Downloads and Organizes Financial Data for Multiple Tickers},
    ##   author = {Marcelo Perlin},
    ##   year = {2016},
    ##   journal = {CRAN Package},
    ##   url = {https://CRAN.R-project.org/package=BatchGetSymbols}
    ## }
    ## }

    set.seed(100)

    ticker.MktIdx <- '^GSPC'
    first.date <- as.Date('2015-01-01')
    last.date <- as.Date('2017-01-01')

    n.chosen.stocks <- 10 # can't be higher than 505

    # get random stocks
    my.tickers <- c(sample(GetSP500Stocks()$tickers,n.chosen.stocks),
                    ticker.MktIdx)

    l.out <- BatchGetSymbols(tickers = my.tickers, 
                                 first.date = first.date,
                                 last.date = last.date)

    ## 
    ## Running BatchGetSymbols for:
    ##    tickers = DUK, CCI, LLY, AAL, HST, IR, SRE, FB, LH, CAH, ^GSPC
    ##    Downloading data for benchmark ticker
    ## Downloading Data for DUK from yahoo (1|11) - Good stuff!
    ## Downloading Data for CCI from yahoo (2|11) - Well done!
    ## Downloading Data for LLY from yahoo (3|11) - Got it!
    ## Downloading Data for AAL from yahoo (4|11) - Nice!
    ## Downloading Data for HST from yahoo (5|11) - Good job!
    ## Downloading Data for IR from yahoo (6|11) - Good job!
    ## Downloading Data for SRE from yahoo (7|11) - Got it!
    ## Downloading Data for FB from yahoo (8|11) - Nice!
    ## Downloading Data for LH from yahoo (9|11) - Nice!
    ## Downloading Data for CAH from yahoo (10|11) - Good job!
    ## Downloading Data for ^GSPC from yahoo (11|11) - Good stuff!

    df.stocks <- l.out$df.tickers

Now, lets check if everything went well with the import process.

    print(l.out$df.control)

    ##    ticker   src download.status total.obs perc.benchmark.dates
    ## 1     DUK yahoo              OK       504                    1
    ## 2     CCI yahoo              OK       504                    1
    ## 3     LLY yahoo              OK       504                    1
    ## 4     AAL yahoo              OK       504                    1
    ## 5     HST yahoo              OK       504                    1
    ## 6      IR yahoo              OK       504                    1
    ## 7     SRE yahoo              OK       504                    1
    ## 8      FB yahoo              OK       504                    1
    ## 9      LH yahoo              OK       504                    1
    ## 10    CAH yahoo              OK       504                    1
    ## 11  ^GSPC yahoo              OK       504                    1
    ##    threshold.decision
    ## 1                KEEP
    ## 2                KEEP
    ## 3                KEEP
    ## 4                KEEP
    ## 5                KEEP
    ## 6                KEEP
    ## 7                KEEP
    ## 8                KEEP
    ## 9                KEEP
    ## 10               KEEP
    ## 11               KEEP

It seems that everything is Ok. All stocks have column
`perc.benchmark.dates` equal to one (100%), meaning that they have the
exact same dates as the benchmark ticker.

Now, lets plot the time series of prices and look for any problem:

    library(ggplot2)

    p <- ggplot(df.stocks, aes(x=ref.date, y=price.adjusted))
    p <- p + geom_line()
    p <- p + facet_wrap(~ticker, scales = 'free')
    print(p)

![](/img/2017-01-15-CalculatingBetas_files/figure-markdown_strict/unnamed-chunk-3-1.png)

Again, we see that all prices seems to be Ok. This is one of the
advantages of working with adjusted (and not closing) prices from yahoo
finance. Artificial effects in the dataset such as ex-dividend prices,
splits and inplits are already taken into account and the result is a
smooth series without any breaks.

Calculating returns
-------------------

In the previous step we downloaded prices from yahoo finance. In the
regression, however, returns (and not prices) are used. We have two
choices here, arithmetic or logarithmic returns. In general, logarithmic
returns are more used in research since they have some special
properties. In practice, though, it makes very little difference. In
this post we will use only log returns. They are given by the following
formula:

![](http://www.sciweavers.org/tex2img.php?eq=R_t%20%3D%20%5Clog%20%5Cfrac%7BP_t%7D%7BP_%7Bt-1%7D%7D%0A&bc=White&fc=Black&im=jpg&fs=12&ff=arev&edit=0)

In order to calculate daily returns for each stock, the first step is to
write a simple function that, given a vector of prices, outputs a vector
of returns. We can write it as:

    calc.ret <- function(p){
      
      my.n <- length(p)
      arit.ret <- c(NA, log(p[2:my.n]/p[1:(my.n-1)]))
      return(arit.ret)
    }

Notice that I used a `NA` for the first element of the return vector. In
the next steps we will column bind the vector of returns in the price
dataframe. Therefore, it is necessary that returns and prices have the
same length. Remember that you always lose the first observation (row)
when calculating returns.

Following up, we will first calculate the returns of all assets and
store them in the same dataframe with the prices. For that, we will use
function `tapply` to calculate the returns of each stock. **A
precaution, however, is necessary**. Function `tapply` returns a list
and sorts its elements using the names. We don't really want that as we
will `unlist` this list and store the result in the dataframe
`df.stocks`. If the order in the list don't match the order in the
dataframe, the returns will be wrongly allocated to the tickers. To
solve that, we will again sort the list to match the order of tickers in
the dataframe.

In the next code we will get the list from `tapply`, sort it to match
the order in the dataframe and unlist it in a new column called `ret`:

    list.ret <- tapply(X = df.stocks$price.adjusted, 
                       INDEX = df.stocks$ticker,
                       FUN = calc.ret)

    list.ret <- list.ret[unique(df.stocks$ticker)]

    df.stocks$ret <- unlist(list.ret)

The final step in preparing the data is to add a column with the returns
of the market index. This is not strictly necessary but I really like to
keep things organized in a tabular way. Since we will match each vector
of returns of the stocks to a vector of returns of the market index, it
makes sense to *synchronize* the rows in the data.frame. First, we
isolate the data for the market index in object `df.MktIdx` and use
function `match` to make an index that matches the dates between the
assets and the market index. We later use this index to build a new
column in `df.stocks`. See the next code:

    df.MktIdx <- df.stocks[df.stocks$ticker==ticker.MktIdx, ]

    idx <- match(df.stocks$ref.date, df.MktIdx$ref.date)

    df.stocks$ret.MktIdx <- df.MktIdx$ret[idx]

Now that we have the data in the correct format and structure, let's
start to calculate some betas. Here is where the different approaches
will differ in syntax. Let's start with the first case, using loops.

Estimating betas
================

Using loops
-----------

Loops are great and (almost) everyone loves then. While they can be a
bit more verbose than fancy on-liners, the structure of a loop is very
flexible and this can help solve complex problems. Let use it in our
problem.

The first step in using loops is the understand the vector that will be
used as iterator in the loop. In our problem we are processing each
stock, so the number of iterations in the loop is simply the number of
stocks in the sample. We can find the unique stocks with the command
`unique`.

    # Check unique tickers
    unique.tickers <- unique(df.stocks$ticker)

    # create a empty vector to store betas
    beta.vec <- c()

    for (i.ticker in unique.tickers){
      
      # message to prompt
      cat('\nRunning ols for',i.ticker)
      
      # filter the data.frame for stock i.ticker
      df.temp <- df.stocks[df.stocks$ticker==i.ticker, ]
      
      # calculate beta with lm
      my.ols <- lm(data = df.temp, formula = ret ~ ret.MktIdx)
      
      # save beta
      my.beta <- coef(my.ols)[2]
      
      # store beta em beta.vec
      beta.vec <- c(beta.vec, my.beta)
    }

    ## 
    ## Running ols for DUK
    ## Running ols for CCI
    ## Running ols for LLY
    ## Running ols for AAL
    ## Running ols for HST
    ## Running ols for IR
    ## Running ols for SRE
    ## Running ols for FB
    ## Running ols for LH
    ## Running ols for CAH
    ## Running ols for ^GSPC

    # print result
    print(data.frame(unique.tickers,beta.vec))

    ##    unique.tickers  beta.vec
    ## 1             DUK 0.4731914
    ## 2             CCI 0.7045035
    ## 3             LLY 0.8846485
    ## 4             AAL 1.3411230
    ## 5             HST 1.2226485
    ## 6              IR 1.1176236
    ## 7             SRE 0.6460290
    ## 8              FB 1.0819643
    ## 9              LH 0.8211889
    ## 10            CAH 0.9816521
    ## 11          ^GSPC 1.0000000

As you can see, the result is a lengthy code, but it works quite well.
The final result is a dataframe with the tickers and their betas. Notice
that, as expected, the betas are all positive and `^GSPC` has a beta
equal to 1.

Using function `by`
-------------------

Another way of solving the problem is to calculate the betas using one
of the functions from the `apply` family. In this case, we will use
function `by`. Be aware that you can also solve the problem using
`tapply` and `lapply`. The code, however, will increase in complexity.

The function `by` works similarly to `tapply`. The difference is that it
is oriented to dataframes. That is, given a grouping variable, the
original dataframe is broken into smaller dataframes and each piece is
passed to a function. This helps a lot our problem since we need to work
with two columns, the vector of returns of the asset and the vector of
returns of the market index.

Given the functional form of `by`, will need to encapsulate a procedure
that takes a dataframe as input and returns a coefficient beta,
calculated from columns `ret` and `ret.MktIdx`. The next code does that.

    get.beta <- function(df.temp){
      
      # estimate model
      my.ols <- lm(data=df.temp, formula = ret ~ ret.MktIdx)
      
      # isolate beta
      my.beta <- coef(my.ols)[2]
      
      # return beta
      return(my.beta)
    }

The previous function accepts a single dataframe called `df.temp`, uses
it to calculate a linear model with function `lm` and then returns the
resulting beta, which is the second coefficient in `coef(my.ols)`. Now,
lets use it with function `by`.

    # get betas with by
    my.l <- by(data = df.stocks, 
               INDICES = df.stocks$ticker, 
               FUN = get.beta)

    # my.l is an objetct of class by. To get only its elements, we can unclass it
    betas <- unclass(my.l)

    # print result
    print(data.frame(betas))

    ##           betas
    ## ^GSPC 1.0000000
    ## AAL   1.3411230
    ## CAH   0.9816521
    ## CCI   0.7045035
    ## DUK   0.4731914
    ## FB    1.0819643
    ## HST   1.2226485
    ## IR    1.1176236
    ## LH    0.8211889
    ## LLY   0.8846485
    ## SRE   0.6460290

Again, it worked well. Needless to say that the results are identical to
the previous case.

Using `dplyr`
-------------

Now, let's solve our problem using package `dplyr`. If you are not
familiar with the *tidyverse* and the work of Hadley Wickham, you will
be a happier person after reading the rest of this post, trust me.

Package `dplyr` is one of my favorites and most used packages. It allows
for the representation of data processing procedures in a simpler and
more intuitive way. It really helps to tackle computational problems if
you can fit it within a flexible structure. This is what, in my opinion,
`dplyr` does best. It combines clever functions with dataframes in the
long (tidy) format.

Have a look in the next set of code.

    library(dplyr)

    ## 
    ## Attaching package: 'dplyr'

    ## The following objects are masked from 'package:stats':
    ## 
    ##     filter, lag

    ## The following objects are masked from 'package:base':
    ## 
    ##     intersect, setdiff, setequal, union

    beta.tab <- df.stocks %>% 
      group_by(ticker) %>% # group by column ticker
      do(ols.model = lm(data = ., formula = ret ~ret.MktIdx)) %>%   # estimate model
      mutate(beta = coef(ols.model)[2]) # get coefficients

    print(beta.tab)

    ## Source: local data frame [11 x 3]
    ## Groups: <by row>
    ## 
    ## # A tibble: 11 × 3
    ##    ticker ols.model      beta
    ##     <chr>    <list>     <dbl>
    ## 1   ^GSPC  <S3: lm> 1.0000000
    ## 2     AAL  <S3: lm> 1.3411230
    ## 3     CAH  <S3: lm> 0.9816521
    ## 4     CCI  <S3: lm> 0.7045035
    ## 5     DUK  <S3: lm> 0.4731914
    ## 6      FB  <S3: lm> 1.0819643
    ## 7     HST  <S3: lm> 1.2226485
    ## 8      IR  <S3: lm> 1.1176236
    ## 9      LH  <S3: lm> 0.8211889
    ## 10    LLY  <S3: lm> 0.8846485
    ## 11    SRE  <S3: lm> 0.6460290

After loading `dplyr`, we use the pipeline operator %&gt;% to streamline
all calculations. This means that we don't need to keep a copy of
intermediate calculations. Also, It looks pretty, don't you agree?

The line `beta.tab <- df.stocks %>%` passes the dataframe `df.stocks`
for the next line, `group_by(ticker) %>%`, which will group the
dataframe according the the values of column `ticker` and pass the
result for the next step. The line
`do(ols.model = lm(data = ., formula = ret ~ret.MktIdx))` estimates the
model by passing a temporary dataframe and saves it in a column called
`ols.model`. Notice that the model is a `S3` object and not a single
value. The dataframe alternative `tibble` is flexible with its content.
The final line, `mutate(beta = coef(ols.model)[2])` retrieves the beta
from each element of the column `ols.model`.

What I really like about `dplyr` is that it makes it easy to extend the
original code. As an example, if I wanted to use a second grouping
variable, I can just add it in the second line as
`group_by(ticker, newgroupingvariable)`. This becomes handy if you need,
lets say, to estimated the model in different time periods.

As an example, let's assume that I want to split the sample for each
stock in half and see if the betas change significantly from time period
to the other. This robustness check is a very common procedure in
scientific research. First, let's build a new column in `df.stocks` that
sets the time periods as `Sample 1` and `Sample 2`. We can use `tapply`
for that;

    set.sample <- function(ref.dates){
      my.n <- length(ref.dates) # get number of dates
      my.n.half <- floor(my.n/2) # get aproximate half of observations
      
      # create grouping variable
      samples.vec <- c(rep('Sample 1', my.n.half ), rep('Sample 2', my.n-my.n.half))
      
      # return
      return(samples.vec)
    }

    # build group
    my.l <- tapply(X = df.stocks$ref.date, 
                   INDEX = df.stocks$ticker,
                   FUN = set.sample )

    # unsort it
    my.l <- my.l[my.tickers]

    # save it in dataframe
    df.stocks$my.sample <- unlist(my.l)

We proceed by calling the same functions as before, but using an
additional grouping variable.

    beta.tab <- df.stocks %>% 
      group_by(ticker,my.sample) %>% # group by column ticker
      do(ols.model = lm(data = ., formula = ret ~ret.MktIdx)) %>%   # estimate model
      mutate(beta = coef(ols.model)[2]) # get coefficients

    print(beta.tab)

    ## Source: local data frame [22 x 4]
    ## Groups: <by row>
    ## 
    ## # A tibble: 22 × 4
    ##    ticker my.sample ols.model      beta
    ##     <chr>     <chr>    <list>     <dbl>
    ## 1   ^GSPC  Sample 1  <S3: lm> 1.0000000
    ## 2   ^GSPC  Sample 2  <S3: lm> 1.0000000
    ## 3     AAL  Sample 1  <S3: lm> 1.1212908
    ## 4     AAL  Sample 2  <S3: lm> 1.6462873
    ## 5     CAH  Sample 1  <S3: lm> 0.9211033
    ## 6     CAH  Sample 2  <S3: lm> 1.0710157
    ## 7     CCI  Sample 1  <S3: lm> 0.6844971
    ## 8     CCI  Sample 2  <S3: lm> 0.7341892
    ## 9     DUK  Sample 1  <S3: lm> 0.5907584
    ## 10    DUK  Sample 2  <S3: lm> 0.3064395
    ## # ... with 12 more rows

As we can see, the output now shows the beta for all combinations
between `ticker` and `my.sample`. It seems that the betas tend to be
higher for *Sample 2*, meaning that the overall systematic risk in the
market has increased over time, at least for the majority of the ten
chosen stocks. Given the small sample of stocks, It might be interesting
to test for this property in a larger dataset.

Back to the model, if you want more information about it, you can just
write new lines in the last call to %&gt;%. Let's say, for example, that
you want to get the value of alpha and the corresponding t-statistic of
both coefficients. We can use the following code for that:

    library(dplyr)

    beta.tab <- df.stocks %>% 
      group_by(ticker) %>% # group by column ticker
      do(ols.model = lm(data = ., formula = ret ~ret.MktIdx)) %>%   # estimate model
      mutate(beta = coef(ols.model)[2],
             beta.tstat = summary(ols.model)[[4]][2,3],
             alpha = coef(ols.model)[1],
             alpha.tstat = summary(ols.model)[[4]][1,3]) # get coefficients

    print(beta.tab)

    ## Source: local data frame [11 x 6]
    ## Groups: <by row>
    ## 
    ## # A tibble: 11 × 6
    ##    ticker ols.model      beta   beta.tstat         alpha alpha.tstat
    ##     <chr>    <list>     <dbl>        <dbl>         <dbl>       <dbl>
    ## 1   ^GSPC  <S3: lm> 1.0000000 1.445949e+16 -2.513794e-19 -0.40203444
    ## 2     AAL  <S3: lm> 1.3411230 1.359647e+01 -4.710243e-04 -0.52817938
    ## 3     CAH  <S3: lm> 0.9816521 1.657973e+01 -3.069863e-04 -0.57348160
    ## 4     CCI  <S3: lm> 0.7045035 1.520931e+01  2.167688e-04  0.51761145
    ## 5     DUK  <S3: lm> 0.4731914 8.938631e+00 -6.718761e-05 -0.14037962
    ## 6      FB  <S3: lm> 1.0819643 1.595227e+01  5.802958e-04  0.94632384
    ## 7     HST  <S3: lm> 1.2226485 1.757965e+01 -4.772003e-04 -0.75890926
    ## 8      IR  <S3: lm> 1.1176236 2.232841e+01  2.317878e-04  0.51219268
    ## 9      LH  <S3: lm> 0.8211889 1.523458e+01  1.443500e-04  0.29619986
    ## 10    LLY  <S3: lm> 0.8846485 1.273551e+01  5.368389e-05  0.08548114
    ## 11    SRE  <S3: lm> 0.6460290 1.190925e+01 -2.093818e-04 -0.42692546

In the previous code, I added line
`beta.tstat = summary(ols.model)[[4]][2,3]` that returns the t-statistic
of the beta coefficient. The location of this parameter is found by
investigating the elements of an object of type `lm`. After calling
`summary`, the t-statistic is available in the fourth element of the
`lm` object, which is a matrix with several information from the
estimation. The t-statistic for the alpha parameter is found in a
similar way.

As you can see, the syntax of `dplyr` make it easy to extend the model
and quickly try new things. It is possible to do the same using other R
functions and a loop, but using `dplyr` is really handy.

Conclusions
===========

As you can probably suspect from the text, I'm a big fan of `dplyr` and
I'm always teaching its use to my students. While loops are ok and I
personally use then a lot in more complex problems, the functions in
`dplyr` allow for an intuitive syntax in data processing, making it easy
to understand and extend code.

Do notice that the code in this example is self contained and
reproducible. If you want to try it for more stocks, just change input
`n.chosen.stocks` to a higher value.
