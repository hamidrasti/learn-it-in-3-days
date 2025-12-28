# Ultimate SQLAlchemy Crash Course for FAANG Interviews (2024-2025)

---

## 1. Core Concepts You Must Know Cold

### **1.1 SQLAlchemy Architecture (Two APIs)**

SQLAlchemy has two distinct layers - understanding this is fundamental:

```
┌─────────────────────────────────────────┐
│           SQLAlchemy ORM                │  ← High-level, object-oriented
├─────────────────────────────────────────┤
│         SQLAlchemy Core                 │  ← Low-level, SQL expression language
├─────────────────────────────────────────┤
│              DBAPI                      │  ← Database drivers (psycopg2, etc.)
└─────────────────────────────────────────┘
```

```python
# Core approach (SQL Expression Language)
from sqlalchemy import create_engine, text, Table, Column, Integer, String, MetaData

engine = create_engine("postgresql://user:pass@localhost/db")
with engine.connect() as conn:
    result = conn.execute(text("SELECT * FROM users WHERE id = :id"), {"id": 1})

# ORM approach (Object-Relational Mapping)
from sqlalchemy.orm import Session, DeclarativeBase, Mapped, mapped_column

class Base(DeclarativeBase):
    pass

class User(Base):
    __tablename__ = "users"
    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(50))

with Session(engine) as session:
    user = session.query(User).filter_by(id=1).first()
```

**Interview Insight:** Know when to use each - ORM for complex domain models, Core for performance-critical bulk operations.

---

### **1.2 Engine & Connection Pool**

The engine is the starting point for any SQLAlchemy application:

```python
from sqlalchemy import create_engine
from sqlalchemy.pool import QueuePool, NullPool

# Basic engine creation
engine = create_engine(
    "postgresql+psycopg2://user:password@localhost:5432/dbname",
    echo=True,           # Log all SQL statements
    pool_size=5,         # Number of connections to keep open
    max_overflow=10,     # Additional connections allowed beyond pool_size
    pool_timeout=30,     # Seconds to wait for available connection
    pool_recycle=1800,   # Recycle connections after 30 minutes
    pool_pre_ping=True   # Test connections before use (handles stale connections)
)

# Connection URL format
# dialect+driver://username:password@host:port/database

# Different database examples
sqlite_engine = create_engine("sqlite:///./local.db")
mysql_engine = create_engine("mysql+pymysql://user:pass@localhost/db")
postgres_engine = create_engine("postgresql+psycopg2://user:pass@localhost/db")

# For testing - disable connection pooling
test_engine = create_engine("sqlite:///:memory:", poolclass=NullPool)
```

**Connection Context Management:**
```python
# Recommended pattern - automatic resource cleanup
with engine.connect() as connection:
    result = connection.execute(text("SELECT 1"))
    connection.commit()  # Explicit commit required in 2.0 style

# Begin transaction block
with engine.begin() as connection:
    connection.execute(text("INSERT INTO users (name) VALUES (:name)"), {"name": "Alice"})
    # Auto-commits if no exception, auto-rollbacks on exception
```

---

### **1.3 Declarative Models & Table Definition**

**Modern SQLAlchemy 2.0 Style (Preferred in 2024-2025):**

```python
from datetime import datetime
from typing import Optional, List
from sqlalchemy import String, ForeignKey, Text, DateTime, func
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column, relationship

class Base(DeclarativeBase):
    pass

class User(Base):
    __tablename__ = "users"
    
    # Primary key with auto-increment
    id: Mapped[int] = mapped_column(primary_key=True)
    
    # Required string field with max length
    email: Mapped[str] = mapped_column(String(255), unique=True, index=True)
    
    # Optional field (nullable)
    name: Mapped[Optional[str]] = mapped_column(String(100))
    
    # Default values
    is_active: Mapped[bool] = mapped_column(default=True)
    
    # Server-side default (executed by database)
    created_at: Mapped[datetime] = mapped_column(
        DateTime(timezone=True), 
        server_default=func.now()
    )
    
    # Relationship - one-to-many
    posts: Mapped[List["Post"]] = relationship(back_populates="author", cascade="all, delete-orphan")
    
    # Optional: table arguments
    __table_args__ = (
        {"schema": "public"},  # PostgreSQL schema
    )
    
    def __repr__(self) -> str:
        return f"User(id={self.id}, email={self.email})"

class Post(Base):
    __tablename__ = "posts"
    
    id: Mapped[int] = mapped_column(primary_key=True)
    title: Mapped[str] = mapped_column(String(200))
    content: Mapped[str] = mapped_column(Text)
    
    # Foreign key
    author_id: Mapped[int] = mapped_column(ForeignKey("users.id", ondelete="CASCADE"))
    
    # Relationship - many-to-one
    author: Mapped["User"] = relationship(back_populates="posts")

# Create all tables
Base.metadata.create_all(engine)
```

**Legacy Style (Still seen in interviews):**
```python
from sqlalchemy import Column, Integer, String
from sqlalchemy.ext.declarative import declarative_base

Base = declarative_base()

class User(Base):
    __tablename__ = "users"
    id = Column(Integer, primary_key=True)
    name = Column(String(50), nullable=False)
```

---

### **1.4 Session Lifecycle & Unit of Work Pattern**

The Session is the central concept for ORM operations:

```python
from sqlalchemy.orm import Session, sessionmaker

# Create session factory
SessionLocal = sessionmaker(bind=engine, autoflush=False, expire_on_commit=False)

# Session lifecycle
session = Session(engine)
try:
    # 1. Create new objects (pending state)
    user = User(email="test@example.com", name="Test")
    session.add(user)  # Object is now "pending"
    
    # 2. Flush - sends SQL to database but doesn't commit
    session.flush()  # Object is now "persistent", has ID assigned
    print(user.id)   # ID is available after flush
    
    # 3. Query existing objects
    existing_user = session.get(User, 1)  # Get by primary key
    
    # 4. Modify objects (automatically tracked)
    existing_user.name = "Updated Name"  # Marked as "dirty"
    
    # 5. Delete objects
    session.delete(existing_user)
    
    # 6. Commit transaction
    session.commit()  # All changes persisted
    
except Exception:
    session.rollback()  # Undo all changes
    raise
finally:
    session.close()  # Return connection to pool

# Better pattern - context manager
with Session(engine) as session:
    with session.begin():  # Auto-commit/rollback
        session.add(User(email="new@example.com"))
```

**Object States:**
```
┌─────────────┐    add()     ┌───────────┐   flush()   ┌────────────┐
│  Transient  │ ──────────▶  │  Pending  │ ─────────▶  │ Persistent │
└─────────────┘              └───────────┘             └────────────┘
                                                              │
                                                              │ expunge() / 
                                                              │ session.close()
                                                              ▼
                                                       ┌────────────┐
                                                       │  Detached  │
                                                       └────────────┘
```

---

### **1.5 Querying (Both Styles)**

**Modern 2.0 Style with `select()`:**

```python
from sqlalchemy import select, and_, or_, func, desc, asc
from sqlalchemy.orm import Session

with Session(engine) as session:
    # Basic SELECT
    stmt = select(User).where(User.is_active == True)
    users = session.scalars(stmt).all()
    
    # Single result
    stmt = select(User).where(User.id == 1)
    user = session.scalar(stmt)  # Returns None if not found
    user = session.execute(stmt).scalar_one()  # Raises if not exactly one
    
    # Filter conditions
    stmt = select(User).where(
        and_(
            User.is_active == True,
            or_(
                User.email.like("%@gmail.com"),
                User.email.like("%@yahoo.com")
            )
        )
    )
    
    # Ordering and limiting
    stmt = select(User).order_by(desc(User.created_at)).limit(10).offset(20)
    
    # Selecting specific columns
    stmt = select(User.id, User.email)
    results = session.execute(stmt).all()  # List of tuples
    
    # Aggregations
    stmt = select(func.count(User.id)).where(User.is_active == True)
    count = session.scalar(stmt)
    
    # Group by
    stmt = select(
        User.is_active, 
        func.count(User.id).label("user_count")
    ).group_by(User.is_active)
```

**Legacy Query API (Still common in codebases):**

```python
with Session(engine) as session:
    # Basic queries
    users = session.query(User).filter(User.is_active == True).all()
    user = session.query(User).filter_by(email="test@example.com").first()
    
    # Chaining
    users = (
        session.query(User)
        .filter(User.is_active == True)
        .order_by(User.created_at.desc())
        .limit(10)
        .all()
    )
```

---

### **1.6 Relationships & Joins**

```python
from sqlalchemy import ForeignKey, Table
from sqlalchemy.orm import relationship, Mapped, mapped_column

# Many-to-Many association table
post_tags = Table(
    "post_tags",
    Base.metadata,
    Column("post_id", ForeignKey("posts.id"), primary_key=True),
    Column("tag_id", ForeignKey("tags.id"), primary_key=True)
)

class Tag(Base):
    __tablename__ = "tags"
    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(50), unique=True)
    posts: Mapped[List["Post"]] = relationship(secondary=post_tags, back_populates="tags")

class Post(Base):
    __tablename__ = "posts"
    id: Mapped[int] = mapped_column(primary_key=True)
    title: Mapped[str] = mapped_column(String(200))
    author_id: Mapped[int] = mapped_column(ForeignKey("users.id"))
    
    author: Mapped["User"] = relationship(back_populates="posts")
    tags: Mapped[List["Tag"]] = relationship(secondary=post_tags, back_populates="posts")

# JOIN queries
with Session(engine) as session:
    # Implicit join through relationship
    stmt = select(Post).join(Post.author).where(User.email == "test@example.com")
    
    # Explicit join
    stmt = select(Post, User).join(User, Post.author_id == User.id)
    
    # Left outer join
    stmt = select(User, Post).outerjoin(Post, User.id == Post.author_id)
    
    # Join with eager loading
    from sqlalchemy.orm import joinedload, selectinload
    
    # joinedload - single query with JOIN (good for one-to-one, many-to-one)
    stmt = select(Post).options(joinedload(Post.author)).where(Post.id == 1)
    
    # selectinload - separate IN query (good for one-to-many, many-to-many)
    stmt = select(User).options(selectinload(User.posts)).where(User.id == 1)
```

---

### **1.7 Transactions & Concurrency**

```python
from sqlalchemy import event
from sqlalchemy.orm import Session

# Nested transactions (savepoints)
with Session(engine) as session:
    session.begin()
    try:
        session.add(User(email="user1@example.com"))
        
        # Savepoint - nested transaction
        with session.begin_nested():
            session.add(User(email="user2@example.com"))
            raise ValueError("Something went wrong")  # This savepoint rolls back
        
        # user1 still exists, user2 rolled back
        session.commit()
    except Exception:
        session.rollback()

# Isolation levels
engine = create_engine(
    "postgresql://...",
    isolation_level="REPEATABLE READ"  # or SERIALIZABLE, READ COMMITTED
)

# Pessimistic locking (SELECT FOR UPDATE)
with Session(engine) as session:
    stmt = select(User).where(User.id == 1).with_for_update()
    user = session.scalar(stmt)  # Row is locked until commit/rollback
    user.balance -= 100
    session.commit()

# Optimistic locking with version column
class Account(Base):
    __tablename__ = "accounts"
    id: Mapped[int] = mapped_column(primary_key=True)
    balance: Mapped[int]
    version_id: Mapped[int] = mapped_column(default=1)
    
    __mapper_args__ = {"version_id_col": version_id}
```

---

### **1.8 Migrations with Alembic**

```bash
# Setup
pip install alembic
alembic init alembic
```

```python
# alembic/env.py - configure target metadata
from models import Base
target_metadata = Base.metadata

# Generate migration
# alembic revision --autogenerate -m "add users table"

# Example migration file
def upgrade():
    op.create_table(
        'users',
        sa.Column('id', sa.Integer(), primary_key=True),
        sa.Column('email', sa.String(255), nullable=False, unique=True),
        sa.Column('created_at', sa.DateTime(), server_default=sa.func.now())
    )
    op.create_index('ix_users_email', 'users', ['email'])

def downgrade():
    op.drop_index('ix_users_email')
    op.drop_table('users')

# Commands
# alembic upgrade head      - Apply all migrations
# alembic downgrade -1      - Rollback one migration
# alembic history           - Show migration history
# alembic current           - Show current revision
```

---

## 2. Most Frequently Asked Interview Questions & Topics

### **Easy**
| Question | Key Points |
|----------|------------|
| What's the difference between ORM and Core? | ORM = object mapping, Core = SQL expressions, ORM built on Core |
| How do you create a session? | `Session(engine)` or sessionmaker factory pattern |
| What is `flush()` vs `commit()`? | flush = SQL sent, commit = transaction finalized |
| How do you filter query results? | `.where()` with 2.0, `.filter()` with legacy |
| What's a foreign key relationship? | `ForeignKey` constraint + `relationship()` for ORM access |

### **Medium**
| Question | Key Points |
|----------|------------|
| Explain the N+1 query problem and solutions | Lazy loading causes extra queries; use `joinedload`/`selectinload` |
| How do you handle database transactions? | Context managers, `begin()`, `commit()`, `rollback()`, savepoints |
| What are the different loading strategies? | lazy, eager (joined, subquery, selectin), raise, noload |
| How do you perform bulk operations efficiently? | `session.bulk_save_objects()`, `insert().values()`, Core operations |
| Explain connection pooling configuration | pool_size, max_overflow, pool_timeout, pool_recycle, pool_pre_ping |

### **Hard / System Design**
| Question | Key Points |
|----------|------------|
| How would you handle high-concurrency writes? | Connection pooling, optimistic/pessimistic locking, isolation levels |
| Design a multi-tenant database layer | Schema-based vs row-based, dynamic engine/session binding |
| How do you optimize a slow SQLAlchemy query? | Eager loading, query analysis, indexing, denormalization, caching |
| Implement soft deletes with SQLAlchemy | Mixin class, query hooks, hybrid properties |
| How do you test SQLAlchemy code? | In-memory SQLite, fixtures, transaction rollback pattern |

### **Deep Dive Questions**
| Question | Expected Answer |
|----------|-----------------|
| Explain the Unit of Work pattern | Session tracks object changes, batches SQL, maintains identity map |
| What happens during `session.commit()`? | flush → send SQL → commit transaction → expire all objects |
| How does identity map work? | Session caches objects by primary key, ensures single instance per identity |
| What is expire/refresh? | expire = mark stale, refresh = reload from DB immediately |
| Explain hybrid properties | Python expression + SQL expression for same logic |

---

## 3. One-Page Cheat Sheet

### **Connection & Session**
```python
engine = create_engine("postgresql://user:pass@host/db", pool_size=5, echo=True)
Session = sessionmaker(bind=engine)
with Session() as session:
    ...
```

### **Model Definition**
```python
class Model(Base):
    __tablename__ = "models"
    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(100), nullable=False)
    parent_id: Mapped[int] = mapped_column(ForeignKey("parents.id"))
    parent: Mapped["Parent"] = relationship(back_populates="children")
```

### **CRUD Operations**
| Operation | Code |
|-----------|------|
| Create | `session.add(obj)` / `session.add_all([...])` |
| Read (PK) | `session.get(Model, id)` |
| Read (Query) | `session.scalars(select(Model).where(...)).all()` |
| Update | `obj.field = value` (auto-tracked) |
| Delete | `session.delete(obj)` |
| Bulk Insert | `session.execute(insert(Model).values([{...}]))` |

### **Common Query Patterns**
```python
# Select all columns
select(Model)
# Select specific
select(Model.id, Model.name)
# Filter
.where(Model.field == value)
.where(and_(cond1, cond2))
.where(or_(cond1, cond2))
.where(Model.field.in_([1, 2, 3]))
.where(Model.name.like("%pattern%"))
.where(Model.field.is_(None))
# Order & Limit
.order_by(desc(Model.created_at))
.limit(10).offset(20)
# Aggregate
select(func.count(Model.id))
.group_by(Model.category)
.having(func.count(Model.id) > 5)
# Join
.join(Model.relationship)
.outerjoin(OtherModel)
.options(joinedload(Model.relationship))
```

### **Loading Strategies**
| Strategy | Use Case | SQL |
|----------|----------|-----|
| `lazy="select"` (default) | Rarely accessed | Separate SELECT on access |
| `joinedload()` | Always needed, to-one | JOIN in same query |
| `selectinload()` | Always needed, to-many | SELECT ... WHERE id IN (...) |
| `lazy="raise"` | Prevent N+1 | Raises exception |

### **Time Complexity Considerations**
| Operation | Complexity |
|-----------|-----------|
| Get by PK | O(1) lookup in identity map / O(log n) DB |
| Query all | O(n) |
| Add to session | O(1) |
| Flush (n dirty objects) | O(n) SQL statements |
| Relationship access (lazy) | O(1) query per access |

---

## 4. Best Practices & Common Mistakes

### ✅ **What Senior Engineers Do**

```python
# 1. Always use context managers
with Session(engine) as session:
    with session.begin():
        # operations here
        pass  # auto-commit on success, auto-rollback on exception

# 2. Use explicit loading strategies to prevent N+1
stmt = select(User).options(
    selectinload(User.posts).selectinload(Post.comments)
)

# 3. Use connection pooling appropriately
engine = create_engine(
    url,
    pool_pre_ping=True,  # Handle stale connections
    pool_recycle=3600    # Recycle connections hourly
)

# 4. Bulk operations for performance
session.execute(
    insert(User),
    [{"email": f"user{i}@example.com"} for i in range(1000)]
)

# 5. Proper dependency injection for testing
def get_db():
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()

# 6. Use hybrid properties for reusable logic
from sqlalchemy.ext.hybrid import hybrid_property

class User(Base):
    first_name: Mapped[str]
    last_name: Mapped[str]
    
    @hybrid_property
    def full_name(self) -> str:
        return f"{self.first_name} {self.last_name}"
    
    @full_name.expression
    def full_name(cls):
        return cls.first_name + " " + cls.last_name
```

### ❌ **Common Mistakes That Get You Rejected**

```python
# 1. N+1 Query Problem (CRITICAL)
# BAD - causes N+1 queries
users = session.scalars(select(User)).all()
for user in users:
    print(user.posts)  # Each access = 1 query!

# GOOD - eager loading
users = session.scalars(
    select(User).options(selectinload(User.posts))
).all()

# 2. Not closing sessions
# BAD - connection leak
session = Session(engine)
users = session.query(User).all()
# session never closed!

# GOOD
with Session(engine) as session:
    users = session.scalars(select(User)).all()

# 3. Using objects outside session
# BAD - DetachedInstanceError
def get_user():
    with Session(engine) as session:
        return session.get(User, 1)  # Session closed after return

user = get_user()
print(user.posts)  # ERROR: detached instance

# GOOD - eager load what you need
def get_user_with_posts():
    with Session(engine) as session:
        stmt = select(User).options(selectinload(User.posts)).where(User.id == 1)
        return session.scalar(stmt)

# 4. Mixing async and sync patterns
# BAD in async context
async def bad_async():
    session = Session(engine)  # WRONG - blocks event loop
    
# GOOD - use async engine and session
from sqlalchemy.ext.asyncio import create_async_engine, AsyncSession
async_engine = create_async_engine("postgresql+asyncpg://...")

# 5. Not handling transactions properly
# BAD - partial commits possible
session.add(user1)
session.commit()
session.add(user2)
# crash here = user1 committed, user2 lost

# GOOD - atomic transaction
with session.begin():
    session.add(user1)
    session.add(user2)
    # all or nothing

# 6. Raw SQL without parameterization (SQL INJECTION!)
# BAD
session.execute(text(f"SELECT * FROM users WHERE email = '{email}'"))

# GOOD
session.execute(text("SELECT * FROM users WHERE email = :email"), {"email": email})
```

---

## 5. Minimal but Complete Code Examples

### **Example 1: Complete CRUD API Pattern**

```python
"""
Complete repository pattern for User model - production-ready pattern
"""
from contextlib import contextmanager
from typing import Optional, List
from sqlalchemy import create_engine, select, String
from sqlalchemy.orm import Session, sessionmaker, DeclarativeBase, Mapped, mapped_column

# Database setup
engine = create_engine("sqlite:///./app.db", echo=True)
SessionLocal = sessionmaker(bind=engine, autoflush=False)

class Base(DeclarativeBase):
    pass

class User(Base):
    __tablename__ = "users"
    id: Mapped[int] = mapped_column(primary_key=True)
    email: Mapped[str] = mapped_column(String(255), unique=True, index=True)
    name: Mapped[Optional[str]] = mapped_column(String(100))
    is_active: Mapped[bool] = mapped_column(default=True)

# Repository pattern
class UserRepository:
    def __init__(self, session: Session):
        self.session = session
    
    def create(self, email: str, name: str = None) -> User:
        user = User(email=email, name=name)
        self.session.add(user)
        self.session.flush()  # Get ID without committing
        return user
    
    def get_by_id(self, user_id: int) -> Optional[User]:
        return self.session.get(User, user_id)
    
    def get_by_email(self, email: str) -> Optional[User]:
        stmt = select(User).where(User.email == email)
        return self.session.scalar(stmt)
    
    def list_active(self, limit: int = 100, offset: int = 0) -> List[User]:
        stmt = (
            select(User)
            .where(User.is_active == True)
            .order_by(User.id)
            .limit(limit)
            .offset(offset)
        )
        return list(self.session.scalars(stmt))
    
    def update(self, user: User, **kwargs) -> User:
        for key, value in kwargs.items():
            setattr(user, key, value)
        return user
    
    def delete(self, user: User) -> None:
        self.session.delete(user)

# Context manager for database sessions
@contextmanager
def get_db():
    session = SessionLocal()
    try:
        yield session
        session.commit()
    except Exception:
        session.rollback()
        raise
    finally:
        session.close()

# Usage
if __name__ == "__main__":
    Base.metadata.create_all(engine)
    
    with get_db() as session:
        repo = UserRepository(session)
        
        # Create
        user = repo.create("alice@example.com", "Alice")
        print(f"Created: {user.id}")
        
        # Read
        found = repo.get_by_email("alice@example.com")
        print(f"Found: {found.name}")
        
        # Update
        repo.update(found, name="Alice Smith")
        print(f"Updated: {found.name}")
        
        # List
        active_users = repo.list_active()
        print(f"Active users: {len(active_users)}")
```

---

### **Example 2: Relationships with Eager Loading**

```python
"""
Complex relationships with proper loading strategies
"""
from typing import List, Optional
from sqlalchemy import create_engine, select, ForeignKey, String, Text
from sqlalchemy.orm import (
    Session, DeclarativeBase, Mapped, mapped_column, 
    relationship, selectinload, joinedload
)

engine = create_engine("sqlite:///:memory:", echo=True)

class Base(DeclarativeBase):
    pass

class Author(Base):
    __tablename__ = "authors"
    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(100))
    
    # One-to-many: Author has many books
    books: Mapped[List["Book"]] = relationship(
        back_populates="author",
        cascade="all, delete-orphan"
    )

class Book(Base):
    __tablename__ = "books"
    id: Mapped[int] = mapped_column(primary_key=True)
    title: Mapped[str] = mapped_column(String(200))
    author_id: Mapped[int] = mapped_column(ForeignKey("authors.id"))
    
    # Many-to-one: Book belongs to one Author
    author: Mapped["Author"] = relationship(back_populates="books")
    
    # One-to-many: Book has many reviews
    reviews: Mapped[List["Review"]] = relationship(
        back_populates="book",
        cascade="all, delete-orphan"
    )

class Review(Base):
    __tablename__ = "reviews"
    id: Mapped[int] = mapped_column(primary_key=True)
    content: Mapped[str] = mapped_column(Text)
    rating: Mapped[int]
    book_id: Mapped[int] = mapped_column(ForeignKey("books.id"))
    
    book: Mapped["Book"] = relationship(back_populates="reviews")

# Usage with proper eager loading
Base.metadata.create_all(engine)

with Session(engine) as session:
    # Create test data
    author = Author(name="J.K. Rowling")
    book1 = Book(title="Harry Potter 1", author=author)
    book1.reviews = [
        Review(content="Amazing!", rating=5),
        Review(content="Good read", rating=4)
    ]
    session.add(author)
    session.commit()
    
    # Query with nested eager loading - prevents N+1 completely
    stmt = (
        select(Author)
        .options(
            selectinload(Author.books)  # Load all books in one query
            .selectinload(Book.reviews)  # Load all reviews in one query
        )
    )
    
    authors = session.scalars(stmt).all()
    
    # Access relationships without additional queries
    for author in authors:
        print(f"Author: {author.name}")
        for book in author.books:
            print(f"  Book: {book.title}")
            for review in book.reviews:
                print(f"    Review: {review.rating}/5 - {review.content}")

# This generates only 3 queries total:
# 1. SELECT * FROM authors
# 2. SELECT * FROM books WHERE author_id IN (...)
# 3. SELECT * FROM reviews WHERE book_id IN (...)
```

---

### **Example 3: Transaction Management & Locking**

```python
"""
Advanced transaction patterns including savepoints and locking
"""
from sqlalchemy import create_engine, select, String
from sqlalchemy.orm import Session, DeclarativeBase, Mapped, mapped_column
from sqlalchemy.exc import IntegrityError

engine = create_engine("sqlite:///:memory:", echo=True)

class Base(DeclarativeBase):
    pass

class Account(Base):
    __tablename__ = "accounts"
    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(100))
    balance: Mapped[int] = mapped_column(default=0)

Base.metadata.create_all(engine)

def transfer_money(
    session: Session, 
    from_account_id: int, 
    to_account_id: int, 
    amount: int
) -> bool:
    """
    Transfer money between accounts with proper locking.
    Returns True if successful, False otherwise.
    """
    # Lock both accounts to prevent concurrent modifications
    # Note: SQLite doesn't support FOR UPDATE, use PostgreSQL in production
    from_account = session.get(Account, from_account_id)
    to_account = session.get(Account, to_account_id)
    
    if not from_account or not to_account:
        raise ValueError("Account not found")
    
    if from_account.balance < amount:
        raise ValueError("Insufficient funds")
    
    from_account.balance -= amount
    to_account.balance += amount
    
    return True

def batch_create_with_savepoint(session: Session, accounts_data: list) -> list:
    """
    Create accounts, continue even if some fail using savepoints.
    """
    created = []
    failed = []
    
    for data in accounts_data:
        try:
            # Savepoint - if this fails, only this savepoint rolls back
            with session.begin_nested():
                account = Account(**data)
                session.add(account)
                session.flush()  # Force insert to check constraints
                created.append(account)
        except IntegrityError as e:
            failed.append({"data": data, "error": str(e)})
    
    return created, failed

# Usage
with Session(engine) as session:
    with session.begin():
        # Setup accounts
        acc1 = Account(name="Alice", balance=1000)
        acc2 = Account(name="Bob", balance=500)
        session.add_all([acc1, acc2])
    
    # Transfer in new transaction
    with session.begin():
        transfer_money(session, acc1.id, acc2.id, 200)
    
    # Verify
    session.refresh(acc1)
    session.refresh(acc2)
    print(f"Alice: ${acc1.balance}, Bob: ${acc2.balance}")  # 800, 700
    
    # Batch create with error handling
    with session.begin():
        created, failed = batch_create_with_savepoint(session, [
            {"name": "Charlie", "balance": 100},
            {"name": "Alice", "balance": 200},  # Will fail - duplicate name if unique
            {"name": "Diana", "balance": 300},
        ])
        print(f"Created {len(created)} accounts, {len(failed)} failed")
```

---

### **Example 4: Async SQLAlchemy (Modern Pattern)**

```python
"""
Async SQLAlchemy pattern - essential for FastAPI/async frameworks
"""
import asyncio
from typing import List
from sqlalchemy import String, select
from sqlalchemy.ext.asyncio import (
    create_async_engine, 
    AsyncSession, 
    async_sessionmaker
)
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column

# Async engine (note the async driver: asyncpg for PostgreSQL)
async_engine = create_async_engine(
    "sqlite+aiosqlite:///./async_app.db",  # Use asyncpg for PostgreSQL
    echo=True
)

async_session_factory = async_sessionmaker(
    async_engine, 
    expire_on_commit=False
)

class Base(DeclarativeBase):
    pass

class Product(Base):
    __tablename__ = "products"
    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(100))
    price: Mapped[int]

async def init_db():
    """Create tables asynchronously"""
    async with async_engine.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)

async def create_product(session: AsyncSession, name: str, price: int) -> Product:
    """Create a product asynchronously"""
    product = Product(name=name, price=price)
    session.add(product)
    await session.flush()
    return product

async def get_products(session: AsyncSession, min_price: int = 0) -> List[Product]:
    """Query products asynchronously"""
    stmt = select(Product).where(Product.price >= min_price).order_by(Product.name)
    result = await session.scalars(stmt)
    return list(result)

async def main():
    await init_db()
    
    # Using async context manager
    async with async_session_factory() as session:
        async with session.begin():
            # Create products
            await create_product(session, "Laptop", 999)
            await create_product(session, "Phone", 599)
            await create_product(session, "Tablet", 399)
        
        # Query
        products = await get_products(session, min_price=400)
        for p in products:
            print(f"{p.name}: ${p.price}")

# FastAPI integration example
"""
from fastapi import FastAPI, Depends

app = FastAPI()

async def get_db():
    async with async_session_factory() as session:
        yield session

@app.get("/products")
async def list_products(db: AsyncSession = Depends(get_db)):
    products = await get_products(db)
    return [{"id": p.id, "name": p.name, "price": p.price} for p in products]
"""

if __name__ == "__main__":
    asyncio.run(main())
```

---

### **Example 5: Testing Patterns**

```python
"""
Testing SQLAlchemy code properly - transaction rollback pattern
"""
import pytest
from sqlalchemy import create_engine, String, select
from sqlalchemy.orm import Session, DeclarativeBase, Mapped, mapped_column

class Base(DeclarativeBase):
    pass

class User(Base):
    __tablename__ = "users"
    id: Mapped[int] = mapped_column(primary_key=True)
    email: Mapped[str] = mapped_column(String(255), unique=True)

# Fixtures for testing
@pytest.fixture(scope="function")
def engine():
    """Create a fresh in-memory database for each test"""
    engine = create_engine("sqlite:///:memory:", echo=False)
    Base.metadata.create_all(engine)
    return engine

@pytest.fixture(scope="function")
def session(engine):
    """
    Creates a session with transaction rollback.
    Changes made in tests are rolled back automatically.
    """
    connection = engine.connect()
    transaction = connection.begin()
    session = Session(bind=connection)
    
    yield session
    
    session.close()
    transaction.rollback()
    connection.close()

# Example tests
class TestUserRepository:
    def test_create_user(self, session):
        user = User(email="test@example.com")
        session.add(user)
        session.flush()
        
        assert user.id is not None
        assert user.email == "test@example.com"
    
    def test_find_user_by_email(self, session):
        # Arrange
        user = User(email="find@example.com")
        session.add(user)
        session.flush()
        
        # Act
        stmt = select(User).where(User.email == "find@example.com")
        found = session.scalar(stmt)
        
        # Assert
        assert found is not None
        assert found.id == user.id
    
    def test_unique_email_constraint(self, session):
        from sqlalchemy.exc import IntegrityError
        
        session.add(User(email="unique@example.com"))
        session.flush()
        
        session.add(User(email="unique@example.com"))
        with pytest.raises(IntegrityError):
            session.flush()
    
    def test_update_user(self, session):
        user = User(email="old@example.com")
        session.add(user)
        session.flush()
        
        user.email = "new@example.com"
        session.flush()
        
        stmt = select(User).where(User.id == user.id)
        updated = session.scalar(stmt)
        assert updated.email == "new@example.com"

# Run with: pytest -v test_example.py
```

---

## 6. Recommended Problems to Solve Today

| # | Problem / Exercise | Difficulty | Focus Area |
|---|-------------------|------------|------------|
| 1 | Build a User-Posts-Comments model with proper relationships | Easy | Model design, relationships |
| 2 | Implement pagination with total count in a single query | Easy | Querying, window functions |
| 3 | Create a generic repository base class with CRUD operations | Medium | Design patterns, generics |
| 4 | Solve N+1 query problem in a 3-level nested relationship | Medium | Eager loading strategies |
| 5 | Implement soft delete with automatic filtering | Medium | Mixins, query events |
| 6 | Build a connection pool monitor that logs pool statistics | Medium | Engine events, monitoring |
| 7 | Implement optimistic locking with retry logic | Hard | Concurrency, version columns |
| 8 | Design a multi-tenant system with row-level isolation | Hard | Query hooks, security |
| 9 | Create an audit log system using SQLAlchemy events | Hard | Events, triggers |
| 10 | Build async API with FastAPI + SQLAlchemy supporting bulk operations | Hard | Async patterns, performance |

---

## 7. 30-Minute Quiz

**Instructions:** Answer without looking at notes. Check answers at the end.

### Questions

**Q1.** What is the difference between `session.flush()` and `session.commit()`?

**Q2.** Write the modern SQLAlchemy 2.0 syntax to select all users where `is_active=True` ordered by `created_at` descending.

**Q3.** What is the N+1 query problem? How do you solve it with `selectinload`?

**Q4.** What are the four states an object can be in during its lifecycle in a Session?

**Q5.** Given this model:
```python
class Order(Base):
    __tablename__ = "orders"
    id: Mapped[int] = mapped_column(primary_key=True)
    items: Mapped[List["OrderItem"]] = relationship(back_populates="order")
```
Write the query to get all orders with their items eagerly loaded.

**Q6.** What's wrong with this code?
```python
def get_user(id):
    with Session(engine) as session:
        return session.get(User, id)

user = get_user(1)
print(user.posts)  # User has a posts relationship
```

**Q7.** How do you configure connection pool recycling to handle stale connections?

**Q8.** What is the difference between `joinedload()` and `selectinload()`? When would you use each?

**Q9.** Write code to perform a bulk insert of 1000 users efficiently.

**Q10.** How do you implement pessimistic locking (SELECT FOR UPDATE) in SQLAlchemy?

---

### Answers

<details>
<summary>Click to reveal answers</summary>

**A1.** 
- `flush()` sends pending SQL to the database within the current transaction (changes visible to current session, not committed)
- `commit()` calls `flush()` then commits the transaction, making changes permanent and visible to other sessions

**A2.**
```python
from sqlalchemy import select, desc
stmt = select(User).where(User.is_active == True).order_by(desc(User.created_at))
users = session.scalars(stmt).all()
```

**A3.** 
N+1 problem: Loading N parent objects then issuing 1 additional query for each parent's relationships = N+1 queries.
```python
# Solution with selectinload
stmt = select(User).options(selectinload(User.posts))
users = session.scalars(stmt).all()
# Now loads all users in 1 query + all posts in 1 query = 2 queries total
```

**A4.** Transient → Pending → Persistent → Detached

**A5.**
```python
stmt = select(Order).options(selectinload(Order.items))
orders = session.scalars(stmt).all()
```

**A6.** The session is closed before accessing `user.posts`. This causes `DetachedInstanceError` because the lazy-loaded relationship can't be fetched without a session. Fix by eager loading:
```python
def get_user(id):
    with Session(engine) as session:
        stmt = select(User).options(selectinload(User.posts)).where(User.id == id)
        return session.scalar(stmt)
```

**A7.**
```python
engine = create_engine(
    "postgresql://...",
    pool_pre_ping=True,  # Test connection validity
    pool_recycle=3600    # Recycle after 1 hour
)
```

**A8.**
- `joinedload()`: Uses SQL JOIN, best for many-to-one or one-to-one (single query)
- `selectinload()`: Uses separate SELECT with IN clause, best for one-to-many or many-to-many (prevents cartesian product)

**A9.**
```python
from sqlalchemy import insert
session.execute(
    insert(User),
    [{"email": f"user{i}@example.com", "name": f"User {i}"} for i in range(1000)]
)
session.commit()
```

**A10.**
```python
stmt = select(User).where(User.id == 1).with_for_update()
user = session.scalar(stmt)  # Row is locked until commit/rollback
```

</details>

---

## 8. Further Learning

### **Top Resource (Article/Video)**
📺 **"SQLAlchemy 2.0 - The One-Point-Four-Ture"** by Mike Bayer (SQLAlchemy creator)
- YouTube: SQLAlchemy 2.0 tutorial from PyCon
- Covers migration from 1.x to 2.0, new patterns, and internals

### **Official Documentation**
📖 **SQLAlchemy 2.0 Documentation**
- https://docs.sqlalchemy.org/en/20/
- Essential sections:
  - ORM Tutorial (2.0 style)
  - Session Basics
  - Relationship Loading Techniques
  - Engine Configuration

---

**Total Study Time Estimate:**
- Core Concepts: 3-4 hours
- Code Examples (typing + experimenting): 2-3 hours
- Practice Problems: 3-4 hours  
- Quiz + Review: 1 hour

**You're interview-ready for SQLAlchemy in ~10-12 focused hours.**
