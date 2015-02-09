Description / Rationale
===================

This is a new CassandraState implementation built on the CQL java driver.  For Cassandra, CQL has better support for lightweight transacations, batching, and collections.  
Also, CQL will likely get more attention than the legacy Thrift interface.  For these reasons, we decided to create a C* state implementation built on CQL.

Design
===================
An application/topology provides implementations of the mapper interfaces. 
For example, the [CqlRowMapper](https://github.com/hmsonline/storm-cassandra-cql/blob/master/src/main/java/com/hmsonline/trident/cql/mappers/CqlRowMapper.java) provides a bidirectional mapping from Keys and Values to statements that can be used to upsert and retrieve data.

Getting Started
===================
You can use the examples to get started.  For the example, you'll want to run a local cassandra instance with the example schema found in 
[schema.cql](https://github.com/hmsonline/storm-cassandra-cql/blob/master/src/test/resources/schema.cql).

You can do this using cqlsh:

```
cat storm-cassandra-cql/src/test/resources/create_keyspace.cql | cqlsh
cat storm-cassandra-cql/src/test/resources/schema.cql | cqlsh
```


## SimpleUpdateTopology

The SimpleUpdateTopology simply emits integers (0-99) and writes those to Cassandra with the current timestamp % 10.  The values are written to the table: mytable.

This is persistence from the topology:
```java
   inputStream.partitionPersist(new CassandraCqlStateFactory(ConsistencyLevel.ONE), new Fields("test"), new CassandraCqlStateUpdater(mapper));
```

During a partition persist, Storm repeatedly calls `updateState()` on the CassandraCqlStateUpdater to update a state object for the batch.  The updater uses the mapper to convert the tuple into a CQL statement, and caches the CQL statement in a CassandraCqlState object.  When the batch is complete, Storm calls commit on the state object, which then executes all of the CQL statements as a batch.

See: 
* [CassandraCqlStateUpdater.updateState](https://github.com/hmsonline/storm-cassandra-cql/blob/master/src/main/java/com/hmsonline/trident/cql/CassandraCqlStateUpdater.java#L37-L41)
* [CassandraCqlState.commit](https://github.com/hmsonline/storm-cassandra-cql/blob/master/src/main/java/com/hmsonline/trident/cql/CassandraCqlState.java#L39-L56)

The SimpleUpdateMapper looksl like this:

```java
public class SimpleUpdateMapper implements CqlRowMapper<Object, Object>, Serializable {
    public Statement map(TridentTuple tuple) {
        long t = System.currentTimeMillis() % 10;
        Update statement = update("mykeyspace", "mytable");
        statement.with(set("col1", tuple.getString(0))).where(eq("t", t));
        return statement;
    }
```

As you can see, it maps tuples to update statements.

When you run the `main()` method in the SimpleUpdateTopology, you should get results in mytable that look like this:
 
```
 t | col1
---+------
 5 |   97
 1 |   99
 8 |   99
 0 |   99
 2 |   99
 4 |   99
 7 |   99
 6 |   99
 9 |   99
 3 |   99
```


## WordCountTopology
The WordCountTopology is slightly more complex in that it uses the CassandraCqlMapState.  The map state assumes you are reading/writing keys and values.  
The topology emits words from two different sources, and then totals the words by source and persists the count to Cassandra.

The CassandraCqlMapState object implements the IBackingMap interface in Storm.
See:
[Blog on the use of IBackingMap](https://svendvanderveken.wordpress.com/2013/07/30/scalable-real-time-state-update-with-storm/)

Have a look at the [WordCountTopology](https://github.com/hmsonline/storm-cassandra-cql/blob/master/src/test/java/com/hmsonline/trident/cql/example/wordcount/WordCountTopology.java).

It uses a FixedBatchSpout that emits sentences over and over again:

```java
   FixedBatchSpout spout1 = new FixedBatchSpout(new Fields("sentence"), 3,
      new Values("the cow jumped over the moon"),
      new Values("the man went to the store and bought some candy"),
      new Values("four score and seven years ago"),
      new Values("how many apples can you eat"));
   spout1.setCycle(true);
```

Then, it splits and groups those words:

```java
TridentState wordCounts =
                topology.newStream("spout1", spout1)
                        .each(new Fields("sentence"), new Split(), new Fields("word"))
                        .groupBy(new Fields("word"))
                        .persistentAggregate(CassandraCqlMapState.nonTransactional(new WordCountMapper()),
                                new IntegerCount(), new Fields("count"))
                        .parallelismHint(6);
```

Instead of a partitionPersist like the other topology, this topology aggregates first, using the persistentAggregate method.  This performs an aggregation, storing the results in the CassandraCqlMapState, on which Storm eventually calls multiPut/multiGet to store/read values.

See:
[CassandraCqlMapState.multiPut/Get](https://github.com/hmsonline/storm-cassandra-cql/blob/master/src/main/java/com/hmsonline/trident/cql/CassandraCqlMapState.java#L122-L187)

In this case, the mapper maps keys and values to CQL statements:

```java
    @Override
    public Statement map(List<String> keys, Number value) {
        Insert statement = QueryBuilder.insertInto(KEYSPACE_NAME, TABLE_NAME);
        statement.value(WORD_KEY_NAME, keys.get(0));
        statement.value(SOURCE_KEY_NAME, keys.get(1));
        statement.value(VALUE_NAME, value);
        return statement;
    }

    @Override
    public Statement retrieve(List<String> keys) {
        // Retrieve all the columns associated with the keys
        Select statement = QueryBuilder.select().column(SOURCE_KEY_NAME)
                .column(WORD_KEY_NAME).column(VALUE_NAME)
                .from(KEYSPACE_NAME, TABLE_NAME);
        statement.where(QueryBuilder.eq(SOURCE_KEY_NAME, keys.get(0)));
        statement.where(QueryBuilder.eq(WORD_KEY_NAME, keys.get(1)));
        return statement;
    }
```

When you run this example, you will find the following counts in Cassandra:

cqlsh> select * from mykeyspace.wordcounttable;

 source | word   | count
--------+--------+-------
 spout2 |      a |    73
 spout2 |    ago |    74
 spout2 |    and |    74
 spout2 | apples |    69
...
 spout1 |   some |    74
 spout1 |  store |    74
 spout1 |    the |   296
 spout1 |     to |    74
 spout1 |   went |    74
```




