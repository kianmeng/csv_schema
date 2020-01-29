# Csv Schema

[![Hex pm](https://img.shields.io/hexpm/v/csv_schema.svg?style=flat)](https://hex.pm/packages/csv_schema)
[![Build Status](https://travis-ci.org/primait/csv_schema.svg?branch=master)](https://travis-ci.org/primait/csv_schema)
[![License](https://img.shields.io/badge/License-MIT-blue.svg)](https://opensource.org/licenses/MIT)

Csv schema is a library helping you to build Ecto.Schema-like modules having a csv file as source.

The idea behind this library is give the possibility to create, at compile-time, a self-contained module exposing functions to retrieve data starting from a CSV.

## Installation

If [available in Hex](https://hex.pm/docs/publish), the package can be installed
by adding `csv_schema` to your list of dependencies in `mix.exs`:

```elixir
def deps do
  [
    {:csv_schema, "~> 0.2.0"}
  ]
end
```

## Usage

Supposing you have a CSV file looking like this:

id  | first_name | last_name  | email                         | gender | ip_address      | date_of_birth
:--:|:----------:|:----------:|:-----------------------------:|:------:|:---------------:|:------------:
1   | Ivory      | Overstreet | ioverstreet0@businessweek.com | Female | 30.138.91.62    | 10/22/2018
2   | Ulick      | Vasnev     | uvasnev1@vkontakte.ru         | Male   | 35.15.164.70    | 01/19/2018
3   | Chloe      | Freemantle | cfreemantle2@parallels.com    | Female | 133.133.113.255 | 08/13/2018
... | ...        | ...        | ...                           | ...    | ...             | ...

Is possible to create an Ecto.Schema-like repository using `Csv.Schema` macro:

```elixir
defmodule Person do
  use Csv.Schema
  alias Csv.Schema.Parser

  schema "path/to/person.csv" do
    field :id, "id", key: true
    field :first_name, "first_name", filter_by: true
    field :last_name, "last_name", sort: :asc
    field :identifier, ["first_name", "last_name"], key: true, join: " "
    field :email, "email", unique: true
    field :gender, "gender", filter_by: true, sort: :desc
    field :ip_address, "ip_address"
    field :date_of_birth, "date_of_birth", parser: &Parser.date!(&1, "{0M}/{0D}/{0YYYY}")
  end
end
```

Note that it's not a requirement to map all fields, but every field mapped must
have a column in csv file.
For example the following field configuration will result in a compilation error:

```elixir
field :id, "non_existing_id", ....
```

Schema could be configured using a custom separator
```elixir
use Csv.Schema, separator: ?,
```

Moreover it's possible to configure if csv file has or has not an header. Depending
on header param value field config changes:
```elixir
# Csv with header
schema "path/to/person.csv" do
  field :id, "id", key: true
  ...
end

# Csv without header. Note that field 1 is binded with the first csv column.
# Index goes from 1 to N
schema "path/to/person.csv" do
  field :id, 1, key: true
  ...
end
```

Now Person module is a struct, defined like this:

```elixir
defmodule Person do
  defstruct id: nil,
            first_name: nil,
            last_name: nil,
            email: nil,
            gender: nil,
            ip_address: nil,
            date_of_birth: nil
end
```

This macro creates for you inside Person module those functions:

```elixir
def by_id(integer_key), do: ...

def filter_by_first_name(string_value), do: ...

def by_email(string_value), do: ...

def filter_by_gender(string_value), do: ...

def get_all, do: ...
```

Where:
- `by_id` returns a `%Person{}` or `nil` if key is not mapped in csv
- `filter_by_first_name` returns a `[%Person{}, %Person{}, ...]` or `[]` if input predicate does not match any person
- `by_email` returns a `%Person{}` or `nil` if no person have provided email in csv
- `filter_by_gender` returns a `[%Person{}, %Person{}, ...]` or `[]` if input predicate does not match any person gender
- `get_all` return all csv rows as a Stream

## Field configuration

Every field should be formed like this:

```
field {struct_field}, {csv_header}, {opts}
```

where:
- `{struct_field}` will be the struct field name. Could be configured as `string` or as `atom`
- `{csv_header}` is the csv column name from where get values. Must be configured using string only
- `{opts}` is a keyword list containing special configurations

opts:
- `:key` : boolean. At most one key could be set. If set to true creates the `by_{name}` function for you.
- `:unique` : boolean. If set to true creates the `by_{name}` function for you. All csv values must be unique or an exception is raised
- `:filter_by` : boolean. If set to true creates the `filter_by_{name}` function
- `:parser` : function. An arity 1 function used to map values from string to a custom type
- `:sort` : `:asc` or `:desc`. It sorts according to Erlang's term ordering with `nil` exception (`number < atom < reference < fun < port < pid < tuple < list < bit-string < nil`)
- `:join` : string. If present it joins the given fields into a binary using the separator


Note that every configuration is optional

## Keep in mind

Compilation time increase in an exponential manner if csv contains lots of lines and you
configure multiple fields candidate for method creation (flags `key`, `unique` and/or `filter_by` set to true).

Because "without data you're just another person with an opinion" here some data:

csv rows | key | unique | filter_by | compile time ms
--------:|:---:|:------:|:---------:|----------------:
1_000    | no  | 0      | 0         |    419 ms
1_000    | yes | 1      | 1         |  1_980 ms
1_000    | yes | 2      | 2         |  2_542 ms
1_000    | yes | 2      | 4         |  3_565 ms
1_000    | yes | 2      | 0         |  1_758 ms
1_000    | yes | 0      | 4         |  2_090 ms
1_000    | no  | 2      | 0         |  1_634 ms
1_000    | no  | 0      | 4         |  1_971 ms
5_000    | no  | 0      | 0         |  2_410 ms
5_000    | yes | 1      | 1         | 15_282 ms
5_000    | yes | 2      | 2         | 22_478 ms
5_000    | yes | 2      | 4         | 28_060 ms
5_000    | yes | 2      | 0         | 16_254 ms
5_000    | yes | 0      | 4         | 15_043 ms
5_000    | no  | 2      | 0         | 14_518 ms
5_000    | no  | 0      | 4         | 12_931 ms
10_000   | no  | 0      | 0         |  4_962 ms
10_000   | yes | 1      | 1         | 28_995 ms
10_000   | yes | 2      | 2         | 42_817 ms
10_000   | yes | 2      | 4         | 54_759 ms
10_000   | yes | 2      | 0         | 37_166 ms
10_000   | yes | 0      | 4         | 29_913 ms
10_000   | no  | 2      | 0         | 33_578 ms
10_000   | no  | 0      | 4         | 29_096 ms

5 compilations average time.

Executed on my machine:

    Lenovo Thinkpad T480
    CPU: Intel(R) Core(TM) i7-8550U CPU @ 1.80GHz
    RAM: 32GB
