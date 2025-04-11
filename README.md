# 📚 DynamoDB Training Guide – Users Table (with Python and Boto3)

This guide covers **all essential DynamoDB operations**. Each function is written in **Python using boto3**, and includes proper error handling using `try/except` blocks.

---

## 🧾 Table Schema

This guide assumes you're working with a DynamoDB table called `Users` with the following structure:

| Attribute           | Type     | Description                           |
|---------------------|----------|---------------------------------------|
| `userId`            | String   | Primary key, unique for each user     |
| `organizationId`    | String   | ID of the organization the user belongs to |
| `firstName`         | String   | User’s first name                     |
| `lastName`          | String   | User’s last name                      |
| `phone`             | Number   | Contact number                        |
| `membershipExpired` | Boolean  | Whether their membership is active    |

---

## ⚙️ Setup

Before using the examples in this guide, make sure you've set up your environment correctly.

### 1. 📄 Create `requirements.txt`

In the same folder where your Lambda function is located, create a file named `requirements.txt` and add the following:

```
boto3
```

This ensures that your Lambda function has access to the AWS SDK for Python.

### 2. 🔌 Initialize DynamoDB Connection

Add the following snippet at the top of your Python file to initialize the DynamoDB connection:

```python
import boto3

dynamodb = boto3.resource("dynamodb")
users_table = dynamodb.Table("Users")
```
---

## 📌 1. Add a New User

This function creates a new user in the table.

```python
def add_user(user_id: str, organization_id: str, first_name: str, last_name: str, phone: int, membership_expired: bool) -> bool:
    try:
        users_table.put_item(
            Item={
                "userId": user_id,
                "organizationId": organization_id,
                "firstName": first_name,
                "lastName": last_name,
                "phone": phone,
                "membershipExpired": membership_expired
            }
        )
        print(f"User {user_id} added.")
        return True
    except Exception as e:
        print(f"Failed to add user: {e}")
        return False
```

---

## 🪪 2. Update a User's membership status

This function updates the `membershipExpired` field for a specific user in the `Users` table.  
It takes the `userId` and a `True` or `False` value to mark whether the user's membership is expired.

For example:
- `True` means the membership is **expired**
- `False` means the membership is **active**

It uses `update_item` to modify only the `membershipExpired` field without affecting other user data.

```python
def update_membership_status(user_id: str, expired: bool) -> bool:
    try:
        users_table.update_item(
            Key={"userId": user_id},
            UpdateExpression="SET membershipExpired = :status",
            ExpressionAttributeValues={":status": expired}
        )
        print(f"✅ Membership status updated for user {user_id} → expired = {expired}")
        return True
    except Exception as e:
        print(f"❌ Failed to update membership status: {e}")
        return False
```
--- 

## 🧑‍💼 3. Update a User's first name and last name

This will change the user's `firstName` and `lastName` while leaving the rest of their data unchanged.

```python
def update_user_name(user_id: str, first_name: str, last_name: str) -> bool:
    try:
        users_table.update_item(
            Key={"userId": user_id},
            UpdateExpression="SET firstName = :fn, lastName = :ln",
            ExpressionAttributeValues={
                ":fn": first_name,
                ":ln": last_name
            }
        )
        print(f"✅ Name updated for user {user_id}: {first_name} {last_name}")
        return True
    except Exception as e:
        print(f"❌ Failed to update name: {e}")
        return False
```

---

## ✏️ 4. Update a User's Info

This updates any combination of first name, last name, or phone.

```python
def update_user(user_id: str, first_name=None, last_name=None, phone=None) -> bool:
    try:
        update_expr = []
        expr_values = {}

        if first_name:
            update_expr.append("firstName = :fn")
            expr_values[":fn"] = first_name
        if last_name:
            update_expr.append("lastName = :ln")
            expr_values[":ln"] = last_name
        if phone:
            update_expr.append("phone = :ph")
            expr_values[":ph"] = phone

        if not update_expr:
            print("Nothing to update.")
            return False

        users_table.update_item(
            Key={"userId": user_id},
            UpdateExpression="SET " + ", ".join(update_expr),
            ExpressionAttributeValues=expr_values
        )
        print(f"User {user_id} updated.")
        return True
    except Exception as e:
        print(f"Failed to update user: {e}")
        return False
```

---

## 🗑️ 5. Delete a User

```python
def delete_user(user_id: str) -> bool:
    try:
        users_table.delete_item(Key={"userId": user_id})
        print(f"User {user_id} deleted.")
        return True
    except Exception as e:
        print(f"Failed to delete user: {e}")
        return False
```

---

## 🔍 6. Get a User by ID

```python
def get_user(user_id: str):
    try:
        response = users_table.get_item(Key={"userId": user_id})
        user = response.get("Item")
        if user:
            print(f"User found: {user}")
        else:
            print("User not found.")
        return user
    except Exception as e:
        print(f"Failed to get user: {e}")
        return None
```

---

## 👥 7. Get Users by Organization ID

```python
def get_users_by_organization(org_id: str):
    try:
        response = users_table.scan(
            FilterExpression="organizationId = :org_id",
            ExpressionAttributeValues={":org_id": org_id}
        )
        users = response.get("Items", [])
        print(f"Found {len(users)} user(s) in organization {org_id}")
        return users
    except Exception as e:
        print(f"Failed to query users: {e}")
        return []
```

---

## 🗑️ 8. Delete All Users of an Organization

```python
def delete_users_by_organization(org_id: str) -> bool:
    try:
        # Step 1: Get all users belonging to the organization
        response = users_table.scan(
            FilterExpression="organizationId = :org_id",
            ExpressionAttributeValues={":org_id": org_id}
        )
        users = response.get("Items", [])
        
        if not users:
            print(f"No users found in organization {org_id}.")
            return False

        # Step 2: Delete each user by their userId
        for user in users:
            user_id = user["userId"]
            users_table.delete_item(Key={"userId": user_id})
            print(f"Deleted user {user_id} from organization {org_id}")

        print(f"✅ Deleted {len(users)} users from organization {org_id}")
        return True

    except Exception as e:
        print(f"❌ Failed to delete users: {e}")
        return False
```

---

### 🧑‍💻 Active vs. Inactive Users in Our System

We have **two types of users** in our `Users` table:

1. **Active Users** – Registered users with normal `userId` values (e.g., `"user123"`).
2. **Inactive Users** – Users who have **not completed registration** yet. Their `userId` starts with a special prefix `";;$#"` followed by a unique invitation code (e.g., `";;$#abc123"`).

## 💤 9. Get All Inactive Users (Invitation Codes)

Inactive users have `userId` values starting with `;;$#`.

```python
def get_inactive_users():
    try:
        response = users_table.scan(
            FilterExpression="begins_with(userId, :prefix)",
            ExpressionAttributeValues={":prefix": ";;$#"}
        )
        users = response.get("Items", [])
        print(f"Found {len(users)} inactive user(s).")
        return users
    except Exception as e:
        print(f"Failed to get inactive users: {e}")
        return []
```

---

## 🔄 10. Check if the invitation code is valid:

```python
def activate_user(invitation_code: str, new_user_id: str) -> bool:
    try:
        # 🔍 Get all inactive users (userIds that start with ';;$#')
        users = get_inactive_users()

        for user in users:
            user_id = user.get("userId", "")

            # ✅ Ensure the userId starts with the expected prefix
            if user_id.startswith(";;$#"):
                # 🧩 Extract the actual code (remove ';;$#')
                extracted_code = user_id[len(";;$#"):]

                # 🔁 Compare with provided invitation_code
                if extracted_code == invitation_code:
                    # 🧬 Copy the user, assign the real userId
                    new_item = user.copy()
                    new_item["userId"] = new_user_id

                    # ✅ Add new user record and delete old invitation user
                    users_table.put_item(Item=new_item)
                    users_table.delete_item(Key={"userId": user_id})

                    print(f"User activated: {invitation_code} → {new_user_id}")
                    return True

        print("❌ Invitation code not found.")
        return False

    except Exception as e:
        print(f"❌ Failed to activate user: {e}")
        return False
```

---

## ⏳ 11. Expire User Membership

```python
def expire_membership(user_id: str) -> bool:
    try:
        users_table.update_item(
            Key={"userId": user_id},
            UpdateExpression="SET membershipExpired = :val",
            ExpressionAttributeValues={":val": True}
        )
        print(f"Membership expired for user {user_id}.")
        return True
    except Exception as e:
        print(f"Failed to expire membership: {e}")
        return False
```

---

## 📊 12. Count Total Users

Supports pagination to get the full count even for large datasets.

```python
def count_all_users() -> int:
    try:
        count = 0

        # 🔍 Perform the initial scan of the table (up to 1MB of data)
        response = users_table.scan()
        count += len(response.get("Items", []))

        # 🔁 DynamoDB returns only up to 1MB of data per scan.
        # If there are more items, the response includes 'LastEvaluatedKey'.
        # This loop continues scanning the table using pagination until all users are counted.
        while 'LastEvaluatedKey' in response:
            # 📌 Continue scanning from where the previous scan left off
            response = users_table.scan(ExclusiveStartKey=response['LastEvaluatedKey'])
            count += len(response.get("Items", []))  # 🧮 Add number of items in this batch

        # ✅ After all pages are scanned, print the total count
        print(f"Total users: {count}")
        return count

    except Exception as e:
        # ❌ Handle and log any errors during the scan process
        print(f"Failed to count users: {e}")
        return 0
```

---

## 🖨️ 13. Print All Rows from Users Table

```python
def print_all_users() -> None:
    try:
        # Step 1: Initial scan
        response = users_table.scan()
        users = response.get("Items", [])

        # Step 2: Print each user in the first batch
        for user in users:
            print(user)

        # Step 3: Keep scanning if there are more items (pagination)
        while 'LastEvaluatedKey' in response:
            response = users_table.scan(ExclusiveStartKey=response['LastEvaluatedKey'])
            users = response.get("Items", [])
            for user in users:
                print(user)

        print("✅ Finished printing all users.")
    
    except Exception as e:
        print(f"❌ Failed to print users: {e}")
```

