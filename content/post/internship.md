---
title: "CockroachDB internship project: Speeding up some interleaved table deletes by a factor of 10 billion"
date: 2018-11-16T04:49:15-08:00
draft: false
aliases:
    - /blog/5
---

<p>Last summer, I did an internship with <a href="https://www.cockroachlabs.com">Cockroach Labs</a>, makers of <a href="https://www.github.com/cockroachdb/cockroach">CockroachDB</a>, a SQL database built for massive distribution. I was working on the SQL language semantics in Cockroach, and I was able to work on many different facets of the project in that area.</p>
<p>Overall, my theme for the summer was finding ways to improve the performance of mutation statements - that's your <code>INSERT</code>s, <code>UPDATE</code>s, and <code>DELETE</code>s. At the tail end of the internship, I was able to contribute a major performance gain by adding a fast path to a particular kind of <code>DELETE</code>, involving a kind of table called an interleaved table. This post is about this particular performance fix and everything about how it works.</p>
<p>All the work described in this post actually comes from <a href="https://github.com/cockroachdb/cockroach/pull/28330">this pull request</a>.</p>
<!--more-->
<h2 id="whats-an-interleaved-table-anyway">What's an interleaved table, anyway?</h2>
<p>When you search &quot;interleaved table&quot;, the first results are <a href="https://www.cockroachlabs.com/docs/stable/interleave-in-parent.html">CockroachDB's own documentation</a> and that of competitor Google Cloud Spanner. Both have the same semantics, and revolve around the data in one table needing to have a strong relationship with another. This is generally the case when one data object &quot;belongs&quot; to another, possibly in a parent-child relationship. Formally, you can specify a &quot;child&quot; table to be interleaved in another &quot;parent&quot; table if and only if the parent's primary key is a prefix of the child's primary key.</p>
<p>Interleaving tables makes reads, joins, writes and other operations much faster when operating on both the parent and child tables, so they primarily should be used if you have two (or more) tables linked to each other, where you frequently have to access rows from both. They shouldn't be used recklessly, however: interleaved tables generally make reads and deletes over ranges a little slower, so when you don't have a strong data locality relationship as described above, specifying them as interleaved may not give you enough benefit for the trouble.</p>
<p>In Cockroach's case at least, specifying a table to be interleaved causes a change in the data's underlying representation to change in storage. CockroachDB uses a key-value store called RocksDB as its storage layer - rows of tables in the database are stored in RocksDB as one or more keys. Cockroach has a very good <a href="https://www.cockroachlabs.com/blog/sql-in-cockroachdb-mapping-table-data-to-key-value-storage/">comprehensive introduction</a> to how this mechanism works and <a href="https://github.com/cockroachdb/cockroach/blob/master/docs/tech-notes/encoding.md">extensive documentation</a> about all its idiosyncrasies.</p>
<p>When a table is specified as interleaved, the keys for the primary index of the child table contain the key corresponding to their associated parent row as a prefix. Here's an example:</p>
<p><img src="https://i.imgur.com/lLnBc6R.png" /></p>
<p>You can also have interleaved tables within interleaved tables, like this:</p>
<p><img src="https://i.imgur.com/xdsAbba.png" /></p>
<p>Because of how RocksDB stores its keys in sorted order (that's a bit of an oversimplification of what it does, actually), interleaving always puts the keys corresponding to the child table's primary key right in between the keys that correspond to the primary key of the row it's interleaved in. This leads to the aforemented speedups in the read, join, and write operations. The keys are much closer together and the database always knows precisely where the interleaved tables' keys were, and doesn't have to search.</p>
<p>At the time of the fix, however, some work still needed to be done to leverage this data transformation for the purpose of improving the performance of deletes.</p>
<h2 id="why-are-deletes-slow">Why are deletes slow?</h2>
<p>A child table in an interleaved relationship usually has a foreign key reference to its parent table. When this foreign key reference is set as <code>ON DELETE CASCADE</code>, rows of the child table are deleted when their parent rows are deleted. In order for CockroachDB to perform a cascading delete on a row from <code>parent</code> table, it performs the following procedure, which is just the same procedure it performs for all foreign key cascading deletes:</p>
<ul>
<li>Check all foreign key references to the row in <code>parent</code>
<ul>
<li>Sees a reference from a row in <code>child</code></li>
</ul></li>
<li>Check the relationship to inspect its cascade clause
<ul>
<li>Sees that it is <code>ON DELETE CASCADE</code></li>
</ul></li>
<li>Check if there are any references to <code>child</code> and continues until it's done</li>
<li>Perform all cascading operations an deletes the row from <code>parent</code>.</li>
</ul>
<p>It's very important to note that all of these checks are happening on a <strong>row-by-row</strong> basis. This is incredibly expensive; delete operations that required many foreign key checks were incredibly slow, even when deleting from the parent table of an interleaved relationship.</p>
<p>With interleaved tables, however, the way the keys are laid out allows for the creation of a fast path when the interleaved relationship is just right.</p>
<h2 id="vroom-vroom">Vroom Vroom</h2>
<p>The fast path works based on the premise that the child table's primary-key keys have the parent table's primary-key keys as a prefix. CockroachDB has a <code>DelRange</code> key-value operation that performs a series of deletes in RocksDB to speedily delete all the keys in a range of values. So, in order to delete a row from the parent table and all its corresponding child table rows, all you need to do is perform this &quot;range delete&quot; from the parent table key to the last key that has the key as its prefix. An illustration of this is below.</p>
<p><img src="https://i.imgur.com/PYW34FH.png" /></p>
<p>Of course, you can't just go using the fast path whenever there is an interleaved table --- when you do a key-value operation such as <code>DelRange</code> you are obviating the need for foreign-key and other safety checks, which need to be done beforehand in order for the fast path to work without possibly leaving you with a corrupted database. Luckily, checking whether the schemas are okay before doing the <code>DelRange</code> is still a lot cheaper than checking for foreign key constraints every time you delete a row.</p>
<p>In order to be able to perform the delete, the database has to make sure the following conditions are met:</p>
<ul>
<li>Every interleaved table and every table interleaved within any of the interleaved table need to have their foreign key reference set to <code>ON DELETE CASCADE</code>.</li>
<li>None of the tables involved should have foreign key references that exist other than the interleaved relationships themselves.</li>
<li>None of the tables involved should have secondary indexes. This needs to be checked because secondary indexes create keys that are outside of the interleaved table key layout, which only concerns the keys concerning the primary key index for each of the tables involved.</li>
</ul>
<p>To detect the fast path, I implemented a tree-walk (breadth-first search) through all of the schema information involved (with the parent table as the root) whenever a delete was performed on a table that had other tables interleaved in it. So this check would run whenever a user would do something like a <code>DELETE FROM parent WHERE id &gt; 4</code> or with any other <code>WHERE</code> clause. The check would check for the three criteria listed above, going through all of the tables interleaved in the parent table and adding any further interleaved relationships into a queue. Here's the relevant function in all its glory:</p>
<!-- HTML generated using hilite.me -->
<div style="background: #ffffff; overflow:auto;width:auto;border:solid gray;border-width:.1em .1em .1em .8em;padding:.2em .6em;">
<pre style="margin: 0; line-height: 125%"><span style="color: #008800; font-weight: bold">func</span> canDeleteFastInterleaved(table TableDescriptor, fkTables sqlbase.TableLookupsByID) <span style="color: #333399; font-weight: bold">bool</span> {
    <span style="color: #888888">// If there are no interleaved tables then don&#39;t take the fast path.</span>
    <span style="color: #888888">// This avoids superfluous use of DelRange in cases where there isn&#39;t as much of a performance boost.</span>
    hasInterleaved <span style="color: #333333">:=</span> <span style="color: #008800; font-weight: bold">false</span>
    <span style="color: #008800; font-weight: bold">for</span> _, idx <span style="color: #333333">:=</span> <span style="color: #008800; font-weight: bold">range</span> table.AllNonDropIndexes() {
        <span style="color: #008800; font-weight: bold">if</span> <span style="color: #007020">len</span>(idx.InterleavedBy) &gt; <span style="color: #0000DD; font-weight: bold">0</span> {
            hasInterleaved = <span style="color: #008800; font-weight: bold">true</span>
            <span style="color: #008800; font-weight: bold">break</span>
        }
    }
    <span style="color: #008800; font-weight: bold">if</span> !hasInterleaved {
        <span style="color: #008800; font-weight: bold">return</span> <span style="color: #008800; font-weight: bold">false</span>
    }
    <span style="color: #888888">// if the base table is interleaved in another table, fail</span>
    <span style="color: #008800; font-weight: bold">for</span> _, idx <span style="color: #333333">:=</span> <span style="color: #008800; font-weight: bold">range</span> fkTables[table.ID].Table.AllNonDropIndexes() {
        <span style="color: #008800; font-weight: bold">if</span> <span style="color: #007020">len</span>(idx.Interleave.Ancestors) &gt; <span style="color: #0000DD; font-weight: bold">0</span> {
            <span style="color: #008800; font-weight: bold">return</span> <span style="color: #008800; font-weight: bold">false</span>
        }
    }
    interleavedQueue <span style="color: #333333">:=</span> []sqlbase.ID{table.ID}
    <span style="color: #008800; font-weight: bold">for</span> <span style="color: #007020">len</span>(interleavedQueue) &gt; <span style="color: #0000DD; font-weight: bold">0</span> {
        tableID <span style="color: #333333">:=</span> interleavedQueue[<span style="color: #0000DD; font-weight: bold">0</span>]
        interleavedQueue = interleavedQueue[<span style="color: #0000DD; font-weight: bold">1</span>:]
        <span style="color: #008800; font-weight: bold">if</span> _, ok <span style="color: #333333">:=</span> fkTables[tableID]; !ok {
            <span style="color: #008800; font-weight: bold">return</span> <span style="color: #008800; font-weight: bold">false</span>
        }
        <span style="color: #008800; font-weight: bold">if</span> fkTables[tableID].Table <span style="color: #333333">==</span> <span style="color: #008800; font-weight: bold">nil</span> {
            <span style="color: #008800; font-weight: bold">return</span> <span style="color: #008800; font-weight: bold">false</span>
        }
        <span style="color: #008800; font-weight: bold">for</span> _, idx <span style="color: #333333">:=</span> <span style="color: #008800; font-weight: bold">range</span> fkTables[tableID].Table.AllNonDropIndexes() {
            <span style="color: #888888">// Don&#39;t allow any secondary indexes</span>
            <span style="color: #888888">// TODO(emmanuel): identify the cases where secondary indexes can still work with the fast path and allow them</span>
            <span style="color: #008800; font-weight: bold">if</span> idx.ID <span style="color: #333333">!=</span> fkTables[tableID].Table.PrimaryIndex.ID {
                <span style="color: #008800; font-weight: bold">return</span> <span style="color: #008800; font-weight: bold">false</span>
            }
            <span style="color: #888888">// interleavedIdxs will contain all of the table and index IDs of the indexes interleaved in this one</span>
            interleavedIdxs <span style="color: #333333">:=</span> <span style="color: #007020">make</span>(<span style="color: #008800; font-weight: bold">map</span>[sqlbase.ID]<span style="color: #008800; font-weight: bold">map</span>[sqlbase.IndexID]<span style="color: #008800; font-weight: bold">struct</span>{})
            <span style="color: #008800; font-weight: bold">for</span> _, ref <span style="color: #333333">:=</span> <span style="color: #008800; font-weight: bold">range</span> idx.InterleavedBy {
                <span style="color: #008800; font-weight: bold">if</span> _, ok <span style="color: #333333">:=</span> interleavedIdxs[ref.Table]; !ok {
                    interleavedIdxs[ref.Table] = <span style="color: #007020">make</span>(<span style="color: #008800; font-weight: bold">map</span>[sqlbase.IndexID]<span style="color: #008800; font-weight: bold">struct</span>{})
                }
                interleavedIdxs[ref.Table][ref.Index] = <span style="color: #008800; font-weight: bold">struct</span>{}{}
            }
            <span style="color: #888888">// The index can&#39;t be referenced by anything that&#39;s not the interleaved relationship</span>
            <span style="color: #008800; font-weight: bold">for</span> _, ref <span style="color: #333333">:=</span> <span style="color: #008800; font-weight: bold">range</span> idx.ReferencedBy {
                <span style="color: #008800; font-weight: bold">if</span> _, ok <span style="color: #333333">:=</span> interleavedIdxs[ref.Table]; !ok {
                    <span style="color: #008800; font-weight: bold">return</span> <span style="color: #008800; font-weight: bold">false</span>
                }
                <span style="color: #008800; font-weight: bold">if</span> _, ok <span style="color: #333333">:=</span> interleavedIdxs[ref.Table][ref.Index]; !ok {
                    <span style="color: #008800; font-weight: bold">return</span> <span style="color: #008800; font-weight: bold">false</span>
                }
                idx, err <span style="color: #333333">:=</span> fkTables[ref.Table].Table.FindIndexByID(ref.Index)
                <span style="color: #008800; font-weight: bold">if</span> err <span style="color: #333333">!=</span> <span style="color: #008800; font-weight: bold">nil</span> {
                    <span style="color: #008800; font-weight: bold">return</span> <span style="color: #008800; font-weight: bold">false</span>
                }
                <span style="color: #888888">// All of these references MUST be ON DELETE CASCADE</span>
                <span style="color: #008800; font-weight: bold">if</span> idx.ForeignKey.OnDelete <span style="color: #333333">!=</span> sqlbase.ForeignKeyReference_CASCADE {
                    <span style="color: #008800; font-weight: bold">return</span> <span style="color: #008800; font-weight: bold">false</span>
                }
            }
            <span style="color: #008800; font-weight: bold">for</span> _, ref <span style="color: #333333">:=</span> <span style="color: #008800; font-weight: bold">range</span> idx.InterleavedBy {
                interleavedQueue = <span style="color: #007020">append</span>(interleavedQueue, ref.Table)
            }
        }
    }
    <span style="color: #008800; font-weight: bold">return</span> <span style="color: #008800; font-weight: bold">true</span>
}
</pre>
</div>
<p>Here's a higher-level description of how the check runs:</p>
<ul>
<li>Start by examining the description of the parent table.
<ul>
<li>If it has no tables interleaved in it, fail the check</li>
</ul></li>
<li>Push the table ID of the table into a queue.</li>
<li>While the queue is nonempty:
<ul>
<li>Pop the table ID at the top of the queue.</li>
<li>If it has any secondary indices, fail the check.</li>
<li>find the tables that interleaved in our current table
<ul>
<li>If any other tables reference the current table other than the interleaved ones, fail the check.</li>
<li>If the <code>ON DELETE</code> clause of the foreign key in the interleaved table is not <code>CASCADE</code>, fail the check.</li>
</ul></li>
<li>push the interleaved tables' IDs in the queue.</li>
</ul></li>
<li>Pass the check if it went through every table in the queue without finding any failing conditions.</li>
</ul>
<p>If the check passes, then instead of going through the foreign key-checking workflow, Cockroach will then issue a single KV delete range operation on the range of keys it needs to delete; it gets this range by doing a scan through of the keys and finding the ranges that satisfy the <code>WHERE</code> clause specified in the <code>DELETE</code> command.</p>
<p>Ordinarily, the scan request will return a bunch of key spans that will end at the key of the parent row that needs to be deleted, in the form of a pair of keys indicating the start and end. For the interleaved fast path we then need to just change the end key to the last key that has the end key as a prefix. Finding the range from a key to its &quot;prefix end&quot; seems like a pretty big dependency but a <code>PrefixEnd</code> function already existed Cockroach's KV API and I was able to use it precisely for this purpose.</p>
<p>That's all there was to it! Now, instead of going through foreign key checks every row, the database now just goes through all of the table schemas before starting to delete anything. Let's see how the performance improved.</p>
<h2 id="show-me-the-numbers">Show me the numbers</h2>
<p>To test out the performance of the fast path, a number of benchmarks were written in deleting large numbers of rows from interleaved tables in the following cases:</p>
<ul>
<li>Single interleaved table (parent &lt;- child)</li>
<li>Two interleaved tables (parent &lt;- 2 children)</li>
<li>&quot;Grandchild&quot; relationship (parent &lt;- child &lt;- grandchild)</li>
</ul>
<p>The benchmark just tests the performance of a delete after inserting 1000 rows into the parent table and one child row per parent row for each child table. Go benchmarks measure performance by running the operation many times and taking how much time on average a single operation takes. Here are the numbers:</p>
<table>
<thead>
<tr class="header">
<th>Test</th>
<th align="center">Non-fast path time</th>
<th align="center">Fast path time</th>
<th align="center">Speedup (slow time / fast time)</th>
</tr>
</thead>
<tbody>
<tr class="odd">
<td>Child</td>
<td align="center">50800162 ns/op</td>
<td align="center">0.08 ns/op</td>
<td align="center">635002025</td>
</tr>
<tr class="even">
<td>Siblings</td>
<td align="center">1557171104 ns/op</td>
<td align="center">0.23 ns/op</td>
<td align="center">6770309147</td>
</tr>
<tr class="odd">
<td>Child and grandchild</td>
<td align="center">2397660316 ns/op</td>
<td align="center">0.22 ns/op</td>
<td align="center">10898455981.8</td>
</tr>
</tbody>
</table>
<p>Disclaimer: the numbers here are somewhat idealized, because the benchmarks here were run on a single node, which is definitely not the situation that Cockroach will normally run in.</p>
<p>However, we still see an performance boost of a factor of 10 billion for the most complex case between the non-fast path and the fast path. This was absolutely huge! We went from second-scale to sub-nanosecond scale in one fell swoop.</p>
<p>Other than the huge performance boost, it's also interesting that while the fast path experiences a predictable slow-down from the single-child case to the cases with more than one interleaved table, the performance for the sibling and grandchild cases perform about the same. This is not true of the non-fast path deletes, which experience a slow down from the sibling to the grandchild case.</p>
<p>This is most likely because the fast path's bottleneck in performance is scanning through all of the keys, which would scale just with the number of keys that need to be deleted in total, while the non-fast path has a bottleneck of checking all of the foreign key relationships of every row being deleted. So not only is there a constant-factor performance boost, there's an algorithmic complexity boost as well.</p>
<h2 id="future-directions">Future directions</h2>
<p>As this is a fast path, the natural next thing to do is to find more cases where this can work, so that we can see similar performance gains for more types of queries. Revisiting the criteria for the fast-path check, we identified that the criteria that could be relaxed first would be the condition that none of the tables could have secondary indices.</p>
<p>The problem with secondary indices is that if an interleaved table has a secondary index, this creates another key that's outside of the prefix range of the parent table. For example, if table with ID <code>73</code> is interleaved in table <code>54</code>, then the child table's primary key would be something like <code>Table/54/1/1/#/Table/73/1/1/0</code>, and its secondary index would be something like <code>Table/73/2/3/0</code>, which isn't in the range. Using some extra scans, it's probably possible to make it so that the interleaved delete will cascade into these secondary index key ranges as well.</p>
<p>On a broader note, other fast paths can be discovered to optimize other types of queries to improve performance in other cases.</p>
<p>If you enjoyed this piece and are excited about this kind of work, I definitely recommend Cockroach Labs as an internship (or full-time) destination! Here's their <a href="https://www.cockroachlabs.com/careers/#jobs">open positions page</a>.</p>

