# PGAdapter and Hibernate

PGAdapter can be used in combination with Hibernate, but with a number of limitations. This sample
shows the command line arguments and configuration that is needed in order to use Hibernate with
PGAdapter.

## Start PGAdapter
You must start PGAdapter before you can run the sample. The following command shows how to start PGAdapter using the
pre-built Docker image. See [Running PGAdapter](../../../README.md#usage) for more information on other options for how
to run PGAdapter.

```shell
export GOOGLE_APPLICATION_CREDENTIALS=/path/to/credentials.json
docker pull gcr.io/cloud-spanner-pg-adapter/pgadapter
docker run \
  -d -p 5432:5432 \
  -v ${GOOGLE_APPLICATION_CREDENTIALS}:${GOOGLE_APPLICATION_CREDENTIALS}:ro \
  -e GOOGLE_APPLICATION_CREDENTIALS \
  -v /tmp:/tmp \
  gcr.io/cloud-spanner-pg-adapter/pgadapter \
  -p my-project -i my-instance \
  -x
```

## Creating the Sample Data Model
Run the following command in this directory. Replace the host, port and database name with the actual host, port and
database name for your PGAdapter and database setup.

```shell
psql -h localhost -p 5432 -d my-database -f sample-schema.sql
```

You can also drop an existing data model using the `drop_data_model.sql` script:

```shell
psql -h localhost -p 5432 -d my-database -f drop-data-model.sql
```

## Data Types
Cloud Spanner supports the following data types in combination with `Hibernate`.

| PostgreSQL Type                        | Hibernate or Java |
|----------------------------------------|-------------------|
| boolean                                | boolean           |
| bigint / int8                          | long              |
| varchar                                | String            |
| text                                   | String            |
| float8 / double precision              | double            |
| numeric                                | BigDecimal        |
| timestamptz / timestamp with time zone | LocalDateTime     |
| bytea                                  | byte[]            |
| date                                   | LocalDate         |
| jsonb                                  | Custom Data Type  |

### JSONB
Hibernate 5.* does not have native support for JSONB. For this, a custom data type
must be added. This sample uses [vladmihalcea/hibernate-types](https://github.com/vladmihalcea/hibernate-types).

## Limitations
The following limitations are currently known:

| Limitation             | Workaround                                                                                                                                                                                                                                                                   |
|------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Hibernate Version      | Version 5.3.20.Final is supported.                                                                                                                                                                                                                                           |
| Postgresql JDBC driver | Version 42.4.3 is supported.                                                                                                                                                                                                                                                 |
| Schema updates         | Cloud Spanner does not support the full PostgreSQL DDL dialect. Automated schema updates using `hibernate` are therefore not supported. It is recommended to set the option `hibernate.hbm2ddl.auto=none` (or `spring.jpa.hibernate.ddl-auto=none` if you are using Spring). |
| Generated primary keys | Cloud Spanner does not support `sequences`. Auto-increment primary key is not supported. Remove auto increment annotation for primary key columns. The recommended type of primary key is a client side generated `UUID` stored as a string.                                 |
| Pessimistic Locking    | Cloud Spanner does not support `LockMode.UPGRADE` and `LockMode.UPGRADE_NOWAIT` lock modes.                                                                                                                                                                                  |


### Schema Updates
Schema updates are not supported as Cloud Spanner does not support the full PostgreSQL DDL dialect. It is recommended to
create the schema manually. Note that PGAdapter does support `create table if not exists` / `drop table if exists`.
See [sample-schema.sql](src/main/resources/sample-schema-sql) for the data model for this example.

### Generated Primary Keys
`Sequences` are not supported. Hence, auto increment primary key is not supported and should be replaced with primary key definitions that
are manually assigned. See https://cloud.google.com/spanner/docs/schema-design#primary-key-prevent-hotspots
for more information on choosing a good primary key. This sample uses UUIDs that are generated by the client for primary
keys.

```java
public class User {
	// This uses auto generated UUID.
    @Id
    @Column(columnDefinition = "varchar", nullable = false)
    @GeneratedValue
    private UUID id;
}
```