---
title: 'How to create tests for Bigquery integration with tests containers and Spring Boot 3'
date: '2024-05-12'
spoiler: "How to create tests for Bigquery integration with tests containers and Spring Boot 3"
---

# How to create tests for Bigquery integration with tests containers and Spring Boot 3

The goal of this article is teach you how to implement integrations tests in applications that use bigquery. The reason
of that is that I need to implement this recently and I would like register my knowledge.

After finish this article you are prepared to:

* Implementing tests with Tests containers
* Make basic operations with Google Bigquery

I assumed you know how to connect with bigquery using java, although if don't you can try
see [this class](dev/danielarrais/bigqueryintegrationtest/config/BigQueryConfig.java).

So, let's go!

## Dependencies

For this article you can need create a basic maven/gradle Spring Boot project and add the bellow dependencies:

```groovy
implementation 'com.google.cloud:google-cloud-bigquery'
testImplementation 'org.testcontainers:junit-jupiter'
testImplementation 'org.testcontainers:gcloud'
```

The first dependency is used to connect on Google Bigquery and the lasts two is used to provide containers for tests
programmatically.

## Insert data into dataset

Before the tests (sorry, it isn't TDD hahaha) we need to create code for test! So I'll create a service for
insert and list data of the bigquery dataset. It's too easy!

```java
@Service
@RequiredArgsConstructor
public class BigQueryService {
    String tableName = "teste";

    private final BigQuery bigquery;
    private final BigQueryProperties properties;

    public InsertAllResponse insertData(Map<String, String> data) {
        var tableId = TableId.of(properties.getProjectId(), properties.getDataset(), tableName);
        var contentForInsert = InsertAllRequest
                .newBuilder(tableId)
                .addRow(data).build();

        return bigquery.insertAll(contentForInsert);
    }

    public List<HashMap<String, String>> getData() throws InterruptedException {
        var fullTableName = String.format("%s.%s.%s", properties.getProjectId(), properties.getDataset(), tableName);
        var query = "SELECT * FROM `" + fullTableName + "` LIMIT 100";

        var results = bigquery.query(QueryJobConfiguration.of(query));
        var schemaResult = results.getSchema();

        return results.streamValues().map(fieldValues -> {
            var map = new HashMap<String, String>();
            IntStream.range(0, fieldValues.size()).forEach(index -> {
                var columnName = schemaResult.getFields().get(index).getName();
                map.put(columnName, fieldValues.get(index).getStringValue());
            });
            return map;
        }).toList();
    }
}
```

## Bigquery container

Now I'll start to create my integration test for my service `BigQueryService`. Basically I need to instantiate a test
container with to bigquery in my class test and set up my dataset and my table in my local bigquery before each
execution. For create the bigquery container you must declare:

```java
@Testcontainers
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
public class BigQueryIntegrationTest {
    @Container
    static BigQueryEmulatorContainer bigQueryContainer = new BigQueryEmulatorContainer("ghcr.io/goccy/bigquery-emulator:0.4.3");
}
```

The attribute `bigQueryContainer` was decelerated as `static` with the annotation `@Container` for the container
starting one time for all tests. If you want a new container for each test you must remove the `static` declaration. The
type of them is `BigQueryEmulatorContainer`, a class of `org.testcontainers:gcloud` dependency.

Furthermore, I need to replace my configuration properties for my service get a connection with my fake bigquery. I can
do this using the annotation `@DynamicPropertySource` and replacing my properties values as bellow:

```java
@DynamicPropertySource
static void setProperties(DynamicPropertyRegistry registry) {
    registry.add("gcp.bigquery.project-id", bigQueryContainer::getProjectId);
    registry.add("gcp.bigquery.host", bigQueryContainer::getEmulatorHttpEndpoint);
    registry.add("gcp.bigquery.dataset", () -> "local-bigquery");
}
```

These properties are custom and mapped for me in
this [class](src%2Fmain%2Fjava%2Fdev%2Fdanielarrais%2Fbigqueryintegrationtest%2Fconfig%2FBigQueryProperties.java).

I created other properties for inject same instances tha I need and store my table name:

```java
@Autowired
private BigQueryService bigQueryService; // service

@Autowired
private BigQueryProperties bigQueryProperties; // bigquery properties

@Autowired
private BigQuery bigQuery; // instance of bigquery
```

## Create dataset for test

Now I'll need create methods to instantiate a dataset with a table required by `BigQueryService`. These methods must
be executed before each test. On method `setupDataSet()` it's called methods for clear and create the dataset, and create a table. I won't
explain other methods because it's simple:

```java
@BeforeEach
public void setupDataSet() {
    clearDataSet();
    createDataSet();
    createTable();
}

public void createDataSet() {
    var datasetInfo = DatasetInfo.newBuilder(bigQueryProperties.getDataset()).build();
    bigQuery.create(datasetInfo);
    log.info("Dataset '{}' created", bigQueryProperties.getDataset());
}

public void createTable() {
    var schema = Schema.of(
            Field.of("column_01", StandardSQLTypeName.STRING),
            Field.of("column_02", StandardSQLTypeName.STRING));
    var tableId = TableId.of(bigQueryProperties.getDataset(), tableName);
    var tableDefinition = StandardTableDefinition.of(schema);
    var tableInfo = TableInfo.newBuilder(tableId, tableDefinition).build();

    bigQuery.create(tableInfo);
    log.info("Table '{}' created", tableInfo.getTableId().getTable());
}

public void clearDataSet() {
    bigQuery.delete(getDatasetId());
    log.info("Dataset '{}' removed", bigQueryProperties.getDataset());
}

private DatasetId getDatasetId() {
    return DatasetId.of(bigQueryContainer.getProjectId(), bigQueryProperties.getDataset());
}
```

## The test

Now the requirements for create the first test for the `BigQueryService` are ready. I'll create once test for call all
methods of the service and assert the return values:

```java
@Test
public void insertItemOnBigQueryTable() throws InterruptedException {
    var dataForInsert = Map.of("column_01", "teste 03", "column_02", "teste");
    var response = bigQueryService.insertData(dataForInsert);
    var data = bigQueryService.getData();
    var tables = bigQueryService.listTables();

    Assertions.assertEquals(data.get(0), dataForInsert);
    Assertions.assertEquals(data.size(), 1);
    Assertions.assertEquals(tables.size(), 1);
    Assertions.assertTrue(CollectionUtils.isEmpty(response.getInsertErrors()));
}
```

# Conclusion

As you see it's too simple create an integration test for Google bigquery using test containers with Java. I don't
explain the details of bigquery codes because isn't complex and the focus is the integration with
containers. But you can see the project in this link and explore more details, as the connection with bigquery, the
insert/read operation, the properties mapping, my test end-to-end using the Google Cloud directly:

* [Properties mapping](src%2Fmain%2Fjava%2Fdev%2Fdanielarrais%2Fbigqueryintegrationtest%2Fconfig%2FBigQueryProperties.java)
* [Test end-to-end](src%2Ftest%2Fjava%2Fdev%2Fdanielarrais%2Fbigqueryintegrationtest%2Fservice%2FBigQueryEndToEndTest.java)
* [Bigquery connection and Bigquery instance injection](src%2Fmain%2Fjava%2Fdev%2Fdanielarrais%2Fbigqueryintegrationtest%2Fconfig%2FBigQueryConfig.java)

# Utils links

* [Testcontainers](https://testcontainers.com/)
* [Testcontainers GCloud Module](https://java.testcontainers.org/modules/gcloud/)
* [Bigquery-emulator](https://github.com/goccy/bigquery-emulator)
* [Use the Bigquery client library](https://cloud.google.com/bigquery/docs/reference/libraries#use)
* [Streaming insert (guide to insert data)](https://cloud.google.com/bigquery/docs/samples/bigquery-table-insert-rows)