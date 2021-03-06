---
layout: post
title: "Want to Work With Databases In Golang?  Let's Try Some gorp."
date: 2013-11-04 18:58
comments: true
categories: [golang,gorp,MySQL,ORM,database,go]
---

# Google's Go

[Go](http://golang.org/) is a new programming language released by [Google](http://www.google.com).  It has an excellent pedigree (see [Rob Pike](http://en.wikipedia.org/wiki/Rob_Pike) and [Ken Thompson](http://en.wikipedia.org/wiki/Ken_Thompson_%28computer_programmer%29)) and it brings a lot of interesting things to the table as a programming tool. Go has been the subject of rave reviews as well as controversy.  As Google is a web company it's no surprise that Go seems hard-wired from the start to be used in the context of the modern web and the standard libaries include everything from [HTTP servers](http://golang.org/pkg/net/http/) to [a templating system](http://golang.org/pkg/html/template/) to address these ends.  A lot of companies and hobbyist hackers seem to enjoy Go as a utility language that replaces components which used to be written in Python or Perl (with Go offering better performance).  


Its supporters emphasize its [performance](http://benchmarksgame.alioth.debian.org/u64q/benchmark.php?test=all&lang=go&lang2=yarv&data=u64q), nifty approach to concurrency (it's [built right in](http://golang.org/doc/effective_go.html#concurrency)), and fast compile times as advantages.  Some of its detractors dislike its lack of exceptions and generics, but the purpose of this article is not to address these concerns, which have already been discussed *ad nauseum*.  Instead, this article will talk about and examine the `gorp` library.  

{% img /images/gorp/gorp.jpg Eh? %}

I don't actually mean GOOD OLD RAISINS & PEANUTS, of course- I mean [gorp](https://github.com/coopernurse/gorp), an "ORM-ish library for Go".  What is it, and how does it work its funny magic?

# ORM-ish?

The README.md from `gorp`'s repository is just too great an introduction to not quote, check it out:

<blockquote>
I hesitate to call gorp an ORM. Go doesn't really have objects, at least not in the classic Smalltalk/Java sense. There goes the "O". gorp doesn't know anything about the relationships between your structs (at least not yet). So the "R" is questionable too (but I use it in the name because, well, it seemed more clever).

The "M" is alive and well. Given some Go structs and a database, gorp should remove a fair amount of boilerplate busy-work from your code.

I hope that gorp saves you time, minimizes the drudgery of getting data in and out of your database, and helps your code focus on algorithms, not infrastructure.
</blockquote>

When I was looking into [Revel](http://www.github.com/robfig/revel) as a possibility for a Go web application framework, I found myself frustrated by its lack of a database solution.  Persistence is just such a key aspect of web applications, and something that we're so accustomed to letting frameworks take care of for us (a la Rails and Django) that it was hard to believe a large framework like Revel didn't even want to touch the problem- especially since [Play](http://www.playframework.com/documentation/1.2.1/model), a large source of inspiration, provides such functionality.  Revel is awesome in a lot of other ways, like its code hotswap feature, but for now at least it is "bring-your-own-ORM" (or other database solution).

So I set off to look into this funny `gorp` business.  As it turns out, `gorp` is pretty straightforward and powerful.  At the time of writing, `gorp` can be used with MySQL, Sqlite3, and PostgreSQL (although there are some known issues that cause different drivers to behave slightly differently).

# Creating Tables

The basic use case for `gorp` is to define some structs and then register them with an instance of `gorp`'s `DbMap` structure.  This structure is responsible for generating the raw SQL to perform basic database operations on a table that will mirror your custom defined structure.  `gorp` can easily create that table for you in the first place.  Check it out:

```go
type Person struct {
    Id      int64    
    Created int64
    Updated int64
    FName   string
    LName   string
}

// connect to db using standard Go database/sql API
// use whatever database/sql driver you wish
db, err := sql.Open("mymysql", "tcp:localhost:3306*mydb/myuser/mypassword")

// construct a gorp DbMap
dbmap := &gorp.DbMap{Db: db, Dialect: gorp.MySQLDialect{"InnoDB", "UTF8"}}

table := dbmap.AddTable(Person{}).SetKeys(true, "Id")
```

You can also use `AddTableWithName` if you don't want the table name to be the same as the structure type's name (in fact, `AddTable` calls `AddTableWithName`):

```go
table := dbmap.AddTableWithName(Person{}, "People").SetKeys(true, "Id")
```

As you can imagine, being able to easily create and drop tables like this is useful for unit tests.

You can use structure field tags if you want to change the name of the columns in the actual SQL (let's say your team has a convention to have only lowercase column names, but all members of a Go struct must be uppercase).  Additionally you can tell `gorp` to ignore fields completely with `db:"-"`:

```go
type Person struct {
    Id       int64                        `id`
    Created  int64                        `created`
    Updated  int64                        `modified`
    FName    string                       `firstName`
    LName    string                       `lastName`
    Comments *SomeNonPersistentStructure  `db:"-"`
}
```

A seemingly undocumented feature is that you can set the size of the table columns manually.  If you don't, `gorp` will automatically figure something out for you that may be a bit too large or too small.  For example, `gorp` turns this structure definition:

```
// A Thing is a post (link submission or a comment)
type Thing struct {
	Id            int64
	Username      string
	Href          string
	Upvotes       int64
	Downvotes     int64
	Description   string
	ParentThingId int64
	Created       int64
	Updated       int64
}
```

into this (with default behavior / MySQL driver):

```
+---------------+--------------+------+-----+---------+-------+
| Field         | Type         | Null | Key | Default | Extra |
+---------------+--------------+------+-----+---------+-------+
| Id            | bigint(20)   | YES  |     | NULL    |       |
| Username      | varchar(255) | YES  |     | NULL    |       |
| Href          | varchar(255) | YES  |     | NULL    |       |
| Upvotes       | bigint(20)   | YES  |     | NULL    |       |
| Downvotes     | bigint(20)   | YES  |     | NULL    |       |
| Description   | varchar(255) | YES  |     | NULL    |       |
| ParentThingId | bigint(20)   | NO   | PRI | NULL    |       |
| Created       | bigint(20)   | YES  |     | NULL    |       |
| Updated       | bigint(20)   | YES  |     | NULL    |       |
+---------------+--------------+------+-----+---------+-------+
```


When you call `gorp.DbMap.AddTableWithName`, it returns you a pointer to a `TableMap` struct that you can use to set the size of the columns.  So you think 255 characters is a bit long for a username?

```
t1 := dbmap.AddTable(Person{}).SetKeys(true, "Id")
t1.ColMap("Username").SetMaxSize(25)
```

The things you learn from reading the unit tests (and digging in the [Revel examples](https://github.com/robfig/revel/blob/master/samples/booking/app/controllers/gorp.go)), huh?

# CRUD

Let's take a look at what CRUD (Create-Read-Update-Delete) looks like using `gorp`-mapped structures.

Inserting a new row is simple (note that you have to declare the structs as pointers so that optional callback hooks can operate on your actual data instead of copies):

```go
person := &Person{
	Created: time.Now().UnixNow(), 
	Updated: time.Now().UnixNow(),
	FName: "Joe",
	LName: "Smith"
}
err := dbmap.Insert(person)
```

Want to select by primary key?

```go
primaryKey := 1
p1, err := dmap.Get(Person{}, primaryKey)
```

How about selecting by arbitrary (non-primary-key) fields?  You can use `dbm.Select` to get a slice, or `dbm.SelectOne` to populate the slice or structure with the revelant data.

```go
var ids []int64
_, err := dbmap.Select(&ids, "select id from Person")

lname = "LeClaire"
var person Person
err := dbmap.SelectOne(&person, "select * from Person where LName=?", lname)
```

Update and delete work similarly :

```go
// count is the # of rows updated / deleted
person.FName = "Nate" 
count, err := dbmap.Update(person)

// or just delete it 
count, err := dbmap.Delete(person)
```


# How does it do all of this crazy voodoo?

Obviously gorp is really cool, and useful.  So how does it work?

{% img /images/gorp/use-the-source-luke.jpg Best way to learn. %}

I had no idea, but I remembered the words of Jeff Atwood and other wise folks and cracked open the [source code on github](https://github.com/coopernurse/gorp/blob/master/gorp.go).  Reading the unit tests also proved useful in understanding how `gorp` should be used (one of the virtues of meticulously tested code - it documents).

Immediately upon cracking open the definition of `DbMap.AddTable` and `DbMap.AddTableWithName`, I had one of those "aha" moments that programmers know so well.

```go
// AddTableWithName has the same behavior as AddTable, but sets
// table.TableName to name.
func (m *DbMap) AddTableWithName(i interface{}, name string) *TableMap {
        t := reflect.TypeOf(i)
        if name == "" {
                name = t.Name()
        }

        // check if we have a table for this type already
        // if so, update the name and return the existing pointer
        for i := range m.tables {
                table := m.tables[i]
                if table.gotype == t {
                        table.TableName = name
                        return table
                }
        }

        tmap := &TableMap{gotype: t, TableName: name, dbmap: m}
        tmap.columns, tmap.version = readStructColumns(t)
        m.tables = append(m.tables, tmap)

        return tmap
}
```

Of course, it uses reflection!  Go's [reflect](http://golang.org/pkg/reflect/) package is what powers this manipulation and mapping of structure metadata (I wasn't aware Go was capable of reflection when I started using `gorp`, so it was a bit of a surprise to find this out). 

Suddenly everything became clearer to me and I feel like the code for `AddTableWithName` is fairly self-explanatory if you are familiar with the usage of the library.  The first part of the method deals with naming the table (user defined or based on the name of the structure).  The middle section checks to see if the table already is in existence and if so it updates the name (consequently, we can set up a table for a structure with one name, then change the table name later on if we want).  Lastly, it adds the table if it doesn't exist and returns a pointer to the `TableMap` structure (we discussed this structure briefly earlier).  

The code for the `readStructColumns` internal method that you see called near the end of the method is pretty cool as well, it powers `gorp`'s ability to deal with struct embedding (a pretty cool feature of the libary IMO).  I won't reproduce it here, but if you are curious [go check it out](https://github.com/coopernurse/gorp/blob/master/gorp.go)!  


# The future

Alas, we developers are never easy to please forever.  Here I will note some things that may become issues for users of `gorp`, and hopefully get the ball rolling on conversation about directions for `gorp`'s future development.

Support for `TEXT` columns (and maybe other, "weirder" column types like PostgreSQL's [json data type](http://www.postgresql.org/docs/9.2/static/datatype-json.html)) seems like something that will be needed to really bring `gorp` into the limelight as a robust and mature tool (see [this issue](https://github.com/coopernurse/gorp/issues/34) on github, where someone brings up `TEXT` specifically).  A `VARCHAR` column arguably would be inappropriate for storing the content of a Reddit comment or a blog post, for example.  I'd be curious how the maintainers are interested in handling this- getting into defining custom data types with `gorp` (e.g. `gorp.Text`) might be dicey, for instance, or it could prove to be a robust solution.  In the long run, it's worth considering how much of `gorp`'s flexibility and power comes from its ability to discern those kinds of things with minimal input from the user, and how much of that we're willing to give up to have a VERY robust database / ORM-ish solution for Golang.

Other tough nuts to crack with `gorp` (Golang's strict/static typing, which is definitely one of its advantages in some ways, is partially what makes some of these so challenging) :

* Handling relational data
* Joins (the existing solution looks pretty workable, but feels a bit stiff - admittedly I haven't tried it though)
* Data migrations

Any ideas?

Also, not to be "that guy", but it could probably stand to be broken up into a few different files (one for each of the different structures, for instance) instead of one large `gorp.go` file.

# Conclusion

`gorp` is a very cool, if still young, tool / library.  I find it to be a good combination of abstraction and practicality.  What do you think?

Thanks for reading, I'll see you next week.

Nate
