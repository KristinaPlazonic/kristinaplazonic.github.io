---
layout: post
title: MCP Server with OpenAPI: Automatically Exposing REST APIs as Tools
date: 2025-12-14

## 1. Introduction

I'm experimenting with standing up MCP servers at work that use REST APIs as targets. While it's possible to define tools one by one manually, FastMCP makes it even easier to dynamically load the entire OpenAPI schema from a REST API and automatically generate MCP tools. This approach scales much better when you have many endpoints, and it keeps your MCP server in sync with your API definitions.

The key insight: if you already have a well-documented REST API with OpenAPI/Swagger, you get MCP tools for free using `FastMCP.from_openapi()`.

## 2. FastAPI App as Our REST API Target

First, I set up a simple FastAPI app with a few routes to serve as our target API. The app runs on port 3000 and automatically generates an OpenAPI schema at `/openapi.json`.

```python
# app.py
from fastapi import FastAPI
from pydantic import BaseModel, Field

app = FastAPI(title="Target API", version="1.0.0")

class Item(BaseModel):
    name: str = Field(description="Name of the item")
    description: str | None = Field(None, description="Optional description of the item")
    price: float = Field(description="Price of the item in USD")

class User(BaseModel):
    username: str = Field(description="Username for the account")
    email: str = Field(description="Email address of the user")
    age: int | None = Field(None, description="Optional age of the user")

class UserResponse(BaseModel):
    id: int = Field(description="Unique identifier for the user")
    username: str = Field(description="Username for the account")
    email: str = Field(description="Email address of the user")
    status: str = Field(description="Current status of the user account")

@app.get("/", operation_id="root")
def root():
    """Get welcome message from the API"""
    return {"message": "Hello World"}

@app.get("/items/{item_id}", operation_id="read_item")
def read_item(item_id: int, q: str | None = None):
    """Retrieve an item by its ID with optional query parameter"""
    return {"item_id": item_id, "q": q}

@app.post("/items", operation_id="create_item")
def create_item(item: Item):
    """Create a new item with name, description, and price"""
    return {"item": item, "created": True}

@app.post("/users", operation_id="create_user", response_model=UserResponse)
def create_user(user: User):
    """Create a new user account with username, email, and optional age"""
    return UserResponse(
        id=123,
        username=user.username,
        email=user.email,
        status="active"
    )

if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=3000)
```

### Key Points

**Pydantic Field Descriptions Matter**: The `Field(description="...")` annotations are critical. These descriptions show up in the OpenAPI schema, which means they'll appear in the MCP Inspector and help MCP understand what each parameter does. Without them, your tools will work but won't have helpful context.

**operation_id Controls Tool Names**: By setting `operation_id` on each endpoint, you control exactly what the MCP tool will be named. Without this, FastAPI generates operation IDs automatically based on the function name and path, which can be verbose (like `create_item_items_post`). With explicit operation IDs, you get clean tool names like `create_item` and `create_user`.

```
Operation IDs in OpenAPI spec:
GET /: root
GET /items/{item_id}: read_item
POST /items: create_item
POST /users: create_user
```

**Docstrings Become Tool Descriptions**: The function docstrings map directly to tool descriptions in the MCP schema, giving Claude context about what each tool does.

## 3. MCP Server with OpenAPI Integration

The MCP server is remarkably simple thanks to FastMCP's `from_openapi()` method. It fetches the OpenAPI spec from the running FastAPI server and automatically generates tools for each endpoint.

```python
import httpx
from fastmcp import FastMCP

client = httpx.AsyncClient(base_url="http://localhost:3000")
openapi_spec = httpx.get("http://localhost:3000/openapi.json").json()

mcp = FastMCP.from_openapi(
    openapi_spec=openapi_spec,
    client=client,
    name="Target API"
)

if __name__ == "__main__":
    mcp.run(transport="http", host="0.0.0.0", port=8000, path="/mcp")
```

### What's Happening Here

1. **OpenAPI Spec Fetch**: We fetch the OpenAPI JSON from our running FastAPI app at startup
2. **Automatic Tool Generation**: `FastMCP.from_openapi()` parses the spec and creates an MCP tool for each endpoint
3. **HTTP Transport**: Running on port 8000 with HTTP transport makes it accessible for testing (you can also use stdio for local CLI usage)

### Gotchas

- The FastAPI server must be running before you start the MCP server (since it needs to fetch the OpenAPI spec)
- If you change your API, restart the MCP server to pick up the new schema
- The `client` base_url should match where your API is actually running

[mcp inspector pointed at the mcp server](assets/images/2025-12-14-mcp_inspector_openapi_mcp.png)

## 4. Test Script

To verify everything works, here's a test client that connects to the MCP server and calls each tool:

```python
import asyncio
from fastmcp import Client
from fastmcp.client.transports import StreamableHttpTransport

async def test_mcp_server():
    transport = StreamableHttpTransport(url="http://localhost:8000/mcp")
    
    async with Client(transport=transport) as client:
        print("✓ Connected to MCP server\n")
        
        tools = await client.list_tools()
        print(f"Available tools: {[t.name for t in tools]}\n")
        
        print("Test 1: Calling root...")
        result = await client.call_tool("root", {})
        print(f"Result: {result.content[0].text}\n")
        
        print("Test 2: Reading item with ID 42...")
        result = await client.call_tool("read_item", {
            "item_id": 42,
            "q": "test query"
        })
        print(f"Result: {result.content[0].text}\n")
        
        print("Test 3: Creating new item...")
    f    result = await client.call_tool("create_item", {
            "name": "Laptop",
            "description": "High-performance laptop",
            "price": 1299.99
        })
        print(f"Result: {result.content[0].text}\n")
        
        print("Test 4: Creating new user...")
        result = await client.call_tool("create_user", {
            "username": "johndoe",
            "email": "john@example.com",
            "age": 30
        })
        print(f"Result: {result.content[0].text}\n")

if __name__ == "__main__":
    asyncio.run(test_mcp_server())
```

### Running the Tests

1. Start the FastAPI server: `python app.py`
2. In another terminal, start the MCP server: `python mcp_server.py`
3. In a third terminal, run the test client: `python mcp_client.py`

### Expected Output

```
✓ Connected to MCP server

Available tools: ['root', 'read_item', 'create_item', 'create_user']

Test 1: Calling root...
Result: {"message":"Hello World"}

Test 2: Reading item with ID 42...
Result: {"item_id":42,"q":"test query"}

Test 3: Creating new item...
Result: {"item":{"name":"Laptop","description":"High-performance laptop","price":1299.99},"created":true}

Test 4: Creating new user...
Result: {"id":123,"username":"johndoe","email":"john@example.com","status":"active"}
```

## 5. References

- [FastMCP Documentation](https://github.com/jlowin/fastmcp)
- [FastMCP OpenAPI Integration](https://github.com/jlowin/fastmcp#openapi-integration)
- [Model Context Protocol Specification](https://modelcontextprotocol.io/)
- [FastAPI Documentation](https://fastapi.tiangolo.com/)
- [OpenAPI Specification](https://swagger.io/specification/)
