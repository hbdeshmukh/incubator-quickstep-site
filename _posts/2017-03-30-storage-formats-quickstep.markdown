---
layout: post
title:  "Storage Formats in Quickstep"
date: 2017-03-30 14:00:30 -0600
categories: guides
author: Harshad
short: Quickstep offers variety of storage formats. This post explains what these formats are and how to use them.
---
One of the strengths of Quickstep is the variety of storage formats it offers to store the relational data. The storage management in Quickstep can work with all these formats, and each format comes with its own strengths and weaknesses. The foundation of this work was laid in a [research paper](http://www.vldb.org/pvldb/vol6/p1474-chasseur.pdf) from the Patel Wisconsin Database group that appeared in [VLDB 2014](http://vldb.org/) (International Conference on Very Large Databases). In this post, I will provide a brief primer on what these storage formats mean, their implication on performance, and finally how to use them in the Quickstep system. I will also provide links to the relevant source code files, so that the readers can start exploring the code base. 

<h1>Storage Formats 101</h1>
A storage format refers to how the data in a table is laid out in the memory. Let's consider a toy relational schema.

{% highlight sql %}
CREATE TABLE employee (ID INT, Age INT, Name VARCHAR(8));
{% endhighlight %}

A tuple in the employee table consists of three attributes, id (an integer), age (an integer) and the name (character of 8 bytes fixed length). Let's also populate our table with few tuples. For simplicity, let us assume that the name is a fixed length attribute; however, in general it could be variable length. 

{% highlight sql %}
INSERT into employee values (1, 30, Jennifer);
INSERT into employee values (2, 25, Jim);
INSERT into employee values (3, 35, David);
{% endhighlight %}

Conceptually, our table looks as follows:

| ID 	| Age 	|   Name   	|
|:--:	|:---:	|:--------:	|
|  100 	|  30 	| Jennifer 	|
|  101 	|  25 	|    Jim   	|
|  102 	|  35 	|   David  	|

Next, let us dig into understanding the different ways in which this table can be stored in memory. 

<h2>Row Store</h2>

The first storage format we will look at is *row store*, which is a popular format, and one that is intuitive too.

The figure below represents how a row store format for our toy example looks like in memory. The offsets indicate how far is the memory location from the beginning of the block. 

![Row Store](../assets/storage-formats-row-store.jpg)

In the row store format, a tuple is stored in the table definition order. In memory, the ID (101) of the first tuple is stored at some address, and it is followed by the age field for that tuple (30), and then the name field ("Jennifer"). The second tuple is stored in memory sequentially after the first tuple, and so on.

<h2>Column Store</h2>

Let's now take a look at the *column store* format. The figure below depicts a column store layout for our toy example. 

![Column Store](../assets/storage-formats-column-store.jpg)

In the column store format, the values from the same column are stored together. The order in which the values from a column are stored remains the same for all the columns. Notice the ID values 101, 102, 1033 are stored contiguously. The corresponding Age fields 30, 25 and 35 are stored contiguously and likewise for the Name column.

<h2>Compression</h2>

Compression is a standard technique to reduce the storage footprint. There is a large body of work on various compression techniques, which we won't cover here. Let's look at how compression can be applied in our toy example. If we look at the ID column, it has three values 101, 102 and 103. If they are stored as regular integers, each of them will occupy 4 bytes of memory. Interestingly, we can reduce the memory consumed in storing these three values. Observe that these three values are very close to 100. If we remember the base value 100 and only record the difference of each value from this base value (of 100), then we only need to store the values 1, 2, and 3. To store these three *deltas*, we don't even need a 4 byte integer, a 1-byte value is enough.

Putting it all together, the picture below shows the compressed column store format in which the compression is applied on the ID column. 

![Column Store with Compression](../assets/storage-formats-compressed-column-store.jpg)

Observe that the memory occupied by the three tuples together is only 43 bytes, compared to 48 bytes in the row store and the column store formats. As the number of tuples increase, there may be more opportunities for compression and thus more memory savings.

There are some aspects related to storage management which deserve a blog post of their own. This includes the block-based storage design, and the impact of the above storage formats on various analytical queries. I hope to write more blog posts to cover these topics in the future.

<h1>Creating tables with various storage formats in Quickstep</h1>

Now I will show you how to use the above storage formats to create tables in Quickstep. The storage format specification is part of the `CREATE TABLE SQL` command. We will continue with our toy example above to illustrate the various ways in which we can use the storage formats in Quickstep. 

{% highlight sql %}
CREATE TABLE employee (
ID INT NOT NULL, 
Age INT NOT NULL, 
Name VARCHAR(8) NOT NULL
) WITH BLOCKPROPERTIES (
  TYPE rowstore,
  BLOCKSIZEMB 4);
{% endhighlight %}

I will now describe some keywords used in the above SQL statement. The keyword `BLOCKPROPERTIES` refers to the storage properties of this table. The above command means that all the blocks in the employee table will use the row store format, with each block sized to a maximum of 4 MB.

{% highlight sql %}
CREATE TABLE employee (
ID INT NOT NULL, 
Age INT NOT NULL, 
Name VARCHAR(8) NOT NULL
) WITH BLOCKPROPERTIES (
  TYPE compressed_columnstore,
  SORT ID,
  COMPRESS ID,
  BLOCKSIZEMB 4);
{% endhighlight %}

In this case, all the blocks in the employee table have compressed column store as their storage format. The values in each block are sorted by the `ID` values, and the compression is applied on the `ID` column (just like we described earlier).

The `COMPRESS` keyword accepts one or more or `ALL` the columns from the table. Thus it can look like:

{% highlight sql %}
CREATE TABLE employee (
ID INT NOT NULL, 
Age INT NOT NULL, 
Name VARCHAR(8) NOT NULL
) WITH BLOCKPROPERTIES (
  TYPE compressed_columnstore,
  SORT ID,
  COMPRESS (ID, Age),
  BLOCKSIZEMB 4);
{% endhighlight %}

The above command indicates that compression will be applied on the `ID` and the `Age` columns only.

Finally, the command below indicates that the compression is applied on all the columns.

{% highlight sql %}
CREATE TABLE employee (
ID INT NOT NULL, 
Age INT NOT NULL, 
Name VARCHAR(8) NOT NULL
) WITH BLOCKPROPERTIES (
  TYPE compressed_columnstore,
  SORT ID,
  COMPRESS ALL,
  BLOCKSIZEMB 4);
{% endhighlight %}

<h1>Code Walkthrough</h1>

For folks interested in looking at the source code for these storage formats, I  provide the links to relevant source code files below. Our code is well documented (doxygen) for the most part, so it should be easier to read.

[Parsing the block properties](https://github.com/apache/incubator-quickstep/blob/master/parser/ParseBlockProperties.hpp)

[Row store implementation](https://github.com/apache/incubator-quickstep/blob/master/storage/SplitRowStoreTupleStorageSubBlock.hpp)

[Basic column store implementation](https://github.com/apache/incubator-quickstep/blob/master/storage/BasicColumnStoreTupleStorageSubBlock.hpp)

[Compressed column store implementation](https://github.com/apache/incubator-quickstep/blob/master/storage/CompressedColumnStoreTupleStorageSubBlock.hpp)

I hope this blog post was useful and gives you some idea about the various storage formats implemented in Quickstep. If you have questions, please shoot us an email on dev@quickstep.incubator.apache.org.
