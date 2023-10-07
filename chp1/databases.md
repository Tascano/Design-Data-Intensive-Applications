• Store data so that they, or another application, can find it again later (databases)

```

from fastapi import FastAPI, HTTPException, Query
from pydantic import BaseModel  # Import the User model

import sqlite3

app = FastAPI()

# Connect to or create a SQLite database file
conn = sqlite3.connect('mydatabase.db')

# Create a cursor object to execute SQL commands
cursor = conn.cursor()

# Create a table to store user data
cursor.execute('''
    CREATE TABLE IF NOT EXISTS users (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        username TEXT NOT NULL,
        email TEXT NOT NULL UNIQUE
    )
''')

# Commit changes and close the connection
conn.commit()
conn.close()

# Define the User model as a Pydantic model
class User(BaseModel):
    username: str
    email: str

@app.post("/user")
async def create_user(user: User):  # Use the User Pydantic model as the request body
    try:
        conn = sqlite3.connect('mydatabase.db')
        cursor = conn.cursor()
        cursor.execute('INSERT INTO users (username, email) VALUES (?, ?)', (user.username, user.email))
        conn.commit()
        conn.close()
        return {"message": "User created successfully"}
    except sqlite3.IntegrityError as e:
        return HTTPException(status_code=400, detail="Email already exists")

@app.get("/users")
async def read_users():
    try:
        conn = sqlite3.connect('mydatabase.db')
        cursor = conn.cursor()
        cursor.execute('SELECT * FROM users')
        users = cursor.fetchall()
        conn.close()
        return users
    except Exception as e:
        return HTTPException(status_code=500, detail="Internal Server Error")

@app.get("/user/{user_id}")
async def read_user(user_id: int):
    try:
        conn = sqlite3.connect('mydatabase.db')
        cursor = conn.cursor()
        cursor.execute('SELECT * FROM users WHERE id = ?', (user_id,))
        user = cursor.fetchone()
        conn.close()
        if user:
            return user
        else:
            raise HTTPException(status_code=404, detail="User not found")
    except Exception as e:
        return HTTPException(status_code=500, detail="Internal Server Error")
```


Use DDB 
```
import boto3
from botocore.exceptions import ClientError
from fastapi import FastAPI, HTTPException, Query
from pydantic import BaseModel

app = FastAPI()

dynamodb = boto3.resource('dynamodb', region_name='your-region')  # Replace 'your-region' with your AWS region
table_name = 'UserDataTable'

# Define the User Pydantic model
class User(BaseModel):
    username: str
    email: str

@app.post("/user")
async def create_user(user: User):
    try:
        table = dynamodb.Table(table_name)
        table.put_item(Item={
            'username': user.username,
            'email': user.email
        })
        return {"message": "User created successfully"}
    except ClientError as e:
        return HTTPException(status_code=500, detail="Failed to create user")

@app.get("/users")
async def read_users():
    try:
        table = dynamodb.Table(table_name)
        response = table.scan()
        return response.get('Items', [])
    except ClientError as e:
        return HTTPException(status_code=500, detail="Failed to fetch users")

# ... other endpoints and error handling ...

if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=8000)

```

Use Posgresql 

```
import asyncpg
from fastapi import FastAPI, HTTPException, Query
from pydantic import BaseModel

app = FastAPI()

# Modify these parameters to match your PostgreSQL configuration
postgres_config = {
    "database": "your-database",
    "user": "your-username",
    "password": "your-password",
    "host": "your-host",
    "port": "your-port"
}

# Define the User Pydantic model
class User(BaseModel):
    username: str
    email: str

async def create_pool():
    return await asyncpg.create_pool(**postgres_config)

@app.on_event("startup")
async def startup():
    app.db_pool = await create_pool()

@app.on_event("shutdown")
async def shutdown():
    await app.db_pool.close()

@app.post("/user")
async def create_user(user: User):
    async with app.db_pool.acquire() as conn:
        try:
            await conn.execute("INSERT INTO users (username, email) VALUES ($1, $2)", user.username, user.email)
            return {"message": "User created successfully"}
        except Exception as e:
            return HTTPException(status_code=500, detail="Failed to create user")

@app.get("/users")
async def read_users():
    async with app.db_pool.acquire() as conn:
        try:
            results = await conn.fetch("SELECT * FROM users")
            return results
        except Exception as e:
            return HTTPException(status_code=500, detail="Failed to fetch users")

# ... other endpoints and error handling ...

if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=8000)


```

Use postgresql on RDS

```
from fastapi import FastAPI, HTTPException, Query
from pydantic import BaseModel
from sqlalchemy import create_engine, Column, Integer, String
from sqlalchemy.ext.declarative import declarative_base
from sqlalchemy.orm import sessionmaker

app = FastAPI()

# Define the SQLAlchemy database models
Base = declarative_base()

class User(Base):
    __tablename__ = "users"
    id = Column(Integer, primary_key=True, index=True)
    username = Column(String, unique=True, index=True)
    email = Column(String, unique=True, index=True)

# Configure the SQLAlchemy connection URL with your RDS details
DATABASE_URL = "postgresql://your-username:your-password@your-rds-host:your-rds-port/your-database"

engine = create_engine(DATABASE_URL)

# Create the database tables
Base.metadata.create_all(bind=engine)

SessionLocal = sessionmaker(autocommit=False, autoflush=False, bind=engine)

# Define the User Pydantic model
class UserCreate(BaseModel):
    username: str
    email: str

@app.post("/user")
async def create_user(user: UserCreate):
    try:
        db = SessionLocal()
        db_user = User(**user.dict())
        db.add(db_user)
        db.commit()
        db.refresh(db_user)
        db.close()
        return {"message": "User created successfully"}
    except Exception as e:
        return HTTPException(status_code=500, detail="Failed to create user")

@app.get("/users")
async def read_users():
    try:
        db = SessionLocal()
        users = db.query(User).all()
        db.close()
        return users
    except Exception as e:
        return HTTPException(status_code=500, detail="Failed to fetch users")

# ... other endpoints and error handling ...

if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=8000)

```
