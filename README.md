# BaltimoreGolang Workshop: Interacting with Databases

## Introduction
Many of the applications we build rely on storing and retrieving data. One of the most common ways of working with data when web applications is by using relational databases and the Structured Query Language (SQL). 

During this workshop, we will look at Go's abstraction of database interaction through the [`database/sql`](https://golang.org/pkg/database/sql/) package and use driver implementations for working with data in [MySQL](https://dev.mysql.com/downloads/) as one of the most popular relational databases.

NoSQL databases have also become very popular for certain use cases. Using [MongoDB](https://www.mongodb.com/), we'll dive into document-oriented storage with Go.

Lastly, we'll eschew database servers altogether and use the popular [BoltDB](https://github.com/boltdb/bolt) package in Go to store and retrieve data using strictly Go types (no tables, no sql, etc).

## Workshop Logistics
This workshop is about hands-on development with Go. Each section will be preceded by a brief introduction of the material and some examples by the instructor followed by you working through the exercises tailored for each section.

### What you'll need:
- [Install Go](https://golang.org/dl/) and check your $GOPATH
- [Install MySQL](https://dev.mysql.com/downloads/)
-  [Install MySQL Workbench](https://dev.mysql.com/downloads/workbench/) (Optional)
- [Install MongoDB](https://www.mongodb.com/download-center#community)

If this is your first time and/or need assistance with setting up any of the above, raise your hand and the instructor (or any other knowledgable student) will give you a hand.

> If you're on a Mac, you may find `brew` to be an easy way to go about pulling down and installing the software listed above (i.e. `brew install go`, `brew install mysql`, `brew install mongodb`).

As is typical of the BaltimoreGolang Workshop Series, we'll be using the [Go Proverbs](https://go-proverbs.github.io/) as our source of data to work with.

When working through the exercises, you have your choice  of exposing the functionality through a CLI tool or through a web server running locally that you can issue HTTP requests to. 

> Should you find any mistakes or errors, submit an issue or better yet, open a pull request with the fix.

Lastly, it's important that you see this as an opportunity to make mistake, ask "stupid" questions and  even collaborate with fellow students to share tips for working through the problems. 

Let's begin!

## The `database/sql` package
The way to work with SQL-like database servers in Go is to use the `database/sql` package. It provides a generic interface that database driver implementers use to expose access to storage and querying functionality.

Let's look at an example:

```
import (
	"database/sql"
	_ "github.com/go-sql-driver/mysql"
)

func main() {
	// get a database object
	db, err := sql.Open("mysql", "user:password@tcp(127.0.0.1:3306)/db")
	check(err)
	defer db.Close()
	
	// check that you can establish a connection
	err = db.Ping()
	if err != nil {
		// do something here
	}
	
	// execute a query	
	rows, err := db.Query("select proverb from proverbs where id = ?", 1)
	check(err)
	defer rows.Close()
	
	var id int
	var	proverb string
	
	// iterate on the results
	for rows.Next() {
		err := rows.Scan(&id, &proverb)
		check(err)
		log.Println(id, proverb)
	}
	err = rows.Err()
	check(err)
}

func check(err error) {
	if err != nil {
		log.Fatal(err)
	}
}
```

There's a lot going on here so let's break it down.

### Import the driver

```
import (
	"database/sql"
	_ "github.com/go-sql-driver/mysql"
)
```

We not only need the `database/sql` package, but we must also import the actual database driver package that implements the interface exposed by the `database/sql` package. In this case, we're using the `github.com/go-sql-driver/mysql` package to talk to a MySQL database server.

> Use "go get github.com/go-sql-driver/mysql" to install the package before using it in your code.

Note that we import it anonymously using an underscore (`_`) package qualifier. This prevents the driver's exported variables and functionality from being exposed to our code--a good thing. Once imported, the driver registers itself as a "mysql" driver with the `database/sql` package.

> Avoid using database drivers directly in your code less you risk tightly coupling your application to that particular implementation of sql database interaction. Always rely on Go's abstraction instead.

### Connecting to the database
Inside of our `main` function...

```
func main(){
	// get a database object
	db, err := sql.Open("mysql", "user:password@tcp(127.0.0.1:3306)/db")
	check(err)
	defer db.Close()
	
	// check that you can establish a connection
	err = db.Ping()
	if err != nil {
		// do something here
	}
	...
}
```

Now that we have a "mysql" driver registered with `databsae/sql`, we can use the `Open` function to create  a database object (not establish a connection, yet).

> func Open(driverName, dataSourceName string) (*DB, error)

Note the `driverName` parameter which should match the string that was registered by the `github.com/go-sql-driver/mysql` driver we imported, in this case `mysql`. The `dataSourceName` param will typically be driver-specific and you should consult the driver's documentation for a proper connection string. In this case, it takes the form of `user:password@host:port/database`.

We use `defer db.Close()` to ensure the `*sql.DB` object returned does not live beyond the scope of the `main` function.

> func (db *DB) Close() error

It's recommended that we ensure we can actually establish a connection to the underlying database server for our `*sql.DB` object and we can do that by using the `Ping` method which establishes a connection if one is not already active. 

> func (db *DB) Ping() error

If the `error` returned is not nil, we know we have some troubleshooting to do.

### Fetch some data

Now that we've got an active connection to a MySQL server, let's retrieve some rows.

```
func main() {
	...
	// execute a query
	rows, err := db.Query("select proverb from proverbs where id = ?", 1)
	check(err)
	defer rows.Close()
	...
}
```

The `Query` method on `*sql.DB` is what allows us to send SQL to the database server and get back `*sql.Rows`. 

> func (db *DB) Query(query string, args ...interface{}) (*Rows, error)

`query` is for our SQL statement and `args` is for any placeholder params in the query string as was the case with our statement above.

### Work with the results

Now that we finally have records to work with, we need to iterate over them in order to get to each row's fields:

```
func main() {
	...
	// iterate on the results
	for rows.Next() {
		err := rows.Scan(&id, &proverb)
		check(err)
		log.Println(id, proverb)
	}
	err = rows.Err()
	...
}
```

Using the `Next` method on the value of type `*sql.Rows` allows us to go row by row until there are no more records. Note that `*sql.Rows.Next()` returns a boolean which will be true up until there are no more rows, so that works for our `for` loop here.

Next, inside each iteration, we use the `*sql.Rows.Scan` method to assign each column in our row to references of variable we declared earlier. Yes, this is the standard way of doing it, particularly because of the strong typing of the language.

> There are community packages that make dealing with some of that necessary boiler-plate easier, the https://github.com/jmoiron/sqlx for example, is a favorite.

From here, you can do anything you wish with the values.

### Inserting data

To mutate data, we'll rely on the `Exec` method of `sql.DB` to execute a SQL statement and get back a response that you can introspect to find out what happened. 

> func (db *DB) Exec(query string, args ...interface{}) (Result, error)

Note that unlike `sql.DB.Query`, `sql.DB.Exec` does not return `sql.Rows`, it returns a `sql.Result` defined as:

```
type Result interface {
	LastInsertId() (int64, error)
	RowsAffected() (int64, error)
}
```

You can use that value to find out the last inserted ID and the number of rows affected by the statement as their names imply.

Insert Example:

```
stmt, err := db.Prepare("INSERT INTO myTable(myColum) VALUES(?)")
check(err)

res, err := stmt.Exec("myData")
check(err)

lastId, err := res.LastInsertId()
check(err)

rowCnt, err := res.RowsAffected()
check(err)

log.Printf("Last Inserted ID: %d, Rows Affected: %d\n", lastId, rowCnt)
```

Note the use of a `sql.DB.Prepare` here. 

> func (db *DB) Prepare(query string) (*Stmt, error)

We use that instead of a raw string passed into `Exec` to both optimize the performance of our execution and to prevent SQL injection attacks in the cases where we're inserting user input.

> Avoid a "Little Bobby Tables" moment - https://xkcd.com/327

Updates and deletes operate in a very similar way and experimentation is left to the reader. On to transactions.

### Working with Transactions

Transactions guarantee that all statements within succeed or none at all. Meaning that if you have a transaction with two insert statements, if one should fail, the other will be rolled back.

Go's `database/sql` package supports transactions using `sql.DB.Begin`, `sql.Tx.Commit` and `sql.Tx.Rollback` methods.

> func (db *DB) Begin() (*Tx, error)

> func (tx *Tx) Commit() error

> func (tx *Tx) Rollback() error

Making use of transactions in Go can be a little daunting at first but make sense with a little practice. 

Let's look at an example:

```
func transMutate() error {
	tx, err := db.Begin()
	if err != nil {
		return err
	}

	// defer the rollback so we don't have to repeat
	// it throughout the function
	defer func() {
		if err != nil {
	    tx.Rollback()
	    return err
    }
		err = tx.Commit()
	}()
	
	// prepare a statement using the transaction
	stmt, err := tx.Prepare("INSERT INTO myTable VALUES (?)")
	if err != nil {
		return err
	}
	
	// execute the statement multiple times
	// under the same transaction
	for _, v := range []int{1, 2, 4} {
		res, err := stmt.Exec(v)
		if err != nil {
			return err
		}
	}

	return err
}
```

Onto exercises!

### Exercise: Interacting with MySQL

In this two-part exercise, your task is to create a tool that populates a local MySQL database with the Go Proverbs found in the `data/proverbs.csv` file of this repo and that subsequently allows us to query for proverbs.

Note that the CSV has two columns, `tags` and `proverb` where by the tags represent metadata on the proverb that will be used in the second part of this exercise. 

#### Part 1
Start your local MySQL server, create a database and a table with two columns (`tags`, `proverb`). Use the GUI or command line for this.

Create an executable that you can invoke from the command line like so:

```
$ ./proverbs import ./data/proverbs.csv
```

Extra Credit: You don't **need** a transaction to populate your database table but using one will give you practice so why not? Import the proverbs using a transaction and a prepared statement for your INSERTs.

#### Part 2
Extend your executable to allow listing and searching based on the proverb itself and on the tags associated with the proverb.

Example: List all proverbs

```
$ ./proverbs all
```

Example: Find proverbs that contain the word "cgo" regardless of case:

```
$ ./proverbs contains "cgo" 
```


Example: Find proverbs tagged with "error":

```
$ ./proverbs tagged "error"
```

### Learning Resources
- https://golang.org/pkg/database/sql
- http://go-database-sql.org
- https://github.com/jmoiron/sqlx

## Interacting with MongoDB

MongoDB, in case you're not already familiar with it, is a document-oriented NoSQL database technology. It allows you to model your data as a document comprised of key-value pairs, similar to JSON objects.

For example, to model our proverbs data structure, we'd need a proverb and tags field as the CSV suggests:

```
{
	proverb: "Cgo is not Go.",
	tags: ["cgo"]
}
```

To learn more about MongoDB itself, I suggest you head over to the docs: [https://docs.mongodb.com/manual/introduction](https://docs.mongodb.com/manual/introduction/).

To interact with a MongoDB server, we'll use the popular `gopkg.in/mgo.v2` package. To install it, use go get:

```
$ go get gopkg.in/mgo.v2
```

Here's a complete example using `mgo`:

```
package main

import (
	"fmt"
	"log"
	"gopkg.in/mgo.v2"
	"gopkg.in/mgo.v2/bson"
)

type Car struct {
	Make string
	Model string
}

func main() {
	session, err := mgo.Dial("server1.example.com,server2.example.com")
	check(err)
	defer session.Close()

	// add data
	c := session.DB("myDB").C("cars")
	err = c.Insert(&Car{"Chevrolet", "Vega"}, &Car{"Ford", "Pinto"})
	check(err)

	// retrieve data
	result := Car{}
	err = c.Find(bson.M{"make": "Ford"}).One(&result)
	check(err)
	fmt.Println("Car:", result.Model)
}
```

### Exercise: Interacting with MongoDB

In this two-part exercise, your task is to create a tool that populates a local MongoDB database with the Go Proverbs found in the `data/proverbs.csv` file of this repo and that subsequently allows us to query for proverbs.

Note that the CSV has two columns, `tags` and `proverb` where by the tags represent metadata on the proverb that will be used in the second part of this exercise. 

#### Part 1
Start your local MongoDB server, create a database in which to store your collections. You can do all this throughout command line client that ships with MongoDB. Check the MongoDB documentation.

Create an executable that you can invoke from the command line like so:

```
$ ./proverbs import ./data/proverbs.csv
```

#### Part 2
Extend your executable to allow listing and searching based on the proverb itself and on the tags associated with the proverb.

Example: List all proverbs

```
$ ./proverbs all
```

Example: Find proverbs that contain the word "cgo" regardless of case:

```
$ ./proverbs contains "cgo" 
```


Example: Find proverbs tagged with "error":

```
$ ./proverbs tagged "error"
```


### Learning Resources
- https://docs.mongodb.com/manual/introduction
- https://labix.org/mgo

## Interacting with BoltDB

In the last portion of our workshop, we'll use BoltDB, a pure go key/value store, to save and retrieve data without the need for a full database server such as MySQL and MongoDB.

To install BoltDB: 

```
$ go get github.com/boltdb/bolt/...
```

The project's README does an excellent job of introducing BoltDB's functionality so I will avoid repeating that here. Head on over to [https://github.com/boltdb/bolt](https://github.com/boltdb/bolt) and come back for this exercise.

### Exercise: Interacting with BoltDB

In this two-part exercise, your task is to create a tool that populates a BoltDB database with the Go Proverbs found in the `data/proverbs.csv` file of this repo and that subsequently allows us to query for proverbs.

Note that the CSV has two columns, `tags` and `proverb` where by the tags represent metadata on the proverb that will be used in the second part of this exercise. 

#### Part 1

Create an executable that you can invoke from the command line to import the proverbs like so:

```
$ ./proverbs import ./data/proverbs.csv
```

#### Part 2
Extend your executable to allow listing and searching based on the proverb itself and on the tags associated with the proverb.

Example: List all proverbs

```
$ ./proverbs all
```

Example: Find proverbs that contain the word "cgo" regardless of case:

```
$ ./proverbs contains "cgo" 
```


Example: Find proverbs tagged with "error":

```
$ ./proverbs tagged "error"
```


### Learning Resources
- https://github.com/boltdb/bolt