Change description on github to "Generates typesafe kotlin APIs from SQL"

SQLDelight
==========

SQLDelight generates typesafe APIs from your SQL statements. It compile-time verifies your schema, statements, and migrations and provides IDE features like autocomplete and refactoring which make writing and maintaining SQL simple. SQLDelight currently supports the SQLite dialect and there are supported SQLite drivers on Android, JVM and iOS.

Example
-------

To use SQLDelight, put your SQL statements in a `.sq` file, like
`src/main/sqldelight/com/example/HockeyPlayer.sq`. Typically the first statement creates a table.

```sql
CREATE TABLE hockeyPlayer (
  player_number INTEGER NOT NULL,
  name TEXT NOT NULL
);

CREATE INDEX playerName ON hockeyPlayer(name);

INSERT INTO hockeyPlayer (player_number, name)
VALUES (15, 'Ryan Getzlaf');
```

From this SQLDelight will generate a `QueryWrapper` kotlin class with an associated `Schema` object that can be used to create your database and run your statements on it.

#### Android
```kotlin
val database: SqlDatabase = AndroidSqlDatabase(QueryWrapper.Schema, context, "test.db")
```

#### iOS (Using kotlin native)
```kotlin
val database: SqlDatabase = NativeSqlDatabase(QueryWrapper.Schema, "test.db")
```

#### JVM
```kotlin
val database: SqlDatabase = SqliteJdbcOpenHelper().apply {
  QueryWrapper.Schema.create(getConnection())
}
```

SQL statements inside a `.sq` file can be labeled to have a typesafe function generated for them available at runtime.

```sql
selectAll:
SELECT *
FROM hockeyPlayer;

insert:
INSERT INTO hockeyPlayer(player_number, name)
VALUES (?, ?);
```

Files with labeled statements in them will have a Queries file generated from them that matches the `.sq` file name - `Player.sq` generates `PlayerQueries.kt`. To get a reference to `PlayerQueries` you need to wrap the database we made above:

```kotlin
// In reality the query wrapper and database above should be created a single time and passed around using your favourite dependency injection/service locator/singleton pattern.
val queryWrapper = QueryWrapper(database)

val playerQueries: PlayerQueries = queryWrapper.playerQueries

println(playerQueries.selectAll().executeAsList())
// Prints [HockeyPlayer.Impl(15, "Ryan Getzlaf")]

playerQueries.insert(player_number = 10, name = "Corey Perry")
println(playerQueries.selectAll().executeAsList())
// Prints [HockeyPlayer.Impl(15, "Ryan Getzlaf"), HockeyPlayer.Impl(10, "Corey Perry")]
```

Custom Types
------------

By default queries will return a data class implementation of the table schema. To override this behavior pass a custom mapper to the query function. Your custom mapper will receive typesafe parameters which are the projection of your select statement.

```kotlin
println(playerQueries.selectAll(mapper = { player_number, name -> name }).executeAsList()
// Prints ["Ryan Getzlaf", "Corey Perry"]
```

In general you should be leveraging SQL to do custom projections whenever possible.

```sql
selectNames:
SELECT name
FROM hockeyPlayer;
```

```kotlin
println(playerQueries.selectNames().executeAsList())
// Prints ["Ryan Getzlaf", "Corey Perry"]
```

But the custom mapping is there when this isn't possible, for example getting a `Parcelable` type for query results on Android.

Query Arguments
---------------

.sq files use the exact same syntax as SQLite, including [SQLite Bind Args](https://www.sqlite.org/c3ref/bind_blob.html). If a statement contains bind args, the associated method will require corresponding arguments.

```sql
selectByNumber:
SELECT *
FROM hockeyPlayer
WHERE player_number = ?;
```

```kotlin
println(playerQueries.selectByNumber(player_number = 10).executeAsOne())
// Prints "Corey Perry"
```

Sets of values can also be passed as an argument.

```sql
selectByNames:
SELECT *
FROM hockeyPlayer
WHERE name IN ?;
```

```kotlin
playerQueries.selectByNames(listOf("Alec", "Jake", "Matt"))
```

Named parameters or indexed parameters can be used.

```sql
firstOrLastName:
SELECT *
FROM hockeyPlayer
WHERE name LIKE '%' || ?1
OR name LIKE ?1 || '%';
```

```java
playerQueries.firstOrLastName(name = "Perry")
```

Types
-----

SQLDelight column definition are identical to regular SQLite column definitions but support an extra column constraint
which specifies the kotlin type of the column in the generated interface. SQLDelight natively supports most jvm primitives.

```sql
CREATE TABLE some_types (
  some_long INTEGER,           -- Stored as INTEGER in db, retrieved as Long
  some_double REAL,            -- Stored as REAL in db, retrieved as Double
  some_string TEXT,            -- Stored as TEXT in db, retrieved as String
  some_blob BLOB,              -- Stored as BLOB in db, retrieved as ByteArray
  some_int INTEGER AS Int,     -- Stored as INTEGER in db, retrieved as Int
  some_short INTEGER AS Short, -- Stored as INTEGER in db, retrieved as Short
  some_float REAL AS Float     -- Stored as REAL in db, retrieved as Float
);
```

Booleans
--------

SQLDelight supports boolean columns and stores them in the db as ints. Since they are implemented
as ints they can be given int column constraints:

```sql
CREATE TABLE hockey_player (
  injured INTEGER AS Boolean DEFAULT 0
)
```

Custom Classes
--------------

If you'd like to retrieve columns as custom types you can specify a kotlin type:

```sql
import kotlin.collections.List;

CREATE TABLE hockeyPlayer (
  cupWins TEXT AS List<String> NOT NULL
);
```

However, creating the `QueryWrapper` will require you to provide a `ColumnAdapter` which knows how
to map between the database type and your custom type:

```kotlin
val listOfStringsAdapter = object : ColumnAdapter<List<String>, String> {
  override fun decode(databaseValue: String) = databaseValue.split(",")
  override fun encode(value: List<String>) = value.joinToString(separator = ",")
}

val queryWrapper: QueryWrapper = QueryWrapper(
  database = database,
  hockeyPlayerAdapter = hockeyPlayer.Adapter(
    cupWinsAdapter = listOfStringsAdapter
  )
)
```

Enums
-----

As a convenience the SQLDelight runtime includes a `ColumnAdapter` for storing an enum as TEXT.

```sql
import com.example.hockey.HockeyPlayer;

CREATE TABLE hockeyPlayer (
  position TEXT AS HockeyPlayer.Position
)
```

```kotlin
val queryWrapper: QueryWrapper = QueryWrapper(
  database = database,
  hockeyPlayerAdapter = hockeyPlayer.Adapter(
    positionAdapter = EnumColumnAdapter()
  )
)
```

RxJava
------

To observe a query depend on the rxjava-extensions artifact and use the extension method it provides:

```groovy
dependencies {
  implementation "com.squareup.sqldelight:rxjava2-extensions:1.0.0
}
```

```kotlin
val players: Observable<List<HockeyPlayer>> = 
  playerQueries.selectAll()
    .asObservable()
    .mapToList()
```

MultiPlatform
-------------

To use SQLDelight in kotlin multiplatform configure the gradle plugin with a package to place the generated code in.

```groovy
apply plugin: "com.squareup.sqldelight"

sqldelight {
  packageName = "com.example.hockey"
}
```

Continue to place your `.sq` files in the `src/main/kotlin` directory, and then `expect` a `SqlDatabase` 

Supported Dialects
------------------

#### SQLite
Full support of dialect including views, triggers, indexes, fts tables, etc. If features are missing please file an issue!

IntelliJ Plugin
---------------

The Intellij plugin provides language-level features for `.sq` files, including:
 - Syntax highlighting
 - Refactoring/Find usages
 - Code autocompletion
 - Generate `Queries` files after edits
 - Right click to copy as valid SQLite
 - Compiler errors in IDE click through to file

Download
--------

For the Gradle plugin:

```groovy
buildscript {
  repositories {
    mavenCentral()
  }
  dependencies {
    classpath 'com.squareup.sqldelight:gradle-plugin:1.0.0'
  }
}

apply plugin: 'com.squareup.sqldelight'
```

The Intellij plugin can be installed from Android Studio by navigating<br>
Android Studio -> Preferences -> Plugins -> Browse repositories -> Search for SQLDelight

Snapshots of the development version (including the IDE plugin zip) are available in
[Sonatype's `snapshots` repository](https://oss.sonatype.org/content/repositories/snapshots/).


License
=======

    Copyright 2016 Square, Inc.

    Licensed under the Apache License, Version 2.0 (the "License");
    you may not use this file except in compliance with the License.
    You may obtain a copy of the License at

       http://www.apache.org/licenses/LICENSE-2.0

    Unless required by applicable law or agreed to in writing, software
    distributed under the License is distributed on an "AS IS" BASIS,
    WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
    See the License for the specific language governing permissions and
    limitations under the License.
