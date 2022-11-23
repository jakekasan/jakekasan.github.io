---
layout: post
title: "Domain Driven Design in Data"
---

# The hard sell
One of the most valuable concepts I think a developer can learn is domain driven design, particularly for a mid to large sized projects.

# The why
The main benefit of following domain driven design is the fact that, if done well, the business logic becomes self documenting. Consider the two examples:

## Example A

```python
Product = Literal["Banana"] | Literal["Apple"]
DateRange = namedtuple("DateRange", "start end")
Field = str

@dataclass
class SalesReport:
    target_products: set[Product]
    date_range: DateRange
    group_by: set[Field]
    aggregations: ...

def generate_report(report: SalesReport):
    ...
```

## Example B

```python

def load_some_data(path) -> pd.DataFrame:
    ...

def filter_on_date(df: pd.DataFrame, start_date: date, end_date: date) -> pd.DataFrame:
    ...

def select_products(df: pd.DataFrame, products: set[Field]) -> pd.DataFrame:
    ...

def filter_between_dates(df: pd.DataFrame, start: date, end: date) -> pd.DataFrame:
    ...
```

Let's say the two approaches do the same thing. Example A is a lot clearer on the intent and how the various bits of information relate to each other. Example B is arguably just organising the use of an external API - the business logic is dependent on an external library.

Some might argue that example B might be more "reusable", but I would argue that they are just really weak abstractions over the Pandas API. In a lot of cases, the examples are far too simple to be reusable in their current form. Take `filter_between_dates`, for example. Immediately, we can think of several ways that this function isn't quite good enough for reuse. Are the dates inclusive or exclusive? What if we want to just the one side of the comparison (either before the end date _or_ after the start date)? We also should be passing in the name of the field that contains the date information. The final implementation would probably look something like this:

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

We have to ask ourselves, is this a worthwhile abstraction? If we're doing things properly, we should have tests for all the usecases that this function covers. So that means:
- Test for no start date inclusive
- Test for no start date exclusive
- Test for no end date inclusive
- Test for no end date exclusive
- Test for start and end inclusive
- Test for start and end exclusive
- Test for field name which isn't in the dataframe (should we raise our own exception? Or let Pandas's bubble up?)
- Test for no date value (err... why would we want to do this?)