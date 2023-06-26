---
layout: post
title: "Hiding DBUtils"
---

# Hiding DBUtils

A common pattern that I've noticed with Databricks notebooks is the gratuitous use of the built-in `dbutils` module in the code.
DBUtils exposes some Databricks functionality such as running one notebook from another, manipulating the mounted storage DBFS and working with widgets.
This functionality can be accessed through the `dbutils` variable provided as a global in every notebook, similar to `spark`.
While the actions mentioned can be useful (except for widgets - these should be used very rarely for accessing very constrained user input), I have found that most
codebases over-use wigets and don't correctly separate them from core business logic.

## Over-availability

The chief reason for the mis-use of `dbutils`, in my opinion, is the fact that it is provided to the user as a global.
When a variable is automatically provided, it tends to be treated as a global, which means that it get's used very liberally, because you don't have to concern yourself with where it comes from.
This is flawed thinking - DBUtils is an external dependency (same as Spark) and should be treated as such: with extreme caution.
External dependencies change, and when they do they might change the arguments they take, their return values or the way that they function internally.
By using them all over the place, you are pretty much guarenteeing that when there is a breaking change, you will spend an awful long time trying to track down
every single place that you used one of these dependencies and fixing any bugs that arise due to the change.
If, on the other hand, you intentionally limit where you allow yourself to use one of these globally-provided variables, the amount of work you'll have to do is very small.
But that's not event the main issue with `dbutils`.

## Avoiding lock-in

The key issue with using pre-provided, non-builtin globals such as `dbutils` or `spark` is that to ever run any of your code, they need to be present.
In fact, to even lint your code, you need to do some fudging of the settings so that the linter "pretends" the variables exist.
This is a huge barrier to testing, debugging and even general development.
By simply just using `dbutils` in your code, your effectively prevent your code from running anywhere except inside a Databricks notebook.
This is more of a problem than you might realise, because now to run any tests you need to be running in Databricks, which means having a cluster spun up and some sort of testing/development workspace.
Forget about having assurance through a simple CI/CD flow that runs as part of your version control with every pull request, you'll have to set up some sort
of complicated pipeline where the code is shipped to a Databricks workspace and waits for a cluster to be spun up for it before anything can be run.
The same thing for development - I often like being able to run an arbitrary part of my code locally and then drop into the debugger to see how different pieces interact.
Forget doing anything of the sort if you have non-builtin globals - you'll need to spin up a cluster ($$$).
The harsh reality is that if linting and testing have such a high barrier to entry, you just won't bother doing them.
Not to mention that even if you are sufficiently motivated, your company's security policy might not let you access your development Databricks workspace from
a Github Actions pipeline.

The auto-inclusion of spark leads to a similar issue.
What if your company decides to migrate back to on-prem (crazier things have happened...) and you now have to create `spark` yourself?
Well if it exists as a global throughout your code, that's not exactly going to be a quick fix.

## So what to do?

So how can we guard against this?
The variables are already provided, surely it's a bit clunky to just pretend they don't exist?
Well actually, that's exactly what you do.
Look at the example below:

```python
spark: SparkSession # some mypy nonsense...
dbutils: DBUtils    # more mypy nonsense...

df = some_transformations(spark.table(...))
df.createOrReplaceTempView("my_view")

results = dbutils.notebook.run("./some/notebook/here", timeout_seconds=0, arguments={"view_name": "my_view"})

```

