# API Documentation

## API Endpoints

### 1. Get User Info
- **Endpoint:** `/api/user`
- **Method:** `GET`
- **Description:** Retrieves information about the current user.
- **Response:** 200 OK
  ```json
  {
    "id": 1,
    "name": "John Doe",
    "email": "john.doe@example.com"
  }
  ```

### 2. Update User Info
- **Endpoint:** `/api/user`
- **Method:** `PUT`
- **Description:** Updates information for the current user.
- **Request Body:**
  ```json
  {
    "name": "Jane Doe",
    "email": "jane.doe@example.com"
  }
  ```
- **Response:** 200 OK
  ```json
  {
    "message": "User information updated successfully"
  }
  ```

### 3. Get All Users
- **Endpoint:** `/api/users`
- **Method:** `GET`
- **Description:** Retrieves a list of all users.
- **Response:** 200 OK
  ```json
  [
    {
      "id": 1,
      "name": "John Doe",
      "email": "john.doe@example.com"
    },
    {
      "id": 2,
      "name": "Jane Smith",
      "email": "jane.smith@example.com"
    }
  ]
  ```

### 4. Delete User
- **Endpoint:** `/api/user/{id}`
- **Method:** `DELETE`
- **Description:** Deletes a user by ID.
- **Response:** 204 No Content

## Examples
### Example Request to Get User Info
```bash
curl -X GET http://localhost:8000/api/user
```

### Example Request to Update User Info
```bash
curl -X PUT http://localhost:8000/api/user\
-H "Content-Type: application/json"\
-d '{"name":"Jane Doe","email":"jane.doe@example.com"}'
```

### Example Request to Get All Users
```bash
curl -X GET http://localhost:8000/api/users
```

### Example Request to Delete User
```bash
curl -X DELETE http://localhost:8000/api/user/1
```
