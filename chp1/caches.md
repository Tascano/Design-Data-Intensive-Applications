Remember the result of an expensive operation, to speed up reads (caches)

Below is a example of a FastAPI service that stores and retrieves user data from DynamoDB with caching using Redis. 
Please note that this example assumes you have set up a DynamoDB table and a Redis server.

```
from fastapi import FastAPI
import redis
import json
import boto3
from botocore.exceptions import ClientError

# Initialize FastAPI app
app = FastAPI()

# Initialize Redis client
redis_client = redis.Redis(host='localhost', port=6379, db=0)

# Initialize DynamoDB client
dynamodb = boto3.resource('dynamodb', region_name='us-east-1')  # Replace with your region
table_name = 'UserDataTable'  # Replace with your DynamoDB table name
table = dynamodb.Table(table_name)

def fetch_user_data_from_db(user_id):
    try:
        response = table.get_item(Key={'user_id': user_id})
        item = response.get('Item')
        return item
    except ClientError as e:
        print(f"Error fetching data from DynamoDB: {e}")
        return None

@app.get("/user/{user_id}")
def get_user_data(user_id: str):
    # Check if data is in the cache
    cached_data = redis_client.get(user_id)
    
    if cached_data:
        return json.loads(cached_data)
    else:
        user_data = fetch_user_data_from_db(user_id)
        if user_data:
            # Cache the data for 5 minutes
            redis_client.setex(user_id, 300, json.dumps(user_data))
            return user_data
        else:
            return {"message": "User not found"}

@app.post("/user")
def save_user_data(user_data: dict):
    # Implement logic to save user data to DynamoDB
    try:
        table.put_item(Item=user_data)
    except ClientError as e:
        print(f"Error saving data to DynamoDB: {e}")
        return {"message": "Failed to save user data"}

    # Invalidate cache for this user since the data is updated
    redis_client.delete(user_data["user_id"])
    return {"message": "User data saved successfully"}

if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="127.0.0.1", port=8000)
```
