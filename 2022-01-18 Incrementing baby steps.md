## Passed my probation!

Yay!

To be honest, aside from some ad-hoc analytical/bug-catching/ingestion tasks, a huge portion of my onboarding has been on a single theme - incremental models.

### Incremental why?

Like most of you reading this, I'm sure the first thought is:

> _If the table is super huge, dropping and recreating can take alot of time. By only upserting new data, we can save time and computation resources!_

This is only part of the story. We'll come back to this later.

Another task that I'm super proud of is solving a bug where Shoppingtrips attribution were disappearing after a certain period ("Shoppingtrips" = when a SB user visits a merchant. "Attribution" = who is the driver behind this Shoppingtrip? facebook ads? refer-a-friend? organic?).

This is the story.

Shoppingtrips table is one of the largest table in SB. At the time of writing this, the table has 1b+ rows. Hence, logically the team had turned this table into incremental mode. When the table refreshes, it only upserts the shoppingtrips for the last 30 days.

Raw shoppingtrips data doesn't have attribution info. They come from another source (Branch and Appsflyer). Hence, there is a `LEFT JOIN` involved between the shoppingtrips table and the attribution table. The attribution data also only "lookback" for 30 days.

Then, users start noticing that for older data in the `JOIN`-ed table, most of the shoppingtrips did not have attribution (default = organic), and we know it is wrong because earlier reports showed otherwise (and also raw data from attribution table). This meant that attributions were "going missing".

The team suspected that it was due to attribution data arriving later than 30 days. To resolve this, the "lookback" period was increased from 30 days to 60 days, and eventualy 90 days. However, it did not solve the problem.

After lots of time spent swimming in shoppingtrips and attribution data, I found a pattern where the `JOIN`-ed table only started showing signs of missing attribution when it is 3 months old. This means that, for recent 3 months shoppingtrip data, the `JOIN`-ed table is consistent with the raw data; however, for shoppingtrips that are older than 3 months, the disappearance rate sky-rocketed.

I suspected that it had to do with the "lookback" period of 90 days.

Long story short, we mananged to solve this issue by making sure that the attribution table has a longer "lookback" period than shoppingtrips table. We switched the shoppingtrips lookback period to 80 days, and it resolved the issue. I even revisited this ticket recently (4 months later) to make sure that it was still working.

Super proud of myself on this one. Ayam Goreng McD to celebrate.

## Coming back to this later

Another huge reason to go for incremental models is because it doesn't involve DDL statements in the updating process. To explain that, first see this example of how a "full refresh" table works:

1. Create `table_A_tmp` using latest data.
2. Rename `table_A` to `table_A_backup`.
3. Rename `table_A_tmp` to `table_A`.
4. Drop `table_A_backup`.

The problem is on Step 2. `ALTER TABLE RENAME` is a DDL statement. This means that it forces a lock on the table that no one else can use it until the DDL statment has ended. So, if the renaming step is unable to execute because (i) somehow it's stuck, highly unlikely, or (2) someone else is using it, and will continue using it for a long time. The renaming step will need to wait for the query ahead to complete before it can start. Also, due to "first-come-first-serve", all subsequent READ queries on this tables can only be executed when the renaming step is completed.

Now, we end up in a deadlock. Dashboards are not working, models are not building, everyone is not happy, it's 12am I'm hungry.

Incremental, on the other hand, only involves `DELETE` and `INSERT` statements. These are not DDL statements. Hence, queries can still run while the upsert process is running.

You might be thinking: How can others still read from this table when it has records being inserted/deleted? Will two people, running the same query on the table while it's upserting, 1 second apart, end up with different results?

### Bonus time

A table never truly "deletes" a record. When we update/delete a row, what's happening backend is actually inserting new records into this table, then "switch off" the outdated/deleted rows. Hence, until the upsert statement is commited, all queries will still read from the old version of the table.

