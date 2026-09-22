# Hands-on L6: Report

**Name:** Soumyanil Ain
**Student ID:** 801488534
**Email:** sain@charlotte.edu

---

## Seed and commands

Seed used for `datagen.py`: **801488534**

The commands I ran, in order (Windows, PowerShell, inside the repository folder):

```bash
py datagen.py 801488534
docker compose up -d

# run the starter main.py once, before filling anything in (all columns read as strings)
docker cp main.py spark-master:/opt/spark/work-dir/
docker exec -it spark-master /opt/spark/bin/spark-submit --master spark://spark-master:7077 /opt/spark/work-dir/main.py /opt/spark/work-dir/shared/input /opt/spark/work-dir/shared/output

# after writing the schema and the four tasks
docker cp main.py spark-master:/opt/spark/work-dir/
docker exec -it spark-master /opt/spark/bin/spark-submit --master spark://spark-master:7077 /opt/spark/work-dir/main.py /opt/spark/work-dir/shared/input /opt/spark/work-dir/shared/output

# final clean run, program output saved to a file, Spark logs left in the terminal
docker cp main.py spark-master:/opt/spark/work-dir/
docker exec spark-master /opt/spark/bin/spark-submit --master spark://spark-master:7077 /opt/spark/work-dir/main.py /opt/spark/work-dir/shared/input /opt/spark/work-dir/shared/output > run_output.txt

docker compose down
```

Where I deviated from the README:

- I used `py` instead of `python3`, because on Windows `python3` only points to the Microsoft Store.
- I wrote each `spark-submit` on one line, because PowerShell doesn't understand the `\` line continuation used in the README.
- For the second run I temporarily added `input("Press Enter to finish...")` before `spark.stop()` so the Spark UI at port 4040 would stay open while I looked at it. I removed it afterwards (see Problems and fixes).
- For the last run I dropped `-it` and redirected stdout to `run_output.txt`, so the schema, tables and plan ended up in one file without the INFO log lines.

---

## Results

Before adding the schema, `printSchema()` showed every column as a string:

```
root
 |-- user_id: string (nullable = true)
 |-- song_id: string (nullable = true)
 |-- timestamp: string (nullable = true)
 |-- duration_sec: string (nullable = true)
```

After adding `logs_schema`:

```
root
 |-- user_id: string (nullable = true)
 |-- song_id: string (nullable = true)
 |-- timestamp: timestamp (nullable = true)
 |-- duration_sec: integer (nullable = true)

1000 log rows, 50 songs
```

### Task 1: favorite genre per user

```
+--------+---------+----------+
|user_id |genre    |play_count|
+--------+---------+----------+
|user_1  |Classical|9         |
|user_10 |Hip-Hop  |5         |
|user_100|Jazz     |5         |
|user_11 |Hip-Hop  |7         |
|user_12 |Rock     |6         |
|user_13 |Rock     |7         |
|user_14 |Pop      |4         |
|user_15 |Classical|6         |
|user_16 |Pop      |6         |
|user_17 |Pop      |4         |
+--------+---------+----------+
```

All 100 users get a favorite. Jazz is the most common favorite (25 users), then Pop (24), Hip-Hop (20), Rock (16) and Classical (15), even though Jazz has the fewest songs in the catalog (7 of 50). The order looks odd (`user_1, user_10, user_100, user_11`) because `user_id` is a string, so it sorts alphabetically, not numerically.

### Task 2: average listening time per song

```
+-------+-------------+----------------+----------+
|song_id|title        |avg_duration_sec|play_count|
+-------+-------------+----------------+----------+
|song_40|Title_song_40|204.81          |21        |
|song_26|Title_song_26|201.94          |16        |
|song_35|Title_song_35|195.4           |25        |
|song_47|Title_song_47|187.25          |16        |
|song_41|Title_song_41|185.06          |17        |
|song_25|Title_song_25|182.55          |20        |
|song_1 |Title_song_1 |180.89          |37        |
|song_3 |Title_song_3 |179.85          |20        |
|song_6 |Title_song_6 |178.23          |31        |
|song_37|Title_song_37|177.08          |12        |
+-------+-------------+----------------+----------+
```

The averages range from about 133 to 205 seconds across the 50 songs, around the midpoint of the 30 to 300 second range in the data. Most songs only have 10 to 25 plays, so a few unusually long or short plays move an average a lot. The top of this list says more about random variation than about which songs people actually like listening to longer. `song_1` is the most played song with 37 plays.

### Task 3: genre loyalty score, top 10

```
+-------+---------+----------+-----------+-------------+
|user_id|genre    |play_count|total_plays|loyalty_score|
+-------+---------+----------+-----------+-------------+
|user_24|Hip-Hop  |8         |8          |1.0          |
|user_83|Pop      |7         |7          |1.0          |
|user_32|Rock     |5         |5          |1.0          |
|user_55|Pop      |4         |4          |1.0          |
|user_29|Jazz     |3         |3          |1.0          |
|user_89|Rock     |14        |15         |0.933        |
|user_67|Classical|13        |14         |0.929        |
|user_81|Jazz     |12        |13         |0.923        |
|user_25|Rock     |8         |9          |0.889        |
|user_94|Rock     |8         |9          |0.889        |
+-------+---------+----------+-----------+-------------+
```

Five users only ever played their favorite genre and get a perfect 1.0. The ties at 1.0 are sorted by `total_plays`, so user_24 with 8 plays comes before user_29 with 3.

Why do users with few plays tend to get a score of 1.0? Would you change the definition of the score to account for that?

With only a handful of plays, it's easy for all of them to land in one genre just by chance, and one genre then counts for 100%. user_29 has 3 plays out of 3, while user_89 has 14 out of 15, and I'd say user_89 is the more convincingly loyal listener even though their score is lower. I would change the score. The simplest fix is to only rank users with a minimum number of plays, for example at least 10. Another option is to smooth the score, for example `(play_count + 1) / (total_plays + 5)`, which pulls users with little data toward an average and lets users with many plays keep a score close to their real share.

### Task 4: night owls

```
+-------+-----------+
|user_id|night_plays|
+-------+-----------+
|user_61|7          |
|user_18|6          |
|user_37|6          |
|user_41|5          |
|user_5 |5          |
|user_45|4          |
|user_54|4          |
|user_6 |4          |
|user_62|4          |
|user_72|4          |
+-------+-----------+
```

89 of the 100 users played at least one song between midnight and 5 AM, 211 plays in total, which is about 21% of all plays. That's almost exactly 5 hours out of 24, so the timestamps look spread evenly over the day, and there aren't any real night owls in this data. user_61 leads with only 7 night plays.

---

## The plan

The `explain()` output of task 1:

```
== Physical Plan ==
AdaptiveSparkPlan isFinalPlan=false
+- Sort [user_id#0 ASC NULLS FIRST], true, 0
   +- Exchange rangepartitioning(user_id#0 ASC NULLS FIRST, 200), ENSURE_REQUIREMENTS, [plan_id=975]
      +- Project [user_id#0, genre#7, play_count#28L]
         +- Filter (rn#38 = 1)
            +- Window [row_number() windowspecdefinition(user_id#0, play_count#28L DESC NULLS LAST, genre#7 ASC NULLS FIRST, specifiedwindowframe(RowFrame, unboundedpreceding$(), currentrow$())) AS rn#38], [user_id#0], [play_count#28L DESC NULLS LAST, genre#7 ASC NULLS FIRST]
               +- WindowGroupLimit [user_id#0], [play_count#28L DESC NULLS LAST, genre#7 ASC NULLS FIRST], row_number(), 1, Final
                  +- Sort [user_id#0 ASC NULLS FIRST, play_count#28L DESC NULLS LAST, genre#7 ASC NULLS FIRST], false, 0
                     +- Exchange hashpartitioning(user_id#0, 200), ENSURE_REQUIREMENTS, [plan_id=968]
                        +- WindowGroupLimit [user_id#0], [play_count#28L DESC NULLS LAST, genre#7 ASC NULLS FIRST], row_number(), 1, Partial
                           +- Sort [user_id#0 ASC NULLS FIRST, play_count#28L DESC NULLS LAST, genre#7 ASC NULLS FIRST], false, 0
                              +- HashAggregate(keys=[user_id#0, genre#7], functions=[count(1)])
                                 +- Exchange hashpartitioning(user_id#0, genre#7, 200), ENSURE_REQUIREMENTS, [plan_id=962]
                                    +- HashAggregate(keys=[user_id#0, genre#7], functions=[partial_count(1)])
                                       +- Project [user_id#0, genre#7]
                                          +- BroadcastHashJoin [song_id#1], [song_id#4], Inner, BuildRight, false, false
                                             :- Filter isnotnull(song_id#1)
                                             :  +- FileScan csv [user_id#0,song_id#1] Batched: false, DataFilters: [isnotnull(song_id#1)], Format: CSV, Location: InMemoryFileIndex(1 paths)[file:/opt/spark/work-dir/shared/input/listening_logs.csv], PartitionFilters: [], PushedFilters: [IsNotNull(song_id)], ReadSchema: struct<user_id:string,song_id:string>
                                             +- BroadcastExchange HashedRelationBroadcastMode(List(input[0, string, false]),false), [plan_id=957]
                                                +- Filter isnotnull(song_id#4)
                                                   +- FileScan csv [song_id#4,genre#7] Batched: false, DataFilters: [isnotnull(song_id#4)], Format: CSV, Location: InMemoryFileIndex(1 paths)[file:/opt/spark/work-dir/shared/input/songs_metadata.csv], PartitionFilters: [], PushedFilters: [IsNotNull(song_id)], ReadSchema: struct<song_id:string,genre:string>
```

My reading of it, from the bottom up:

**File scans.** The two `FileScan csv` lines at the bottom read `listening_logs.csv` and `songs_metadata.csv`. Spark only reads the columns it needs (`user_id, song_id` from the logs, `song_id, genre` from the songs), so `timestamp`, `duration_sec`, `title`, `artist` and `mood` are never read for this query. Each scan has a `Filter isnotnull(song_id)` that Spark added by itself, since a row with a null key can never match in an inner join.

**The join.** The operator is `BroadcastHashJoin` with `BuildRight`. Spark chose a broadcast join because the songs table is tiny (50 rows), so it sends a full copy of it to every executor through the `BroadcastExchange` and builds a hash table from it. The 1,000 log rows then stay where they are and never need to be shuffled for the join.

**Shuffles.** There are three `Exchange` operators:

1. `hashpartitioning(user_id, genre)`: after a partial count on each executor (`partial_count`), rows are shuffled so that all rows for the same (user, genre) pair end up in the same partition for the final count.
2. `hashpartitioning(user_id)`: the window function ranks each user's genres, so all of a user's rows have to be in one partition. Before this shuffle there's a partial `WindowGroupLimit`, where Spark notices that I only keep `rn = 1` and drops rows that can't be the top one before they're sent over the network.
3. `rangepartitioning(user_id)`: for the final `orderBy("user_id")`, rows are split into sorted ranges, and each partition is sorted locally, which gives a globally sorted result.

The `BroadcastExchange` also moves data, but it copies the small table to every executor instead of repartitioning anything.

**Comparison with the Spark UI.** In the SQL / DataFrame tab, task 1 shows up as two queries: query 2 (`showString`, from `show()`) and query 3 (`csv`, from the write). The diagram for query 2 has the same scans, `BroadcastExchange`, `BroadcastHashJoin`, `HashAggregate`, `WindowGroupLimit` and `Window` nodes, and the row counts on the edges add up: 1,000 log rows and 50 songs scanned, 1,000 rows after the join, 318 (user, genre) pairs after the aggregation, one row per user after the filter. There are two differences:

- The query 2 diagram has no third `Exchange rangepartitioning` or final `Sort`. Instead there is a `TakeOrderedAndProject`. `show()` only needs the first 20 rows (it takes 21, so it knows whether to print "only showing top 20 rows"), so Spark turns "sort everything, then take 21" into one top-k step without a full shuffle. Only 21 rows leave it. `explain()` runs on `favorite` without a limit, so the full plan above matches the write query instead.
- The diagram has `AQEShuffleRead` nodes after each `Exchange`, which are not in the `explain()` text. `explain()` printed the plan before it ran (`isFinalPlan=false`), while the UI shows the final plan after adaptive query execution. `AQEShuffleRead` is AQE merging the default 200 shuffle partitions into a few, since there is very little data.

---

## Transformations and actions

The actions in my `main.py` are:

- line 57: `logs.count()` and `songs.count()`
- line 33: `df.show(20, truncate=False)` inside `save()`, called once for each of the four tasks
- line 34: `df.coalesce(1).write...csv(...)` inside `save()`, also called once per task

That's 10 actions. Everything else is a transformation or only planning: `spark.read.csv` with a schema, `join`, `groupBy`, `agg`, `withColumn`, `filter`, `orderBy`, `limit`, `printSchema()` and `explain()`.

The Spark UI shows **42 completed jobs**. The SQL / DataFrame tab lists exactly 10 queries, one per action, with the jobs each one launched:

| Query | Action          | Jobs     |
| ----- | --------------- | -------- |
| 0     | `logs.count()`  | 0, 1     |
| 1     | `songs.count()` | 2, 3     |
| 2     | task 1 `show()` | 4 to 7   |
| 3     | task 1 write    | 8 to 13  |
| 4     | task 2 `show()` | 14 to 16 |
| 5     | task 2 write    | 17 to 21 |
| 6     | task 3 `show()` | 22 to 28 |
| 7     | task 3 write    | 29 to 35 |
| 8     | task 4 `show()` | 36, 37   |
| 9     | task 4 write    | 38 to 41 |

This isn't what I expected at first. I expected one job per action, so 10. The difference comes from adaptive query execution, which is on by default in Spark 4: each shuffle stage runs as its own job first, so Spark can look at the real data sizes before planning the next step, and then one more job produces the result. That's visible in the Jobs tab: the jobs marked `1/1` are those shuffle jobs, and the ones marked `(1 skipped)` or `(2 skipped)` are result jobs that reuse shuffle output that was already computed. A global `orderBy` also adds a job that samples the data to pick the range boundaries. Task 3 launches the most jobs because it runs the whole task 1 pipeline again, computes total plays per user, and then joins and sorts the two.

The `show()` and the write for the same task each rerun the full query from the CSV files, because nothing is cached. Calling `.cache()` on the task results would avoid that.

Two other things I noticed:

- Every job in the Jobs tab has the same description, `$anonfun$withThreadLocalCaptured$1 at CompletableFuture.java:1768`, because AQE submits the jobs from a background thread and the UI loses the line in my code. The SQL / DataFrame tab is where I could match jobs to lines (`count`, `showString` for `show()`, `csv` for the write).
- In the run of the starter code, without a schema, loading the logs launched a job on its own (`csv at ...`), which read the first line of the file to get the column names. With the schema declared, loading launched no job at all.

---

## Problems and fixes

1. **Placeholder brackets.** I first typed `python3 datagen.py <801488534>` and PowerShell refused it with `The '<' operator is reserved for future use.` The brackets were part of the placeholder, so I removed them.

2. **`python3` not found on Windows.** `python3 datagen.py 801488534` gave `Python was not found; run without arguments to install from the Microsoft Store, or disable this shortcut from Settings > Apps > Advanced app settings > App execution aliases.` On Windows, `python3` is an alias to the Store installer. Running `py datagen.py 801488534` (the Python launcher for Windows) worked.

3. **The Spark UI closed too fast, and my workaround hung.** The application UI on port 4040 disappears as soon as the program ends, so I added `input("Press Enter to finish...")` before `spark.stop()`. The program paused as planned, but pressing Enter did nothing. With `spark-submit`, the Python script runs as a child process of Spark's Java launcher, and keyboard input doesn't reach it. I used the paused UI to look at the Jobs and SQL / DataFrame tabs, then stopped the program with Ctrl + C and removed the `input()` line. All four outputs had already been written before the pause, so nothing was lost.
