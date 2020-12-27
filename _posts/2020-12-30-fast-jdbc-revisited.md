---
layout: post
title: Fast JDBC access in Python using pyarrow.jvm (2020 edition)
feature_image: "/images/gerson-cifuentes-W1VqcpcnSHk-unsplash.jpg"
---

About a year ago, [I have benchmarked access databases through JDBC in Python](https://uwekorn.com/2019/11/17/fast-jdbc-access-in-python-using-pyarrow-jvm.html).
Recently, the maintainer of [`jpype`](https://pypi.org/project/JPype1/) gave [me a heads-up that they significantly improved performance on their side](https://github.com/Thrameos/jpype/issues/52).
While this is actually the library I'm comparing my `pyarrow.jvm`-based approach to, I have a high appreciation for any performance tuning that is done.
Thus I'm happily trying to recreate the setup I used a year ago to see how performance changed.

It needs to be noted here that while the performance improvements in `jpype` mean that [JayDeBeApi](https://github.com/baztian/jaydebeapi) should get a lot better performance, it should improve the performance of `pyarrow.jvm` a bit.
While the main payload data is just handed over by the passing the memory pointer from Java to Python, we use `jpype` to access the metadata about the stored data in Java.
Thus this metadata lookup will also improve a bit.

## Benchmark setup with updated versions

Overall, we have done the following version updates:

 * Apache Drill: `1.16 -> 1.18`
 * Apache Arrow: `0.15.1 -> 2.0.0`
 * `jpype1`: `0.6.3 -> 1.2.0`
 * `JayDeBeApi`: `1.1.1 -> 1.2.3`

We had to do some changes to the code as on the Arrow side a new SQL API was implemented that also support iterating over chunks of the result and reusing previously allocated buffers.
Also thanks to [some tips from the `jpype` maintainer](https://github.com/Thrameos/jpype/issues/52#issuecomment-741509830), we switched the access of Java classes to using the `jpype.imports` "magic" as this gave a much better performance.
Additionally, we have added the `jpype.dbapi2` module as a third possibility to access databases with a JDBC driver from Python.

To have a single JAR that we can use to start JVM as in the initial post, we update the `pom.xml` to the new version of the dependencies and build that JAR using `mvn assembly:single`.

```xml
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.uwekorn</groupId>
    <artifactId>drill-odbc</artifactId>
    <version>0.2-SNAPSHOT</version>
    <packaging>jar</packaging>

    <name>drill-odbc</name>

    <properties>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    </properties>

    <dependencies>
        <dependency>
            <groupId>org.apache.arrow</groupId>
            <artifactId>arrow-jdbc</artifactId>
            <version>2.0.0</version>
        </dependency>
        <dependency>
            <groupId>org.apache.arrow</groupId>
            <artifactId>arrow-memory</artifactId>
            <version>2.0.0</version>
            <type>pom</type>
        </dependency>
        <dependency>
            <groupId>org.apache.drill.exec</groupId>
            <artifactId>drill-jdbc-all</artifactId>
            <version>1.18.0</version>
            <exclusions>
              <exclusion>
                <groupId>org.slf4j</groupId>
                <artifactId>log4j-over-slf4j</artifactId>
              </exclusion>
        </exclusions>
        </dependency>

    </dependencies>
    
     <build>
      <plugins>
        <plugin>
          <artifactId>maven-assembly-plugin</artifactId>
          <configuration>
            <archive>
              <manifest>
                <mainClass>com.uwekorn.Main</mainClass>
              </manifest>
            </archive>
            <descriptorRefs>
              <descriptorRef>jar-with-dependencies</descriptorRef>
            </descriptorRefs>
          </configuration>
        </plugin>
          <plugin>
              <groupId>org.apache.maven.plugins</groupId>
              <artifactId>maven-compiler-plugin</artifactId>
              <version>3.8.0</version>
              <configuration>
                  <source>11</source>
                  <target>11</target>
              </configuration>
          </plugin>
      </plugins>
    </build>
</project>
```

We then start the JVM as before using the following code.

```python
import jpype
import os

classpath = os.path.join(os.getcwd(), "all-jar/target/drill-odbc-0.2-SNAPSHOT-jar-with-dependencies.jar")
jpype.startJVM(jpype.getDefaultJVMPath(), f"-Djava.class.path={classpath}")
```

Before we can connect to the database, we must first start it using `./bin/drill-embedded`.
Then we can connect to it using the client libraries:

```python
import jaydebeapi
import jpype.dbapi2

try:
    # May fail on first attempt
    jdbapi2_conn = jpype.dbapi2.connect("jdbc:drill:drillbit=127.0.0.1")
except:
    jdbapi2_conn = jpype.dbapi2.connect("jdbc:drill:drillbit=127.0.0.1")
jdbapi2_cursor = jdbapi2_conn.cursor()

try:
    # May fail on first attempt
    conn = jaydebeapi.connect('org.apache.drill.jdbc.Driver', 'jdbc:drill:drillbit=127.0.0.1')
except:
    conn = jaydebeapi.connect('org.apache.drill.jdbc.Driver', 'jdbc:drill:drillbit=127.0.0.1')
cursor = conn.cursor()
```

With the newest combination of Drill and `jpype1`, you sadly get the following error when establishing the connection the first.
On every second call, it passes though without any issues.

```
Exception: Java Exception

The above exception was the direct cause of the following exception:

java.lang.NoClassDefFoundError            Traceback (most recent call last)
<ipython-input-5-51b73f9fbbd7> in <module>
----> 1 jdbapi2_conn = jpype.dbapi2.connect("jdbc:drill:drillbit=127.0.0.1")

…/env/lib/python3.8/site-packages/jpype/dbapi2.py in connect(dsn, driver, driver_args, adapters, converters, getters, setters, **kwargs)
    418     # User supplied nothing
    419     elif driver_args is None:
--> 420         connection = DM.getConnection(dsn)
    421 
    422     # Otherwise use the kwargs

java.lang.NoClassDefFoundError: java.lang.NoClassDefFoundError: oadd/org/apache/drill/exec/store/StoragePluginRegistry
```

To later run the benchmarks, we implement a generic DB-API2 `select_to_pandas` function and use `functools.partial` to create `jaydebeapi` and `jpype.dbapi2` variants of it.

```python
def select_to_pandas(cursor, query):
    cursor.execute(query)
    columns = [c[0] for c in cursor.description]
    data = cursor.fetchall()
    df = pd.DataFrame(data, columns=columns)
    
select_jaydebeapi = partial(select_to_pandas, cursor=cursor)
select_jpype_dbapi2 = partial(select_to_pandas, cursor=jdbapi2_cursor)
```

For the Apache Arrow variant, we only instantiate a connection on the Java side and configure the `JdbcToArrowUtils` setup.

```python
import jpype.imports

from java.sql import DriverManager

from org.apache.arrow.adapter.jdbc import JdbcToArrowUtils, JdbcToArrowConfigBuilder
from org.apache.arrow.memory import RootAllocator

connection = DriverManager.getConnection("jdbc:drill:drillbit=127.0.0.1")

ra = RootAllocator(sys.maxsize)
calendar = JdbcToArrowUtils.getUtcCalendar()
config_builder = JdbcToArrowConfigBuilder()
config_builder.setAllocator(ra)
config_builder.setCalendar(calendar)
config_builder.setTargetBatchSize(-1)
pyarrow_jdbc_config = config_builder.build()
```

To retrieve data using Apache Arrow, we instantiate a connection and retrieve a `ResultSet` instance.
We also instantiate an empty `VectorSchemaRoot`, the Java equivalent of a `pyarrow.RecordBatch`.
Using `JdbcToArrowUtils.jdbcToArrowVectors`, this `VectorSchemaRoot` is then filled with the data from the `ResultSet`.
As we have set the `targetBatchSize` in the config above to `-1` (unlimited), the full result is transferred.

Due to the nature of Arrow buffers in Java, we must also take care to explicitly release the memory on the Java side by calling `VectorSchemaRoot.clear()`.
This will reduce the reference count of these buffers by one.
If we are still using one of these buffers after we have converted the data to `pandas`, we have instantiated a `pyarrow.jvm._JvmBufferNanny` instance for each buffer.
This will decrease the reference count on the Java side if the object gets deallocated on the Python side.
At the end, one can ask the root allocator using `ra.getAllocatedMemory()` to check that all allocated memory was free'd again or how much is still referenced if e.g. it was used in the constructed `pandas.DataFrame`.

```python
from org.apache.arrow.vector import VectorSchemaRoot

def select_pyarrow_jvm(query):
    stmt = connection.createStatement()
    result_set = stmt.executeQuery(query)
    
    root = VectorSchemaRoot.create(
        JdbcToArrowUtils.jdbcToArrowSchema(
            result_set.getMetaData(),
            pyarrow_jdbc_config
        ),
        pyarrow_jdbc_config.getAllocator()
    )
    try:
        JdbcToArrowUtils.jdbcToArrowVectors(result_set, root, pyarrow_jdbc_config)
        df = pyarrow.jvm.record_batch(root).to_pandas()
    finally:
        # Ensure that we clear the JVM memory.
        root.clear()
        stmt.close()
```

## Timings

To get an understanding of the performance we run the same benchmarks as last year.
This means, we fetch the top N rows from the `yellow_tripdata_2016-01` file (converted to Parquet) from the New York TLC Taxi trips dataset, i.e. we run the below SQL query for `N = {10000, 100000, 1000000}` and compare to last years values.

```
SELECT * FROM dfs.`/…/data/yellow_tripdata_2016-01.parquet` LIMIT <N>
```

|   LIMIT |                            JayDeBeApi (2019) |                          JayDeBeApi (2020) |                         jpype.dbapi2 (2020) | pyarrow.jvm (2019) | pyarrow.jvm (2020) |
|---------|----------------------------------------------|--------------------------------------------|---------------------------------------------|--------------------|--------------------|
|   10000 |   <nobr><code>7.11 s ± 58.6 ms</code></nobr> | <nobr><code>2.27 s ± 64.8 ms</code></nobr> |   <nobr><code>1.1 s ± 36.3 ms</code></nobr> | <nobr><code>165 ms ± 5.86 ms</code></nobr> | <nobr><code>181 ms ± 5.93 ms</code></nobr> |
|  100000 |   <nobr><code>1min 9s ± 1.07 s</code></nobr> |  <nobr><code>18.7 s ± 182 ms</code></nobr> |    <nobr><code>8.37 s ± 64 ms</code></nobr> | <nobr><code>538 ms ± 29.6 ms</code></nobr> | <nobr><code>502 ms ± 20.7 ms</code></nobr> |
| 1000000 | <nobr><code>11min 31s ± 4.76 s</code></nobr> | <nobr><code>3min 6s ± 848 ms</code></nobr> | <nobr><code>1min 22s ± 473 ms</code></nobr> | <nobr><code>5.05 s ± 596 ms</code></nobr> | <nobr><code>3.68 s ± 34.8 ms</code></nobr> |

<br />
In the above table, one can clearly see that `jpype` made really good progress if used either via `jaydebeapi` or `jpype.dbapi2`.
The overhead if significantly less now with a 10x improvement in the case for `N = 1000000` when going from JayDeBeApi (2019) to jpype.dbapi2 (2020).
Still, it is quite far away from the `pyarrow.jvm` variant that completely avoids creating an temporary Python objects for the payload data before converting it to `pandas`.
In that variant, we see slight speedups compared from last year but they are much smaller than those achieved by the better object conversion methods in `jpype`.

Overall the improvements in Apache Arrow Java still mean that a result with the unlimited query goes down from 50.2.s in the 2019 setup to 44.1 s in the current one.

## Conclusion

With a speedup of upto 10x in the recent `jpype` releases, the overall benefit of using `pyarrow.jvm` vanishes a bit for small to medium results.
There it might be simpler to stick to the more recent `jpype.dbapi2` interface and have one dependency less.
If your results are though bigger than 100000 rows, you should definitely take a look at setting up `pyarrow.jvm` for accessing a database behind a JDBC driver.
As it omits the intermediate conversion from Java to Python objects and instead directly writes into native memory on the Java side that can be used on the C/C++ side to construct a `pandas.DataFrame`, it reduces the memory peak and provides an enormous speedup for very large result sets.
