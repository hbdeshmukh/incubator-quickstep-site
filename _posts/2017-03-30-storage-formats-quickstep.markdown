---
layout: post
title:  "Storage Formats in Quickstep"
date: 2017-03-30 14:00:30 -0600
categories: guides
author: Harshad
short: Quickstep offers variety of storage formats. This post explains what these formats are and how to use them.
---
One of the strengths of Quickstep is the variety of storage formats it offers to store the relational data. The storage management in Quickstep can work with all these formats, and each format comes with its own strengths and weaknesses. The foundation of this work was laid in a [research paper](http://www.vldb.org/pvldb/vol6/p1474-chasseur.pdf) from the Patel's Wisconsin Database group that appeared in [VLDB 2014](http://vldb.org/) (Very Large Databases), which is a premier database conference. In this post, I will provide a brief primer on what these storage formats mean, their implication on performance and finally how to use them while using Quickstep system. For the curious ones, I will also provide links to the relevant source code files, so that the readers can start exploring the code base. 

<h2>Storage Formats 101</h2>
A storage format refers to how the data in a table is laid out in the memory. Let's consider a toy relational schema.

{% highlight sql %}
CREATE TABLE employee (id INT, age INT, name VARCHAR(8));
{% endhighlight %}

A tuple in the employee table consists of three attributes, id (an integer), age (an integer) and the name (character of 8 bytes fixed length). Let's also populate our table with few tuples. We assume the name to be fixed length for simplicity, however in general it could be variable length. 

{% highlight sql %}
INSERT into employee values (1, 30, Jennifer);
INSERT into employee values (2, 25, Jim);
INSERT into employee values (3, 35, David);
{% endhighlight %}

Conceptually, our table looks as below:

| ID 	| Age 	|   Name   	|
|:--:	|:---:	|:--------:	|
|  1 	|  30 	| Jennifer 	|
|  2 	|  25 	|    Jim   	|
|  3 	|  35 	|   David  	|

Now we will dive into understanding the different ways in which this table would be stored in memory. The first storage format we will look at is row store, which is one of the more popular formats and intuitive too.

The figure below represents how a row store format for our toy example would look like in the memory. 

In the row store format, a tuple is stored as it looks in the above table. In the memory, the ID (1) of the first tuple is storead at some address, and it is followed by the age field for that tuple (30), and then the name field (Jennifer). In the memory location right after the first tuple, the second tuple is stored, and so on.
