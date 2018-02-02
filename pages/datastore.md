---
permalink: datastore.html
sidebar: mydoc_sidebar
title: Data Store
keywords: Data Store, In, Memory, Acceleration
toc: false
Tags: Data Store
previous: advanced_features.html
next: enterprise_plugins.html
---

{% include prev_next.html %}

## What is DataStore?
The Speedment Enterprise Datastore is a proprietary module for Speedment that stores database entities in-memory, allowing database queries to be performed extremely fast, utilizing the Java 8 Stream API to the fullest.

A Stream does not describe any details about how data is retrieved, in fact this is delegated to the framework defining the pipeline source and termination. There is nothing in the design of a stream entailing data must come from a SQL query. This fact is used by Speedment Enterprise that contains an in-JVM-memory analytics engine called DataStore, allowing streams to connect directly to RAM instead of remote databases.

The engine provides streams with exactly the same API semantics as for databases but will execute queries with orders of magnitude lower latencies. This creates a new way to write high performance data applications whereby the actual source-of-truth can remain with an existing database. It is possible to provision terabytes of data in the JVM with no garbage collection impact because data is stored off heap and can optionally be mapped to SSD files. Streams can have a latency well under one microsecond. Comparing this to a traditional application with a database connection, just the TCP round-trip delay in a high-performance network is hardly ever under 40 microseconds and then database latency and data transfer times have to be added on top.

Thus, the DataStore module is best suited for read-intensive applications, like analytics. 

## Enabling DataStore
In order to use DataStore you need a commercial Speedment license or a trial license key. Download a free trial license using the Speedment [Initializer](https://www.speedment.com/initializer/).

The DataStore module needs to be referenced both in your pom.xml file and in you application.

### POM File
Use the [Initializer](https://www.speedment.com/initializer/) to get a POM file template. To use DataStore, add it as a dependency to the speedment-enterprise-maven-plugin and mention it as a component:
``` xml
    <plugin>
        <groupId>com.speedment.enterprise</groupId>
        <artifactId>speedment-enterprise-maven-plugin</artifactId>
        <version>${speedment.enterprise.version}</version>
        <dependencies>
            <dependency>
                <groupId>com.speedment.enterprise</groupId>
                <artifactId>datastore-tool</artifactId>
                <version>${speedment.enterprise.version}</version>
            </dependency>
        </dependencies> 
        <configuration>
            <components>
                <component>com.speedment.enterprise.datastore.tool.DataStoreToolBundle</component>
            </components>
        </configuration>
    </plugin>
```

You also have to depend on DataStore as a runtime dependency to your application:
``` xml
    <dependencies>
        <dependency>
            <groupId>com.speedment.enterprise</groupId>
            <artifactId>datastore-runtime</artifactId>
            <version>${speedment.enterprise.version}</version>
        </dependency>
    </dependencies>
```

### Application
When you build the application, the DataStoreBundle needs to be added to the runtime like this:
``` java
    SakilaApplicationBuilder builder = new SakilaApplicationBuilder()            
        .withPassword(password)
        .withBundle(DataStoreBundle.class);
```

## Using DataStore

### Load from the Database
Before DataStore can be used, it has to load all database content into the JVM. This is how it is done:
``` java
    // Load the in-JVM-memory content into the DataStore from the database
    app.get(DataStoreComponent.class)
        .ifPresent(DataStoreComponent::load);
```

After the DataStore module has been added and loaded, all stream queries will be made towards RAM instead of the remote database. No other change in the application is needed.

### Synchronizing with the Database
If you want to update the DataStore to the latest state of the underlying database, do like this:
``` java
    // Refresh the in-JVM-memory content from the database
    app.get(DataStoreComponent.class)
        .ifPresent(DataStoreComponent::reload);
```

This will load a new version of the database in the background and when completed, new streams will use the new data. Old ongoing streams will continue to use the old version of the DataStore content. Once all old streams are completed, the old version of the DataStore content will be released.

### Loading/Reloading with a Particular ExecutorService
By default, DataStore will load and structure data from the database using the common `ForkJoinPool`. Sometimes loading can take minutes and then it might be desirable to perform loading using a custom executor with perhaps a limited number of threads so that the application can continue to serve requests while loading in the background.
You can provide any executor to the `load()` and `reload()` methods like this:
``` java
    ExecutorService myExecutorService = Executors.newFixedThreadPool(3);
    DataStoreComponent dsc = app.getOrThrow(DataStoreComponent.class);
    dsc.load(myExecutorService);
    ...
    // Later on
    dsc.reload(myExecutorService);
```

### Load/Reload Individual Rows
If you only want to pull in a subset of the database rows, you can use a variant of the load/reload method as shown hereunder:
``` java
    final StreamSupplierComponentDecorator decorator = StreamSupplierComponentDecorator.builder()
        .withStreamDecorator(Film.FILM_ID.identifier().asTableIdentifier(), s -> s.limit(100))
        .build();

    DataStoreComponent dataStoreComponent = app.getOrThrow(DataStoreComponent.class);
    dataStoreComponent.load(ForkJoinPool.commonPool(), decorator);
```
This will only load the first 100 films from the database. Any stream operation(s) that returns the same stream type (i.e. filter(), sorted(), distinct(), limit() and skip() but not map() and flatMap()) may be applied in the decorator. Specifically, applying the operation `s -> s.limit(0)` will prevent DataStore from loading any data into memory.

This is useful, for example when working on time based data, in micro service deployments or in various test scenarios.

{% include warning.html content = "
Providing a custom `StreamSupplierComponentDecorator` means that you are assuming the responsibility of ensuring referential integrity. If the number of entities are reduced, for example using `filter()` or `limit()` operations, then these skipped entities may be referenced by other entities. This must now be handled by your application.
" %}


### Load/Reload Individual Tables
Sometimes it makes sense to just put a limited set of tables in the DataStore while other tables can be reached via the underlying database. By using the Speedment Enterprise module Meta Stream Supplier, we can select which tables are retrieved from the the DataStore and which tables will be retrieved from the database.

In order to run, the module first needs to be configured using a class that implements the interface `MetaStreamSupplierConfigurator`. This is how a custom configurator can look like:
``` java
public static class MyMetaStreamConfigurator implements MetaStreamSupplierConfigurator {
        @Override
        public Stream<TableMapping<Class<? extends StreamSupplierComponent>>> tableMappings() {
            return Stream.of(
                TableMapping.of(Film.FILM_ID.identifier().asTableIdentifier(), DataStoreStreamSupplierComponent.class),
                TableMapping.of(Artist.ARTIST_ID.identifier().asTableIdentifier(), SqlStreamSupplierComponent.class)
            );
        }
    }
```

This will configure the Meta Stream Supplier to explicitly use the Data Store for the film table and the database for the artist table. Unconfigured tables will default to the top most `StreamSupplierComponent` (usually the DataStore) but if you like another behavior, just override the `MetaStreamSupplierConfigurator::defaultStreamSupplierComponentClass` method.

By installing the bundle `MetaStreamSupplierBundle` we activate the module. Here is an example of how to install the Meta Stream Supplier module:
``` java
    SakilaApplication app = new SakilaApplicationBuilder()
        .withBundle(DataStoreBundle.class);
        .withComponent(MyMetaStreamConfigurator.class)
        .withBundle(MetaStreamSupplierBundle.class)
        .build();
```

When you elect to used some tables from the database rather than the DataStore then you usually do not want those tables to take up valuable space in the DataStore since you are not going to use them anyhow. Read more on how to control what data goes into the DataStore [here](#selecting-rows).

{% include warning.html content = "
Providing a custom `MetaStreamSupplierComponent` means that you are assuming the responsibility of ensuring referential integrity. If the `MetaStreamSupplierComponent` are using components that are from different transaction states, then these component views might violate referential integrity. This must now be handled by your application.
" %}

### Custom Loading/Reloading
Since 1.1.15, there is a general way of configuring the load process. This way will eventually replace other ways of loading. Loading/reloading is invoked using the `DataStoreComponent.createSnapshotJob(LoadConfiguration loadConfiguration)`. The `LoadConfiguration` can be obtained using a builder as exemplified below:

``` java
        
        StreamSupplierComponentDecorator myDecorator = ...;
        Transaction myTransaction = ...;
        ExecutorService myExecutor = ...;
        
        LoadConfiguration loadConfiguration = LoadConfiguration.builder()
            .withChunkSize(10_000)
            .withChunkSize(Films.FILM_ID.identifier().asTableIdentifier(), 5_000)
            .withDecorator(myDecorator)
            .withExecutor(myExecutor)
            .withTransaction(myTransaction)
            .build();
        
        DataStoreComponent ds = app.getOrThrow(DataStoreComponent.class);
        CompletableFuture<Void> job = ds.createSnapshotJob(loadConfiguration);
```

|   Method             | Parameter type             | Action
| :------------------- | :------------------------- | ------------------------------- |
|   withChunkSize      | chunkSize                  | Sets the positive chunk size that shall be used for all tables when loading data from the data source unless specifically overridden by the `Builder.withChunkSize(tableIdentifier, chunkSize)` method. The default chunk size is `Long.MAX_VALUE` meaning no chunk loading shall be used. Chunk loading can sometimes improve load performance for some database types.
|   withChunkSize      | tableIdentifier, chunkSize | Sets the positive chunk size that shall be used for the given `tableIdentifier` when loading data from the data source. The default chunk size is `Long.MAX_VALUE` meaning no chunk loading shall be used. Chunk loading can sometimes improve load performance for some database types.
|   withDecorator      | decorator                  | Sets the decorator to use when creating a data snapshot. The default decorator is the `StreamSupplierComponentDecorator.identity()` decorator that does not modify any stream.
|   withExecutor       | executor                   | Sets the executor to use when creating a data snapshot. The default executor is the `ForkJoinPool.commonPool()`
|   withExecutor       | executor                   | Sets the Transaction that shall be used when creating a data snapshot. The default transaction is no transaction.



### Showing The Load/Reload Progress
The load and organize process can be viewed in the log by enabling `APPLICATION_BUILDER` logging as shown hereunder:
``` java
    SakilaApplicationBuilder builder = new SakilaApplicationBuilder()        
        .withPassword(password)
        .withLogging(LogType.APPLICATION_BUILDER)
        .withBundle(DataStoreBundle.class);
        .build();
```

When the DataStore is loaded, information on the loading progress will be shown in the logs:
``` text
2017-05-16T01:46:16.901Z DEBUG [pool-1-thread-3] (#APPLICATION_BUILDER) - db0.sakila.category : Loading from database completed (took 136.76 ms).
2017-05-16T01:46:16.939Z DEBUG [pool-1-thread-1] (#APPLICATION_BUILDER) -    db0.sakila.actor : Loading from database completed (took 195.63 ms).
2017-05-16T01:46:16.948Z DEBUG [pool-1-thread-1] (#APPLICATION_BUILDER) -    db0.sakila.actor : Building entity cache with 200 rows completed (took 7.98 ms). Density is 31 bytes/entity
2017-05-16T01:46:16.948Z DEBUG [pool-1-thread-3] (#APPLICATION_BUILDER) - db0.sakila.category : Building entity cache with 16 rows completed (took 46.12 ms). Density is 20 bytes/entity
2017-05-16T01:46:16.985Z DEBUG [pool-1-thread-1] (#APPLICATION_BUILDER) -  db0.sakila.country : Loading from database completed (took 36.65 ms).
2017-05-16T01:46:16.995Z DEBUG [pool-1-thread-1] (#APPLICATION_BUILDER) -  db0.sakila.country : Building entity cache with 109 rows completed (took 8.78 ms). Density is 24 bytes/entity
2017-05-16T01:46:17.007Z DEBUG [pool-1-thread-2] (#APPLICATION_BUILDER) -  db0.sakila.address : Loading from database completed (took 252.67 ms).
...
2017-05-16T01:46:18.732Z DEBUG [pool-1-thread-1] (#APPLICATION_BUILDER) -  rental.rental_date : Loading column store
2017-05-16T01:46:18.743Z DEBUG [pool-1-thread-3] (#APPLICATION_BUILDER) - rental.inventory_id : Building column cache with 16,044 rows completed (took 53.60 ms). Density is 4 bytes/entity
2017-05-16T01:46:18.743Z DEBUG [pool-1-thread-3] (#APPLICATION_BUILDER) -    rental.rental_id : Loading column store
2017-05-16T01:46:18.745Z DEBUG [pool-1-thread-1] (#APPLICATION_BUILDER) -  rental.rental_date : Building column store
2017-05-16T01:46:18.748Z DEBUG [pool-1-thread-2] (#APPLICATION_BUILDER) -  rental.return_date : Building column cache with 16,044 rows completed (took 97.70 ms). Density is 4 bytes/entity
2017-05-16T01:46:18.750Z DEBUG [pool-1-thread-3] (#APPLICATION_BUILDER) -    rental.rental_id : Building column store
2017-05-16T01:46:18.756Z DEBUG [pool-1-thread-3] (#APPLICATION_BUILDER) -    rental.rental_id : Building column cache with 16,044 rows completed (took 12.33 ms). Density is 4 bytes/entity
2017-05-16T01:46:18.781Z DEBUG [pool-1-thread-1] (#APPLICATION_BUILDER) -  rental.rental_date : Building column cache with 16,044 rows completed (took 47.96 ms). Density is 4 bytes/entity
Finished reloading in 2.05 s.
```

### Disable Unnecessary Indexes
By default, the Datastore indexes every column of every table. However, since the indexes can sometimes take up quite a bit of extra space, you might want to disable indexing on columns that you are not filtering or sorting on directly.

Open the Speedment Tool and go to the column you want to disable indexing for and select "Disable In-Memory Index".

{% include image.html file="disableindex.png" url="https://www.speedment.com/" alt="Disable In-Memory Index in Speedment Tool" caption="Select Disable In-Memory Index" %}

{% include tip.html content = "
You should take a moment to look over your indexes in the Speedment Tool to see how they are mapped. If you have low-cardinality columns (like `gender`, `city`, `category` etc), you might want to use the [Enum Serializer Plugin](enterprise_enums#top) for Datastore to convert them into enums. You should also try and use `(To Primitive)` as the Type Mapper wherever possible.
" %}

### Optimizing for Low Cardinality
**Requires Speedment Enterprise 1.1.15 or later.** If you have a column in your database with a very low cardinality, you can get faster load and query times if you mark it as a "Low Cardinality" column in the Speedment Tool. 

{% include image.html file="low_cardinality.png" url="https://www.speedment.com/" alt="Mark Column as Low Cardinality in Speedment Tool" caption="Mark Column as Low Cardinality" %}

This tells Speedment to do two things:

1. Try to optimize storage by removing duplicates
2. Create separate buckets for every distinct value in the index

There are however no guarantees that these optimizations will be done, since other factors might make them unnescessary or even slower. Marking columns as "Low Cardinality" is usually a good thing if you have up to a few hundred disinct values.

{% include tip.html content = "
If you have columns with low cardinality where the distinct values possible never changes, it will be even better to map it to an Java enum class using the [Enum Serializer Plugin](enterprise_enums#top) for Datastore.
" %}

### Creating Multi-Indexes
**Speedment Enterprise 1.1.10** introduced the concept of Multi-Indexes which can significantly increase the performance of filters that involve two columns of the same (or similar) type. Multi-Indexes take up the same amount of memory as a single-column index, but can filter both dimensions in `O(log N)` time-complexity.

To create a Multi-Index for a pair of columns, open the Speedment Tool and right-click on the table.

{% include image.html file="create_multi_index.png" url="https://www.speedment.com/" alt="Creating Multi-Indexes in Speedment Tool" caption="Step 1: Right-Click Table and select Create Multi-Index" %}

In the dropdown menu, select "Create Multi Index". Scroll to the bottom of the table and select the newly created index.

On the right side, you can now enter the name of the columns to use as Primary and Secondary sort order. The name must match perfectly the "Database Name" of the columns in question. You may also want to set a custom name for the index. To enable editing, right-click the "MultiIndex Name"-textfield and select "Enable editing".

{% include image.html file="configure_multi_index.png" url="https://www.speedment.com/" alt="Configuring the Primary and Secondary Column of Multi-Indexes" caption="Step 2: Configure Primary and Secondary Column" %}

Currently, only very specific filters are being considered when interpreting a Speedment Stream for a Multi-Index. To make sure a filter is optimized correctly, it should be written as a `.and()`-combination of two regular Speedment filters.

**Example Usage:**
```java
flights.stream()
    .filter(Flight.ORIGIN.equal("SFO")
       .and(Flight.DESTINATION.equal("LAX"))
    )
    .skip(100_000).limit(1_000)
    .collect(toList());
```

Only `equal`- and `in`-operators can be optimized using multi-indexes.

### Obtaining Statistics
You can obtain statistics on how tables, columns and memory segments are used by invoking the DataStoreComponent::getStatistics method. Here is an example of how to print out DataStore statistics.
``` java
    app.get(DataStoreComponent.class)
        .map(DataStoreComponent::getStatistics)
        .map(StatisticsUtil::prettyPrint)
        .ifPresent(s -> s.forEachOrdered(System.out::println));
```

This will produce an output that starts like this:

|   Table  |  Column    | Off_Heap [B]| Rows | Nulls | Cardinality | Type  |         Class              | Distinct  |
| :------- | :--------- | ----------: | ---: | ----: | :---------- | :---- | :------------------------- | :-------- |
|   actor  |    -       |        6307 |  200 |     - |          -  | Table | SingleSegmentEntityStore   |      -    |
|   actor  | actor_id   |         400 |    - |     0 |          -  | Column| IndexedIntFieldCache       | true      |     
|   actor  | first_name |         400 |    - |     0 |          -  | Column| IndexedStringFieldCache    | false     |
|   actor  | last_name  |         400 |    - |     0 |          -  | Column| IndexedStringFieldCache    | false     |
|   actor  | last_update|         400 |    - |     0 |          -  | Column| IndexedComparableFieldCache| false     |
| address  |       -    |       66286 |  603 |     - |          -  | Table | SingleSegmentEntityStore   |      -    |
| address  | address    |        2412 |    - |     0 |          -  | Column| IndexedStringFieldCache    | false     |
| address  | address2   |        2404 |    - |     4 |          -  | Column| IndexedStringFieldCache    | false     |
| address  | address_id |        2412 |    - |     0 |          -  | Column| IndexedIntFieldCache       | true      |

...

## Aggregating Columns

A common use case in analytical applications is to aggregate many results into a few. This can be done very efficiently using the specialized collectors built into Speedment Enterprise by leveraging the standard Java Streams API.

An even more efficient way to perform off-heap aggregation using Speedment is enabled by the AggregatorBuilder which is designed to perform all steps of the aggragation with minimal heap memory footprint. 

The two methods are described in the following two sections, starting with the most efficient in [Aggregation using the dedicated Speedment AggregatorBuilder](#AggregatorBuilder) and then [Aggregation using the standard Java Streams API](#AggregationCollections) in the following section.

### <a name="AggregatorBuilder"></a> Aggregation using the dedicated Speedment AggregatorBuilder 
**Requires Speedment Enterprise 1.1.18 or later.** 
In the following you will find two examples of how to use the AggregatorBuilder API for super-fast off-heap aggregation.

In both examples we need an `EntityStore<X>` that holds the entities of type `X` over which the aggregation 
will take place. In the two examples below we will be using `Film` entities. 
The entity store can be retrieved from a holder of the data store component as follows.

```java
      DataStoreComponent dataStore = app.getOrThrow(DataStoreComponent.class);
      FilmManager filmManager = app.getOrThrow(FilmManager.class);
      DataStoreHolder holder = dataStore.currentHolder();
      EntityStore<Film> store = holder.getEntityStore(filmManager.getTableIdentifier());
```

#### A simple aggregation

In the first example we will compute the average length and sum of replacement cost of all films in the database, grouped on rating and release year. To capture the data to be aggregated for each unique aggregate key we define an aggregator object. It has
* a constructor that takes a reference to an entity in an entity store
as parameter
* and data fields used to accumulate the aggregation results per key.

The `@Data` annotation of [Project Lombok](https://projectlombok.org/) generates getter for all fields, setter for non-final fields and a constructor that takes the final field as parameter.

```java
      @Data // See Project Lombok
      class LengthAndCost {
          float length, replacementCost;
      }
```

With this class and an entity store `store` the aggregation can be expressed as follows.

```java      
List<LengthAndCost> aggregate = store.entityRefStream()
    .collect(
        Aggregator.builder(
                EntityRef.of(Salary.class), // This helps us get the generics right
                LengthAndCost::new)         // How we want the results
            .key(EntityRefUtil.ofEnum(Film.RATING))
            .key(EntityRefUtil.ofInt(Film.RELEASE_YEAR))
            .mean(EntityRefUtil.ofInt(Film.LENGTH), LengthAndCost::setLength)
            .sum(EntityRefUtil.ofInt(Film.REPLACEMENT_COST), LengthAndCost::setReplacementCost)
            .build()
    ).collect(toList());
```

The class methods of `com.speedment.enterprise.datastore.runtime.aggregator.EntityRefUtil.*` can be imported statically to make the code a bit more readable.

```java      
List<LengthAndCost> aggregate = store.entityRefStream()
    .collect(
        Aggregator.builder(
                EntityRef.of(Film.class), // This helps us get the generics right
                LengthAndCost::new)         // How we want the results
            .key(ofEnum(Film.RATING))
            .key(ofInt(Film.RELEASE_YEAR))
            .mean(ofInt(Film.LENGTH), LengthAndCost::setLength)
            .sum(ofInt(Film.REPLACEMENT_COST), LengthAndCost::setReplacementCost)
            .build()
    ).collect(toList());
```

The aggregation is collected into a list of `LengthAndCost` instances, one for each unique aggregate key in the entity store. Thus, for each unique pair of rating and release year we get a `LengthAndCost` instance holding the corresponding aggregate values.

#### Using Expressions
Sometimes, the value that is to be aggregated is stored in one way in the database but should be used in a different way in the aggregation. Say for an example that there is a table of Salaries and that each row contains information about an employee-number, the date when that employee started and a date when the salary was last updated. To determine the average time from when an employee started to when the salary was first updated, an expression must be defined. The Speedment aggregator has a number of helper methods to make it easier to declare this kind of expressions.

First, import the methods of this utility class:
```java
import static com.speedment.enterprise.aggregator.core.expression.Expressions.*;
```

// Secondly, define a class to hold the results of the aggregation:

```java
@Data // Generate setters and getters with Lombok
class DaysSinceRaise {
    double averageDaysSinceRaise;
    Employee.Gender gender; // Enum column
}
```

Then the aggregation can be defined like this:

```java
List<DaysSinceRaise> aggregate = store.entityRefStream()
    .collect(
        Aggregator.builder(
                EntityRef.of(Employee.class),
                DaysSinceRaise::new)
            .key(ofEnum(Employee.GENDER))
            .first(ofEnum(Employee.GENDER), DaysSinceRaise::setGender)
            .mean(
                minusAsInt(
                    ofInt(Employee.DATE_LAST_RAISE)
                        .orElseGet( // Might never have gotten a raise
                            minusAsInt(  // In that case, return -1
                                ofInt(Employee.DATE_STARTED),
                                1
                            )
                        ),
                    ofInt(Employee.DATE_STARTED)
                ),
                DaysSinceRaise::setAverageDaysSinceRaise
            )
            .build()
    ).collect(toList());
```

Expressions is a very powerful tool to express custom fields. They are strictly type-safe and prefer primitive values of performance reasons. They also separate between nullable and non-nullable values by having two alternative interfaces for each type (for an example `ToInt` and `ToIntOrNull`). A nullable field can be converted into a non-nullable field by using any of the methods `orElse`, `orElseGet` or `orThrow()`. This is similar to how `java.util.Optional` works, except that `Optional` holds a single value where `ToInt`, `ToLong` etc represents an expression.

#### Computing Correlation Coefficients
A more advanced example of the capabilities of the aggregator is to compute Pearsson's Correlation Coefficient using the online covariance method.

To compute a correlation between the salary and days employed between men and women, first define the expression and aggregator that we are going to use. This can be done statically since both these objects are immutable.

```java
private final static int SECONDS_IN_A_DAY = 86_400;

private final static ToInt<EntityRef<Salary>> DAYS_EMPLOYED =
    minusAsInt(
        ofInt(Salary.FROM_DATE).orThrow(),
        ofInt(Salary.HIRE_DATE).orThrow()
    ).map(days -> days / SECONDS_IN_A_DAY);

private final static Aggregator<EntityRef<Salary>, ?, Result> AGGREGATOR =
    Aggregator.builder(EntityRef.of(Salary.class), Result::new)
        .key(ofEnum(Salary.GENDER))
        .first(ofEnum(Salary.GENDER), Result::setGender)
        .count(Result::setCount)
        .mean(ofInt(Salary.SALARY), Result::setSalaryMean)
        .mean(DAYS_EMPLOYED, Result::setDaysEmployedMean)
        .variance(ofInt(Salary.SALARY), Result::setSalaryVariance)
        .variance(DAYS_EMPLOYED, Result::setDaysEmployedVariance)
        .covariance(ofInt(Salary.SALARY), DAYS_EMPLOYED, Result::setCovariance)
        .build();
```

The `Result` object needs to hold a number of fields to display to the user. It is defined like this:

```java
@Data
private static class Result {
    private Gender gender;
    private long count;
    private double salaryMean;
    private double daysEmployedMean;
    private double salaryVariance;
    private double daysEmployedVariance;

    @JsonIgnore
    private double covariance;
    private double correlation;
}
```

Once these has been created, use the aggregator to collect the stream of entity references. The collector returns a new stream of result holders (the `Result` class defined above). The actual coefficient can be computed by simply mapping this stream.

```java
Map<Gender, Result> resultByGender = entityStore().entityRefStream()
    .collect(AGGREGATOR)  // This is where all the offheap aggregations are done
    .map(result -> {
        if (result.salaryVariance == 0
        ||  result.daysEmployedVariance == 0) {
            result.setCorrelation(Double.NaN);
        } else {
            result.setCorrelation(
                result.covariance / (
                    Math.sqrt(result.salaryVariance) *
                    Math.sqrt(result.daysEmployedVariance)
                )
            );
        }
        return result;
    })
    .collect(toMap(  // Use a regular java Collector to get the result as a map
        Result::getGender,
        Function.identity()
    ))
```

Aggregator also supports parallel streams. To utilize this, simply make the stream parallel before collecting.

```java
entityStore().entityRefStream()
    .parallel()   // Simple!
    .collect(AGGREGATOR);
```

**Requires Speedment Enterprise 1.1.10 or later.** Aggregation can also be done using the specialized collectors built into Speedment Enterprise.

The following example will calculate the minimum, maximum and average rental duration for each "rating" category.

```java
System.out.println(Json.toJson(
    films.stream()
        .collect(groupingBy(
            entityFunction(
                Film::getRating,
                store -> ref -> store.deserializeReference(ref, Film.RATING)
            ),
            mergeBuilder(Film.class, LinkedHashMap::new)
                .first(Film.RATING, (map, rating) -> map.put("rating", rating.name()))
                .min(Film.RENTAL_DURATION, (map, duration) -> map.put("minDuration", duration))
                .max(Film.RENTAL_DURATION, (map, duration) -> map.put("maxDuration", duration))
                .average(Film.RENTAL_DURATION, (map, duration) -> map.put("averageDuration", duration))
                .build(),
            toList()
        ))
    )
);
```

The code above outputs the following:

```json
[
  {
    "rating" : "PG13",
    "minDuration" : 3,
    "maxDuration" : 7,
    "averageDuration" : 4.6
  },
  {
    "rating" : "R",
    "minDuration" : 3,
    "maxDuration" : 7,
    "averageDuration" : 4.857142857142857
  },
  {
    "rating" : "G",
    "minDuration" : 3,
    "maxDuration" : 7,
    "averageDuration" : 4.6
  },
  {
    "rating" : "NC17",
    "minDuration" : 3,
    "maxDuration" : 6,
    "averageDuration" : 4.0
  },
  {
    "rating" : "PG",
    "minDuration" : 3,
    "maxDuration" : 7,
    "averageDuration" : 4.666666666666667
  }
]
```

The collection is done in a few steps. The `groupingBy` method is statically imported from the `EntityCollectors`-class. It takes three arguments. 

**Grouping Function**
The first argument is the grouping method that will be used to categorize the incomming entities. Here we use the `entityFunction` method statically imported from the `EntityFunction`-class. This allows us to specify two different lambdas; one that operates on an already deserialized entity and one that operats directly on an in-memory reference. In the best case scenario, we can perform the categorization without deserializing more than one field.

**Inner Collector**
For the second argument of `groupingBy` is used to describe how entities within one category are to be reduced. The method is imported statically from `EntityCollectors`. `mergeBuilder` takes the class of the incomming objects as well as a `Supplier` for the resulting type. In this case, we want to collect the inner stream into a `LinkedHashMap` so that it can be turned into JSON. `mergeBuilder` works as a Builder Pattern, allowing every field of the entity to be collected individually. The builder is terminated by calling `.build()`.

**Outer Collector**
The third argument of `groupingBy` is a regular Java `Collector` that is used to collect the results for each category. In this case, we use the standard Java `Collectors.toList()` method so that we get a `java.util.List`.

The entire operation will complete in `O(N)` time complexity, without deserializing more than exactly the fields needed for the result.

### Clear the DataStore
You can explicitly clear the content of the DataStore by calling the `clear()` method as shown below. After the clear method has been called, streams are not available and a DataStoreNotLoadedException will be thrown if a stream is requested.
``` java
    // Clear the DataStore and release all in-JVM-memory resources
    app.get(DataStoreComponent.class)
        .ifPresent(DataStoreComponent::clear);
    // Streams are now served by the database instead of DataStore
```
If `load()` is called after `clear()`, streams can again be served by DataStore.

## Performance
The DataStore module will sort each table and each column upon load/re-load. This means that you can benefit from low latency regardless on which column you use in stream filters, sorters, etc.

When the DataStore module is being used, Stream latency will be orders of magnitudes better.

{% include prev_next.html %}

## Discussion
Join the discussion in the comment field below or on [Gitter](https://gitter.im/speedment/speedment)

{% include messenger.html page-url="datastore.html" %}
