---
layout: post
title: Fast JDBC access in Python using pyarrow.jvm
feature_image: "/images/gerson-cifuentes-W1VqcpcnSHk-unsplash.jpg"
---

While most databases are accessible via ODBC where we have an efficient way via [turbodbc](https://github.com/blue-yonder/turbodbc) to turn results into a `pandas.DataFrame`, there are nowadays a lot of databases that either only come solely with a JDBC driver or the non-JDBC drivers are not part of free or open-source offering.
To access these databases, you can use [JayDeBeApi](https://github.com/baztian/jaydebeapi) which is using [JPype](https://pypi.org/project/JPype1/) to call the JDBC driver.
JPype starts a JVM inside the Python process and exposes the Java APIs as plain Python objects.
While the convenience of use is really nice, this Java-Python bridge sadly comes at a high serialisation cost.

One of the main goals of Apache Arrow is to remove the serialisation cost of tabular data between different languages.
A typical example where this already is successfully used is the Scala-Python bridge in PySpark.
Here the communication between the JVM and Python is done via [Py4J](http://www.py4j.org/), a bridge between Python and JVM process.
As there are multiple processes involved, the serialisation cost is reduced but communication and data copy between the ecosystems still exists.

In the following, we want to present an alternative approach to retrieve data via JDBC where the overhead between the JVM and `pandas` is kept as minimal as possible.
This includes retrieving the whole data on the JVM side, transforming it to an Arrow Record Batch and then passing the memory pointer to that Record Batch over to Python.
The important detail here is that we only pass a pointer to the data to Python, not the data itself.

### Benchmark setup

In this benchmark, we will use [Apache Drill](http://drill.apache.org/) as the database using its official JDBC driver.
For the data, we will use the January 2017 Yellow Cab New York City trip data converted to Parquet.
We start Drill in its embedded mode using `./bin/drill-embedded`. There we can already peak into the data using 

```sql
SELECT * FROM dfs.`/…/data/yellow_tripdata_2016-01.parquet` LIMIT 1
```

As the main aspect here is to show how to access databases using JDBC in Python, we will use `JayDeBeApi` now to connect to this running Drill instance.
Therefore we start a JVM with `jpype` and then connect using `jaydebeapi` and the `drill-jdbc-all-1.16.0.jar` JAR to the database.
For the JDBC connections, it is important that we have either a classpath with all Java dependencies or as in this case, a JAR that already bundles all dependencies.
Finally, we execute the query and use the result to construct a `pandas.DataFrame`.

```python
import jaydebeapi
import jpype
import os

classpath = os.path.join(os.getcwd(), "apache-drill-1.16.0/jars/jdbc-driver/drill-jdbc-all-1.16.0.jar")
jpype.startJVM(jpype.getDefaultJVMPath(), f"-Djava.class.path={classpath}")
conn = jaydebeapi.connect('org.apache.drill.jdbc.Driver', 'jdbc:drill:drillbit=127.0.0.1')
cursor = conn.cursor()

query = """
    SELECT * 
    FROM dfs.`/…/data/yellow_tripdata_2016-01.parquet`
    LIMIT 1
"""

cursor.execute(query)
columns = [c[0] for c in cursor.description]
data = cursor.fetchall()
df = pd.DataFrame(data, columns=columns)
```

To measure the performance, we have tried initially to run the full query to measure the retrieval performance but as this didn't finish after 10min, we reverted to running the SELECT query with different `LIMIT` sizes.
This lead to the following response times on my laptop (mean ± std. dev. of 7 runs):

| LIMIT n | Time               |
|---------|--------------------|
|   10000 |   7.11 s ± 58.6 ms |
|  100000 |   1min 9s ± 1.07 s |
| 1000000 | 11min 31s ± 4.76 s |

Out of curiosity, we have retrieved the full result set once and this came down to an overall time of `2h 42min 59s` _on a warm JVM_.

### pyarrow.jvm and combined jar

As the above times were quite frustrating, we have high hopes that using Apache Arrow could bring a decent speedup for this operation.
To use Apache Arrow Java and the Drill ODBC driver together, we need to bundle both together on the JVM classpath.
The simplest way to do this is generate a new JAR that includes all dependencies using a build tool like Apache Maven.
With the following `pom.xml` you get a fat JAR using `mvn assembly:single`.
It is important here that your Apache Arrow Java version matches the `pyarrow` version, in this case here, both are at `0.15.1`.
It might still work when they differ but as there is limited API stability between the two implementations, this could otherwise lead to crashes.

```xml
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.uwekorn</groupId>
    <artifactId>drill-odbc</artifactId>
    <version>0.1-SNAPSHOT</version>
    <packaging>jar</packaging>

    <name>drill-odbc</name>

    <properties>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    </properties>

    <dependencies>
        <dependency>
            <groupId>org.apache.arrow</groupId>
            <artifactId>arrow-jdbc</artifactId>
            <version>0.15.1</version>
        </dependency>
        <dependency>
            <groupId>org.apache.arrow</groupId>
            <artifactId>arrow-memory</artifactId>
            <version>0.15.1</version>
        </dependency>
        <dependency>
            <groupId>org.apache.drill.exec</groupId>
            <artifactId>drill-jdbc-all</artifactId>
            <version>1.16.0</version>
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

After the JAR has been built, we now want to start the JVM with it loaded.
Sadly, `jpype` has the limitation that you need to restart your Python process when you want to restart the JVM with different parameters.
We thus adjust the JVM startup command to:

```python
classpath = os.path.join(os.getcwd(), "all-jar/target/drill-odbc-0.1-SNAPSHOT-jar-with-dependencies.jar")
```

To use Apache Arrow Java to retrieve the result, we need to instantiate a `RootAllocator` that is used in Arrow Java to allocate the off-heap memory and also construct a `DriverManager` instance to connect to the database.

```python
ra = jpype.JPackage("org").apache.arrow.memory.RootAllocator(sys.maxsize)
dm = jpype.JPackage("java").sql.DriverManager
connection = dm.getConnection("jdbc:drill:drillbit=127.0.0.1")
```

Once this is setup, we can use the Java method `sqlToArrow` to query a database using JDBC, retrieve the result and convert it to an Arrow RecordBatch on the Java side.
With the helper `pyarrow.jvm.record_batch` we can take the `jpype` reference to the Java object, extract the memory address of the RecordBatch and create a matching Python `pyarrow.RecordBatch` object that points to the same memory.

```python
batch = jpype.JPackage("org").apache.arrow.adapter.jdbc.JdbcToArrow.sqlToArrow(
    connection,
    query,
    ra
)

df = pyarrow.jvm.record_batch(batch).to_pandas()
```

Using these commands, we can now execute the same queries again and compare them to the `jaydebeapi` times:

| LIMIT n | Time  (JayDeBeApi) | Time (pyarrow.jvm) | Speedup |
|---------|--------------------|--------------------|---------|
|   10000 |   7.11 s ± 58.6 ms |   165 ms ± 5.86 ms |     43x |
|  100000 |   1min 9s ± 1.07 s |   538 ms ± 29.6 ms |    128x |
| 1000000 | 11min 31s ± 4.76 s |    5.05 s ± 596 ms |    136x |

With the `pyarrow.jvm` approach, we not get similar times to `turbodbc.fetchallarrow()` on other databases that come with an open ODBC driver.
This also leads to the retrieval of the whole being a more sane `50.2 s` instead of the hours-long wait with `jaydebeapi`.

### Conclusion

By moving the row-to-columnar conversion to the JVM and avoiding to create intermediate Python objects before creating a `pandas.DataFrame` again, we can speedup the retrieval times for JDBC drivers in Python by over *100x`*.
As a user, you need to change your calls to `jaydebeapi` to the Apache Arrow Java API and `pyarrow.jvm`.
Additionally, you will have to take care that the Apache Arrow Java and the JDBC drivers are on the Java classpath.
By using a common Java build tool, this can be achieved by simply declaring them as dependencies of a dummy package.
