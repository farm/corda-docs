+++
date = "2020-01-08T09:59:25Z"
title = "Database management"
aliases = [ "/releases/4.0/database-management.html",]
menu = [ "corda-enterprise-4-0",]
tags = [ "database", "management",]
+++



# Database management


{{< note >}}
For details on how to connect your node to a database see [here](node-database.md).

{{< /note >}}
Corda - the platform, and the installed third-party CorDapps store their data in a relational database (see
            [API: Persistence](api-persistence.md)). When Corda is first installed, or when a new CorDapp is installed, associated tables, indexes,
            foreign-keys, etc. must be created. Similarly, when Corda is upgraded, or when a new version of a CorDapp is installed,
            their database schemas may have changed, but the existing data needs to be preserved or changed accordingly.

Corda can run on most major SQL databases, so CorDapp developers need to keep this database portability
            requirement in mind when writing and testing the code. To address these concerns, Corda Enterprise provides a mechanism
            to make it straightforward to migrate from the old schemas to the new ones whilst preserving data. It does this by
            integrating a specialised database migration library. Also Corda Enterprise makes it easy to “lift” a CorDapp that does
            not handle the database migration (e.g.: the CorDapp developers did not include database migration scripts).

This document is addressed to node administrators and CorDapp developers.


* Node administrators need to understand how the manage the underlying database.


* CorDapp Developers need to understand how to write migration scripts.


“Database migrations” (or schema migrations) in this document, refers to the evolution of the database schema or the
            actual data that a Corda Node uses when new releases of Corda or CorDapps are installed. On a high level, this means
            that the Corda binaries will ship with scripts that cover everything from the creation of the schema for the initial
            install to changes on subsequent versions.

A Corda node runs on top of a database that contains internal node tables, vault tables and CorDapp tables.
            The database migration framework will handle all of these in the same way, as evolutions of schema and data.
            As a database migration framework, we use the open source library [Liquibase](http://www.liquibase.org/).


{{< note >}}
This advanced feature is only provided in Corda Enterprise.
                Whenever an upgraded version of Corda or a new version of a CorDapp is shipped that requires a different database schema to its predecessor,
                it is the responsibility of the party shipping the code (R3 in the case of Corda; the app developer in the case of a CorDapp) to also provide the migration scripts.
                Once such a change has been applied to the actual database, this fact is recorded in the database by the database migration library (see below),
                hence providing a mechanism to determine the ‘version’ of any given schema.

{{< /note >}}

{{< warning >}}
Contract state tables created by CorDapps must remain backwards compatible between releases.
                This means that they can evolve only by adding columns to them, and not by repurposing or deleting existing ones.
                The reason for this is that this part of the database schema constitutes part of the CorDapp API, and third-party systems can integrate at the database level with Corda.
                If you need to break compatibility, you have the option of creating a new version of the `MappedSchema` with new tables, but you would then have to write to both the old and the new version.

{{< /warning >}}


## About Liquibase

Liquibase is a tool that implements an automated, version based database migration framework with support for a
                large number of databases. It works by maintaining a list of applied changesets. A changeset can be something very
                simple like adding a new column to a table. It stores each executed changeset with columns like id, author, timestamp,
                description, md5 hash, etc in a table called `DATABASECHANGELOG`. This changelog table will be read every time a
                migration command is run to determine what change-sets need to be executed. It represents the “version” of the database
                (the sum of the executed change-sets at any point). Change-sets are scripts written in a supported format (xml, yml,
                sql), and should never be modified once they have been executed. Any necessary correction should be applied in a new
                change-set.

For documentation around Liquibase see: [The Official website](http://www.liquibase.org) and [Tutorial](https://www.thoughts-on-java.org/database-migration-with-liquibase-getting-started).

(Understanding how Liquibase works is highly recommended for understanding how database migrations work in Corda.)


## Integration with the Corda node

By default, a node will *not* attempt to execute database migration scripts at startup (even when a new version has been
                deployed), but will check the database “version” and halt if the database is not in sync with the node, to
                avoid data corruption. To bring the database to the correct state we provide an advanced [Database management tool](#database-management-tool-ref).

Running the migration at startup automatically can be configured by specifying true in the `database.runMigration`
                node configuration setting (default behaviour is false). We recommend node administrators to leave the default behaviour
                in production, and use the database management tool to have better control. It is safe to run at startup if you have
                implemented the usual best practices for database management (e.g. running a backup before installing a new version, etc.).


## Migration scripts structure

Corda provides migration scripts in an XML format for its internal node and vault tables. CorDapps should provide
                migration scripts for the tables they manage. In Corda, `MappedSchemas` (see [API: Persistence](api-persistence.md)) manage JPA
                Entities and thus the corresponding database tables. So `MappedSchemas` are the natural place to point to the
                changelog file(s) that contain the change-sets for those tables. Nodes can configure which `MappedSchemas` are included
                which means only the required tables are created. To follow standard best practices, our convention for structuring the
                change-logs is to have a “master” changelog file per `MappedSchema` that will only include release change-logs (see example below).

Example:

As a hypothetical scenario, let’s suppose that at some point (maybe for security reasons) the `owner` column of the
                `PersistentCashState` entity needs to be stored as a hash instead of the X500 name of the owning party.

This means, as a CorDapp developer we have to do these generic steps:


* In the `PersistentCashState` entity we need to replace


```kotlin
@Column(name = "owner_name")
var owner: AbstractParty,
```
with:

```kotlin
@Column(name = "owner_name_hash", length = MAX_HASH_HEX_SIZE)
var ownerHash: String,
```

* Add a `owner_key_hash` column to the `contract_cash_states` table. (Each JPA Entity usually defines a table name as a @Table annotation.)


* Run an update to set the `owner_key_hash` to the hash of the `owner_name`. This is needed to convert the existing data to the new (hashed) format.


* Delete the `owner_name` column


Steps 2. 3. and 4. can be expressed very easily like this:

```xml
<changeSet author="R3.Corda" id="replace owner_name with owner_hash">
    <addColumn tableName="contract_cash_states">
        <column name="owner_name_hash" type="nvarchar(130)"/>
    </addColumn>
    <update tableName="contract_cash_states">
        <column name="owner_name_hash" valueComputed="hash(owner_name)"/>
    </update>
    <dropColumn tableName="contract_cash_states" columnName="owner_name"/>
</changeSet>
```
The `PersistentCashState` entity is included in the `CashSchemaV1` schema, so based on the above mentioned convention we create a file `cash.changelog-v2.xml` with the above changeset and include in *cash.changelog-master.xml*.

```kotlin
@CordaSerializable
object CashSchemaV1 : MappedSchema(
        schemaFamily = CashSchema.javaClass, version = 1, mappedTypes = listOf(PersistentCashState::class.java)) {

    override val migrationResource = "cash.changelog-master"
```
```xml
<databaseChangeLog>
    <!--the original schema-->
    <include file="migration/cash.changelog-init.xml"/>

    <!--added now-->
    <include file="migration/cash.changelog-v2.xml"/>
</databaseChangeLog>
```
As we can see in this example, database migrations can “destroy” data, so it is therefore good practice to backup the
                database before executing the migration scripts.


## Database management tool

The database management tool is distributed as a standalone JAR file named `tools-database-manager-${corda_version}.jar`.
                It is intended to be used by Corda Enterprise node administrators.

The following sections document the available subcommands.


### Creating SQL migration scripts for CorDapps

The `create-migration-sql-for-cordapp` subcommand can be used to create migration scripts for each `MappedSchema` in
                    a CorDapp. Each `MappedSchema` in a CorDapp installed on a Corda Enterprise node requires the creation of new tables
                    in the node’s database. It is generally considered bad practice to apply changes to a production database automatically.
                    Instead, migration scripts can be generated for each schema, which can then be inspected before being applied.

Usage:

```shell
database-manager create-migration-sql-for-cordapp [-hvV] [--jar]
                                                  [--logging-level=<loggingLevel>]
                                                  -b=<baseDirectory>
                                                  [-f=<configFile>]
                                                  [<schemaClass>]
```
The `schemaClass` parameter can be optionally set to create migrations for a particular class, otherwise migration
                    schemas will be created for all classes found.

Additional options:


* `--base-directory`, `-b`: (Required) The node working directory where all the files are kept (default: `.`).


* `--config-file`, `-f`: The path to the config file. Defaults to `node.conf`.


* `--jar`: Place generated migration scripts into a jar.


* `--verbose`, `--log-to-console`, `-v`: If set, prints logging to the console as well as to a file.


* `--logging-level=<loggingLevel>`: Enable logging at this level and higher. Possible values: ERROR, WARN, INFO, DEBUG, TRACE. Default: INFO.


* `--help`, `-h`: Show this help message and exit.


* `--version`, `-V`: Print version information and exit.



### Executing SQL migration scripts

The `execute-migration` subcommand runs migration scripts on the node’s database.

Usage:

```shell
database-manager execute-migration [-hvV] [--doorman-jar-path=<doormanJarPath>]
                                   [--logging-level=<loggingLevel>]
                                   [--mode=<mode>] -b=<baseDirectory>
                                   [-f=<configFile>]
```

* `--base-directory`, `-b`: (Required) The node working directory where all the files are kept (default: `.`).


* `--config-file`, `-f`: The path to the config file. Defaults to `node.conf`.


* `--mode`: The operating mode. Possible values: NODE, DOORMAN. Default: NODE.


* `--doorman-jar-path=<doormanJarPath>`: The path to the doorman JAR.


* `--verbose`, `--log-to-console`, `-v`: If set, prints logging to the console as well as to a file.


* `--logging-level=<loggingLevel>`: Enable logging at this level and higher. Possible values: ERROR, WARN, INFO, DEBUG, TRACE. Default: INFO.


* `--help`, `-h`: Show this help message and exit.


* `--version`, `-V`: Print version information and exit.



### Executing a dry run of the SQL migration scripts

The `dry-run` subcommand can be used to output the database migration to the specified output file or to the console.
                    The output directory is the one specified by the `--base-directory` parameter.

Usage:

```shell
database-manager dry-run [-hvV] [--doorman-jar-path=<doormanJarPath>]
                         [--logging-level=<loggingLevel>] [--mode=<mode>]
                         -b=<baseDirectory> [-f=<configFile>] [<outputFile>]
```
The `outputFile` parameter can be optionally specified determine what file to output the generated SQL to, or use
                    `CONSOLE` to output to the console.

Additional options:


* `--base-directory`, `-b`: (Required) The node working directory where all the files are kept (default: `.`).


* `--config-file`, `-f`: The path to the config file. Defaults to `node.conf`.


* `--mode`: The operating mode. Possible values: NODE, DOORMAN. Default: NODE.


* `--doorman-jar-path=<doormanJarPath>`: The path to the doorman JAR.


* `--verbose`, `--log-to-console`, `-v`: If set, prints logging to the console as well as to a file.


* `--logging-level=<loggingLevel>`: Enable logging at this level and higher. Possible values: ERROR, WARN, INFO, DEBUG, TRACE. Default: INFO.


* `--help`, `-h`: Show this help message and exit.


* `--version`, `-V`: Print version information and exit.



### Releasing database locks

The `release-lock` subcommand forces the release of database locks. Sometimes, when a node or the database management
                    tool crashes while running migrations, Liquibase will not release the lock. This can happen during some long
                    database operations, or when an admin kills the process (this cannot happen during normal operation of a node,
                    only during the migration process - see: <[http://www.liquibase.org/documentation/databasechangeloglock_table.html](http://www.liquibase.org/documentation/databasechangeloglock_table.html)>)

Usage:

```shell
database-manager release-lock [-hvV] [--doorman-jar-path=<doormanJarPath>]
                              [--logging-level=<loggingLevel>] [--mode=<mode>]
                              -b=<baseDirectory> [-f=<configFile>]
```
Additional options:


* `--base-directory`, `-b`: (Required) The node working directory where all the files are kept (default: `.`).


* `--config-file`, `-f`: The path to the config file. Defaults to `node.conf`.


* `--mode`: The operating mode. Possible values: NODE, DOORMAN. Default: NODE.


* `--doorman-jar-path=<doormanJarPath>`: The path to the doorman JAR.


* `--verbose`, `--log-to-console`, `-v`: If set, prints logging to the console as well as to a file.


* `--logging-level=<loggingLevel>`: Enable logging at this level and higher. Possible values: ERROR, WARN, INFO, DEBUG, TRACE. Default: INFO.


* `--help`, `-h`: Show this help message and exit.


* `--version`, `-V`: Print version information and exit.



### Database Manager shell extensions

The `install-shell-extensions` subcommand can be used to install the `database-manager` alias and auto completion for
                    bash and zsh. See [Shell extensions for CLI Applications](cli-application-shell-extensions.md) for more info.


{{< note >}}
When running the database management tool, prefer using absolute paths when specifying the “base-directory”.

{{< /note >}}

{{< warning >}}
It is good practice for node operators to backup the database before upgrading to a new version.

{{< /warning >}}


## Examples

The first time you set up your node, you will want to create the necessary database tables. Run the normal installation
                steps. Using the database management tool, attempt a dry-run to inspect the output SQL:

```kotlin
java -jar tools-database-manager-3.0.0.jar --base-directory /path/to/node --dry-run
```
The output sql from the above command can be executed directly on the database or this command can be run:

```kotlin
java -jar tools-database-manager-3.0.0.jar --base-directory /path/to/node --execute-migration
```
At this point the node can be started successfully.

When upgrading, deploy the new version of Corda. Attempt to start the node. If there are database migrations in the new
                release, then the node will exit and will show how many changes are needed. You can then use the same commands
                as above, either to do a dry run or execute the migrations.

The same is true when installing or upgrading a CorDapp. Do a dry run, check the SQL, then trigger a migration.


## Node administrator installing a CorDapp

If a CorDapp does not include the required migration scripts for each `MappedSchema`, these can be generated and inspected before
                being applied as follows:


* Deploy the CorDapp on your node (copy the JAR into the `cordapps` folder)


* Find out the name of the `MappedSchema` containing the new contract state entities


* Call the database management tool: `java -jar tools-database-manager-${corda_version}.jar --base-directory /path/to/node --create-migration-sql-for-cordapp com.example.MyMappedSchema`.
                        This will generate a file called `my-mapped-schema.changelog-master.sql` in a folder called `migration` in the `base-directory`.
                        In case you don’t specify the actual `MappedSchema` name, the tool will generate one SQL file for each schema defined in the CorDapp


* Inspect the file(s) to make sure it is correct. This is a standard SQL file with some Liquibase metadata as comments


* Create a JAR with the `migration` folder (by convention it could be named: `originalCorDappName-migration.jar`),
                        and deploy this JAR in the node’s `cordapps` folder together with the CorDapp (e.g. run the following command in the node base directory
                        `jar cvf /path/to/node/cordapps/MyCordapp-migration.jar migration`)


* To make sure that the new migration will be used, do a dry run with the database management tool and inspect the output file



## Node administrator deploying a new version of a CorDapp developed by the OS community

This is a slightly more complicated scenario. You will have to understand the changes (if any) that happened in the latest version. If there are changes that require schema adjustments, you will have to write and test those migrations. The way to do that is to create a new changeset in the existing changelog for that CorDapp (generated as above). See  [Liquibase Sql Format](http://www.liquibase.org/documentation/sql_format.html)


## CorDapp developer developing a new CorDapp

CorDapp developers who decide to store contract state in custom entities can create migration files for the `MappedSchema` they define.

There are 2 ways of associating a migration file with a schema:


* By overriding `val migrationResource: String` and pointing to a file that needs to be in the classpath.


* By putting a file on the classpath in a `migration` package whose name is the hyphenated name of the schema (all supported file extensions will be appended to the name).


CorDapp developers can use any of the supported formats (XML, SQL, JSON, YAML) for the migration files they create. In
                case CorDapp developers distribute their CorDapps with migration files, these will be automatically applied when the
                CorDapp is deployed on a Corda Enterprise node. If they are deployed on an open source Corda node, then the
                migration will be ignored, and the database tables will be generated by Hibernate. In case CorDapp developers don’t
                distribute a CorDapp with migration files, then the organisation that decides to deploy this CordApp on a Corda
                Enterprise node has the responsibility to manage the database.

During development or demo on the default H2 database, then the CorDapp will just work when deployed even if there are
                no migration scripts, by relying on the primitive migration tool provided by Hibernate, which is not intended for
                production.


{{< warning >}}
A very important aspect to be remembered is that the CorDapp will have to work on all supported Corda databases.
                    It is the responsibility of the developers to test the migration scripts and the CorDapp against all the databases.
                    In the future we will provide additional tooling to assist with this aspect.

{{< /warning >}}

When developing a new version of an existing CorDapp, depending on the changes to the `PersistentEntities`, a
                changelog will have to be created as per the Liquibase documentation and the example above.


## Troubleshooting

When seeing problems acquiring the lock, with output like this:

```kotlin
Waiting for changelog lock....
Waiting for changelog lock....
Waiting for changelog lock....
Waiting for changelog lock....
Waiting for changelog lock....
Waiting for changelog lock....
Waiting for changelog lock....
Liquibase Update Failed: Could not acquire change log lock.  Currently locked by SomeComputer (192.168.15.X) since 2013-03-20 13:39
SEVERE 2013-03-20 16:59:liquibase: Could not acquire change log lock.  Currently locked by SomeComputer (192.168.15.X) since 2013-03-20 13:39
liquibase.exception.LockException: Could not acquire change log lock.  Currently locked by SomeComputer (192.168.15.X) since 2013-03-20 13:39
        at liquibase.lockservice.LockService.waitForLock(LockService.java:81)
        at liquibase.Liquibase.tag(Liquibase.java:507)
        at liquibase.integration.commandline.Main.doMigration(Main.java:643)
        at liquibase.integration.commandline.Main.main(Main.java:116)
```
then the advice at [this StackOverflow question](https://stackoverflow.com/questions/15528795/liquibase-lock-reasons)
                may be useful. You can run `java -jar tools-database-manager-3.0.0.jar --base-directory /path/to/node --release-lock` to force Liquibase to give up the lock.

