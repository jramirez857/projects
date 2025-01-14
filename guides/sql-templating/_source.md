---
jupyter:
  jupytext:
    text_representation:
      extension: .md
      format_name: markdown
      format_version: '1.3'
      jupytext_version: 1.13.0
  kernelspec:
    display_name: Python 3 (ipykernel)
    language: python
    name: python3
---

# SQL templating

<!-- start description -->
Introductory tutorial teaching how to develop modular SQL pipelines.
<!-- end description -->

## Basic templating

SQL templating is a powerful way to make your SQL scripts more concise. It works by using a templating language (Ploomber uses [jinja](https://github.com/pallets/jinja)) to generate SQL code on the fly.

If you've followed the SQL pipelines tutorial in the *Get started* section, you've already used SQL templating. Let's take a look at the structure of a SQL script in Ploomber:

```postgresql
{% set product = SQLRelation(['my_schema', 'my_table', 'table']) %}

DROP TABLE IF EXISTS {{product}};

CREATE TABLE {{product}} AS
SELECT * FROM {{upstream['clean']}}
WHERE x > 10
```

The `{% set .. %}` statement defines a `product` variable, this serves two purposes. First, it tells Ploomber that this script is going to create a table named `my_table` in a schema named `my_schema`, for any script that declares the current script as upstream dependency, Ploomber will pass this information so it can use it as input. Second, it allows us to declare the value once and re-use it in two places via the `{{product}}` placeholder.

If you modify this script and re-execute your pipeline, you'd want to overwrite any existing table, this is why we have the `DROP TABLE IF EXISTS ...;` statement before `CREATE TABLE ..;`, since both of them take the table name as argument, we can use the `{{product}}` placeholder.

Finally, the `{{upstream['clean']}}` placeholders tells Ploomber that the current script uses the product from a task named `clean` as input data. This defines the dependency relationship between these two scripts and also implies that the placeholder will be replaced by the actual table/view generated by the `clean` task.

These are the basic elements for templated SQL scripts, you don't have to use more if you don't want to but sometimes is convenient to do so to write concise code and maximize reusability.

## Control structures

jinja offers control structures that help us write SQL code on the fly. Say we want to compute summary statistics on a given column:

```postgresql
SELECT
    some_column,
    AVG(another_column) as avg_another_column,
    STDEV(another_column) as stdev_another_column,
    COUNT(another_column) as count_another_column,
    SUM(another_column) as sum_another_column,
    MAX(another_column) as max_another_column,
    MIN(another_column) as min_another_column,
FROM some_table
GROUP BY some_column
```

This code is very repetitive, now imagine how repetitive. We can generate the same code succinctly using a for loop:

```postgresql
SELECT
    some_column,
-- loop over aggregation functions
{% for fn in ['AVG', 'STDEV', 'COUNT', 'SUM', 'MAX', 'MIN'] %}
    -- apply function to the column, name the column
    -- and only add a comma if we are not in the last loop element
    {{fn}}({{col_agg}}) as {{fn}}_{{col_agg}}{{ ',' if not loop.last else '' }}
{% endfor %}
FROM some_table
GROUP BY some_column
```


## Macros

Macros let us to maximize SQL code reusability by defining snippets that we can "import" in other files. To define a macro, enclose your snippet between the  `{% macro MACRO_NAME %} ... {% endmacro %}` tags. Let's create a macro using our previous snippet:


<% expand('sql/macros.sql') %>

The `{% macro %}` tag defines the macro name and parameters (if any). To use our macro in a different file, we have to import it. Let's say we define the previous macro in a `macros.sql` file:

<% expand('sql/create-table.sql') %>

### Configuring support for macros

To work with macros, we have to make a small change to our `pipeline.yaml` file. So far, to specify which SQL files to use, we've just passed the path the file in the `source` key. To be able to import macros in our scripts we have to configure a source loader.

A source loader is simply a folder with files, with a small addition: it defines an "jinja environment" that makes imports work (to know more about jinja environments, [click here](https://jinja.palletsprojects.com/en/2.11.x/api/#basics).

Let's say all the scripts in our pipeline are in a `sql/` directory. `sql/` has two scripts, which correspond to the files shown in the previous section:

```sh
tree sql
```

To configure our source loader. We just need to add a `source_loader` section like this:

<% expand('pipeline.yaml') %>

## Printing rendered code

Templated SQL help us write more concise SQL code, but if your template renders to an invalid SQL script, you'll get syntax errors, only use it when the benefits outweigh this risk. One way to debug SQL templates is to see how the rendered code looks like, you can do so from the command line:

```sh
ploomber task sql-task --source
```

As we can see, our template is generating a valid SQL script. But if it didn't it'd be easier to spot errors in the rendered code than in the templated source.

## Where to go next

* [Jinja documentation](https://jinja.palletsprojects.com/en/2.11.x/templates/)

