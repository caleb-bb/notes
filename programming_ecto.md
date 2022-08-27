# Ecto 

## Schema
The `schema ... do` expression defines a struct, fundamentally, but it goes further and specifies what data type is accepted by each struct field, and it does this for the purpose of mapping things to the schema. This expression uses the `schema` macro, which is granted courtesy of `use Ecto.Schema`. Then `field/2` takes the  names of fields and their data types. We don't need a one-to-one mapping with the table in the db. We can just specify the fields we want in this schema. 

`timestamps/0` grants `inserted_at` and `updated_at` fields, which Ecto automatically updates for every update and insertion of a record.

We don't need to include an `id` column among the calls to `field/2`, since Ecto automatically generates an `id` column and sets it as our primary key. Most databases have an `is` column which is assumed to be the primary key. This, however, is not universally consistent. If our db uses something else as a primary key, say, `thing_id` in table `things`, then we want:

    field :thing_id, :id, primary_key: true

which tells Ecto that there's a field called `:thing_id` which is the primary key for this schema. `:id` indicates an integer-based primary key, and `:uuid` indicates a primary key using Ecto's UUID utility.

### Types
Ecto has 15 types, which correspond to Elixir types. The Ecto types are on the left. The Elixir types are on the right.

#### Ecto Types <-> Elixir Types
- :id <-> integer
- :binary\_id <-> binary
- :integer <-> integer
- :float <-> float 
- :boolean <-> boolean
- :string <-> UTF-8 Encoded String
- :binary <-> binary
- {:array, inner\_type} <-> list
- {:map} <-> map 
- :decimal <-> Decimal 
- :date <-> Date
- :time <-> Time 
- :naive_datetime <-> NaiveDateTime 
- :utc_datetime <-> DateTime

The `map`


こんにちわ！
$ \leftrightarrow $
