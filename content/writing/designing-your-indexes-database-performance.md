---
title: "Designing Your Indexes for Better Database Performance"
date: 2023-03-20T00:00:00Z
draft: false
tags: ["databases", "performance", "indexing", "optimization", "storage"]
originalPost:
  site: "Polar Signals"
  url: "https://www.polarsignals.com/blog/posts/2023/03/20/designing-your-indexes"
---

At Polar Signals, we improved our database query performance by 80% by changing the primary index in our main database table. In this blog post, we’ll go through why the performance improved in our specific use case and try to develop an intuition as to how to design indexes for better query performance. I hope that by reading this blog post, you will come away with a better intuition around index design to improve your own database queries.

## What our schema looked like

In [Parca](https://github.com/parca-dev/parca) and at Polar Signals, we use [FrostDB](https://github.com/polarsignals/frostdb) to store profile samples. Although these samples are stored using a columnar layout under the hood, the ideas covered in this blog post generalize to row-oriented databases as well, so for conceptual simplicity, the data will be described as rows from now on.

In any database table, the primary index describes a set of columns that rows are ordered by. The simplified primary index for our samples looked like this:

\![The profile samples table schema.](data:image/gif;base64,R0lGODlhAQABAIAAAAAAAP///yH5BAEAAAAALAAAAAABAAEAAAIBRAA7)\![The profile samples table schema.](/_next/image?url=https%3A%2F%2Fcdn.sanity.io%2Fimages%2Fu4y7d6w6%2Fproduction%2F0f5731d413809036a9133f320f4e75c8da877e60-662x69.webp&w=3840&q=75)The profile samples table schema.
The **profile type** column describes what type of sample this is. For example, whether it is a memory profile or CPU profile. The **labels** column describes a set of labels associated with the sample. For example, the IP address of the host this sample was collected on, or a kubernetes pod name. The **stacktrace** column is a hash that describes the stack trace of the sample. The **timestamp** describes the time at which this sample was taken. The interpretation of the **value** column depends on the profile type. For memory profiles, for example, it is the number of bytes allocated by the stack trace and for cpu profiles, the number of times that stack trace was observed in a profile.

The real schema is a bit more complex; profile types are represented using multiple columns, and the labels column is what we call a ‘dynamic’ column, aka typed wide-columns. Refer to the [FrostDB readme](https://github.com/polarsignals/frostdb#readme) for more information. For the purposes of this blog post, the simplified schema suffices to illustrate the key ideas.

Let’s put some example data into our table:
\![The profile samples table with some example data.](data:image/gif;base64,R0lGODlhAQABAIAAAAAAAP///yH5BAEAAAAALAAAAAABAAEAAAIBRAA7)\![The profile samples table with some example data.](/_next/image?url=https%3A%2F%2Fcdn.sanity.io%2Fimages%2Fu4y7d6w6%2Fproduction%2F7e518db959bc900117c64cbb7830d2ccca25871e-773x469.webp&w=3840&q=75)The profile samples table with some example data.
One of the most important queries we execute against this data is an aggregation to display a timeseries for a given profile type and optional label set (e.g. a given host or an app). We call this a range query:
SELECT sum(value) GROUP BY labels, timestamp WHERE profile_type = $1 AND timestamp > $2 AND timestamp < $3 [AND labels = $4]
What this query does is filter by the desired profile type (e.g. CPU), timestamp range, and label set and perform an aggregation to get a given value grouped by the set of labels at each timestamp. The label set is optional because it is dependent on what the user is looking for. For example, a user might look at the CPU usage over the last 15 minutes for a given app on a given host, or an app on any given host.

Pushing filters down to the storage layer

To understand query performance, it’s useful to have a mental model of how a database is structured. There are many “layers” to a database that we will not cover in this blog post. You might have heard about the planning/optimization layer or the transactional layer. The two layers we care about for our purposes are the storage and execution layer. The storage layer is what physically stores your data (i.e. on memory or disk) and returns bytes according to some basic query parameters. The execution layer takes those bytes and executes arbitrarily complex data transformations (e.g. aggregations).

The key idea I want to transmit in this blog post is that a huge driver of database performance is making the query read as little data from the storage layer as possible. This matters because there is always an overhead associated with reading rows from the storage layer before they are processed by the execution layer; for example disk seeks or decompression of data.

Additionally, even if we were to assume that loading rows from storage into the execution layer is free, there is a key difference in the granularity at which filters can be applied between the storage and execution layer. In the storage layer, data is usually grouped together into larger chunks (e.g. SSTables or pages), enabling filtering of a chunk at a time based on the min and max values (as defined by the ordering) in that chunk. In the execution layer however, rows are generally filtered a row (or a batch of rows in very specific cases) at a time. The advantage of execution layer filtering is that it can execute arbitrary filters while the storage layer, although it allows for more efficient filtering, can only generally filter data if the filter expression specifies a prefix of the columns that data is ordered by on disk (there are also some cases where data can be filtered out even if this is not true, but that’s left as an exercise to the reader).

With these two ideas, it becomes clear that filtering as much as possible at the storage layer should improve query performance. To do this, we need to make sure that our filter plays nicely with how our data is ordered in the storage layer, which is defined by our primary index.

## Improving our schema

First of all, let’s see how well an example query does on the example data. To simplify the range query described above, we’ll only use the filter portion of the query above with some placeholders filled in:
SELECT * WHERE profile_type = 'CPU' AND timestamp > 1 AND timestamp < 4 AND labels.app = 'App 2'

For illustration purposes, assume the storage layer stores data in chunks of 5 rows. In the following diagram, rows are colored red if they are read from disk and green if they are rows that contribute to the final result:
\![An example of an inefficient range scan of the profile samples table.](data:image/gif;base64,R0lGODlhAQABAIAAAAAAAP///yH5BAEAAAAALAAAAAABAAEAAAIBRAA7)\![An example of an inefficient range scan of the profile samples table.](/_next/image?url=https%3A%2F%2Fcdn.sanity.io%2Fimages%2Fu4y7d6w6%2Fproduction%2F46389b9626e98d91d2c7b2b7a94cfcc913b85b64-905x483.webp&w=3840&q=75)An example of an inefficient range scan of the profile samples table.
As the diagram illustrates, both chunks (i.e. all rows) had to be read from the storage layer since each chunk has a row that satisfies the filter. This is the worst case scenario: we are paying the overhead of loading eight rows into memory that are not needed, as well as the cost to filter those rows out in the execution layer.

The problem with the original schema is that it doesn’t work very well with our filter. There is a labels column (the node name) that is not contained in the filter expression, which implies that the query would like results for all nodes. This makes it difficult to filter chunks based on the app column filter since min/max values of the app column in data chunks lose meaning. The first chunk in the above diagram, for example, has “App 1” as app values in the first and last row which does not satisfy the filter, but also contains “App 2” somewhere in the middle which does. This causes the storage layer to only be able to filter out chunks based on the profile type. For example, if there was a chunk of data with only memory profiles, that chunk will be filtered out completely by the storage layer, but that’s about it.

There is also a very constrained filter on the timestamp column that is essentially useless as a filter at the storage layer since the column is so low in the sort order that the filter cannot be meaningfully applied on a chunk of data. A casual observer might remark that these timestamp values look randomly generated.

Given these issues, it seems like a good decision to change the primary index so that rows sort by timestamp before the labels. Ideally we would also switch the order between the app and node columns, but as mentioned before, filters on these columns can be arbitrarily expressed by a user depending on which label set they are interested in visualizing. Changing this order would work well for our specific example query, but would not generalize to better performance across the board.

The following diagram illustrates executing the same example query with the proposed schema:
\![An example of an efficient range scan of the profile samples table.](data:image/gif;base64,R0lGODlhAQABAIAAAAAAAP///yH5BAEAAAAALAAAAAABAAEAAAIBRAA7)\![An example of an efficient range scan of the profile samples table.](/_next/image?url=https%3A%2F%2Fcdn.sanity.io%2Fimages%2Fu4y7d6w6%2Fproduction%2F75bff215315e4953787f4f8883d3f5be83d375c8-903x484.webp&w=3840&q=75)An example of an efficient range scan of the profile samples table.
This is clearly an improvement over the previous schema. Now that the storage layer can apply the filter on the timestamp column, only three rows are unnecessarily decompressed and filtered in the execution layer.

## Results

Theoretically, this seems like a significant improvement. How does this translate to real world numbers in our case? The result of benchmarking just the filter portion of a range query (no aggregation for emphasis) for the last 5 minutes over 30 minutes of data with the old schema and the new schema shows an 80% improvement:
name           old time/op    new time/op    delta
Query/Range-8    24.0ms ± 4%     5.0ms ± 1%  -79.09%  (p=0.000 n=8+9)

name           old alloc/op   new alloc/op   delta
Query/Range-8    77.0MB ± 4%    12.8MB ± 0%  -83.41%  (p=0.000 n=8+10)

name           old allocs/op  new allocs/op  delta
Query/Range-8     1.78M ± 5%     0.35M ± 0%  -80.60%  (p=0.000 n=8+10)
Note that this query is run on an in-memory dataset, so the theoretical performance improvement from applying these ideas to an on-disk dataset is likely larger given that the associated disk I/O costs are not reflected here.

## Conclusion

This blog post covered how we achieved an 80% performance improvement of our database queries by developing and applying some intuition around storage layer filter efficiency. The key ideas presented in this blog post are that storage layer filters are more efficient because larger chunks of data are filtered at a time, and that the less data that is loaded from the storage layer to the execution layer, the more efficient database queries become.

Hopefully this blog post was useful in building an intuition around how to design your indexes for better database performance. Let us know on [twitter](https://twitter.com/PolarSignalsIO), or [discord](https://discord.com/invite/ZgUpYgpzXy) if these ideas were useful to you!