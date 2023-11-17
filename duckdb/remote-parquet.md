# Summing columns in remote Parquet files using DuckDB

[vivym/midjourney-messages](https://huggingface.co/datasets/vivym/midjourney-messages) on Hugging Face is a large (~8GB) dataset consisting of 55,082,563 Midjourney images - each one with the prompt and a URL to the image hosted on Discord.

TLDR: Each record links to a Discord CDN URL, and the total size of all of those images is 148.07 TB - so Midjourney has cost Discord a LOT of money in CDN costs!

Read on for the details of how I figured this out.

Each of the records looks like this (converted to JSON):

```json
{
  "id": "1144508197969854484",
  "channel_id": "989268300473192561",
  "content": "**adult Goku in Dragonball Z, walking on a beach, in a Akira Toriyama anime style** - Image #1 <@1016225582566101084>",
  "timestamp": "2023-08-25T05:46:58.330000+00:00",
  "image_id": "1144508197693046875",
  "height": 1024,
  "width": 1024,
  "url": "https://cdn.discordapp.com/attachments/989268300473192561/1144508197693046875/anaxagore54_adult_Goku_in_Dragonball_Z_walking_on_a_beach_in_a__987e6fd5-64a1-43f6-83dd-c58d2eb42948.png",
  "size": 1689284
}
```
The records are hosted on Hugging Face as Parquet files - 56 of them, each around 160MB. Here's [the full list](https://huggingface.co/datasets/vivym/midjourney-messages/tree/main/data) - they're hosted in Git LFS, but Hugging Face also offers an HTTP download link.

I wanted to total up the `size` column there to see how space on Discord's CDN is taken up by these Midjourney images. But I didn't want to download all 8GB of data just to run that query.

DuckDB can query remotely hosted Parquet files and use HTTP Range header tricks to retrieve just the subset of data needed to answer a query. It should be able to sum up the `size` column without downloading the whole dataset.

The URL to each Parquet file looks like this (this actually redirects to a final URL, but DuckDB seems to be able to follow those redirects transparently):

    https://huggingface.co/datasets/vivym/midjourney-messages/resolve/main/data/000000.parquet

## Querying a single file

Here's a DuckDB query that calculates the sum of that `size` column without retrieving the entire file:

```sql
SELECT SUM(size) FROM 'https://huggingface.co/datasets/vivym/midjourney-messages/resolve/main/data/000000.parquet';
```
Running that in the `duckdb` CLI tool gives me this result:

```
┌───────────────┐
│   sum(size)   │
│    int128     │
├───────────────┤
│ 3456458790156 │
└───────────────┘
```
That's 3.14TB, just for the first file.

## Tracking network usage with nettop

How many bytes of data did that retrieve? We can find out on macOS using the `nettop` command.

First we need the PID of our `duckdb` process:

```
ps aux | grep duckdb
simon            19992   0.0  0.0 408644352   1520 s114  S+    2:30PM   0:00.00 grep duckdb
simon            19985   0.0  0.0 408527616   4752 s118  S+    2:30PM   0:00.01 duckdb
```
Now we can run the following, before we execute that SQL query:
```bash
nettop -p 19985
```
Then I ran the SQL query, and saw this in the output from `nettop`:
```
5331 KiB
```
So DuckDB retrieved 5.3MB of data (from a file that was 159MB) to answer that query.

## Using UNION ALL

How about the total across all 56 files? I generated this `UNION ALL` query to answer that question:

```sql
SELECT SUM(size_total)
FROM (
    SELECT SUM(size) as size_total FROM 'https://huggingface.co/datasets/vivym/midjourney-messages/resolve/main/data/000000.parquet' UNION ALL
    SELECT SUM(size) as size_total FROM 'https://huggingface.co/datasets/vivym/midjourney-messages/resolve/main/data/000001.parquet' UNION ALL
    SELECT SUM(size) as size_total FROM 'https://huggingface.co/datasets/vivym/midjourney-messages/resolve/main/data/000002.parquet' UNION ALL
    SELECT SUM(size) as size_total FROM 'https://huggingface.co/datasets/vivym/midjourney-messages/resolve/main/data/000003.parquet' UNION ALL
    SELECT SUM(size) as size_total FROM 'https://huggingface.co/datasets/vivym/midjourney-messages/resolve/main/data/000004.parquet' UNION ALL
    SELECT SUM(size) as size_total FROM 'https://huggingface.co/datasets/vivym/midjourney-messages/resolve/main/data/000005.parquet' UNION ALL
    SELECT SUM(size) as size_total FROM 'https://huggingface.co/datasets/vivym/midjourney-messages/resolve/main/data/000006.parquet' UNION ALL
    SELECT SUM(size) as size_total FROM 'https://huggingface.co/datasets/vivym/midjourney-messages/resolve/main/data/000007.parquet' UNION ALL
    SELECT SUM(size) as size_total FROM 'https://huggingface.co/datasets/vivym/midjourney-messages/resolve/main/data/000008.parquet' UNION ALL
    SELECT SUM(size) as size_total FROM 'https://huggingface.co/datasets/vivym/midjourney-messages/resolve/main/data/000009.parquet' UNION ALL
    SELECT SUM(size) as size_total FROM 'https://huggingface.co/datasets/vivym/midjourney-messages/resolve/main/data/000010.parquet' UNION ALL
    SELECT SUM(size) as size_total FROM 'https://huggingface.co/datasets/vivym/midjourney-messages/resolve/main/data/000011.parquet' UNION ALL
    SELECT SUM(size) as size_total FROM 'https://huggingface.co/datasets/vivym/midjourney-messages/resolve/main/data/000012.parquet' UNION ALL
    SELECT SUM(size) as size_total FROM 'https://huggingface.co/datasets/vivym/midjourney-messages/resolve/main/data/000013.parquet' UNION ALL
    SELECT SUM(size) as size_total FROM 'https://huggingface.co/datasets/vivym/midjourney-messages/resolve/main/data/000014.parquet' UNION ALL
    SELECT SUM(size) as size_total FROM 'https://huggingface.co/datasets/vivym/midjourney-messages/resolve/main/data/000015.parquet' UNION ALL
    SELECT SUM(size) as size_total FROM 'https://huggingface.co/datasets/vivym/midjourney-messages/resolve/main/data/000016.parquet' UNION ALL
    SELECT SUM(size) as size_total FROM 'https://huggingface.co/datasets/vivym/midjourney-messages/resolve/main/data/000017.parquet' UNION ALL
    SELECT SUM(size) as size_total FROM 'https://huggingface.co/datasets/vivym/midjourney-messages/resolve/main/data/000018.parquet' UNION ALL
    SELECT SUM(size) as size_total FROM 'https://huggingface.co/datasets/vivym/midjourney-messages/resolve/main/data/000019.parquet' UNION ALL
    SELECT SUM(size) as size_total FROM 'https://huggingface.co/datasets/vivym/midjourney-messages/resolve/main/data/000020.parquet' UNION ALL
    SELECT SUM(size) as size_total FROM 'https://huggingface.co/datasets/vivym/midjourney-messages/resolve/main/data/000021.parquet' UNION ALL
    SELECT SUM(size) as size_total FROM 'https://huggingface.co/datasets/vivym/midjourney-messages/resolve/main/data/000022.parquet' UNION ALL
    SELECT SUM(size) as size_total FROM 'https://huggingface.co/datasets/vivym/midjourney-messages/resolve/main/data/000023.parquet' UNION ALL
    SELECT SUM(size) as size_total FROM 'https://huggingface.co/datasets/vivym/midjourney-messages/resolve/main/data/000024.parquet' UNION ALL
    SELECT SUM(size) as size_total FROM 'https://huggingface.co/datasets/vivym/midjourney-messages/resolve/main/data/000025.parquet' UNION ALL
    SELECT SUM(size) as size_total FROM 'https://huggingface.co/datasets/vivym/midjourney-messages/resolve/main/data/000026.parquet' UNION ALL
    SELECT SUM(size) as size_total FROM 'https://huggingface.co/datasets/vivym/midjourney-messages/resolve/main/data/000027.parquet' UNION ALL
    SELECT SUM(size) as size_total FROM 'https://huggingface.co/datasets/vivym/midjourney-messages/resolve/main/data/000028.parquet' UNION ALL
    SELECT SUM(size) as size_total FROM 'https://huggingface.co/datasets/vivym/midjourney-messages/resolve/main/data/000029.parquet' UNION ALL
    SELECT SUM(size) as size_total FROM 'https://huggingface.co/datasets/vivym/midjourney-messages/resolve/main/data/000030.parquet' UNION ALL
    SELECT SUM(size) as size_total FROM 'https://huggingface.co/datasets/vivym/midjourney-messages/resolve/main/data/000031.parquet' UNION ALL
    SELECT SUM(size) as size_total FROM 'https://huggingface.co/datasets/vivym/midjourney-messages/resolve/main/data/000032.parquet' UNION ALL
    SELECT SUM(size) as size_total FROM 'https://huggingface.co/datasets/vivym/midjourney-messages/resolve/main/data/000033.parquet' UNION ALL
    SELECT SUM(size) as size_total FROM 'https://huggingface.co/datasets/vivym/midjourney-messages/resolve/main/data/000034.parquet' UNION ALL
    SELECT SUM(size) as size_total FROM 'https://huggingface.co/datasets/vivym/midjourney-messages/resolve/main/data/000035.parquet' UNION ALL
    SELECT SUM(size) as size_total FROM 'https://huggingface.co/datasets/vivym/midjourney-messages/resolve/main/data/000036.parquet' UNION ALL
    SELECT SUM(size) as size_total FROM 'https://huggingface.co/datasets/vivym/midjourney-messages/resolve/main/data/000037.parquet' UNION ALL
    SELECT SUM(size) as size_total FROM 'https://huggingface.co/datasets/vivym/midjourney-messages/resolve/main/data/000038.parquet' UNION ALL
    SELECT SUM(size) as size_total FROM 'https://huggingface.co/datasets/vivym/midjourney-messages/resolve/main/data/000039.parquet' UNION ALL
    SELECT SUM(size) as size_total FROM 'https://huggingface.co/datasets/vivym/midjourney-messages/resolve/main/data/000040.parquet' UNION ALL
    SELECT SUM(size) as size_total FROM 'https://huggingface.co/datasets/vivym/midjourney-messages/resolve/main/data/000041.parquet' UNION ALL
    SELECT SUM(size) as size_total FROM 'https://huggingface.co/datasets/vivym/midjourney-messages/resolve/main/data/000042.parquet' UNION ALL
    SELECT SUM(size) as size_total FROM 'https://huggingface.co/datasets/vivym/midjourney-messages/resolve/main/data/000043.parquet' UNION ALL
    SELECT SUM(size) as size_total FROM 'https://huggingface.co/datasets/vivym/midjourney-messages/resolve/main/data/000044.parquet' UNION ALL
    SELECT SUM(size) as size_total FROM 'https://huggingface.co/datasets/vivym/midjourney-messages/resolve/main/data/000045.parquet' UNION ALL
    SELECT SUM(size) as size_total FROM 'https://huggingface.co/datasets/vivym/midjourney-messages/resolve/main/data/000046.parquet' UNION ALL
    SELECT SUM(size) as size_total FROM 'https://huggingface.co/datasets/vivym/midjourney-messages/resolve/main/data/000047.parquet' UNION ALL
    SELECT SUM(size) as size_total FROM 'https://huggingface.co/datasets/vivym/midjourney-messages/resolve/main/data/000048.parquet' UNION ALL
    SELECT SUM(size) as size_total FROM 'https://huggingface.co/datasets/vivym/midjourney-messages/resolve/main/data/000049.parquet' UNION ALL
    SELECT SUM(size) as size_total FROM 'https://huggingface.co/datasets/vivym/midjourney-messages/resolve/main/data/000050.parquet' UNION ALL
    SELECT SUM(size) as size_total FROM 'https://huggingface.co/datasets/vivym/midjourney-messages/resolve/main/data/000051.parquet' UNION ALL
    SELECT SUM(size) as size_total FROM 'https://huggingface.co/datasets/vivym/midjourney-messages/resolve/main/data/000052.parquet' UNION ALL
    SELECT SUM(size) as size_total FROM 'https://huggingface.co/datasets/vivym/midjourney-messages/resolve/main/data/000053.parquet' UNION ALL
    SELECT SUM(size) as size_total FROM 'https://huggingface.co/datasets/vivym/midjourney-messages/resolve/main/data/000054.parquet' UNION ALL
    SELECT SUM(size) as size_total FROM 'https://huggingface.co/datasets/vivym/midjourney-messages/resolve/main/data/000055.parquet'
);
```

## Using read_parquet()

**Update:** Alex Monahan [tipped me off](https://twitter.com/__AlexMonahan__/status/1724605446689108459) to a more concise alternative for the same query:

```sql
SELECT SUM(size)
FROM read_parquet([
    'https://huggingface.co/datasets/vivym/midjourney-messages/resolve/main/data/000000.parquet',
    'https://huggingface.co/datasets/vivym/midjourney-messages/resolve/main/data/000001.parquet',
    'https://huggingface.co/datasets/vivym/midjourney-messages/resolve/main/data/000002.parquet',
    'https://huggingface.co/datasets/vivym/midjourney-messages/resolve/main/data/000003.parquet',
    'https://huggingface.co/datasets/vivym/midjourney-messages/resolve/main/data/000004.parquet',
    'https://huggingface.co/datasets/vivym/midjourney-messages/resolve/main/data/000005.parquet',
    'https://huggingface.co/datasets/vivym/midjourney-messages/resolve/main/data/000006.parquet',
    'https://huggingface.co/datasets/vivym/midjourney-messages/resolve/main/data/000007.parquet',
    'https://huggingface.co/datasets/vivym/midjourney-messages/resolve/main/data/000008.parquet',
    'https://huggingface.co/datasets/vivym/midjourney-messages/resolve/main/data/000009.parquet',
    'https://huggingface.co/datasets/vivym/midjourney-messages/resolve/main/data/000010.parquet',
    'https://huggingface.co/datasets/vivym/midjourney-messages/resolve/main/data/000011.parquet',
    'https://huggingface.co/datasets/vivym/midjourney-messages/resolve/main/data/000012.parquet',
    'https://huggingface.co/datasets/vivym/midjourney-messages/resolve/main/data/000013.parquet',
    'https://huggingface.co/datasets/vivym/midjourney-messages/resolve/main/data/000014.parquet',
    'https://huggingface.co/datasets/vivym/midjourney-messages/resolve/main/data/000015.parquet',
    'https://huggingface.co/datasets/vivym/midjourney-messages/resolve/main/data/000016.parquet',
    'https://huggingface.co/datasets/vivym/midjourney-messages/resolve/main/data/000017.parquet',
    'https://huggingface.co/datasets/vivym/midjourney-messages/resolve/main/data/000018.parquet',
    'https://huggingface.co/datasets/vivym/midjourney-messages/resolve/main/data/000019.parquet',
    'https://huggingface.co/datasets/vivym/midjourney-messages/resolve/main/data/000020.parquet',
    'https://huggingface.co/datasets/vivym/midjourney-messages/resolve/main/data/000021.parquet',
    'https://huggingface.co/datasets/vivym/midjourney-messages/resolve/main/data/000022.parquet',
    'https://huggingface.co/datasets/vivym/midjourney-messages/resolve/main/data/000023.parquet',
    'https://huggingface.co/datasets/vivym/midjourney-messages/resolve/main/data/000024.parquet',
    'https://huggingface.co/datasets/vivym/midjourney-messages/resolve/main/data/000025.parquet',
    'https://huggingface.co/datasets/vivym/midjourney-messages/resolve/main/data/000026.parquet',
    'https://huggingface.co/datasets/vivym/midjourney-messages/resolve/main/data/000027.parquet',
    'https://huggingface.co/datasets/vivym/midjourney-messages/resolve/main/data/000028.parquet',
    'https://huggingface.co/datasets/vivym/midjourney-messages/resolve/main/data/000029.parquet',
    'https://huggingface.co/datasets/vivym/midjourney-messages/resolve/main/data/000030.parquet',
    'https://huggingface.co/datasets/vivym/midjourney-messages/resolve/main/data/000031.parquet',
    'https://huggingface.co/datasets/vivym/midjourney-messages/resolve/main/data/000032.parquet',
    'https://huggingface.co/datasets/vivym/midjourney-messages/resolve/main/data/000033.parquet',
    'https://huggingface.co/datasets/vivym/midjourney-messages/resolve/main/data/000034.parquet',
    'https://huggingface.co/datasets/vivym/midjourney-messages/resolve/main/data/000035.parquet',
    'https://huggingface.co/datasets/vivym/midjourney-messages/resolve/main/data/000036.parquet',
    'https://huggingface.co/datasets/vivym/midjourney-messages/resolve/main/data/000037.parquet',
    'https://huggingface.co/datasets/vivym/midjourney-messages/resolve/main/data/000038.parquet',
    'https://huggingface.co/datasets/vivym/midjourney-messages/resolve/main/data/000039.parquet',
    'https://huggingface.co/datasets/vivym/midjourney-messages/resolve/main/data/000040.parquet',
    'https://huggingface.co/datasets/vivym/midjourney-messages/resolve/main/data/000041.parquet',
    'https://huggingface.co/datasets/vivym/midjourney-messages/resolve/main/data/000042.parquet',
    'https://huggingface.co/datasets/vivym/midjourney-messages/resolve/main/data/000043.parquet',
    'https://huggingface.co/datasets/vivym/midjourney-messages/resolve/main/data/000044.parquet',
    'https://huggingface.co/datasets/vivym/midjourney-messages/resolve/main/data/000045.parquet',
    'https://huggingface.co/datasets/vivym/midjourney-messages/resolve/main/data/000046.parquet',
    'https://huggingface.co/datasets/vivym/midjourney-messages/resolve/main/data/000047.parquet',
    'https://huggingface.co/datasets/vivym/midjourney-messages/resolve/main/data/000048.parquet',
    'https://huggingface.co/datasets/vivym/midjourney-messages/resolve/main/data/000049.parquet',
    'https://huggingface.co/datasets/vivym/midjourney-messages/resolve/main/data/000050.parquet',
    'https://huggingface.co/datasets/vivym/midjourney-messages/resolve/main/data/000051.parquet',
    'https://huggingface.co/datasets/vivym/midjourney-messages/resolve/main/data/000052.parquet',
    'https://huggingface.co/datasets/vivym/midjourney-messages/resolve/main/data/000053.parquet',
    'https://huggingface.co/datasets/vivym/midjourney-messages/resolve/main/data/000054.parquet',
    'https://huggingface.co/datasets/vivym/midjourney-messages/resolve/main/data/000055.parquet'
]);
```
This version displays a useful progress bar while the query is executing:

![The results of the query with a progress bar at 100%](https://github.com/simonw/til/assets/9599/59dbc802-5fb7-4638-8248-9079a796811f)

## Using list_transform()

[@adityawarmanfw shared](https://twitter.com/adityawarmanfw/status/1724775939048182158) this much more elegant solution:

```sql
SELECT
    SUM(size) AS size
FROM read_parquet(
    list_transform(
        generate_series(0, 55),
        n -> 'https://huggingface.co/datasets/vivym/midjourney-messages/resolve/main/data/' ||
            format('{:06d}', n) || '.parquet'
    )
);
```
This is using a [DuckDB lambda function](https://duckdb.org/docs/sql/functions/nested.html#lambda-functions) - really neat!

To measure them, I ran a query in a fresh DuckDB instance with `nettop` watching the network traffic. Here's what that looked like while it was running:

![Animated GIF of nettop showing different connections being made and how much bandwidth is used for each one](https://github.com/simonw/til/assets/9599/6e7f1e07-4d76-43d7-a2b7-81daba8c99ca)

The total data transferred was 287 MiB - still a lot of data, but a big saving on 8GB.

That's also around what I'd expect for 56 files, given that a single file fetched 5.3MB earlier and 5.3 * 56 = 296.8.

## The end result

The result it gave me was:
```
┌─────────────────┐
│ sum(size_total) │
│     int128      │
├─────────────────┤
│ 162800469938172 │
└─────────────────┘
```
That's 162800469938172 bytes - or 148.07 TB of CDN space used by Midjourney images!

(I got ChatGPT to build me a tool for converting bytes to KB/MB/GB/TB: [Byte Size Converter](https://til.simonwillison.net/tools/byte-size-converter).)

## CTEs and views work too

You can run this query using a CTE to make it nicer to read:

```sql
with midjourney_messages as (
    select
        *
    from read_parquet(
        list_transform(
            generate_series(0, 2),
            n -> 'https://huggingface.co/datasets/vivym/midjourney-messages/resolve/main/data/' ||
                format('{:06d}', n) || '.parquet'
        )
    )
)
select sum(size) as size from midjourney_messages;
```
(I used `generate_series(0, 2)` here instead of `(0, 55)` to speed up these subsequent experiments.)

Or you can define a view, which lets you refer to `midjourney_messages` in multiple queries. This is a bad idea though, as each time you execute the query against the view it will download the data again:

```sql
create view midjourney_messages as
select * from read_parquet(
    list_transform(
        generate_series(0, 2),
        n -> 'https://huggingface.co/datasets/vivym/midjourney-messages/resolve/main/data/' ||
            format('{:06d}', n) || '.parquet'
    )
);
```
Running `create view` here transferred 37 KiB according to `nettop` - presumably from loading metadata in order to be able to answer `describe` queries like this one:
```sql
describe midjourney_messages;
```
Output:
```
┌─────────────┬─────────────┬─────────┬─────────┬─────────┬───────┐
│ column_name │ column_type │  null   │   key   │ default │ extra │
│   varchar   │   varchar   │ varchar │ varchar │ varchar │ int32 │
├─────────────┼─────────────┼─────────┼─────────┼─────────┼───────┤
│ id          │ VARCHAR     │ YES     │         │         │       │
│ channel_id  │ VARCHAR     │ YES     │         │         │       │
│ content     │ VARCHAR     │ YES     │         │         │       │
│ timestamp   │ VARCHAR     │ YES     │         │         │       │
│ image_id    │ VARCHAR     │ YES     │         │         │       │
│ height      │ BIGINT      │ YES     │         │         │       │
│ width       │ BIGINT      │ YES     │         │         │       │
│ url         │ VARCHAR     │ YES     │         │         │       │
│ size        │ BIGINT      │ YES     │         │         │       │
└─────────────┴─────────────┴─────────┴─────────┴─────────┴───────┘
```
`nettop` confirmed that each time I ran this query:
```sql
select count(*) from midjourney_messages;
```
Approximately 114 KiB of data was fetched.