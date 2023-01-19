## Today I learned

### Mind-blowing locks 🔐

Today I saw a dbt `backup` table that couldn't acquire a `AccessExclusiveLock` to be dropped. This blew my mind and my first thought was "why dafuq is someone querying from a dbt backup table? (dbt backup table is the backup of the table to be refreshed with newest data, it will be dropped when the new table refresh is completed)

So dug around all the logs for that datetime, and found that no one was explicitly querying from the backup table; it was a delayed effect of the `ALTER TABLE original_table RENAME TO original_table_dbt_backup`. (if you can't understand, go find out how dbt fully-refreshes a table)

Imagine this scenario, in this sequence:

1. `ai_jialerchew` is selecting from `ai_accounts`.
2. dbt want to rename `ai_accounts`  to `ai_accounts_dbt_backup`, but blocked by 1.
3. `ai_xiaojun` is selecting from `ai_accounts`, but blocked by 2 (and 1).
4. New `ai_accounts` done refreshing. Dbt wants to drop `ai_accounts_dbt_backup`.

Once (1) completes, (2) will immediately complete. Will (3) return error? Can (4) run immediately after (2)?

I tested it out. (3) doesn’t return error wtf. It will automatically read from `ai_account_dbt_backup`. So while (3) is running, (4) cannot run. Hence there is a stuck `AccessExclusiveLock` on `ai_accounts_dbt_backup`. 🧠 = 💥


### CPU time ⏰

Also, today was the first time that we moved our BI tool from Redshift Provisioned, a data warehouse that has fixed compute resource, to Redshift Serverless (pay per use, automatically scales based on traffic).

We set WLM to allow maximum query time of 300s and maximum CPU time of 300s. What's "CPU time", you asked? I have no idea either.

Then, there was this one report that was killed by WLM after running for ~30s, because its CPU time exceeded 300s. I was thinking how do we adjust to a "reasonable" CPU time limit?

My strategy was: Run the query on Redshift Provisioned (no WLM limit), then see how much CPU time it consumed. Then we can use it as a reference.

#### How to get CPU time?

1. Try running the query on local IDE till completion.
2. Get the query statistics via: `select * from STL_QUERY_METRICS a left join stl_querytext b on a.query = b.query where a.userid = <you> and starttime > '2023-01-19 11:07:12.472'`
3. Get value of cpu_time for the row where segment and step_type = -1. It means the sum at query level.
