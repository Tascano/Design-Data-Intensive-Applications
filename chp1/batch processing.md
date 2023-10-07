Periodically crunch a large amount of accumulated data (batch processing)

```
# main.py
from fastapi import FastAPI
from sqlalchemy.orm import Session
from datetime import datetime, timedelta
from apscheduler.schedulers.background import BackgroundScheduler
from models import DATABASE_URL, SessionLocal, User

app = FastAPI()

# Function to calculate the average age of users created/updated in the last 24 hours
def calculate_average_age():
    db = SessionLocal()
    yesterday = datetime.now() - timedelta(days=1)
    users = (
        db.query(User)
        .filter(User.created_at >= yesterday)
        .all()
    )
    if not users:
        return None
    total_age = sum(user.age for user in users)
    average_age = total_age / len(users)
    return average_age

# Endpoint to get the average age
@app.get("/average_age")
def get_average_age():
    average_age = calculate_average_age()
    if average_age is None:
        return {"message": "No users found in the last 24 hours."}
    return {"average_age": average_age}

# Initialize and configure the APScheduler to run the job every 24 hours
scheduler = BackgroundScheduler()
scheduler.add_job(calculate_average_age, "interval", hours=24)
scheduler.start()

# Run the FastAPI app
if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=8000)

```
