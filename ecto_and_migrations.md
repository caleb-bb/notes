# Ecto And Migrations

## Backfilling

Backfilling data is when we change data in bulk, possibly inferring the backfilled data from what's already there. For example, if we have one field for `student_name` and we want to split it into two fields, we would be backfilling first and last name by extrapolating from the full name in the `student_name` field. Ideally, we would like for our code to be safe to re-run. So if we can make it idempotent when the data is already good (i.e. doing nothing if the first and last name fields exist) that wouldd be one way to make it safe to run. We can accomplish bulk data manipulation via Ecto migrations (see **Data vs. Schema** below).

Examples of backfilling include,
- populating data in (a) new column(s)
- changing a column to make it required
- splitting a table into several 
- fixing bad data 

There are four keys to backfilling safely:

1. Running outside a transaction
2. batching
3. throttling
4. resiliency

To accomplish point 1, we use

    @disable_ddl_transaction true
    @disable_migration_lock true

to disable migration transactions.

The second key, batching, is necessary because databases can be, like, really big. So if you don't batch, you can eat up CPU, cause crashes, and various other horrible things. Batching basically means processing the data a few rows at a time, which means some kind of pagination. The kind of pagination we're stuck with outside of transactions is keyset pagination (see **keyset pagination**).

To query and update the data, which we need to do if we're gonna backfill, we prefer using `execute(query)` with raw SQL that represents the table at the moment that we use this. We avoid referencing existing Ecto schemas because, if the schemas change, then we get a different result the next time we run this migration. A second approach is to write a small Ecto schema *inside* of our migration that only uses what we need. This makes the migration self-contained so that it doesn't create issues upon being contaminated by a changed external schema that it references. 

For throttling, we just use `Process.sleep(@throttle)` for each page.

Resiliency means we should be able to handle errors without the whole migration failing. Most of the time, you're gonna find some records in a state you don't expect, e.g. a `first_name` field with a middle initial tacked on. In those cases, the migration will fail, and we have to revise our code. Resiliency means that our migration will pick up where it left off insteadf of starting all the way over.

Data migrations should be stored separately from schema migrations and run manually. 

### Batching in detail

#### Deterministic batching
If data is deterministic, we can backfill as follows:

1. Query in such a way that we only get results that need to be updated
2. Update those results (thus they can no longer be returned by the query)
3. If our query still returns results, return to step 1

In more detail:

1. Disable migration transactions 
2. Use keyset pagination: order the data, find rows greater than the last one we did, and limit by whatever batch size we want (see **keyset pagination**)
3. Mutate the records on the current page
4. Check for failed updates and handle appropriately
5. Use the last mutated record's ID as the starting point for the next page. This achieves resiliency because we save the argument to the `WHERE` clause even if the migration fails, meaning we can always pick up where we left off.
6. Sleep so we don't exhaust the database. 100ms is a decent amount of time to start.
7. If there's anything left, return to step 1

An example mifration for backfilling deterministic data:

    defmodule MyApp.Repo.DataMigrations.BackfillPosts do
      use Ecto.Migration
      import Ecto.Query

      @disable_ddl_transaction true
      @disable_migration_lock true
      @batch_size 1000
      @throttle_ms 100

      def up do
        throttle_change_in_batches(&page_query/1, &do_change/1)
      end

      def down, do: :ok

      def do_change(batch_of_ids) do
        {_updated, results} = repo().update_all(
          from(r in "weather", select: r.id, where: r.id in ^batch_of_ids),
          [set: [approved: true]],
          log: :info
        )
        not_updated = MapSet.difference(MapSet.new(batch_of_ids), MapSet.new(results)) |> MapSet.to_list()
        Enum.each(not_updated, &handle_non_update/1)
        results
      end

      def page_query(last_id) do
        # Notice how we do not use Ecto schemas here.
        from(
          r in "weather",
          select: r.id,
          where: is_nil(r.approved) and r.id > ^last_id,
          order_by: [asc: r.id],
          limit: @batch_size
        )
      end

      # If you have BigInt or Int IDs, fallback last_pos = 0
      # If you have UUID IDs, fallback last_pos = "00000000-0000-0000-0000-000000000000"
      defp throttle_change_in_batches(query_fun, change_fun, last_pos \\ 0)
      defp throttle_change_in_batches(_query_fun, _change_fun, nil), do: :ok
      defp throttle_change_in_batches(query_fun, change_fun, last_pos) do
        case repo().all(query_fun.(last_pos), [log: :info, timeout: :infinity]) do
          [] ->
            :ok

          ids ->
            results = change_fun.(List.flatten(ids))
            next_page = results |> Enum.reverse() |> List.first()
            Process.sleep(@throttle_ms)
            throttle_change_in_batches(query_fun, change_fun, next_page)
        end
      end

      defp handle_non_update(id) do
        raise "#{inspect(id)} was not updated"
      end
    end

#### Arbitrary batching

Okay, so we have arbitrary data. Since it's not deterministic, there's nothing intrinsic to the data that tells us whether we updated it or not. So, we have to introduce some external way of tracking that. We will backfill as follows:

1. Create a temporary table, i.e. a real table we drop at the end. We don't use a postgres temporary table because those don't offer resiliency. We need something that is still there in case our migration runs into an error. Temporary tables fall away when that happens, so we need a "real" table that isn't dropped if an error happens halfway through.
2. Populate our temporary table with the IDs of everything that needs to be updated. 
3. Index the temporary table so we can quickly delete IDs.
4. Pull batches of IDs from the temporary table uing keyset pagination.
5. For each batch (page) of records, make the required changes, if any.
6. Upsert changes records to the "real" table, meaning update them if they're already there and insert them if they're not. This will conflict with existing records, so we'll tell Postgres to deal with conflicts via replacement.
7. Delete those IDs from the temporary table that we just upserted.
8. Throttle.
9. If the temporary table is not empty, return to step 1. Otherwise, drop it.

So in this case, the temporary table functions as a list of all the things we need to update. Since this is arbitrary data and we can't tell just from looking at a record if that record should be updated, the temporary table is our external means of tracking the data that still needs to be dealt with. If it's still on the table, then it needs updating, and if the table is empty, then we're done.

We could create our backfilling migration by running `mix ecto.gen.migration --migrations-path=priv/repo/data_migrations backfill_weather`.

# Both of these modules are in the same migration file
# In this example, we'll define a new Ecto Schema that is a snapshot
# of the current underlying table and no more.

    defmodule MyApp.Repo.DataMigrations.BackfillWeather.MigratingSchema do
      use Ecto.Schema

      # Copy of the schema at the time of migration
      schema "weather" do
        field :temp_lo, :integer
        field :temp_hi, :integer
        field :prcp, :float
        field :city, :string

        timestamps(type: :naive_datetime_usec)
      end
    end

    defmodule MyApp.Repo.DataMigrations.BackfillWeather do
      use Ecto.Migration
      import Ecto.Query
      alias MyApp.Repo.DataMigrations.BackfillWeather.MigratingSchema

      @disable_ddl_transaction true
      @disable_migration_lock true
      @temp_table_name "records_to_update"
      @batch_size 1000
      @throttle_ms 100

      def up do
        repo().query!("""
        CREATE TABLE IF NOT EXISTS "#{@temp_table_name}" AS
        SELECT id FROM weather WHERE inserted_at < '2021-08-21T00:00:00'
        """, [], log: :info, timeout: :infinity)
        flush()

        create_if_not_exists index(@temp_table_name, [:id])
        flush()

        throttle_change_in_batches(&page_query/1, &do_change/1)

        # You may want to check if it's empty before dropping it.
        # Since we're raising an exception on non-updates
        # we don't have to do that in this example.
        drop table(@temp_table_name)
      end

      def down, do: :ok

      def do_change(batch_of_ids) do
        # Wrap in a transaction to momentarily lock records during read/update
        repo().transaction(fn ->
          mutations =
            from(
              r in MigratingSchema,
              where: r.id in ^batch_of_ids,
              lock: "FOR UPDATE"
            )
            |> repo().all()
            |> Enum.reduce([], &mutation/2)

          # Don't be fooled by the name `insert_all`, this is actually an upsert
          # that will update existing records when conflicting; they should all
          # conflict since the ID is included in the update.

          {_updated, results} = repo().insert_all(
            MigratingSchema,
            mutations,
            returning: [:id],
            # Alternatively, {:replace_all_except, [:id, :inserted_at]}
            on_conflict: {:replace, [:temp_lo, :updated_at]},
            conflict_target: [:id],
            placeholders: %{now: NaiveDateTime.utc_now()},
            log: :info
          )
          results = Enum.map(results, & &1.id)

          not_updated =
            mutations
            |> Enum.map(& &1[:id])
            |> MapSet.new()
            |> MapSet.difference(MapSet.new(results))
            |> MapSet.to_list()

          Enum.each(not_updated, &handle_non_update/1)
          repo().delete_all(from(r in @temp_table_name, where: r.id in ^results))

          results
        end)
      end

      def mutation(record, mutations_acc) do
        # This logic can be whatever you need; we'll just do something simple
        # here to illustrate
        if record.temp_hi > 1 do
          # No updated needed
          mutations_acc
        else
          # Upserts don't update autogenerated fields like timestamps, so be sure
          # to update them yourself. The inserted_at value should never be used
          # since all these records are already inserted, and we won't replace
          # this field on conflicts; we just need it to satisfy table constraints.
          [%{
            id: record.id,
            temp_lo: record.temp_hi - 10,
            inserted_at: {:placeholder, :now},
            updated_at: {:placeholder, :now}
          } | mutations_acc]
        end
      end

      def page_query(last_id) do
        from(
          r in @temp_table_name,
          select: r.id,
          where: r.id > ^last_id,
          order_by: [asc: r.id],
          limit: @batch_size
        )
      end

      defp handle_non_update(id) do
        raise "#{inspect(id)} was not updated"
      end

      # If you have BigInt IDs, fallback last_pod = 0
      # If you have UUID IDs, fallback last_pos = "00000000-0000-0000-0000-000000000000"
      # If you have Int IDs, you should consider updating it to BigInt or UUID :)
      defp throttle_change_in_batches(query_fun, change_fun, last_pos \\ 0)
      defp throttle_change_in_batches(_query_fun, _change_fun, nil), do: :ok
      defp throttle_change_in_batches(query_fun, change_fun, last_pos) do
        case repo().all(query_fun.(last_pos), [log: :info, timeout: :infinity]) do
          [] ->
            :ok

          ids ->
            case change_fun.(List.flatten(ids)) do
              {:ok, results} ->
                next_page = results |> Enum.reverse() |> List.first()
                Process.sleep(@throttle_ms)
                throttle_change_in_batches(query_fun, change_fun, next_page)
              error ->
                raise error
            end
        end
      end
    end



## Executing migrations

### change, down, up
The `up` keyword denotes actions to be taken during migration, while `down` denotes actions to be taken during rollback. `up` and `down` are to be used when the actions taken in a simple `change` are too complex for Ecto to infer how to reverse them.

### flush/0 
Recall that instructions inside of a migration are like transactions: they do not execute immediately, but only when the relevant `up`, `down`, or `change` callback terminates. 

`flush/0` is a kind of checkpoint, which will refuse to continue with a migration until the prior steps have been exe uted. So suppose we have steps like this:

1. Create a column 
2. Create another column 
3. Backfill those two columns 

In this case, we would alter our migration like so:

1. Create a column 
2. Create another column
3. flush (make sure 1 and 2 have been applied to the table before proceeding)
4. Backfill those two columns 

We would need to use `flush/0` for this because you cannot backfill columns that don't exist yet.

## Keyset Pagination
There are plenty of ways to paginate database tables and most of them suck. LIMIT/OFFSET is very expensive CPU-wise and can lead to duplication and other errors because the table can be written to as we're traversing pages. Cursors are better, but they require transactions. 

If we can't use transactions, we need keyset pagination. The general pattern for keyset pagination is thus:

    SELECT * FROM t AS OF SYSTEM TIME ${time}
        WHERE key > ${value}
        ORDER BY key
        LIMIT ${amount}

So we're taking `amount` rows at a time (given by the `LIMIT` clause) and looking at that many records, with the lowest one being specified by our `WHERE` clause. Think of each "page" as having `amount` number of entries, and of the first entry on each `page` having a value of `value` + 1. That's not precisely correct, but it gives you the gist.  

Some additional resources on keyset pagination:
https://www.cockroachlabs.com/docs/stable/pagination.html#keyset-pagination
https://www.citusdata.com/blog/2016/03/30/five-ways-to-paginate/

## Locks
In the broadest sense, a process "locks" a resource when that process prevents other processes from touching that resource. There are many different kinds of lock, and you can read about them at https://www.postgresql.org/docs/7.2/locking-tables.html

Some types of lock are supplied by Postgres so our app can use them and some of the locks modes are acquired by Postgres automatically before it executes certain statements. The point of locks is to manage concurrency, so different kinds of locks can (or can't) happen at the same time. For example, `AccessShareLock` is the weakest lock because it only conflicts with one other (`Access Exclusive`). An entire table can be locked, or just a single row at a time. When a row-level lock is acquired, the locked row cannot be written to, but can be queried.

Different locks are acquired by different database actions. For example, `UPDATE`, `DELETE`, and `INSERT` all acquire a `RowExclusiveLock`. This means that, if Postgres acquires any lock that conflicts with `RowExclusiveLock`, then we can't `UPDATE` or `DELETE` or `INSERT` while that other lock is active. 

Locks have the capacity to cause problems, because they can stop processes from getting to the database when they need to. We can get around this with safeguards. For example, we can use `lock_timeout` inside of a transaction to automatically cancel a statement if a lock is held too long. Inside of our migrations, we can use something like `execute("SET LOCAL lock_timeout TO '5s'", "SET LOCAL lock_timeout TO '10s'")` to set lock timeouts. We can also set this as a hook (see hooks below) that happens after a transaction begins. Since transactions need to set locks in order to prevent race conditions, it makes sense to set a lock timeout as a hook that happens after a transaction begins. Since migrations are transactions, they must acquire locks in order to run properly. And since locks can cause issues, we want ways of timing out a migration that waits too long.

We can also ensure safety using statement timeouts, which apply to all statements, not just locks. 


## Migrations
Ecto migrations fundamentally do two things: change the structure of the database, and change what's in the database. So migrations can introduce both syntactic (structural) and semantic (content) changes into the database. It can do the latter by adding new data or modifying data that already exists.

### Data vs. Schema 
A data migration is different from a schema migration. A schema migration is for setting up the database and is generally run by automatic workflows whenever we spin up our app. A data migration is for putting data into the database, and is generally a one-off deal

### Migration On Prod

Elixir used to be set up so you had to copy all your code to the server, install Elixir on it, and then run `mix ecto.migrate` and `mix phx.server`. These days, however, things are pre-compiled. This speeds up deployment, but raises the question of how we do the stuff on the server that we used to do using `mix` commands, since `mix` is no longer included on the server. We want to be able to check the status of migrations, migrate *up to* some specific migration (default latest migration), and rollback *down to* some specific migration for a particular repo.

In order to do all that, we generally have a `MyApp.Release` module that acts as an entry point for release-related tasks when the application is running on a server. So if there's something we want to be able to do on prod or staging, or some logs we would like to see ,`MyApp.Release` is where we would write our code. If, for example, we want a one-off function for running data migrations stored somewhere different from our schema migrations, then we would write that function into `MyApp.Release`.

### Mix Commands For Migrations
The basic generator for a migration is `mix ecto.gen.migration name_of_migration`, which will give you a time-stamped migration file in the appropriate directory. Phoenix also gives us a fancier generator, `mix phx.gen.schema` which gives you a migration that *also* lets you specify fields and types. You can type `mix help phx.gen.schema` to see exactly how that works. `mix ecto.migrate` will run the migration for you once it's created. Well, actually, it runs *all* of your migrations. If we want to see the exact SQL that Ecto is using, we can run `mix ecto.migrate --log-sql` and ecto will happily display the SQL it's using. However, the only SQL we see is the SQL used to implement our migrations. The `Ecto.Migrator` SQL that manages the migration is not shown. To see the migrator SQL, we can, appropriately, use `mix ecto.migrate --log-migrator-sql`. Easy-peasy. Note that migrations are actually SQL transactions, which will wind up being rolled back if any part of the migration fails. There is a `schema_migrations` table which is locked while migrations are run, so no other process can touch it during that time. Once the transaction is committed, the lock comes off.

There is also a `--migrations-path=WHATEVER/PATH/YOU/WOULD/LIKE` flag for `ecto.gen.migration`. This lets us stick our migration into a different folder than the default `priv/repo/migrations` folder. `mix ecto.migrate` looks in that folder, as does `mix ecto.reset`. So if we have a one-off data migration, this is a convenient way to stash them somewhere where they won't be taken up into automatic workflows.

### Recipes

There's a body of recipes at https://github.com/fly-apps/safe-ecto-migrations for common scenarios. These scenarios include adding an index, adding a reference/foreign key, and more.

#### Adding a column w/t default value
(skipping this one because the recipe for a safe migration is not needer in any version of Postgres > 11, and we're running 14)

#### Adding an index 
By default, creating an index in Postgres blocks reads and writes because of the locks it acquires. So as long as a migration that creates an index is running, nothing else can even read the table on which that index is being created. To get around this, we disable the migration transactions and run the index concurrently to avoid blocking reads, like so:

    @disable_ddl_transaction true
    @disable_migration_lock true

    def change do
      create index("posts", [:slug], concurrently: true)
    end

Since it's not being run as a transaction, it no longer locks the table. Since indexing a large database (~100 million rows) can take time on the order of minutes, locking the table can create serious problems. Note that, if we choose not to run a migration as a transaction, it is imperative to *only* create an index, and nothing else! Migrations are run as as traansactions by default for a reason: if the migration does more than one thing and it's not run as a transaction, we can find ourselves in a situation where some steps succeed and others fail. Imagine if your bank account balance went down, but the ATM didn't give you any money! 

#### Adding a reference or constraint 

When we add a foreign key constraint, we're making our table depend on the primary key for another table. Two things happen with this. First, we create a new constraint on how records can change going forward. Second, we validate that constraint for all existing records. These are two separate actions, so if they run in the same migration, they run as a transaction and lock the table while the whole thing is being scanned. To keep from blocking writes, we can split the migration up piecemeal. The first migration can just add the new constraint, with validation turned off:

    def change do
      alter table("posts") do
        add :group_id, references("groups", validate: false)
      end
    end

The second migration will just execute some SQL to validate the constraint.

    def change do
      execute "ALTER TABLE posts VALIDATE CONSTRAINT group_id_fkey", ""
    end
    
An analogous process can be used to add a check constraint.

For not-null constraints, which have the same effect of scanning the entire table, we can do somehting similar: add the constraint for new records, and then validate the old ones.

First migration:

    def change do
      create constraint("products", :active_not_null, check: "active IS NOT NULL"), validate: false
    end

Second migration:

    def change do
      execute "ALTER TABLE products VALIDATE CONSTRAINT active_not_null", ""
    end

In Postgres 12+, we can add a `NOT NULL` constraint to a column *after* validating said constraint. 
    
#### Changing column types 
Changing a column type may cause a table to be rewritten, which requires a lock. However, in Postgres, there are certain changes of column type that are safe i.e. will not cause a rewrite and lock the table. The safe changes are:

- increase length on varchar or removing the limit
- changing varchar to text
- changing text to varchar with no length limit
- postgres 9.2+ increasing precision (not scale) of decimal or numeric columns
- postgres 9.2+ changing decimal or numeric to be unconstrained
- postgres 12+ changing timestamp to timestamptz when session TZ is UTC

If we're making a change besides one of these in our pgdb, then we use the following process:

1. Create a new column 
2. In application code, write to both columns 
3. Backfill data from old column to new column 
4. In application code, move reads from old column to the new column
5. In application code, remove old column from Ecto schemas
6. Drop the old column

#### Removing a column
Removing a column can create bugs because old production code is querying based on the old column still being there. There are ways to remove old columns so nothing breaks when we roll out the change. We accomplish this by first removing the reference to the column in the schema, and then running the migration that removes the column from the table. Simple.

#### Renaming a column
Renaming a column can cause all sorts of headaches because Mr. Data no longer lives at his old address and that makes it hard on the mailman. A better strategy is to use this:

    defmodule MyApp.MySchema do
      use Ecto.Schema

      schema "weather" do
        field :temp_lo, :integer
        field :temp_hi, :integer
    -   field :prcp, :float
    +   field :precipitation, :float, source: :prcp
        field :city, :string

        timestamps(type: :naive_datetime_usec)
      end
    end

That `source: :prcp` makes it so all the code using this schema can reference a column called "precipitation" but still get its data from the old column and send updates to the old column and so on. So, Mr. Data stays at the same address, but he sets up mail forwarding.

If we really, really want to rename the old column, then we have to do so using a multi-stage approach, as follows:

1. Create a new column
2. In application code, write to both columns 
3. Backfill data from old column to new column
4. In application code, move reads from old column to new 
5. In application code, remove old column from Ecto schemas 
6. Drop the old column 

Step 1 gives us our new column. Step 2 lets us write everything to both columns. So far, this is seamless: the new column is being written, but only the old column is still being read from, so no bugs occur. Step 3 backfills all the data from the old column to the new one. Now that both columns are being written to, and both have the same historical data, there's no functional difference from them. That means we can proceed to Step 4, and begin reading from the new column instead of the old one; with all of our old data backfilled, there will be no change in functionality. Now we can go through and remove the old column from the Ecto schemas to complete Step 5, and then go to Step 6 and drop the old column from the DB. If this seems like a lot of work just to change a name, well, it is! That's why we prefer the first strategy here. 


## Misc
In databases, a "hook" is an action that occurs before or after a database operation. So, if I hash a password before committing it to the db, that's a hook. Note that beginning or ending a transaction is a database operation, so we can have hooks that run after the transaction begins (since the beginning is an operation) or before the transaction is committed.
