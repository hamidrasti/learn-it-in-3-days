# Ultimate gRPC Crash Course for FAANG Interviews (2024–2025)

---

## 1. Core Concepts You Must Know Cold

### 1.1 What is gRPC?

**gRPC (Google Remote Procedure Call)** is a high-performance, open-source RPC framework that uses HTTP/2 for transport and Protocol Buffers (protobuf) as its interface definition language (IDL) and serialization format.

```
┌─────────────────┐                    ┌─────────────────┐
│   gRPC Client   │◄──── HTTP/2 ──────►│   gRPC Server   │
│  (Stub/Client)  │   Binary Protobuf  │    (Service)    │
└─────────────────┘                    └─────────────────┘
```

**Key Interview Points:**
- Created by Google, open-sourced in 2015
- Language-agnostic (supports 10+ languages)
- Uses `.proto` files to define services and messages
- Generates client and server code automatically

```protobuf
// Example: Basic service definition
syntax = "proto3";

service UserService {
  rpc GetUser(GetUserRequest) returns (GetUserResponse);
}

message GetUserRequest {
  string user_id = 1;
}

message GetUserResponse {
  string name = 1;
  string email = 2;
  int32 age = 3;
}
```

---

### 1.2 Protocol Buffers (Protobuf)

**Protobuf** is Google's language-neutral, platform-neutral serialization mechanism.

**Why Protobuf over JSON?**

| Aspect | Protobuf | JSON |
|--------|----------|------|
| Size | ~3-10x smaller | Larger (text-based) |
| Speed | ~5-100x faster | Slower parsing |
| Schema | Strongly typed | Schema-less |
| Human-readable | No (binary) | Yes |

**Field Numbers & Wire Types:**
```protobuf
message Person {
  string name = 1;    // Field number 1
  int32 id = 2;       // Field number 2
  string email = 3;   // Field number 3
  
  // NEVER reuse field numbers after deletion!
  // Use 'reserved' keyword instead
  reserved 4, 5;
  reserved "old_field";
}
```

**Scalar Types:**
```protobuf
// Numeric
int32, int64, uint32, uint64, sint32, sint64
float, double
bool

// String/bytes
string, bytes

// Special
fixed32, fixed64, sfixed32, sfixed64
```

**Complex Types:**
```protobuf
message ComplexExample {
  // Enums
  enum Status {
    UNKNOWN = 0;  // First enum value must be 0
    ACTIVE = 1;
    INACTIVE = 2;
  }
  Status status = 1;
  
  // Nested messages
  message Address {
    string city = 1;
    string country = 2;
  }
  Address address = 2;
  
  // Repeated (arrays)
  repeated string tags = 3;
  
  // Maps
  map<string, int32> scores = 4;
  
  // Oneof (union type)
  oneof contact {
    string phone = 5;
    string email = 6;
  }
  
  // Optional (proto3)
  optional string nickname = 7;
}
```

---

### 1.3 The Four Communication Patterns

This is **THE MOST ASKED** concept in gRPC interviews.

```protobuf
service StreamingService {
  // 1. Unary RPC
  rpc Unary(Request) returns (Response);
  
  // 2. Server Streaming
  rpc ServerStream(Request) returns (stream Response);
  
  // 3. Client Streaming  
  rpc ClientStream(stream Request) returns (Response);
  
  // 4. Bidirectional Streaming
  rpc BiDiStream(stream Request) returns (stream Response);
}
```

**Visual Comparison:**

```
1. UNARY (Request-Response)
   Client ──Request──► Server
   Client ◄──Response── Server

2. SERVER STREAMING
   Client ──Request──────────► Server
   Client ◄──Response 1─────── Server
   Client ◄──Response 2─────── Server
   Client ◄──Response N─────── Server

3. CLIENT STREAMING
   Client ──Request 1──────► Server
   Client ──Request 2──────► Server
   Client ──Request N──────► Server
   Client ◄──Response─────── Server

4. BIDIRECTIONAL STREAMING
   Client ◄──────────────────► Server
   (Both send streams independently)
```

**Use Cases:**

| Pattern | Use Case |
|---------|----------|
| Unary | Simple CRUD, Auth, single queries |
| Server Streaming | Real-time feeds, large file downloads, notifications |
| Client Streaming | File uploads, telemetry, log aggregation |
| Bidirectional | Chat apps, gaming, collaborative editing |

---

### 1.4 HTTP/2 Fundamentals

gRPC requires HTTP/2. You MUST understand why.

**HTTP/2 Features gRPC Leverages:**

```
┌─────────────────────────────────────────────────────┐
│                    TCP Connection                    │
├─────────────────────────────────────────────────────┤
│  Stream 1 │  Stream 3 │  Stream 5 │  Stream 7       │
│  (RPC 1)  │  (RPC 2)  │  (RPC 3)  │  (RPC 4)        │
├─────────────────────────────────────────────────────┤
│            MULTIPLEXING - No Head-of-Line Blocking  │
└─────────────────────────────────────────────────────┘
```

| HTTP/1.1 | HTTP/2 |
|----------|--------|
| Text-based | Binary framing |
| One request per connection | Multiplexed streams |
| Header repetition | Header compression (HPACK) |
| No server push | Server push support |
| New connection per request | Single persistent connection |

**Interview Question:** *"Why can't gRPC work over HTTP/1.1?"*
- Streaming requires multiplexed connections
- Binary framing needed for protobuf efficiency
- HTTP/1.1 lacks proper stream cancellation

---

### 1.5 gRPC vs REST

**The comparison table interviewers love:**

| Aspect | gRPC | REST |
|--------|------|------|
| Protocol | HTTP/2 | HTTP/1.1 (usually) |
| Payload | Protobuf (binary) | JSON/XML (text) |
| API Contract | Strict (.proto) | Loose (OpenAPI optional) |
| Code Generation | Built-in | External tools |
| Streaming | Native 4 patterns | Limited (SSE, WebSocket) |
| Browser Support | Limited (grpc-web) | Native |
| Caching | Complex | Native HTTP caching |
| Human Debugging | Hard (binary) | Easy (text) |

**When to Use gRPC:**
- Microservices communication
- Low-latency, high-throughput systems
- Polyglot environments
- Real-time bidirectional streaming

**When to Use REST:**
- Public APIs
- Browser-first applications
- Simple CRUD operations
- When caching is critical

---

### 1.6 Channel and Stub Architecture

```
┌─────────────────────────────────────────────────────────┐
│                     CLIENT SIDE                          │
├─────────────────────────────────────────────────────────┤
│  Application Code                                        │
│       │                                                  │
│       ▼                                                  │
│  ┌─────────────────┐                                     │
│  │     STUB        │  (Generated from .proto)           │
│  │  - Blocking     │                                     │
│  │  - Async        │                                     │
│  │  - Future       │                                     │
│  └────────┬────────┘                                     │
│           │                                              │
│           ▼                                              │
│  ┌─────────────────┐                                     │
│  │    CHANNEL      │  (Manages connection)              │
│  │  - Load Balance │                                     │
│  │  - Retry Logic  │                                     │
│  │  - Connection   │                                     │
│  └────────┬────────┘                                     │
│           │                                              │
│           ▼                                              │
│      HTTP/2 Transport                                    │
└─────────────────────────────────────────────────────────┘
```

**Python Example:**
```python
import grpc
from user_pb2_grpc import UserServiceStub

# Create channel (connection manager)
channel = grpc.insecure_channel('localhost:50051')

# Create stub (client)
stub = UserServiceStub(channel)

# Make RPC call
response = stub.GetUser(GetUserRequest(user_id="123"))
```

**Go Example:**
```go
// Create connection
conn, err := grpc.Dial("localhost:50051", grpc.WithInsecure())
defer conn.Close()

// Create client
client := pb.NewUserServiceClient(conn)

// Make call
resp, err := client.GetUser(ctx, &pb.GetUserRequest{UserId: "123"})
```

---

### 1.7 Interceptors (Middleware)

Interceptors are gRPC's middleware pattern. **Critical for system design answers.**

```
Request Flow:
┌──────────────────────────────────────────────────────────┐
│  Client                                                   │
│    │                                                      │
│    ▼                                                      │
│  Client Interceptor 1 (Auth)                             │
│    │                                                      │
│    ▼                                                      │
│  Client Interceptor 2 (Logging)                          │
│    │                                                      │
│    ▼                                                      │
│  ═══════════════ NETWORK ════════════════                │
│    │                                                      │
│    ▼                                                      │
│  Server Interceptor 1 (Auth Verify)                      │
│    │                                                      │
│    ▼                                                      │
│  Server Interceptor 2 (Rate Limit)                       │
│    │                                                      │
│    ▼                                                      │
│  Service Handler                                          │
└──────────────────────────────────────────────────────────┘
```

**Types:**
- **Unary Interceptors**: For request-response RPCs
- **Stream Interceptors**: For streaming RPCs

**Python Server Interceptor Example:**
```python
class AuthInterceptor(grpc.ServerInterceptor):
    def intercept_service(self, continuation, handler_call_details):
        # Extract metadata
        metadata = dict(handler_call_details.invocation_metadata)
        token = metadata.get('authorization', '')
        
        if not self.validate_token(token):
            # Return error
            return grpc.unary_unary_rpc_method_handler(
                lambda req, ctx: ctx.abort(
                    grpc.StatusCode.UNAUTHENTICATED,
                    'Invalid token'
                )
            )
        
        return continuation(handler_call_details)
```

---

### 1.8 Metadata (Headers)

Metadata is gRPC's equivalent of HTTP headers.

```python
# Client: Sending metadata
metadata = [
    ('authorization', 'Bearer token123'),
    ('x-request-id', 'uuid-here'),
    ('x-custom-header', 'value')
]

response = stub.GetUser(
    request,
    metadata=metadata
)

# Server: Reading metadata
def GetUser(self, request, context):
    metadata = dict(context.invocation_metadata())
    token = metadata.get('authorization')
    request_id = metadata.get('x-request-id')
    
    # Send response metadata
    context.set_trailing_metadata([
        ('x-response-time', '42ms')
    ])
    
    return response
```

---

### 1.9 Error Handling and Status Codes

**gRPC Status Codes (MUST MEMORIZE):**

| Code | Name | HTTP Equivalent | Use Case |
|------|------|-----------------|----------|
| 0 | OK | 200 | Success |
| 1 | CANCELLED | 499 | Client cancelled |
| 2 | UNKNOWN | 500 | Unknown error |
| 3 | INVALID_ARGUMENT | 400 | Bad request |
| 4 | DEADLINE_EXCEEDED | 504 | Timeout |
| 5 | NOT_FOUND | 404 | Resource missing |
| 6 | ALREADY_EXISTS | 409 | Duplicate |
| 7 | PERMISSION_DENIED | 403 | Forbidden |
| 8 | RESOURCE_EXHAUSTED | 429 | Rate limited |
| 9 | FAILED_PRECONDITION | 400 | State error |
| 10 | ABORTED | 409 | Concurrency conflict |
| 11 | OUT_OF_RANGE | 400 | Invalid range |
| 12 | UNIMPLEMENTED | 501 | Not implemented |
| 13 | INTERNAL | 500 | Server error |
| 14 | UNAVAILABLE | 503 | Service down |
| 16 | UNAUTHENTICATED | 401 | Auth required |

**Error Handling Example:**
```python
# Server-side error
def GetUser(self, request, context):
    user = db.get_user(request.user_id)
    if not user:
        context.abort(
            grpc.StatusCode.NOT_FOUND,
            f'User {request.user_id} not found'
        )
    return user

# Client-side handling
try:
    response = stub.GetUser(request)
except grpc.RpcError as e:
    if e.code() == grpc.StatusCode.NOT_FOUND:
        print(f"User not found: {e.details()}")
    elif e.code() == grpc.StatusCode.DEADLINE_EXCEEDED:
        print("Request timed out")
    else:
        print(f"RPC failed: {e.code()}: {e.details()}")
```

---

### 1.10 Deadlines and Timeouts

**Critical for production systems and system design interviews.**

```python
# Client sets deadline
from datetime import timedelta

# Option 1: Timeout (duration)
response = stub.GetUser(
    request,
    timeout=5.0  # 5 seconds
)

# Option 2: Deadline (absolute time)
deadline = time.time() + 5.0
response = stub.GetUser(
    request,
    timeout=deadline - time.time()
)
```

**Deadline Propagation:**
```
┌─────────┐     ┌─────────┐     ┌─────────┐
│ Client  │────►│Service A│────►│Service B│
│ DL: 5s  │     │ DL: 4s  │     │ DL: 3s  │
└─────────┘     └─────────┘     └─────────┘
                (1s used)       (1s used)
```

```go
// Go: Check remaining deadline
deadline, ok := ctx.Deadline()
if ok {
    remaining := time.Until(deadline)
    if remaining < 100*time.Millisecond {
        return nil, status.Error(codes.DeadlineExceeded, "not enough time")
    }
}
```

---

### 1.11 Load Balancing

**Client-Side vs Proxy Load Balancing:**

```
CLIENT-SIDE LB (gRPC Native):
┌──────────┐
│  Client  │──────┬──────────► Server 1
│  with LB │──────┼──────────► Server 2
│          │──────┴──────────► Server 3
└──────────┘
    │
    └── Client discovers all backends
        and balances directly

PROXY-BASED LB (Envoy, nginx):
┌──────────┐      ┌───────┐
│  Client  │─────►│ Proxy │───► Server 1
└──────────┘      │  (LB) │───► Server 2
                  └───────┘───► Server 3
```

**Built-in Policies:**
- `pick_first`: Use first available (default)
- `round_robin`: Rotate through backends

```python
# Python round-robin example
channel = grpc.insecure_channel(
    'dns:///my-service.example.com',
    options=[
        ('grpc.lb_policy_name', 'round_robin'),
    ]
)
```

---

### 1.12 Security (TLS/mTLS)

```python
# Server with TLS
server_credentials = grpc.ssl_server_credentials(
    [(private_key, certificate_chain)],
    root_certificates=client_ca,  # For mTLS
    require_client_auth=True       # For mTLS
)
server.add_secure_port('[::]:443', server_credentials)

# Client with TLS
channel_credentials = grpc.ssl_channel_credentials(
    root_certificates=server_ca,
    private_key=client_key,        # For mTLS
    certificate_chain=client_cert   # For mTLS
)
channel = grpc.secure_channel('localhost:443', channel_credentials)
```

**Authentication Patterns:**
1. **Token-based**: JWT in metadata
2. **mTLS**: Mutual certificate verification
3. **Google Auth**: OAuth2/OIDC integration

---

## 2. Most Frequently Asked Interview Questions & Topics

### Easy (Conceptual / Definition)

| # | Question | Key Points |
|---|----------|------------|
| 1 | What is gRPC and how does it differ from REST? | Binary vs text, HTTP/2, streaming, code-gen |
| 2 | What are Protocol Buffers? | IDL, serialization, schema evolution |
| 3 | Explain the 4 communication patterns | Unary, server/client/bidi streaming + use cases |
| 4 | What is a `.proto` file? | Service/message definitions, field numbers |
| 5 | What are the benefits of HTTP/2 for gRPC? | Multiplexing, binary, header compression |

### Medium (Implementation / Troubleshooting)

| # | Question | Key Points |
|---|----------|------------|
| 1 | How do you handle errors in gRPC? | Status codes, error details, client handling |
| 2 | Explain interceptors and their use cases | Auth, logging, metrics, rate limiting |
| 3 | How does deadline propagation work? | Timeout inheritance, context cancellation |
| 4 | How do you implement authentication? | Metadata, mTLS, token interceptors |
| 5 | What happens when you add/remove fields in protobuf? | Backward/forward compatibility, reserved |
| 6 | How do you load balance gRPC? | Client-side vs proxy, round-robin, service mesh |
| 7 | Explain channel and connection management | Reuse channels, connection pooling |

### Hard (System Design / Deep Dive)

| # | Question | Key Points |
|---|----------|------------|
| 1 | Design a real-time notification system using gRPC | Server streaming, reconnection, scaling |
| 2 | How would you implement a chat service? | Bidi streaming, presence, message ordering |
| 3 | Design gRPC gateway for REST compatibility | grpc-gateway, transcoding |
| 4 | Handle partial failures in streaming RPCs | Retry logic, checkpointing, idempotency |
| 5 | Implement rate limiting across gRPC services | Interceptors, distributed rate limiting |
| 6 | How would you trace requests across services? | OpenTelemetry, context propagation, metadata |
| 7 | Design schema evolution strategy for large systems | Versioning, deprecation, field reservation |

---

## 3. One-Page Cheat Sheet

### Proto3 Quick Reference

```protobuf
syntax = "proto3";
package mypackage;
import "google/protobuf/timestamp.proto";

// Message
message User {
  string id = 1;
  repeated string roles = 2;
  map<string, string> metadata = 3;
  optional string nickname = 4;
  google.protobuf.Timestamp created_at = 5;
  
  oneof contact {
    string email = 6;
    string phone = 7;
  }
}

// Service
service UserService {
  rpc Get(GetReq) returns (User);
  rpc List(ListReq) returns (stream User);
  rpc Upload(stream Chunk) returns (Status);
  rpc Chat(stream Msg) returns (stream Msg);
}
```

### Status Codes Quick Reference

| Code | Meaning | Retry? |
|------|---------|--------|
| OK (0) | Success | N/A |
| INVALID_ARGUMENT (3) | Bad input | No |
| NOT_FOUND (5) | Missing | No |
| ALREADY_EXISTS (6) | Duplicate | No |
| PERMISSION_DENIED (7) | Forbidden | No |
| DEADLINE_EXCEEDED (4) | Timeout | Yes |
| RESOURCE_EXHAUSTED (8) | Rate limited | Yes (backoff) |
| UNAVAILABLE (14) | Transient | Yes (backoff) |
| INTERNAL (13) | Server bug | Maybe |
| UNAUTHENTICATED (16) | No auth | No (re-auth) |

### Performance Rules of Thumb

| Metric | Guideline |
|--------|-----------|
| Message size | Keep < 1MB (default limit) |
| Streaming batch | 100-1000 messages before yield |
| Connection reuse | Always reuse channels |
| Deadline | Always set (default: no timeout!) |
| Keep-alive | Configure for long-lived connections |

### Best Practices Bullets

```
✓ Reuse channels (expensive to create)
✓ Set deadlines on ALL calls
✓ Use streaming for large payloads
✓ Reserve deleted field numbers
✓ Handle all error codes
✓ Use interceptors for cross-cutting concerns
✓ Enable retries with backoff

✗ Don't create new channel per request
✗ Don't use streaming for simple req/res
✗ Don't reuse field numbers
✗ Don't ignore deadline propagation
✗ Don't block in streaming handlers
```

---

## 4. Best Practices & Common Mistakes

### What Senior Engineers Do ✓

```python
# 1. Reuse channels (create once, use everywhere)
class GrpcClientManager:
    _channels = {}
    
    @classmethod
    def get_channel(cls, address):
        if address not in cls._channels:
            cls._channels[address] = grpc.insecure_channel(
                address,
                options=[
                    ('grpc.keepalive_time_ms', 30000),
                    ('grpc.keepalive_timeout_ms', 5000),
                ]
            )
        return cls._channels[address]
```

```python
# 2. Always set deadlines
response = stub.GetUser(
    request,
    timeout=5.0  # NEVER call without timeout in production
)
```

```python
# 3. Implement proper retry with exponential backoff
retry_policy = {
    'maxAttempts': 4,
    'initialBackoff': '0.1s',
    'maxBackoff': '1s',
    'backoffMultiplier': 2,
    'retryableStatusCodes': ['UNAVAILABLE', 'DEADLINE_EXCEEDED']
}
```

```protobuf
// 4. Use proper schema evolution
message User {
  string id = 1;
  string name = 2;
  // Field 3 was 'age', removed in v2
  reserved 3;
  reserved "age";
  
  // New field added in v3
  optional int32 birth_year = 4;
}
```

```python
# 5. Graceful shutdown
server.stop(grace=30)  # Allow 30s for ongoing RPCs
```

### What Gets You Rejected ✗

```python
# ❌ Creating new channel per request
def get_user(user_id):
    channel = grpc.insecure_channel('localhost:50051')  # BAD!
    stub = UserServiceStub(channel)
    return stub.GetUser(GetUserRequest(user_id=user_id))

# ❌ No timeout (can hang forever)
response = stub.GetUser(request)  # No timeout!

# ❌ Ignoring errors
try:
    response = stub.GetUser(request)
except:  # Too broad!
    pass

# ❌ Blocking in async contexts
async def handler():
    stub.GetUser(request)  # Blocking call in async!
    
# ❌ Reusing field numbers
message Bad {
  string name = 1;
  // string old_name = 1;  // NEVER reuse numbers!
}
```

### Schema Evolution Rules

```
SAFE CHANGES:
✓ Add new fields (use new field numbers)
✓ Remove fields (reserve the number!)
✓ Rename fields (wire format uses numbers)
✓ Change int32 ↔ int64 (compatible)
✓ Add enum values

BREAKING CHANGES:
✗ Change field numbers
✗ Change field types (incompatible)
✗ Change field to repeated or vice versa
✗ Remove field without reserving
```

---

## 5. Minimal but Complete Code Examples

### Example 1: Complete Unary RPC (Python)

```protobuf
// user.proto
syntax = "proto3";

package user;

service UserService {
  rpc GetUser(GetUserRequest) returns (GetUserResponse);
  rpc CreateUser(CreateUserRequest) returns (CreateUserResponse);
}

message GetUserRequest {
  string user_id = 1;
}

message GetUserResponse {
  string user_id = 1;
  string name = 2;
  string email = 3;
}

message CreateUserRequest {
  string name = 1;
  string email = 2;
}

message CreateUserResponse {
  string user_id = 1;
  bool success = 2;
}
```

```python
# server.py
import grpc
from concurrent import futures
import user_pb2
import user_pb2_grpc

class UserServicer(user_pb2_grpc.UserServiceServicer):
    def __init__(self):
        self.users = {}
    
    def GetUser(self, request, context):
        user_id = request.user_id
        
        if user_id not in self.users:
            context.abort(
                grpc.StatusCode.NOT_FOUND,
                f'User {user_id} not found'
            )
        
        user = self.users[user_id]
        return user_pb2.GetUserResponse(
            user_id=user_id,
            name=user['name'],
            email=user['email']
        )
    
    def CreateUser(self, request, context):
        import uuid
        user_id = str(uuid.uuid4())
        
        self.users[user_id] = {
            'name': request.name,
            'email': request.email
        }
        
        return user_pb2.CreateUserResponse(
            user_id=user_id,
            success=True
        )

def serve():
    server = grpc.server(futures.ThreadPoolExecutor(max_workers=10))
    user_pb2_grpc.add_UserServiceServicer_to_server(UserServicer(), server)
    server.add_insecure_port('[::]:50051')
    server.start()
    print("Server started on port 50051")
    server.wait_for_termination()

if __name__ == '__main__':
    serve()
```

```python
# client.py
import grpc
import user_pb2
import user_pb2_grpc

def run():
    # Create channel (reuse this!)
    with grpc.insecure_channel('localhost:50051') as channel:
        stub = user_pb2_grpc.UserServiceStub(channel)
        
        # Create user
        create_response = stub.CreateUser(
            user_pb2.CreateUserRequest(
                name="Alice",
                email="alice@example.com"
            ),
            timeout=5.0  # Always set timeout!
        )
        print(f"Created user: {create_response.user_id}")
        
        # Get user
        try:
            get_response = stub.GetUser(
                user_pb2.GetUserRequest(user_id=create_response.user_id),
                timeout=5.0
            )
            print(f"Got user: {get_response.name}, {get_response.email}")
        except grpc.RpcError as e:
            print(f"RPC failed: {e.code()}: {e.details()}")

if __name__ == '__main__':
    run()
```

---

### Example 2: Server Streaming (Real-time Feed)

```protobuf
// feed.proto
syntax = "proto3";

service FeedService {
  rpc SubscribeFeed(FeedRequest) returns (stream FeedItem);
}

message FeedRequest {
  string user_id = 1;
  repeated string topics = 2;
}

message FeedItem {
  string id = 1;
  string topic = 2;
  string content = 3;
  int64 timestamp = 4;
}
```

```python
# server.py
import time
import random

class FeedServicer(feed_pb2_grpc.FeedServiceServicer):
    def SubscribeFeed(self, request, context):
        """Server streaming: sends continuous updates to client"""
        user_id = request.user_id
        topics = request.topics
        
        print(f"User {user_id} subscribed to: {topics}")
        
        item_id = 0
        while context.is_active():  # Check if client still connected
            # Check deadline
            if context.time_remaining() <= 0:
                break
                
            # Simulate generating feed items
            topic = random.choice(topics) if topics else "general"
            
            yield feed_pb2.FeedItem(
                id=str(item_id),
                topic=topic,
                content=f"Content for {topic} #{item_id}",
                timestamp=int(time.time())
            )
            
            item_id += 1
            time.sleep(1)  # Rate limit: 1 item/second
        
        print(f"User {user_id} disconnected")
```

```python
# client.py
def subscribe_to_feed():
    with grpc.insecure_channel('localhost:50051') as channel:
        stub = feed_pb2_grpc.FeedServiceStub(channel)
        
        request = feed_pb2.FeedRequest(
            user_id="user123",
            topics=["tech", "sports"]
        )
        
        try:
            # Stream with timeout
            for item in stub.SubscribeFeed(request, timeout=60.0):
                print(f"[{item.topic}] {item.content}")
                
        except grpc.RpcError as e:
            if e.code() == grpc.StatusCode.DEADLINE_EXCEEDED:
                print("Feed subscription timeout")
            else:
                print(f"Error: {e.code()}: {e.details()}")
```

---

### Example 3: Client Streaming (File Upload)

```protobuf
// upload.proto
syntax = "proto3";

service UploadService {
  rpc UploadFile(stream FileChunk) returns (UploadResponse);
}

message FileChunk {
  string filename = 1;
  bytes data = 2;
  int32 chunk_number = 3;
  bool is_last = 4;
}

message UploadResponse {
  bool success = 1;
  string file_id = 2;
  int64 total_bytes = 3;
  string checksum = 4;
}
```

```python
# server.py
import hashlib

class UploadServicer(upload_pb2_grpc.UploadServiceServicer):
    def UploadFile(self, request_iterator, context):
        """Client streaming: receives file chunks from client"""
        file_data = bytearray()
        filename = None
        chunk_count = 0
        
        for chunk in request_iterator:
            if filename is None:
                filename = chunk.filename
                print(f"Receiving file: {filename}")
            
            file_data.extend(chunk.data)
            chunk_count += 1
            
            if chunk.is_last:
                break
        
        # Calculate checksum
        checksum = hashlib.md5(file_data).hexdigest()
        
        # Save file (simulated)
        file_id = f"file_{checksum[:8]}"
        
        return upload_pb2.UploadResponse(
            success=True,
            file_id=file_id,
            total_bytes=len(file_data),
            checksum=checksum
        )
```

```python
# client.py
def upload_file(filepath, chunk_size=64*1024):
    with grpc.insecure_channel('localhost:50051') as channel:
        stub = upload_pb2_grpc.UploadServiceStub(channel)
        
        def chunk_generator():
            with open(filepath, 'rb') as f:
                chunk_number = 0
                while True:
                    data = f.read(chunk_size)
                    if not data:
                        break
                    
                    is_last = len(data) < chunk_size
                    
                    yield upload_pb2.FileChunk(
                        filename=filepath,
                        data=data,
                        chunk_number=chunk_number,
                        is_last=is_last
                    )
                    chunk_number += 1
        
        response = stub.UploadFile(chunk_generator(), timeout=300.0)
        print(f"Upload complete: {response.file_id}")
        print(f"Size: {response.total_bytes}, Checksum: {response.checksum}")
```

---

### Example 4: Bidirectional Streaming (Chat)

```protobuf
// chat.proto
syntax = "proto3";

service ChatService {
  rpc Chat(stream ChatMessage) returns (stream ChatMessage);
}

message ChatMessage {
  string user_id = 1;
  string room_id = 2;
  string content = 3;
  int64 timestamp = 4;
  MessageType type = 5;
  
  enum MessageType {
    TEXT = 0;
    JOIN = 1;
    LEAVE = 2;
  }
}
```

```python
# server.py
from collections import defaultdict
import threading

class ChatServicer(chat_pb2_grpc.ChatServiceServicer):
    def __init__(self):
        self.rooms = defaultdict(list)  # room_id -> list of queues
        self.lock = threading.Lock()
    
    def Chat(self, request_iterator, context):
        """Bidirectional streaming: real-time chat"""
        import queue
        
        message_queue = queue.Queue()
        room_id = None
        user_id = None
        
        def receive_messages():
            nonlocal room_id, user_id
            try:
                for message in request_iterator:
                    if room_id is None:
                        room_id = message.room_id
                        user_id = message.user_id
                        
                        with self.lock:
                            self.rooms[room_id].append(message_queue)
                    
                    # Broadcast to all clients in room
                    self.broadcast(room_id, message, message_queue)
            except Exception as e:
                print(f"Receive error: {e}")
            finally:
                # Cleanup on disconnect
                if room_id:
                    with self.lock:
                        if message_queue in self.rooms[room_id]:
                            self.rooms[room_id].remove(message_queue)
                message_queue.put(None)  # Signal to stop sending
        
        # Start receiving in background
        receiver = threading.Thread(target=receive_messages)
        receiver.start()
        
        # Send messages from queue
        while context.is_active():
            try:
                message = message_queue.get(timeout=1.0)
                if message is None:
                    break
                yield message
            except queue.Empty:
                continue
        
        receiver.join()
    
    def broadcast(self, room_id, message, sender_queue):
        with self.lock:
            for q in self.rooms[room_id]:
                if q != sender_queue:  # Don't echo to sender
                    q.put(message)
```

---

### Example 5: Interceptor for Auth + Logging

```python
# interceptors.py
import grpc
import time
import logging

class AuthInterceptor(grpc.ServerInterceptor):
    """Server-side authentication interceptor"""
    
    def __init__(self, valid_tokens):
        self.valid_tokens = valid_tokens
    
    def intercept_service(self, continuation, handler_call_details):
        # Extract metadata
        metadata = dict(handler_call_details.invocation_metadata)
        token = metadata.get('authorization', '')
        
        if not token.startswith('Bearer '):
            return self._abort_handler(
                grpc.StatusCode.UNAUTHENTICATED,
                'Missing Bearer token'
            )
        
        token = token[7:]  # Remove 'Bearer ' prefix
        
        if token not in self.valid_tokens:
            return self._abort_handler(
                grpc.StatusCode.UNAUTHENTICATED,
                'Invalid token'
            )
        
        return continuation(handler_call_details)
    
    def _abort_handler(self, code, message):
        def abort(request, context):
            context.abort(code, message)
        return grpc.unary_unary_rpc_method_handler(abort)


class LoggingInterceptor(grpc.ServerInterceptor):
    """Server-side logging interceptor"""
    
    def intercept_service(self, continuation, handler_call_details):
        method = handler_call_details.method
        start = time.time()
        
        logging.info(f"RPC Start: {method}")
        
        response = continuation(handler_call_details)
        
        duration = time.time() - start
        logging.info(f"RPC End: {method} ({duration:.3f}s)")
        
        return response


# Client-side interceptor
class ClientAuthInterceptor(grpc.UnaryUnaryClientInterceptor):
    def __init__(self, token):
        self.token = token
    
    def intercept_unary_unary(self, continuation, client_call_details, request):
        # Add auth metadata
        metadata = list(client_call_details.metadata or [])
        metadata.append(('authorization', f'Bearer {self.token}'))
        
        new_details = grpc.ClientCallDetails(
            client_call_details.method,
            client_call_details.timeout,
            metadata,
            client_call_details.credentials,
            client_call_details.wait_for_ready,
            client_call_details.compression
        )
        
        return continuation(new_details, request)


# Usage
def serve_with_interceptors():
    interceptors = [
        AuthInterceptor({'secret-token-123'}),
        LoggingInterceptor(),
    ]
    
    server = grpc.server(
        futures.ThreadPoolExecutor(max_workers=10),
        interceptors=interceptors
    )
    # ... add services and start
```

---

## 6. Recommended Problems to Solve Today

### Coding Exercises

| # | Problem | Difficulty | Description |
|---|---------|------------|-------------|
| 1 | Basic CRUD Service | Easy | Implement User CRUD with proper error handling |
| 2 | File Upload with Progress | Medium | Client streaming with progress callback |
| 3 | Real-time Leaderboard | Medium | Server streaming for live score updates |
| 4 | Chat Room System | Hard | Bidi streaming with multiple rooms |
| 5 | Rate Limiting Interceptor | Medium | Implement token bucket in interceptor |

### System Design Problems

| # | Problem | Difficulty | Key gRPC Concepts |
|---|---------|------------|-------------------|
| 1 | Design Notification Service | Medium | Server streaming, reconnection, scaling |
| 2 | Design Logging Pipeline | Medium | Client streaming, batching, backpressure |
| 3 | Design Real-time Collaboration | Hard | Bidi streaming, conflict resolution |
| 4 | Design API Gateway | Hard | gRPC-gateway, transcoding, routing |
| 5 | Design Service Mesh | Hard | Load balancing, mTLS, observability |

### Practice Implementations

```
1. [ ] Implement all 4 streaming patterns
2. [ ] Add authentication interceptor
3. [ ] Add retry logic with exponential backoff
4. [ ] Implement deadline propagation across services
5. [ ] Add Prometheus metrics interceptor
6. [ ] Implement graceful shutdown
7. [ ] Add health checking (grpc_health_v1)
8. [ ] Implement reflection for debugging
```

---

## 7. 30-Minute Quiz

Test yourself! Answers at the end.

### Questions

**Q1.** What serialization format does gRPC use by default?
- A) JSON
- B) XML
- C) Protocol Buffers
- D) MessagePack

**Q2.** Which HTTP version does gRPC require?
- A) HTTP/1.0
- B) HTTP/1.1
- C) HTTP/2
- D) HTTP/3

**Q3.** What's the correct order of streaming patterns from client to server perspective?
- A) Unary sends: 1 request, receives: 1 response
- B) Server streaming sends: 1 request, receives: N responses
- C) Client streaming sends: N requests, receives: 1 response
- D) All of the above are correct

**Q4.** What gRPC status code should you return for "resource not found"?
- A) UNKNOWN
- B) INVALID_ARGUMENT
- C) NOT_FOUND
- D) UNAVAILABLE

**Q5.** In Protocol Buffers, what happens when you delete a field?
- A) You can reuse the field number immediately
- B) You should reserve the field number
- C) The proto file won't compile
- D) Old clients will crash

**Q6.** What is the purpose of interceptors in gRPC?
- A) To serialize messages faster
- B) To implement cross-cutting concerns like auth and logging
- C) To define service contracts
- D) To generate code from proto files

**Q7.** Which gRPC status codes are generally safe to retry?
- A) NOT_FOUND, ALREADY_EXISTS
- B) UNAVAILABLE, DEADLINE_EXCEEDED
- C) INVALID_ARGUMENT, PERMISSION_DENIED
- D) INTERNAL, UNKNOWN

**Q8.** What is the main advantage of HTTP/2 multiplexing for gRPC?
- A) Better security
- B) Multiple RPCs over single TCP connection
- C) Faster serialization
- D) Automatic code generation

**Q9.** In a microservices architecture, how should gRPC channels be managed?
- A) Create new channel for each request
- B) Reuse channels across requests
- C) Create new channel for each service
- D) Channels don't matter in microservices

**Q10.** What is the default behavior when no deadline is set on a gRPC call?
- A) 5 second timeout
- B) 30 second timeout
- C) No timeout (can wait forever)
- D) Immediate failure

---

### Answers

<details>
<summary>Click to reveal answers</summary>

1. **C) Protocol Buffers** - gRPC uses protobuf by default for serialization

2. **C) HTTP/2** - gRPC requires HTTP/2 for multiplexing and streaming

3. **D) All of the above are correct** - These are the exact definitions

4. **C) NOT_FOUND** - Maps to HTTP 404

5. **B) You should reserve the field number** - To prevent future reuse causing compatibility issues

6. **B) To implement cross-cutting concerns like auth and logging** - Interceptors are middleware

7. **B) UNAVAILABLE, DEADLINE_EXCEEDED** - These are transient errors safe to retry

8. **B) Multiple RPCs over single TCP connection** - Eliminates connection overhead

9. **B) Reuse channels across requests** - Channels are expensive to create

10. **C) No timeout (can wait forever)** - Always set deadlines in production!

**Scoring:**
- 9-10: Ready for interviews!
- 7-8: Review weak areas
- 5-6: Need more practice
- <5: Start from the beginning

</details>

---

## 8. Further Learning

### Top Resource (Video/Article)
**"gRPC Crash Course" by Hussein Nasser** on YouTube (~2 hours)
- Covers all concepts with practical demos
- Great for visual learners
- Link: Search "Hussein Nasser gRPC" on YouTube

### Official Documentation
**gRPC Official Documentation**: https://grpc.io/docs/
- **Start with**: "What is gRPC?" → "Core concepts"
- **Then**: Language-specific tutorials (Go/Python/Java)
- **For interviews**: "Guides" section (auth, error handling, performance)

---

## Quick Reference Card (Print This!)

```
┌─────────────────────────────────────────────────────────┐
│                  gRPC INTERVIEW CHEAT SHEET              │
├─────────────────────────────────────────────────────────┤
│ PATTERNS:                                                │
│   Unary:      1 req → 1 res    (Simple CRUD)            │
│   Server:     1 req → N res    (Feeds, Downloads)       │
│   Client:     N req → 1 res    (Uploads, Logs)          │
│   Bidi:       N req ↔ N res    (Chat, Gaming)           │
├─────────────────────────────────────────────────────────┤
│ STATUS CODES (Retry?):                                   │
│   NOT_FOUND (5)         - No                            │
│   INVALID_ARGUMENT (3)  - No                            │
│   UNAVAILABLE (14)      - Yes (backoff)                 │
│   DEADLINE_EXCEEDED (4) - Yes (backoff)                 │
│   UNAUTHENTICATED (16)  - No (re-auth)                  │
├─────────────────────────────────────────────────────────┤
│ MUST DO:                                                 │
│   ✓ Reuse channels                                      │
│   ✓ Set deadlines ALWAYS                                │
│   ✓ Reserve deleted field numbers                       │
│   ✓ Handle all error codes                              │
│   ✓ Use interceptors for cross-cutting                  │
├─────────────────────────────────────────────────────────┤
│ DON'T:                                                   │
│   ✗ New channel per request                             │
│   ✗ Skip timeout                                        │
│   ✗ Reuse field numbers                                 │
│   ✗ Block in async handlers                             │
└─────────────────────────────────────────────────────────┘
```

---

**Good luck with your interviews! 🚀**
