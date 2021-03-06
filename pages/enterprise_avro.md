---
permalink: enterprise_avro.html
sidebar: mydoc_sidebar
title: Avro Plugin
keywords: Enterprise, Avro, Binary, File
enterprise: true
toc: false
Tags: Enterprise, Avro, Binary, File
previous: enterprise_virtualcolumns.html
next:
---

{% include prev_next.html %}

## About
When working with large datavolumes, it is sometimes unnescessary to have a database at all. It can be more performant to read binary data directly from a file and then analyze it in Speedment. One such format for storing binary data is [Avro](https://avro.apache.org/).

## Generating Code
Speedment uses the metadata in a database as the domain model when generating code. The metadata is stored in a `speedment.json`-file, and unless you call `mvn speedment:reload`, it will only connect to the database if that file doesn't exist. When working with Avro-files, we use this to our advantage. Instead of using the database metadata to generate the `speedment.json`-file, we can use a Maven plugin called `speedment-avro-maven-plugin` to create it from a number of Avro-schemas. We can then run `mvn speedment:generate` as usual to generate Java code.

```xml
<plugin>
    <groupId>com.speedment.enterprise.plugins</groupId>
    <artifactId>speedment-avro-maven-plugin</artifactId>
    <version>${speedment.enterprise.version}</version>
    
    <configuration>
        <projectName>sakila</projectName>
        <deleteExisting>true</deleteExisting>
        <enableEnumPlugin>true</enableEnumPlugin>
        <overridesFile>src/main/json/speedment_override.json</overridesFile>
        <schemas>
            <schema>
                <tableId>film</tableId>
                <schemaFile>src/main/avro/Film.avsc</schemaFile>
                <overridesFile>src/main/json/speedment_film_override.json</overridesFile>
            </schema>
        </schemas>
    </configuration>
    
    <!-- Execute automatically -->
    <executions>
        <execution>
            <id>Reload From Avro File</id>
            <phase>generate-sources</phase>
            <goals>
                <goal>avro</goal>
            </goals>
        </execution>
    </executions>
</plugin>
```

In the example above, the plugin is configured to generate a project called "sakila" with one table called "Film" based on the Avro schema `src/main/avro/Film.avsc`. There are two so called `<overridesFile>` specified; one for the project as a whole and one that is local only to the `Film` table. These are optional `.json`-files that you can create manually that will override any settings generated by the plugin. This is useful for configuring things that the Avro-plugin can't yet figure out automatically, like the package structure of your project or which [type mappers](https://speedment.github.io/speedment-doc/maven.html#adding-a-type-mapper) to use.

### Override Generated speedment.json
Here is an example of how `src/main/json/speedment_override.json` could look:

```json
{
  "config" : {
    "id" : "sakila",
    "name" : "sakila",
    "companyName" : "speedment",
    "packageName" : "com.speedment.example.sakila",
    "appId" : "b9c8eb47-3810-4f45-972b-f2cf64f43d71",
    "dbmses" : [
      {
        "id" : "sakila",
        "alias" : "my_dbms",
        "schemas" : [
          {
            "id" : "sakila"
            "alias" : "my_schema"
          }
        ]
      }
    ]
  }
}
```

We set a custom `company` and `packageName` on the project-level and also specify a custom `alias` for both the generated `dbms` and the `schema`. Note that the `id` of both `project`, `dbms` and `schema` will be the value you specified as `projectName` in the maven plugin configuration.

### Override Table Specific Settings
Here is an example of how the table-specific `src/main/json/speedment_film_override.json` could look:

```json
{
  "config" : {
    "id" : "sakila",
    "dbmses" : [
      {
        "id" : "sakila",
        "schemas" : [
          {
            "id" : "sakila",
            "tables" : [
              {
                "id" : "film",
                "avroDataFile" : "Film.avro",
                "columns" : [
                  {
                    "id" : "film_id",
                    "alias" : "id"
                  },
                  {
                    "id" : "rating",
                    "enabled" : false
                  }
                ]
              }
            ]
          }
        ]
      }
    ]
  }
}
```

We specify the location of the binary data file for the `Film` table, and also change the default settings for two of the columns. The first one (`film_id`) we give an alias to so that it will be called just `id` in the code. The second one (`rating`) we disable since we don't want it to be generated at all.

### Automating Build Process
The next step is to invoke `mvn speedment:generate` automatically **after** the `speedment-avro-maven-plugin` has been invoked. This can be done like this:

```xml
<plugin>
    <groupId>com.speedment.enterprise</groupId>
    <artifactId>speedment-enterprise-maven-plugin</artifactId>
    <version>${speedment.enterprise.version}</version>

    <configuration>
        <components>
            <component>com.speedment.enterprise.datastore.tool.DataStoreToolBundle</component>
            <component>com.speedment.enterprise.plugins.avro.generator.AvroGeneratorBundle</component>
        </components>
    </configuration>

    <executions>
        <execution>
            <id>Generate Speedment Sources</id>
            <phase>generate-sources</phase>
            <goals>
                <goal>generate</goal>
            </goals>
        </execution>
    </executions>
</plugin>
```

Finally, you need to make sure that you have the avro-dependencies on the classpath. To fix this, simply add the `avro-runtime`-dependency to the `pom.xml`-file.

```xml
<dependency>
    <groupId>com.speedment.enterprise.plugins</groupId>
    <artifactId>avro-runtime</artifactId>
</dependency>
```

## Using Avro Files at Runtime
To load data from the `.avro`-files instead of a database, you need to add the `AvroRuntimeBundle` to the Speedment application builder and append `withSkipCheckDatabaseConnectivity()`.

```java
new AmericanApplicationBuilder()
    .withSkipCheckDatabaseConnectivity()
    .withBundle(AvroRuntimeBundle.class)
    .withBundle(DataStoreBundle.class)
    .build();
```

Note that the **order is important**! The `AvroRuntimeBundle` needs to come before the `DataStoreBundle` since both specify a custom `StreamSupplierComponent`.

## Customizing Generated Code
When the code is generated, it will look a bit different than if a regular SQL database was used. First, you no longer have any `SqlAdapter.java`-files, but instead a number of `AvroAdapter.java`-files. These handle the deserialization from Avro to Speedment.

To change the location of the data file (the one that ends with `.avro`) at runtime, you can override the `dataFile`-method in `XXXAvroAdapter.java`. For an example, in the Sakila demo, I could override the method like this:

```java
public class FilmAvroAdapter extends GeneratedFilmAvroAdapter {

    @Config(name = "film.data.file", value = "Film.avro")
    private String filmDataFile;
    
    @Override
    protected Path dataFile(ProjectComponent projects) {
        return Paths.get(filmDataFile);
    }
    
}
```

You can now change the location of the data file without recompiling by simply creating a `settings.properties`-file in the root of my project with the line:

```properties
film.data.file = another_dir/another_film.avro
```

And it will be used instead.

{% include prev_next.html %}

## Discussion
Join the discussion in the comment field below or on [Gitter](https://gitter.im/speedment/speedment)

{% include messenger.html page-url="integration.html" %}
