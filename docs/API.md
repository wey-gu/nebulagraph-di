# API Reference

## Table of contents

- [NebulaGraphConfig](#NebulaGraphConfig)
- [NebulaReader](#NebulaReader)
- [engines](#engines)
- [NebulaDataFrameObject](#NebulaDataFrameObject)
- [NebulaGraphObject](#NebulaGraphObject)
- [NebulaAlgorithm](#NebulaAlgorithm)
- [NebulaWriter](#NebulaWriter)
- [NebulaGNN](#NebulaGNN)

## NebulaGraphConfig

`ngdi.NebulaGraphConfig` is the configuration for `ngdi.NebulaReader`, `ngdi.NebulaWriter` and `ngdi.NebulaAlgorithm`.

See code for details: [ngdi/config.py](../ngdi/config.py)

## NebulaReader

`ngdi.NebulaReader` reads data from NebulaGraph and constructs `NebulaDataFrameObject` or `NebulaGraphObject`.
It supports different engines, including Spark and NebulaGraph. The default engine is Spark.

Per each engine, `ngdi.NebulaReader` supports different read modes, including query, scan, and load.
For now, Spark Engine supports query, scan and load modes, while NebulaGraph Engine supports query and scan modes.

In Spark Engine:
- All modes are underlyingly implemented by NebulaGraph Spark Connector nGQL Reader, while in the future, query mode will be optionally done by opencypher/morpheus to bypass the Nebula-GraphD.
- The `NebulaDataFrameObject` returned by `ngdi.NebulaReader.read()` is a Spark DataFrame, which can be further processed by Spark SQL or Spark MLlib.

In NebulaGraph Engine:
- Query mode is implemented by Python Nebula-GraphD Client, while scan mode the Nebula-StoreageD Client.
- The `NebulaDataFrameObject` returned by `ngdi.NebulaReader.read()` is a Pandas DataFrame, which can be further processed by Pandas. And the graph returned by ngdi.

### Functions

- `ngdi.NebulaReader.query()` sets the query statement.
- `ngdi.NebulaReader.scan()` sets the scan statement.
- `ngdi.NebulaReader.load()` sets the load statement. (not yet implemented)
- `ngdi.NebulaReader.read()` executes the read operation and returns a DataFrame or `NebulaGraphObject`.
- `ngdi.NebulaReader.show()` shows the DataFrame returned by `ngdi.NebulaReader.read()`.

### Examples

#### Spark Engine

- Scan mode

```python
from ngdi import NebulaReader
# read data with spark engine, scan mode
reader = NebulaReader(engine="spark")
reader.scan(edge="follow", props="degree")
df = reader.read()
```

- Query mode

```python
from ngdi import NebulaReader
# read data with spark engine, query mode
reader = NebulaReader(engine="spark")
query = """
    MATCH ()-[e:follow]->()
    RETURN e LIMIT 100000
"""
reader.query(query=query, edge="follow", props="degree")
df = reader.read()
```

- Load mode

> not yet implemented

```python
# read data with spark engine, load mode (not yet implemented)
reader = NebulaReader(engine="spark")
reader.load(source="hdfs://path/to/edge.csv", format="csv", header=True, schema="src: string, dst: string, rank: int")
df = reader.read() # this will take some time
df.show(10)
```

## engines

- `ngdi.engines.SparkEngine` is the Spark Engine for `ngdi.NebulaReader`, `ngdi.NebulaWriter` and `ngdi.NebulaAlgorithm`.

- `ngdi.engines.NebulaEngine` is the NebulaGraph Engine for `ngdi.NebulaReader`, `ngdi.NebulaWriter`.

- `ngdi.engines.NetworkXEngine` is the NetworkX Engine for `ngdi.NebulaAlgorithm`.

## `NebulaDataFrameObject`

ngdi.`NebulaDataFrameObject` is a Spark DataFrame or Pandas DataFrame, which can be further processed by Spark SQL or Spark MLlib or Pandas.

### Functions

- `ngdi.NebulaDataFrameObject.algo.pagerank()` runs the PageRank algorithm on the Spark DataFrame.
- `ngdi.NebulaDataFrameObject.to_graphx()` converts the DataFrame to a GraphX Graph. not yet implemented.

## NebulaGraphObject

`ngdi.NebulaGraphObject` is a GraphX Graph or NetworkX Graph, which can be further processed by GraphX or NetworkX.

### Functions

- `ngdi.NebulaGraphObject.algo.get_all_algo()` returns all algorithms supported by the engine.
- `ngdi.NebulaGraphObject.algo.pagerank()` runs the PageRank algorithm on the NetworkX Graph. not yet implemented.

## NebulaAlgorithm

`ngdi.NebulaAlgorithm` is a collection of algorithms that can be run on ngdi.`NebulaDataFrameObject`(spark engine) or `ngdi.NebulaGraphObject`(networkx engine).

## NebulaWriter

`ngdi.NebulaWriter` writes the computed or queried data to different sinks.
Supported sinks include:
- NebulaGraph(Spark Engine, NebulaGraph Engine)
- CSV(Spark Engine, NebulaGraph Engine)
- S3(Spark Engine, NebulaGraph Engine), not yet implemented.

### Functions

- `ngdi.NebulaWriter.options()` sets the options for the sink.
- `ngdi.NebulaWriter.write()` writes the data to the sink.
- `ngdi.NebulaWriter.show_options()` shows the options for the sink.

### Examples

#### Spark Engine

- NebulaGraph sink

Assume that we have a Spark DataFrame `df_result` computed with `df.algo.louvain()` with the following schema:

```python
df_result.printSchema()
# result:
root
 |-- _id: string (nullable = false)
 |-- louvain: string (nullable = false)
```

We created a TAG `louvain` in NebulaGraph on same space with the following schema:

```ngql
CREATE TAG IF NOT EXISTS louvain (
    cluster_id string NOT NULL
);
```

Then, we could write the louvain result to NebulaGraph, map the column `louvain` to `cluster_id` with the following code:

```python
from ngdi import NebulaWriter
from ngdi.config import NebulaGraphConfig

config = NebulaGraphConfig()

properties = {
    "louvain": "cluster_id"
}

writer = NebulaWriter(data=df_result, sink="nebulagraph_vertex", config=config, engine="spark")
writer.set_options(
    tag="louvain",
    vid_field="_id",
    properties=properties,
    batch_size=256,
    write_mode="insert",
)
writer.write()
```

Then we could query the result in NebulaGraph:

```cypher
MATCH (v:louvain)
RETURN id(v), v.louvain.cluster_id LIMIT 10;
```


## NebulaGNN

`ngdi.NebulaGNN` is a collection of graph neural network models that can be run on ngdi. not yet implemented.
