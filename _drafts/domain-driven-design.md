---
layout: post
title: "Domain Driven Design in Data"
---

# The hard sell
One of the most valuable concepts I think a developer can learn is domain driven design, particularly for a mid to large sized projects.

# The why
The main benefit of following domain driven design is the fact that, if done well, the business logic becomes self documenting and becomes easier for people unfamiliar with a codebase to understand what it is trying to do. Consider the two examples:

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

Let's see what we can learn from just looking at the code. Example A has a class called `SalesReport`, which has a member called `products`, which is a set of some `Product` object. It also has a member called `date_range` which is a `DateRange` object, a member called `factors` which is a set of `Field` types and a member called `aggregations` which is a dictionary with `str` keys and value of type `Aggregate`. I've deliberately left the implementations blank.

Even without knowing what the custom objects like `Product` or `DateRange` are, it's pretty clear what this code is about. The class is called `SalesReport`, so it concerns some reporting of sales data. Given the member `products`, the report must be limited to specific products, which are represented by this `Product` class. The `DateRange` object must describe some period of time, judging by it's name, while the factors must be some aspects of the sales data that we are looking at. Finally the `aggregations` member is a dictionary, and judging by it's name it is some collected statistic based on the factors. We might not know _how_ the individual objects have been implemented, but just from their names and the fact that they all belong to this `SalesReport` class, we can get a pretty good idea about what this code is used for.

Contrast that with example B. It obviously deals with Pandas dataframes. It obviously has some notion about loading data into a Pandas dataframe. It can filter this dataframe between two dates, filter where a field contains a certain value, save the results. But why is it doing that? What is the goal?

The intent isn't clear. We know _how_ the code is doing something, but not _what_ it is doing. It's just a showcase of Pandas functionality.

Putting some thought into our domain also helps to tidy up our code. Take `filter_between_dates`, for example. It contains arguments for both the start date, end date and then whether the start and end dates are inclusive. Both start and end are optional. The problem that you might spot here is that `start_inclusive` and `end_inclusive` are meaningless if either `start_date` or `end_date` haven't been passed in. Consider the below:

```python
def filter_between_dates(
    df: pd.DataFrame,
    start_date: date | None,
    end_date: date | None,
    start_inclusive: bool,
    end_inclusive: bool,
    field_name: str
    ) -> pd.DataFrame:
    return df.loc[df[field_name].between(start_date, end_date, start_inclusive, end_inclusive)]
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

Here, we've made it clear that the `inclusive` aspect of the date `start` and `end` is tied to those member being non-null. No start date? No `start_inclusive`. This means that we can rewrite the function signature to the following:

```python
def filter_between_dates(
    df: pd.DataFrame,
    date_range: DateRange,
    field_name: str
    ) -> pd.DataFrame:
    ...
```

Instead of someone having to reason about 4 separate arguments, it's just one. If they want to focus on the implementation of the `DateRange` object, it's there, but the details are compartmentalised and don't litter some other bit of code.
