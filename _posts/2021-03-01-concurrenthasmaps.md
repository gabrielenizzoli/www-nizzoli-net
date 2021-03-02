---
title: "ConcurrentHashMap > Collections.synchronizedMap"
categories:
  - Code
tags:
  - spark
  - java
  - concurrency
---

In Java there are generally 2 options for a thread-safe Map:

* wrap it using a `Collections.synchronizedMap`, or
* use a `ConcurrentHashMap`.

The `Collections.synchronizedMap` type is a wrapper around a map with an identical interface, and simply has a global lock guarding every operation (read or write).
Generally speaking, this is not needed, and in the majority of the cases it is actually detrimental.

An example of this is in a configuration class in Apache Spark, where a simple map (that mostly holds static data) is frequently accessed to verify the existence of a flag.
This map is currently implemented as:

```scala
  private[sql] val sqlConfEntries = java.util.Collections.synchronizedMap(
    new java.util.HashMap[String, ConfigEntry[_]]())
```

Since it is a `Collections.synchronizedMap`, it has a global lock, so every read or write access requires waiting on the lock: this imposes a untenable delay on the whole system when the concurrency on the map is high.
On an executor with, let say, 40 cores, this is a problem.
[The solution to fix this is fairly trivial](https://issues.apache.org/jira/browse/SPARK-34573):

```scala
  private[sql] val sqlConfEntries =
    new ConcurrentHashMap[String, ConfigEntry[_]]()
```

Every read operation will be lock free. while very write will only lock a local portion of the map.
Small changes can go a long way to make the code better :)
