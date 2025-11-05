---
title: "SQL on FHIR"
date: 2025-08-30T08:45:41-07:00
description: "Part 2: Analyzing FHIR Data 101"
---

# Introduction to SQL on FHIR

FHIR resources are JSON documents with nested structures and arrays. This is great for supporting solutions and applications but makes them difficult to query with SQL or run analytical processes against. As Data Analysts, we want to be able to define mappings from JSON fields to tabular columns so as we can change FHIR resources into SAS Datasets, Pandas/Polars dataframes and other tabular formats we know and love. 

[SQL on FHIR](https://build.fhir.org/ig/FHIR/sql-on-fhir-v2/) is a common standard that defines these mappings. SQL on FHIR does this through View Definitions - specifications that define how to flatten common FHIR resources. ViewDefintions are themselves JSON files and can be exchanged and stored by FHIR Servers. 

As a recap, a FHIR Resource for a Patient may look like:
```json
{
  "resourceType": "Patient",
  "id": "stuart-mackle",
  "name": [
    {
      "use": "official",
      "family": "Mackle",
      "given": ["Stuart"]
    }
  ],
  "gender": "male",
  "birthDate": "1900-01-01",
  "address": [
    {
      "use": "home",
      "line": ["100 Example Street"],
      "city": "Seattle",
      "state": "WA",
      "postalCode": "98102",
      "country": "US"
    },
    {
      "use": "work",
      "line": ["100 SAS Campus Drive"],
      "city": "Cary",
      "state": "NC",
      "postalCode": "27513",
      "country": "US"
    }
  ]
}
```

# SQL on FHIR Specification

SQL on FHIR defines how to create tabular views by processing FHIR resources and ViewDefinitions. View Definitions support `select` and `column` operations for mapping fields in JSON, as well as `union` and `forEach` which allow us to iterate over data (such as printing out every address for a patient).

Here's a simple ViewDefinition that creates a patient demographics table:

```json
{
  "resourceType": "ViewDefinition",
  "id": "patient-demographics",
  "name": "patient_demographics",
  "status": "active",
  "resource": "Patient",
  "select": [
    {
      "column": [
        {
          "name": "id",
          "path": "id",
          "type": "string"
        },
        {
          "name": "given_name",
          "path": "name.where(use='official').given.first()",
          "type": "string"
        },
        ...
      ]
    }
  ]
}
```

ViewDefinitions can extract basic information and flatten it into a simple table with columns for ID, name, birth date, and gender. Note that each ViewDefinition operates on a single resource, this means that creating a table like `patient_x_observation` will require two ViewDefinitions and running a query in a database or analytics runtime of your choice.

Notice the query language: `name.where(use='official').given.first()`. This is FHIRPath, a JSON query language also supported by HL7 and commonly used within FHIR applications. SQL on FHIR uses this as a way to map the columns (in this case `given_name`) to objects within the FHIR resources.

# Functions

When we're defining a SQL on FHIR View Definitions we're using the column function in combination with forEach, select, unionAll to produce a set of results. If we only want one output row per resource generally we're going to do a select function with a list of configured columns.

Column is the most basic function of SQL on FHIR, it is a list of mappings (name: output column name, path: FHIRPath to select from resource, column type).
```json
"select": {
  "column": [
    {
      "name": "id",
      "path": "id",
      "type": "string"
    }
  ]
}
```
This will operate at the resource level if we define it as above. It emits a table with a single row "id" for each resource.

However if we use a forEach parameter such as "address" we can produce multiple rows:
```json
"select": [
  {
    "column": {
      "name": "first_name",
      "path": "Patient.name[0].given.first()"
    }
  },
  {
    "forEach": "address",
    "column": [
      {
        "name": "zip_code",
        "path": "postalCode"
      }
    ]
  }
]
```
This can produce multiple rows for each resource as a Patient resource can have multiple [addresses](https://www.hl7.org/fhir/patient-definitions.html#Patient.address). It can iterate over any defined fields which have multiple values or many cardinality. As such, the output data set will contain multiple rows for some patients who have multiple addresses (like my example above) in their record:
```
+------------+----------+
| first_name | zip_code |
+------------+----------+
| Stuart     | 98102    |
| Stuart     | 27513    |
| John       | 90210    |
+------------+----------+
```

Similarly to this we can unionAll to emit multiple rows when we have sub-selects that have the same fields:
```json
"select": {
  "unionAll": [
    {
      "forEach": "address",
      "column": [
        {"path": "postalCode", "name": "zip_code"},
      ]
    },
    {
      "forEach": "contact.address",
      "column": [
        {"path": "postalCode", "name": "zip_code"},
      ]
    }
  ]
}
```
This will produce one row for each entry of `address` and `contact.address`, e.g. if we have a patient with one contact address and an address history containing 4 addresses we will have 5 output rows.

Note this uses forEach in the nested select but there is no requirement to use forEach, they can be simple selects which only contain column mappings.

# Implementation 

I recently open sourced an [SQL on FHIR implementation](https://github.com/sassoftware/sqlonfhir/tree/main) in Python at SAS. This library can be installed with:
{{< highlight bash>}}
pip install sqlonfhir
{{< / highlight >}}

The interface is simple consisting of one function `evaluate(resources, view_definition)`, which is demonstrated in this example:
{{<  highlight python >}}
from sqlonfhir import evaluate

view_definition = {
    "resource": "Patient",
    "column": [
        {"name": "id", "path": "id"},
        {"name": "name", "path": "name.given.first()"}
    ]
}

resources = [
    {
        "resourceType": "Patient",
        "id": "patient1",
        "name": [{"given": ["John"], "family": "Doe"}]
    }
]

result = evaluate(resources, view_definition)
print(result)
{{< / highlight >}}

Other [implementations](https://sql-on-fhir.org/extra/impls.html) also exist and support translating FHIR data into other formats (such as parquet, MSSQL) or running on other platforms (such as DuckDB or Spark).

I would really recommend using the [Synthea Project](https://synthea.mitre.org/) to generate or download some mock FHIR data to use with this library as a first introduction to SQL on FHIR.