
Allow users to search data by keyword or filter it in various ways (search indexes)


```
from fastapi import FastAPI, Query
from pydantic import BaseModel
from sqlalchemy import create_engine, Column, Integer, String, select
from databases import Database
from sqlalchemy.ext.declarative import declarative_base
from sqlalchemy.orm import sessionmaker

# Create a FastAPI app
app = FastAPI()

# Define a Pydantic model for tasks
class Task(BaseModel):
    id: int
    title: str
    description: str

# SQLite Database setup
DATABASE_URL = "sqlite:///./test.db"
database = Database(DATABASE_URL)
Base = declarative_base()

# Define a SQLAlchemy model for tasks
class TaskModel(Base):
    __tablename__ = "tasks"
    id = Column(Integer, primary_key=True, index=True)
    title = Column(String, index=True)
    description = Column(String, index=True)

engine = create_engine(DATABASE_URL, connect_args={"check_same_thread": False})
Base.metadata.create_all(bind=engine)

SessionLocal = sessionmaker(autocommit=False, autoflush=False, bind=engine)

# Create a new task
@app.post("/tasks/", response_model=Task)
async def create_task(task: Task):
    async with database.transaction():
        query = TaskModel.insert().values(**task.dict())
        task_id = await database.execute(query)
        return {**task.dict(), "id": task_id}

# Search for tasks by keyword
@app.get("/tasks/", response_model=list[Task])
async def search_tasks(keyword: str = Query(None)):
    async with database.transaction():
        if keyword:
            query = select(TaskModel).where(
                TaskModel.title.contains(keyword) | TaskModel.description.contains(keyword)
            )
        else:
            query = select(TaskModel)
        results = await database.fetch_all(query)
        return results

if __name__ == "__main__":
    import uvicorn

    uvicorn.run(app, host="0.0.0.0", port=8000, reload=True)


```



1. **Index Creation**: Indexes were created in the SQLAlchemy model for the `title` and `description` columns of the `TaskModel` class:

   ```python
   class TaskModel(Base):
       __tablename__ = "tasks"
       id = Column(Integer, primary_key=True, index=True)
       title = Column(String, index=True)
       description = Column(String, index=True)
   ```

By adding `index=True` to these columns, you instructed SQLAlchemy to create indexes for them in the SQLite database. These indexes improve the speed of searching for tasks by `title` and `description`.

2. **Index Usage in Search**: Indexes were used when searching for tasks by keyword in the `/tasks/` endpoint:

   ```python
   # Search for tasks by keyword
   @app.get("/tasks/", response_model=list[Task])
   async def search_tasks(keyword: str = Query(None)):
       async with database.transaction():
           if keyword:
               query = select(TaskModel).where(
                   TaskModel.title.contains(keyword) | TaskModel.description.contains(keyword)
               )
           else:
               query = select(TaskModel)
           results = await database.fetch_all(query)
           return results
   ```

Here, when you perform a search with a keyword, the code generates an SQL query that utilizes the indexes on the `title` and `description` columns to efficiently search for matching records.
This usage of indexes improves search performance by reducing the time required to retrieve relevant data from the database.

So, in summary, indexes were created in the SQLAlchemy model to improve database performance when searching by specific columns (`title` and `description`), 
and these indexes were used in the search query to speed up the search process.

Description usually is a larger string and as such not prefered for indexes. 
