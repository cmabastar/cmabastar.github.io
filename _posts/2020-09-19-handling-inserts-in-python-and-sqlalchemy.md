---
layout: post
title: Handling Inserts In Python and Sqlalchemy
---

Dealing database inserts is one of the most crucial parts of your data lifecycle. It could range from a single insert and multiple inserts. However, oftentimes we face some bottlenecks on handling inserts depending on the size of the data we are inserting.


Here we will show you how to do many ways of insertions using postgres as a backend. Also, for the example, I’m assuming we are using the t2.micro instance which has 1GB of RAM.

Let’s start!


### Installing Dependencies


```python
pip install sqlalchemy==1.39
pip install Flask-SQLAlchemy==2.4.1
```


We define our example model as the following:


```python
from sqlalchemy import func
from flask_sqlalchemy import SQLAlchemy
db = SQLAlchemy()

class Post(db.Model):
	__tablename__ = "posts"

	id = db.Column(db.Integer(), primary_key=True)
	message = db.Column(db.String(1028), nullable=False)
	created_at = Column(
		db.DateTime(timezone=True),
		default=func.now(),
		server_default=text("now()"),
		nullable=False,
	)
```



### Inserting for 1-100 entries

Single Insertion via ORM, This is a great way to maintain code readability in your code.


```python
post = Post(message='Hello world')
db.session.add(post)
db.session.commit()
```


For multiple entries this may look like the following:


```python
for i in range(0,100):
    post = Post(message=f"hello world {i}")
)
    db.session.add(post)
db.session.commit()
```


Since there are only few data generated and everything perfectly fits in our memory, this is still fast.


### Inserting for 100-5000 entries

For each sqlalchemy model we create, sqlalchemy maps them on a python object. If we are to insert large quantities of data, it might not fit into our memory and may take some time to generate the objects. (ref: [https://docs.sqlalchemy.org/en/13/faq/performance.html](https://docs.sqlalchemy.org/en/13/faq/performance.html))

For this case, we can use the following example for inserting:


```python
db.session.bulk_insert_mappings(
        Post,
        [
            dict(
                message=f"hello world {i}",
            )
            for i in range(0,5000)
        ],
    )
 db.session.commit()
```


Or, Another way is to use a core level insert which offers faster insertion performance based on the sqlalchemy’s benchmark.


```python
to_insert = []
for i in range(0, 5000):
	to_insert.append({"message" : f"hello world {i}"})
stmt = Post.__table__.insert().values(to_insert)
db.session.execute(stmt)
db.session.commit()
```



### Inserting for 5000 - 100K+ entries

This is where it gets tricky depending your model size and the server size you have.  One solution I used for this is to chunk the insertions.

We could use the following example below to make sure we don’t get OOM killed.


```python
to_insert = []
ctr = 0
chunk = 100000
for i in range(0, 1000000):
	to_insert.append({"message": f"hello world {i}"})
	ctr += 1
	if ctr % chunk == 0:
		stmt = Post.__table__.insert().values(to_insert)
		db.session.execute(stmt)
		db.session.commit()
		to_insert = []

# Any leftovers are saved
if to_insert:
	stmt = Post.__table__.insert().values(to_insert)
	db.session.execute(stmt)
	db.session.commit()
```



### Inserting 1M+ and beyond.

The previous example above should suffice in most of your workloads, but there is another way to insert bulk loads of data by using psycopg2’s **_copy_from_** function.  By doing it this way, we don’t use python dict objects but a binary level of insertion.


```python
import io
import csv

iodata = io.StringIO()
writer = csv.writer(iodata, delimiter="\t")
for i in range(0, 1000000):
    writer.writerow([f"hello world {i}"])
# Reset the cursor to the top of the file
iodata.seek(0)

cursor = db.session.connection().connection.cursor()
cursor.copy_from(
	iodata,
	Post.__table__.name,
	columns=("message"),
	sep="\t",
)
```


I tried running and generating the 1M entries of this string to a csv and it only generated a 18MB of file. Which means that should be our estimated memory consumption for inserting this to our db.
