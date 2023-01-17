## More about incrementals

It's not all rainbows and butterflies when it comes to building incremental models. Imagine this: `table_a` has upstreams `table_b`, `table_c`, and `table_d`, all joined together. How do you check if a record is considered "more recent" than the existing record, and needs to be updated?

### Prerequisite: When did this table last refreshed?

Every table needs to have a `commit_time` column where it specifies the most recent insert/update datetime of this record.

Back to the table A,B,C,D example. We proceed to build `table_a_tmp` using `table_b`, `table_c`, and `table_d`. Since we have the `commit_time` of each row from each upstream table, the output of `GREATEST(table_b.commit_time, table_c.commit_time, table_d.commit_time)` will return the "most recent update" datetime of this particular row. If the output is more recent than the previous update datetime (checkpoint) of `table_a`, then it will be updated/inserted into `table_a`.

Of course I'm oversimplying the problem here. There's still alot of possibility such as aggregates, window functions, deletions, etc. Maybe I'll share the guide I wrote here sometime in the future. It still needs more _incremental_ improvements 😆.

Ok I'm really sick of writing about incremental. Let's stop here.

## Culture of writing

Coming from a team with zero documentation, I can feel the power of documenting our works. I can't emphasize how much has the docs written by my fellow SB-ers enabled me to work independently without disturbing others. I can simply search and read most of the info I need.

Not to mention, my boss said "documenting your tasks is part of actual work". Give him a like.

Guess I'll stop here today. Signing off.
