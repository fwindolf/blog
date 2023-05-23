# Using SQLModel instead of SQLAlchemy

After quite some time in the world of SQLAlchemy at work, I decided to try out the ORM in the FastAPI world. SQLModel combines SQLAlchemy and Pydantic into an ORM that behaves just like a class, but with the support of the SQLAlchemy ecosystem. 

## Model Definition

Models are defined similar to SQLAlchemy. The difference is that instead of using Columns, we can use the Pydantic-style `Field` and `Relationship`.

### Base Model

First, we want to define our Base Model that gives us a primary key as well as some extra columns that contain the information about when an instance was created and/or updated. In the real world, you would probably split those up, or use a different BaseModel alltogether.

Note, that unlike in SQLAlchemy, we don't need to create a `DeclarativeBase` base model.

```python
import uuid
from datetime import datetime

from pydantic import BaseModel
from sqlalchemy import text
from sqlmodel import Field, SQLModel


class BaseModel(SQLModel):
    """Mixin for models.
    
    Adds a UUID primary key and created_at/updated_at timestamps.
    """

    id: uuid.UUID = Field(
        default_factory=uuid.uuid4,
        primary_key=True,
        index=True,
        nullable=False,
        sa_column_kwargs={
            "server_default": text("gen_random_uuid()"), 
            "unique": True,
        },
    )
    
    created_at: datetime = Field(
        default_factory=datetime.utcnow,
        nullable=False,
        sa_column_kwargs={"server_default": text("current_timestamp(0)")},
    )

    updated_at: datetime = Field(
        default_factory=datetime.utcnow,
        nullable=False,
        sa_column_kwargs={
            "server_default": text("current_timestamp(0)"),
            "onupdate": text("current_timestamp(0)"),
        },
    )
```

As this BaseModel inherits from SQLModel, how does the engine behind it know that no table should be created? Easy: if we want an actual table being created, we add a `table=True`.

### Book and Author

Now let's define some actual tables for the class `Author` and `Book`:

```python
import uuid
from .base import BaseModel
from sqlmodel import Relationship, Field

class Author(BaseModel, table=True):
    name: str = Field(nullable=False)
    books: list["Book"] = Relationship(back_populates="author")

class Book(BaseModel, table=True):
    title: str = Field(nullable=False)
    author_id: uuid.UUID = Field(foreign_key="author.id")
    author: Author = Relationship(back_populates="books")
```

Here, an `Author` has a name and has written at least one book. Similarily, a book has a title, an author referenced by its id and the relationship. This is a pretty straightforward setup, with one quirk: As we need to define the relationship in both direction, we have to forward declare the book type as `"Book"`.

## Database Queries

### Get Single Author

To retrieve a single author from the database, you can use the `get` method provided by the `Session` object:

```python
from sqlmodel import Session

def get_author(session: Session, author_id: uuid.UUID):
    author = session.get(Author, author_id)
    return author
```

In this easy example, we use a synchronous session to query the database. In a real API, we would probably move this method into a repository that groups all author queries, and use automatic injection of the session, so you don't have to pass it along to every query. We will get to that eventually in another blog post, I promise!

### Get all Books from an Author

But first, we want to write some more elaborate queries. To retrieve all books written by a particular author, you can use the `select` method along with the `where` method:

```python
from sqlmodel import select

def get_author_books(session: Session, author_id: uuid.UUID):
    books = session.exec(select(Book).where(Book.author_id == author_id)).all()
    return books
```

You can see that we can simply chain the calls to build our query, starting from the `select` method from SQLModel. With this, we first state the model that we want to query (just like with our `.get`). On that, we chain a `where` method that uses a field of the model we query and some constraint like `== author_id`. 

If we would have more models and wanted to add more constraints (eg. querying all authors called Fred), we can't just use a where anymore, as our Book has no direkt relationship with the author's name. Therefore, we would first `join` the Author, then add the `Author.name == "Fred"`:

```python
session.exec(select(Book).join(Author).where(Author.name == "Fred")).all()
```

## The Advanced Stuff

With that problem solved, we can continue to two other topics that took me a while to figure out.

### Multiple Files - Circular Dependencies

When your models start to grow and you begin to separate them into multiple files, circular dependencies can become a problem. Python's forward references can help solve this issue. In the `Author` and `Book` models above, we have already used forward references. 

This starts to break at the point where we split both up into their own files. The solution is to provide the type during type checking, but relying on forward references during runtime.

This is a much better solution than the method often suggested that adds the relationships after the related model is defined. 

```python
import uuid
from typing import TYPE_CHECKING
from .base import BaseModel
from sqlmodel import Relationship, Field

if TYPE_CHECKING:
    from .book import Book

class Author(BaseModel, table=True):
    name: str = Field(nullable=False)
    books: list["Book"] = Relationship(back_populates="author")
```

```python

from .author import Author

class Book(BaseModel, table=True):
    title: str = Field(nullable=False)
    author_id: uuid.UUID = Field(foreign_key="author.id")
    author: Author = Relationship(back_populates="books")
```

The order in which those files are imported should not matter anymore. The `list[]` is enough for the type to work as forward references, and during type checking and in our IDE.


### Migrations with Alembic

Alembic is a database migration tool for SQLAlchemy and works well with SQLModel. To generate a migration script with Alembic, you would typically do:

```bash
alemb