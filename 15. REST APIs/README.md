# REST APIs

## What are REST APIs?

REST (Representational State Transfer) is an architectural style for designing web services that provides a standardized way for applications to communicate over the internet. Think of REST APIs like a well-organized restaurant menu - each item has a clear name, description, and price, and there are standard ways to order (GET), modify your order (PUT), add items (POST), or cancel items (DELETE). Just as the menu provides a consistent interface between customers and the kitchen, REST APIs provide a consistent interface between different software applications.

REST was introduced by Roy Fielding in his doctoral dissertation in 2000 and has since become the dominant architectural style for web APIs. It leverages existing web protocols and conventions, making it intuitive for developers who are already familiar with how the web works. REST APIs use standard HTTP methods and status codes, making them easy to understand, implement, and debug.

## Core REST Principles

### Statelessness

One of the fundamental principles of REST is that each request must contain all the information necessary to process it. The server doesn't store any information about previous requests from the client. Think of it like ordering from a drive-through restaurant - each time you place an order, you need to provide complete information (what you want, how you'll pay, where to deliver) because the staff doesn't remember your previous visits.

This statelessness makes REST APIs highly scalable because any server can handle any request without needing to know about the client's history. It also makes the system more reliable because there's no session state that can be lost if a server fails.

### Resource-Based URLs

REST treats everything as a resource that can be identified by a URL. Instead of thinking about actions or procedures, you think about the things (resources) your application manages and how to interact with them. For example, instead of having URLs like `/getUserById` or `/updateUserEmail`, you have `/users/123` where the HTTP method determines what action to perform.

This approach makes APIs more intuitive and predictable. Once you understand the resource structure, you can often guess how to interact with new resources without reading extensive documentation.

### HTTP Methods as Actions

REST uses standard HTTP methods to indicate what action to perform on a resource:
- **GET** retrieves data without changing anything
- **POST** creates new resources
- **PUT** updates entire resources or creates them if they don't exist
- **PATCH** partially updates existing resources
- **DELETE** removes resources

This standardization means that developers can understand what an API does just by looking at the HTTP method and URL, making REST APIs self-documenting to a large extent.

### Uniform Interface

REST APIs should have a consistent, predictable interface across all resources. Similar operations should work similarly regardless of which resource you're working with. If you know how to retrieve a user with `GET /users/123`, you should be able to retrieve an order with `GET /orders/456` using the same pattern.

## HTTP Status Codes in REST

REST APIs use standard HTTP status codes to communicate the result of operations, providing a universal language for describing what happened with each request.

### Success Codes (2xx)
- **200 OK**: The request was successful and the server returned the requested data
- **201 Created**: A new resource was successfully created (typically used with POST)
- **204 No Content**: The request was successful but there's no content to return (often used with DELETE)

### Client Error Codes (4xx)
- **400 Bad Request**: The request was malformed or invalid
- **401 Unauthorized**: Authentication is required but wasn't provided or was invalid
- **403 Forbidden**: The request is understood but the server refuses to authorize it
- **404 Not Found**: The requested resource doesn't exist
- **409 Conflict**: The request conflicts with the current state of the resource

### Server Error Codes (5xx)
- **500 Internal Server Error**: The server encountered an unexpected error
- **502 Bad Gateway**: The server received an invalid response from an upstream server
- **503 Service Unavailable**: The server is temporarily unable to handle requests

## Real-World REST API Examples

### Social Media Platform

A social media platform's REST API might be organized around resources like users, posts, and comments:

- `GET /users/123` retrieves user profile information
- `POST /users/123/posts` creates a new post for that user
- `GET /posts/456/comments` retrieves all comments on a specific post
- `PUT /users/123/profile` updates the user's profile information
- `DELETE /posts/456` removes a specific post

This structure makes it easy for mobile apps, web clients, and third-party integrations to interact with the platform using familiar, predictable patterns.

### E-commerce API

An e-commerce platform might organize its API around products, customers, orders, and inventory:

- `GET /products?category=electronics&price_max=500` searches for products
- `POST /customers/789/orders` creates a new order for a customer
- `GET /orders/101/status` checks the status of a specific order
- `PATCH /products/202` updates specific product details like price or description
- `DELETE /customers/789/cart/items/303` removes an item from a shopping cart

### Banking API

A banking API needs to handle accounts, transactions, and transfers securely:

- `GET /accounts/555/balance` retrieves current account balance
- `POST /accounts/555/transactions` creates a new transaction (deposit or withdrawal)
- `GET /transactions?from_date=2023-01-01&to_date=2023-12-31` retrieves transaction history
- `POST /transfers` initiates a transfer between accounts
- `GET /transfers/777/status` checks the status of a pending transfer

## Simple REST API Implementation

```python
# Flask REST API example
from flask import Flask, jsonify, request

app = Flask(__name__)

# In-memory data store (in real apps, use a database)
users = {
    1: {"id": 1, "name": "John Doe", "email": "john@example.com"},
    2: {"id": 2, "name": "Jane Smith", "email": "jane@example.com"}
}
next_user_id = 3

# GET /users - Retrieve all users
@app.route('/users', methods=['GET'])
def get_users():
    return jsonify(list(users.values())), 200

# GET /users/<id> - Retrieve specific user
@app.route('/users/<int:user_id>', methods=['GET'])
def get_user(user_id):
    user = users.get(user_id)
    if user:
        return jsonify(user), 200
    else:
        return jsonify({"error": "User not found"}), 404

# POST /users - Create new user
@app.route('/users', methods=['POST'])
def create_user():
    global next_user_id
    data = request.get_json()
    
    if not data or 'name' not in data or 'email' not in data:
        return jsonify({"error": "Name and email are required"}), 400
    
    new_user = {
        "id": next_user_id,
        "name": data['name'],
        "email": data['email']
    }
    users[next_user_id] = new_user
    next_user_id += 1
    
    return jsonify(new_user), 201

# PUT /users/<id> - Update entire user
@app.route('/users/<int:user_id>', methods=['PUT'])
def update_user(user_id):
    if user_id not in users:
        return jsonify({"error": "User not found"}), 404
    
    data = request.get_json()
    if not data or 'name' not in data or 'email' not in data:
        return jsonify({"error": "Name and email are required"}), 400
    
    users[user_id] = {
        "id": user_id,
        "name": data['name'],
        "email": data['email']
    }
    
    return jsonify(users[user_id]), 200

# DELETE /users/<id> - Delete user
@app.route('/users/<int:user_id>', methods=['DELETE'])
def delete_user(user_id):
    if user_id not in users:
        return jsonify({"error": "User not found"}), 404
    
    del users[user_id]
    return '', 204

if __name__ == '__main__':
    app.run(debug=True)
```

## Common Interview Questions

**Q: What is REST and what are its main principles?**

REST (Representational State Transfer) is an architectural style for web services that uses standard HTTP methods and follows key principles: statelessness (each request contains all necessary information), resource-based URLs (everything is treated as a resource with a unique identifier), uniform interface (consistent patterns across all resources), and leveraging HTTP methods for actions (GET for retrieval, POST for creation, PUT for updates, DELETE for removal). REST makes APIs predictable, scalable, and easy to understand because it builds on familiar web concepts.

**Q: What's the difference between PUT and PATCH methods?**

PUT is used for complete resource replacement - you send the entire resource representation and the server replaces the existing resource completely. PATCH is used for partial updates where you only send the fields you want to change. For example, PUT /users/123 with {"name": "John", "email": "john@example.com"} replaces the entire user record, while PATCH /users/123 with {"email": "newemail@example.com"} only updates the email field. PUT is idempotent (calling it multiple times has the same effect), while PATCH may or may not be idempotent depending on implementation.

**Q: How do you handle authentication in REST APIs?**

Common authentication methods include: API keys (passed in headers or query parameters), JWT tokens (stateless tokens containing user information), OAuth 2.0 (industry standard for authorization), and Basic Authentication (username/password in Authorization header). Most modern APIs use Bearer tokens where the client includes "Authorization: Bearer <token>" in request headers. The choice depends on security requirements, scalability needs, and integration complexity. REST's stateless nature means authentication information must be included with each request.

**Q: What are the advantages and disadvantages of REST APIs?**

Advantages include: simplicity and ease of understanding, wide language and platform support, statelessness enabling scalability, caching capabilities through HTTP, and flexibility in data formats (JSON, XML). Disadvantages include: over-fetching or under-fetching data (getting too much or too little), multiple round trips for complex operations, limited real-time capabilities, and potential for chatty interfaces requiring many API calls. These limitations have led to alternatives like GraphQL for complex data requirements and WebSockets for real-time communication.

## REST API Design Best Practices

### Use Consistent Naming Conventions

Use nouns for resource names and make them plural for collections. For example, use `/users` instead of `/user` for a collection of users, and `/users/123` for a specific user. This consistency makes your API intuitive and predictable.

Keep URLs simple and hierarchical to represent relationships between resources. For nested resources, use patterns like `/users/123/posts` to represent posts belonging to a specific user.

### Implement Proper Error Handling

Always return appropriate HTTP status codes and include meaningful error messages in the response body. This helps API consumers understand what went wrong and how to fix it.

```json
{
  "error": "Validation failed",
  "message": "Email address is required",
  "code": "MISSING_EMAIL",
  "details": {
    "field": "email",
    "value": null
  }
}
```

### Support Filtering, Sorting, and Pagination

For collection endpoints, support query parameters for filtering, sorting, and pagination to handle large datasets efficiently:

- `GET /products?category=electronics&sort=price&order=asc&page=2&limit=20`

This allows clients to retrieve only the data they need and improves performance for both client and server.

### Version Your APIs

Plan for API evolution by including version information in your URLs or headers. This allows you to make breaking changes while maintaining backward compatibility for existing clients:

- `GET /v1/users/123` or `GET /users/123` with `Accept: application/vnd.api+json;version=1`

### Use HATEOAS (Hypermedia as the Engine of Application State)

Include links in your API responses to help clients discover available actions and navigate the API:

```json
{
  "id": 123,
  "name": "John Doe",
  "email": "john@example.com",
  "_links": {
    "self": "/users/123",
    "posts": "/users/123/posts",
    "edit": "/users/123"
  }
}
```

## REST vs Other API Styles

### REST vs SOAP

SOAP (Simple Object Access Protocol) is a protocol that uses XML for message format and can work over various transport protocols. While SOAP provides strong typing and built-in error handling, REST is simpler, lighter weight, and easier to cache. REST has largely replaced SOAP for new web services due to its simplicity and better performance.

### REST vs GraphQL

GraphQL allows clients to request exactly the data they need in a single request, solving REST's over-fetching and under-fetching problems. However, REST is simpler to implement and cache, making it still suitable for many use cases. The choice depends on your specific requirements for data flexibility versus simplicity.

### REST vs RPC

RPC (Remote Procedure Call) APIs focus on actions or functions rather than resources. While RPC can be more intuitive for certain operations, REST's resource-based approach tends to be more scalable and maintainable for complex systems with many interrelated entities.

Understanding REST APIs is essential for modern backend development because they provide a standard, scalable way for different systems to communicate. While newer alternatives like GraphQL and gRPC have their place, REST remains the foundation for most web services due to its simplicity, broad support, and alignment with web architecture principles.
