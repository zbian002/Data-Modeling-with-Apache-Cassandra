# Sparkify: Data Modeling with Cassandra

Sparkify is a startup with a music streaming app. Data scientists at Sparkify are particularly interested in understanding what songs users are listening to.

The goal of this project is to define a data model that will help our data scientists answer questions about user activity. In this case, we are working with a NoSQL database, [Apache Cassandra](http://cassandra.apache.org/), which means we have to be especially intentional in our data modeling. We first need to understand what queries our data scientists would like to run, and then we can design tables to fit those queries. We will be aiming for "1 query, 1 table".

## File Overview
- `event_data/` - contains our data for user sessions in the streaming app in csv format. files are partitioned by day. (e.g. `2018-11-02-events.csv`)
- `images` - contains an image of what our data looks like after an ETL process
- `data_modeling_with_cassandra.ipynb` - Has two parts:
  - ETL pipeline for processing the event files and compiling them into a single csv with the desired columns
  - Data modeling examples that show three queries and the creation of three tables to serve those queries

Note: the data is not included in this repo, but can be shared upon request.

To run this, you simply need to open the Notebook and run all cells.

## The Data Model: 1 Query, 1 Table

**Subset of our data after the ETL pipeline:**

![Final Data](images/image_event_datafile_new.jpg)

### Queries to model for

**Song plays by session**

Query:
> Give me the artist, song title and song's length in the music app history that was heard during sessionId = 338, and itemInSession = 4

Required fields: artist, song, length, sessionid, itemInSession

Partition Key: should be `sessionId`, because this is the primary way our data is being filtered in the query. We are also filtering on `itemInSession`, but that field is less specific and would result in a less logical distribution across nodes.

Primary Key: `sessionId` is not sufficient for a PK, because it is not unique. However, adding `itemInSession` as a clustering column with give us uniqueness.

**Song plays by User's session**

Query:
> Give me only the following: name of artist, song (sorted by itemInSession) and user (first and last name) for userid = 10, sessionid = 182

Required fields: artist, song, itemInSession, firstName, lastName, userId, sessionId

Partition Key: should be `userId` in this case, because this is the primary way our data is being filtered in the query. We are also filtering on `sessionId`, but again that field is less specific and would result in a less logical distribution across nodes.

Primary Key: In this table `userId` would not be unique, and adding `sessionId` would still not guarantee uniqueness. However, the query also asks for sorting by `itemInSession`, which means we need it as a clustering column anyway. So a PK that is both unique and sorts the way we would like is `(userId, sessionId, itemInSession)`.

**Users by Song listens**

Query:
> Give me every user name (first and last) in my music app history who listened to the song 'All Hands Against His Own'

Required fields: firstName, lastName, song, userId

Partition Key: we're filtering by `song` in this case, so we'll use that as our partition key

Primary Key: It's not necessarily required given our query, but I've chosen to include the `userId` field so that we can add it to our PK. I've done this for two reasons, the first being that we need a unique PK and `(song)` or `(song, firstName)` don't suffice. Another reason would be to give us some sorting by `userId`, which feels like a logical way to order. If instead we wanted to sort alphabetically we could do `(song, firstName, userId)`.

### Why was a NoSQL Database chosen?

Well, the real answer is, to get experience with a NoSQL database. But if think about this as if it were a real production system, we might ask:

**When would we want NoSQL over a relational DB?**

Perhaps you have LARGE amounts of data. Because a relational database can only be scaled by adding machine memory, its very possible to have a table too large for a single machine. So if you want distributed tables, NoSQL is what you want. Some other potential reasons are: need for high throughput that isn't slowed down by ACID transactions or high availability. In our case Apache Cassandra is an AP tolerant system, so we're optimizing for availability and partition tolerance.

**What are the caveats of a NoSQL database like Apache Cassandra?**

So first off, based on CAP Theorem, we know that what's being sacrificed by our AP tolerant system is _consistency_. Which means it's possible that a read from our DB might not give us the most up to date information, instead we settle for _eventual consistency_. Another thing we must take into account is the lack of `JOIN`s. One of the nice aspects of a relational database is the query flexibility and capability to do aggregations. With Cassandra, we don't have that ability, so instead we must know about the queries our Data Scientists would like before hand so we can model our tables to fit them. And if our queries change, so must our tables.