---
layout: post
title: Fun in Rlang
---

Understanding why "100000" and 100000 are not equal... and a deep dive into the
`factor` logic of Rlang.

<!--more-->

## Introduction

Why was I even working with `Rlang`? That's a good question! The data scientists that I worked with were all quite familiar with the language and were quite happy with the performance of a particular ML algorithm implementation in `Rlang`.

## Backstory

In most model training environments you will have data which will need to be transformed into _factors_. What does this mean? For _categorical_ data types to help the model capture the differences in features correctly they are assigned a factor - all of them being on the same order of magnitude, and having a few other characteristics, such as improving data storage required.

In our case we had a *big* data frame with a few columns that contain categorical data, such as *day-of-week*, *month* and a *zone_id* column, which we want to transform into factors.

```R
df <- data.table(
  dow = c("Mon", "Tue", "Mon", "Fri"),
  zone = c(100, 100, 101, 200),
  x = 1:4
)
```

Transforming `dow` into a factor we would have to do the following:

```R
df$dow_factor <- factor(df$dow, levels=unique(df$dow))
```

This works fine for `character` or `string` data types, however, when you want to transform `zone_id` into a factor, well...

## Let the fun begin!

In our particular case, we wanted to include a fallback value of `0` in our levels. So our transformation code would be:

```R
df$zone_factor <- factor(df$zone, levels=c(0, unique(df$zone)))
```

This would seem like perfectly fine code that should work - to the **untrained** eye. From here on I will go very deep into the inner workings of how `factor` and *data types* work in this particular case - if you just want to see the TL;DR - browse to the end.

In R when you type `0` in your code you don't get a value of type `integer` rather you get
a `numerical`. Why is this problematic? Because, under the hood, deep in the C code that transforms `numerical` into `string` there's something which does not correctly transforms them back to `string`. This means that `match("100000", 100000L) == TRUE`, because the 2nd value is now an `integer` which gets correctly transformed back into a string.

## How we stumbled into this

However, little did we know that`class(0) == numerical`, `class(unique(df$zone)) == integer`, however calling `c` will cast **all** of the following values to the type of the first value in the list; meaning that `class(c(0, unique(x))) == numerical`. Under the hood `factor` calls `match` to find the value in the levels that matches the current array element. `factor` has some code that eventually ends up transforming the current array element into a `character` data type (by calling **as.character**).

This means we were now in a position where we were calling `match(<character>, <numerical>)`, and this is how we found out that `is.na(match("100000", 100000)) == TRUE`, meaning no match.

## Lesson learnt

**Dont** use `factor(<type>, levels=<other_type>)` or `match(<type>, <other_type>)` and strive to use `factor(<character>, levels=<character>)` or R somehow find a way to change your data types into **character**, but not do this consistently and you'll end up in a similar debugging   nightmare.

Code to exemplify the issue with `match`:

```R
> is.na(match("100000", 100000))
[1] TRUE
> is.na(match("100001", 100001))
[1] FALSE
> is.na(match("100000", 100000L))
[1] FALSE
```

First two comparisons is between `character` and `numeric` - and no match is found, whilst the 3rd comparison is between `character` and `integer` and a match is found. I have not tested this extensively but I found that all multiples of **100k** are facing this problem...

I'm sure someone braver than me will dive deeper and understand this problem!
