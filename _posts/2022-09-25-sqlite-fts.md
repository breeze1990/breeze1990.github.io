---
title: "A Beginner's journey with SQLite FTS5 - External Content Tables"
date: 2022-09-25
---

I recently came to find [SQLite FTS5](https://sqlite.org/fts5.html) quite interesting. The online document gives a brief introduction with some examples explaining how it works. However, I didn't have a clear understading on the [External Content Tables](https://www.sqlite.org/fts5.html#external_content_tables) section until I played with it and it started to make sense to me.

# Table Setup
SQLite tables are set up like below:
``` sql
# create content table
CREATE TABLE tbl (id INTEGER PRIMARY KEY, title TEXT, body TEXT);
# create index table
CREATE VIRTUAL TABLE tbl_idx USING fts5(body, content=tbl, content_rowid=id);
# create trigger to sync data from content table to index table
# trigger on update statement is omitted on purpose for demonstration
CREATE TRIGGER tbl_ai AFTER INSERT ON tbl BEGIN
  INSERT INTO tbl_idx(rowid, body) VALUES (new.id, new.body);
END;
CREATE TRIGGER tbl_ad AFTER DELETE ON tbl BEGIN
  INSERT INTO tbl_idx(tbl_idx, rowid, body) VALUES('delete', old.id, old.body);
END;
```
# Query
## Interaction with content and index tables
First, run a simple `INSERT` statement and check the result
``` sql
INSERT INTO tbl VALUES (1, 'hello', 'hello world');
```
> sqlite> SELECT * FROM tbl_idx WHERE tbl_idx MATCH 'world';  
hello world

It's working as expected here. Next, something interesting happens when I run an `UPDATE` statement.

``` sql
UPDATE tbl SET body = 'hello sqlite' WHERE id=1;
```
> sqlite> SELECT * FROM tbl_idx WHERE tbl_idx MATCH 'world';  
  hello sqlite

As shown here, inconsistency occurs. The result shows updated content, but the query actually is still using old keyword. Because I didn't include a trigger to handle `UPDATE` statement, the index table isn't updated. As a result, SQLite still finds the rowid with the old keyword and then by rowid it finds the updated record in the content table, which is returned.

Hence, in the document it says:
>It is still the responsibility of the user to ensure that the contents of an external content FTS5 table are kept up to date with the content table. One way to do this is with triggers.

## Useful query
``` sql
# SQL to list all triggers
SELECT * FROM sqlite_master WHERE type = 'trigger';
```

# Thoughts
The "content" option is very useful in practice. As for an application, it's rarely a good design to couple content and index in one table. Often, we first have some tables to support some other forms of business functionalities. In addition, we can build a search feature with this FTS extension in a loosely coupled way.
The user just has to take the responsibility of keeping data in sync as in the document. "trigger" is one simple way to automatically do that within SQLite. Also, we can implement an application component that can have more complicated capabilities. e.g., we can join data from multiple tables and index them all together in one place.
