---
layout: post
title: 'PostgreSQL: How to use it?'
date: 2018-06-02 00:01:30 +0000
categories: [Postgresql, C/C++, Qt]
tags: [python, postgresql, qt, database, базы данных]
seo:
  date_modified: 2020-01-28 23:54:15 +0500
---

This is rather not a post, but a small cheat sheet required to work with **PostgreSQL** operations. Impossible to keep all the information in your head, so you need to do for yourself handbooks to be able to access them at any time.

----------------

## PostgreSQL installation

I work on Linux [Manjaro](http://manjaro.org/) and the installation of **PostgreSQL** was reduced to what I opened the package Manager, found **PostgreSQL** in the search engine and installed. In other distributions, you can install in a similar way.

----------------

## Working with PostgreSQL

----------------

### Initial settings

Server startup.

```bash
sudo systemctl enable --now postgresql.service
```

To view the status.

```bash
systemctl status postgresql.service
```

The superuser console.

```bash
$ sudo su - postgres
[sudo] пароль для ainur: 
[postgres@ainur-pc ~]$ psql 
psql (10.4)
Введите "help", чтобы получить справку.
```

psql — the PostgreSQL interactive terminal.

Connect to the database server.

```bash
[ainur]$ psql --username=ainur --host=localhost --dbname=settings
```

Connection information.

```bash
settings=> \conninfo
```

----------------

### Get information in PostgreSQL

List of commands for information.

```sql
postgres=> \l		/* list of all databases		*/
postgres=> \l+		/* list of all databases with add. inf. */
postgres=> \dn		/* list of scheme			*/
postgres=> \dn+		/* list of scheme with add. inf. 	*/
postgres=> \du		/* view roles				*/
postgres=> \du+		/* view roles with descriptionм		*/
postgres=> \dt		/* list of tables			*/
postgres=> \dt+		/* list of tables с описанием		*/
postgres=> \d table 	/* information about a specific table	*/
```

----------------

### Users

Creating user with some rights and password.

```sql
CREATE ROLE ainur WITH CREATEROLE CREATEDB LOGIN PASSWORD 'pass';
```

Remove user.

```sql
postgres=# DROP ROLE user;
```

----------------

#### User privilege

Give the user the privilege on the database.

```sql
postgres=# GRANT ALL privileges ON DATABASE settings TO ainur;
```

Revocation of privileges.

```sql
postgres=# REVOKE ALL privileges ON DATABASE settings FROM ainur;
```

----------------

### Why do we need a scheme and why create it?

The [database schema](https://en.wikipedia.org/wiki/Database_schema) includes descriptions of the content, structure, and integrity constraints used to create and maintain the database.

Create a schema.

```sql
postgres=# CREATE SCHEMA test;
CREATE SCHEMA
```

Delete the scheme.

```sql
postgres=# DROP SCHEMA test;
DROP SCHEMA
```

----------------

### Databases

Create a database.

```sql
postgres=# CREATE DATABASE settings;
CREATE DATABASE
```

Create a database based on another database.

```sql
CREATE DATABASE dbname TEMPLATE template0;
```

Or create a database for a specific user.

```sql
CREATE DATABASE имя_базы OWNER имя_роли;
```

Remote database.

```sql
postgres=# DROP DATABASE testdb;
DROP DATABASE
```

----------------

### Tables

A table consists of columns that are defined by type and name, and rows that contain data. At least one of the columns must have a primary key.
A primary key is a field or set of fields that uniquely identify a record. The primary key should be minimal sufficient: it should not contain fields whose removal from the primary key will not affect its uniqueness.

Create a table.

```sql
settings=> CREATE TABLE settingsInt (
id SERIAL PRIMARY KEY,
name text,
value integer
);
CREATE TABLE
```

Remove table.

```sql
settings=> DROP TABLE settingsInt;
```

Get a table.

```sql
settings=> SELECT * FROM settingsint;
 id |     name      | value 
----+---------------+-------
  1 | vol           |    10
  2 | node quantity |    10
  3 | papers        |    20
  4 | words         |    55
  5 | files         |  2435
(5 строк)
```

Get a table with order by a certain column.

```sql
settings=> SELECT * FROM settingsint ORDER BY name ;
 id |     name      | value 
----+---------------+-------
  5 | files         |  2435
  2 | node quantity |    10
  3 | papers        |    20
  1 | vol           |    10
  4 | words         |    55
(5 строк)
```

Get a specific line of table.

```sql
settings=> SELECT * FROM settingsint WHERE name = 'vol';
 id | name | value 
----+------+-------
  1 | vol  |    10
(1 строка)
```

----------------

### Inserting and deleting data and tables

Inserting data to the table.

```sql
settings=> INSERT INTO settingsint (name, value)
VALUES 
( 'vol' , 10 ), 
( 'node quantity', 10 ), 
( 'papers', 20  ), 
( 'words', 55  ), 
( 'files', 2435  );
INSERT 0 5
```

Delete rows containing name equal to **'words'** from the table.

```sql
settings=> DELETE FROM settingsint WHERE name = 'words';
DELETE 1
```

Delete all data from the table.

```sql
settings=> DELETE FROM settingsint;
```

Add an additional column named **new_column** with the type **text** and default value **'default text'**

```sql
settings=> ALTER TABLE settingsint ADD COLUMN new_colunm text DEFAULT 'default text';
ALTER TABLE
```

Delete column.

```sql
settings=> ALTER TABLE settingsint DROP COLUMN new_colunm;
ALTER TABLE
```

Change the column of a specific row.

```sql
settings=> UPDATE settingsint SET value = 200 WHERE name = 'vol';
UPDATE 1
```

----------------

### Links with potentially useful information

1. [SQL commands - reference guide](https://postgrespro.ru/docs/postgresql/10/sql-commands)
2. [Data types in PostgreSQL](https://postgrespro.ru/docs/postgresql/10/datatype)
3. [Another SQL command reference](https://www.w3schools.com/sql/default.asp)

----------------

## PostgreSQL and Python

----------------

### As their friends?

For work with **PostgreSQL** from **Python** I found the library [psycopg2](http://initd.org/psycopg/docs/install.html). 

First of all, you need to import the library.

```python
import psycopg2
```

Then you need to connect to the database. This makes the **connect** method where you must specify the required parameters:

* dbname
* user
* password
* host

At the end of the program you need to close the connection.

```python
>> # Connect to the database
>> conn = psycopg2.connect("dbname=settings \
                            user=ainur      \
                            password=pass   \
                            host=localhost")
>> # Doing something with the database...
>> # Close the connection
>> conn.close()
```

To execute SQL commands, you need to create a so-called cursor, which also needs to be closed after executing the necessary commands.

```python
>> cur = conn.cursor()
>> # Doing something with the database...
>> cur.close()
```

To execute SQL commands, you need to call the **execute** method, which takes the necessary commands into arguments. To get results or data, use methods:

* fetchall 
* fetchone
* fetchmany(n)

For example, get the whole table.

```python
>> cur.execute('SELECT * FROM settingsint;')
>> data = cur.fetchall()
>> print(data)
[
	(2, 'node quantity', 10), 
	(3, 'papers', 20), 
	(5, 'files', 2435), 
	(1, 'vol', 200)
]
```

The data output is in the format of an array, which can be parsed by accessing the index and get the individual elements.

For example, get value **value** of line **'vol'**.

```python
>> cur.execute("SELECT value FROM settingsint WHERE name = %s;", ('vol',))
>> print(cur.statusmessage)	# get execution status
>> data = cur.fetchone()
>> print(data[0])
SELECT 1
200
```

Add data to the table.

```python
>> cur.execute("INSERT INTO settingsint (name, value)  \
                VALUES                                 \
                ( 'name1' , 42 ),                      \
                ( 'name2' , 56 )")
>> print(cur.statusmessage)
>> conn.commit()	# commit changes
INSERT 0 2
```

After making changes, you must commit them by calling the method **conn.commit()**, otherwise, the data will not be saved.

----------------

## PostgreSQL and C++

----------------

### As their friends?

----------------

#### ODB C++

As far as I know, the application of [ODB](https://www.codesynthesis.com/products/odb/) - this is a correct way to work with the database through **C++**. He allows you to use the database as an object. On the official website there is all information on installation and work with ODB. Unfortunately, at the moment I have not managed to install it properly on my Manjaro Linux, so maybe I will write about it later.

----------------

#### libpqxx

[libpqxx](http://pqxx.org/development/libpqxx/) - an [open source library](https://github.com/jtv/libpqxx) for working with PostgreSQL that provides methods for executing SQL queries.

It is installed simply:

```bash
$ sudo pacman -S libpqxx
```

To work enough to connect the library **<pqxx/pqxx>**.

Well, give an example to display the entire table.

```cpp
#include <iostream>
#include <string>
#include <pqxx/pqxx>

int main()
{
    try
    {
        pqxx::connection c("dbname=settings user=ainur host=localhost");
        if(c.is_open())
        {
            std::cout << "Opened database successfully: " 
		      << c.dbname() << std::endl;
        }
        else
        {
            std::cout << "Can't open database" << std::endl;
            return 1;
        }
        std::string SQL = "SELECT * FROM settingsint;";
        pqxx::nontransaction n(c);
        r = n.exec(SQL);
        std::cout << r.column_name(0) << "\t"
                  << r.column_name(1) << "\t"
                  << r.column_name(2) << "\t"
                  << std::endl;
        for(pqxx::result::const_iterator i = r.begin(); i != r.end(); i++)
        {
            std::cout << i.at(0) << "\t"
                      << i.at(1) << "\t"
                      << i.at(2) << std::endl;
        }

        c.disconnect();
    } 
    catch (const std::exception &e) 
    {
        std::cerr << e.what() << std::endl;
        return 1;
    }
}
```

The flags used for compilation.

```bash
$ g++ main.cpp -o main -lpqxx -lpq
```

The output will be something like this.

```sh
$ ./main 
id      name    value
3       papers  20
5       files   2435
1       vol     200
12      name1   42
13      name2   56
14      name1   42
15      name2   56
16      name1   42
17      name2   56
2       node    10
```

It is worth noting that if the database is not changed, it is better to use **pqxx::nontransaction**.
If you need to insert data into the table, use **pqxx:: work**.
All work must be done in the **try** block to catch the exception and quickly fix the problem.
The following example shows how to insert additional rows into a table.

```cpp
#include <iostream>
#include <string>
#include <pqxx/pqxx>

int main()
{
    try
    {
        pqxx::connection c("dbname=settings user=ainur host=localhost");
        if(c.is_open())
        {
            std::cout << "Opened database successfully: " 
		      << c.dbname() << std::endl;
        }
        else
        {
            std::cout << "Can't open database" << std::endl;
            return 1;
        }
        pqxx::work w(c);
        std::string SQL =  "INSERT INTO settingsint (name, value) \
                            VALUES                                \
                            ( 'c++', 100 );";
        pqxx::result r = w.exec(SQL);        
        w.commit();
        c.disconnect();
    } 
    catch (const std::exception &e) 
    {
        std::cerr << e.what() << std::endl;
        return 1;
    }
}
```

This should be enough for the job. All the queries necessary for working with the database should be formed in **std::string SQL**.

-----------------

#### QtSql

Qt provides its own library for working with databases, which is called [QtSql](http://doc.qt.io/qt-5/sql-programming.html).

To work with it, first of all, you need to add a line to the **pro** file:

```bash
QT += sql
```

And connect the **QtSql** library.

```cpp
#include <QtSql>
```

The essence of the work is about the same as in the previous examples - send SQL query, get the execution status and response.

```cpp
#include <QCoreApplication>
#include <QDebug>
#include <QtSql>

int main(int argc, char *argv[])
{
    Q_UNUSED(argc);
    Q_UNUSED(argv);
    
    // Connect to database server
    QSqlDatabase db = QSqlDatabase::addDatabase("QPSQL");
    db.setHostName("localhost");
    db.setDatabaseName("settings");
    db.setUserName("ainur");
    db.setPassword("pass");

    if(!db.open())  qDebug() << "Unable to open database";
    
    bool res = 0;

    QSqlQuery query;
    
    // Create a request and send it
    res = query.exec("SELECT * FROM settingsint ORDER BY id");
    
    // Check the status
    if(!res)    qDebug() << "sql response error";
    
    // Output the result
    while(query.next())
    {
        qDebug() << query.value(0).toInt() << "\t"
                 << query.value(1).toString() << "\t"
                 << query.value(2).toInt();
    }
}
```

For queries with data changes, you can simply make a SQL query with the required data.

```cpp 
QSqlQuery query;
res = query.exec("INSERT INTO settingsint (name, value) "
                 "VALUES ('qt_data', 100)");        
if(!res)    qDebug() << "sql response error";
```

It is also possible to first prepare the request form using the **prepare** method and, if necessary, set the values for the data by the **bindValue** method.

```cpp
QSqlQuery query;
query.prepare("INSERT INTO settingsint (name, value) "
              "VALUES (:name, :value)");
query.bindValue(":name", "qt_data");
query.bindValue(":value", 200);
res = query.exec();
if(!res)    qDebug() << "sql response error";
```


