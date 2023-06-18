---
layout: post
title: "Domain Driven Design in Data"
---

# The hard sell
One of the most valuable concepts I think a developer can learn is domain driven design, particularly for a mid to large sized project.

# Why?
The primary benefit of domain driven design is that you consciously split __what__ you are doing from __how__ you are doing it.
This leads to much cleaner looking code that is easier understand and maintain.

## Example A

```python
Product = ...
DateRange = ...
Field = ...
Aggregation = ...

@dataclass
class SalesReport:
    products: set[Product]
    date_range: DateRange
    factors: set[Field]
    aggregations: dict[str, Aggregation]

def generate_report(report: SalesReport):
    ...
```

## Example B

```python

def load_some_data() -> pd.DataFrame:
    ...

def filter_where_contains(df: pd.DataFrame, field: str, values: list[str]) -> pd.DataFrame:
    ...

def filter_between_dates(df: pd.DataFrame, start: date, end: date, start_inclusive: bool, end_inclusive: bool, field_name: str) -> pd.DataFrame:
    ...

def group_by_fields(df: pd.DataFrame, fields: list[str]) -> pd.DataFrame:
    ...

def save_results(df: pd.DataFrame) -> None:
    ...
```

Let's see what we can learn from just looking at the code.
Example A has a class called `SalesReport`, which has a member called `products`, which is a set of some `Product` object.
It also has a member called `date_range` which is a `DateRange` object, a member called `factors` which is a set of `Field` types and a member called `aggregations` which is a dictionary with `str` keys and value of type `Aggregate`.
I have deliberately left the implementations blank.

Even without knowing what the custom objects like `Product` or `DateRange` are, it's pretty clear what this code is about.
The class is called `SalesReport`, so an educated guess might be that it has something to do with reporting on sales.
The class also has a member called `products`, the type of which is `set[Product]`, so we can assume that the reporting on sales can be in relation to products.
The type called `DateRange` sounds like it deals with a range of dates, so possible the sales report might only deal with data that fall within this range.
Finally the `aggregations` member is a dictionary of type `str` and `Aggregation`, which suggests that we have some string key by which refer to an aggregation.
We might not know _how_ the individual objects have been implemented, but just from their names and the fact that they all belong to this `SalesReport` class, we can get a pretty good idea about what this code is trying to do.

Contrast that with example B.
It obviously deals with Pandas dataframes.
It obviously has some notion about loading data into a Pandas dataframe.
It can filter this dataframe between two dates, filter where a field contains a certain value, save the results.
But why is it doing that? What is the goal? What are the important bits of information that might change the outcome?

The intent isn't clear - we know _how_ the code is doing something, but not _what_ it is doing.
The code is just a number of convenience functions that wrap the existing Pandas API.

Putting some thought into our domain also helps to tidy up our code.
Take `filter_between_dates`, for example.
It contains arguments for both the start date, end date (both optional) and then whether the start and end dates are inclusive.
The problem that you might spot here is that `start_inclusive` and `end_inclusive` are meaningless if either `start_date` or `end_date` haven't been passed in.
Consider the below:

```python
def filter_between_dates(
    df: pd.DataFrame,
    start_date: date | None,
    end_date: date | None,
    start_inclusive: bool,
    end_inclusive: bool,
    field_name: str
    ) -> pd.DataFrame:
    locator = (
        df[field_name].between(
            start_date,
            end_date,
            start_inclusive,
            end_inclusive)
    )
    return df.loc[locator]
```

Compare that to this:

```python

@dataclass
class DateBound:
    value: date
    inclusive: bool

@dataclass
class DateRange:
    start: DateBound | None
    end: DateBound | None
```

Here, we've made it clear that the `inclusive` aspect of the date `start` and `end` depends on whether they are included.
No start date? No `start_inclusive`.
This makes sense, because how can a not-a-date be inclusive?
This also means that we can rewrite the original date filter function signature to the following:

```python
def filter_between_dates(
    df: pd.DataFrame,
    date_range: DateRange,
    field_name: str
    ) -> pd.DataFrame:
    ...
```

Instead of someone having to reason about 4 separate arguments concerning dates, it's just one.
If the user wants to delve into the implementation of `DateRange`, they can do so in that type's implementation.
That code, however, is not obfuscating this function's implementation.
What the Pandas code inside will do now depends on our custom business logic, rather than our business logic depending on what Pandas can do.
This is the core of domain driven design - we define entities and relationships that describe our processes and any actual infrastructure code only serves to carry these processes out as we define them.

# Summary

This short example hopefully demonstrated some benefits of trying to apply concepts from domain driven design when working with data.
This approach might feel unnatural to someone coming from working with jupyter notebooks, but I would argue that notebooks are an entirely unnatural way to develop code.
Notebooks are a presentation tool.
They are useful when you want to present some data along with just enough interactivity to let the target audience adjust a few values here and there.
The side effect that they have had, however, is that they encourage writing functions that are heavily dependend on the APIs of the libraries they are wrapping.
The signatures of those functions therefore have very little to do with the subject matter (i.e. reports, models, analysis) and everything to do with what the library functions need.
This is short-term convenience at the expense of readability and maintainability.
What we should be striving for instead is developing objects and classes that describe the situation we are trying model.

