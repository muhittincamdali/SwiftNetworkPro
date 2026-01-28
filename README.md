# SwiftNetworkPro

<p align="center">
  <img src="Assets/banner.png" alt="SwiftNetworkPro" width="800">
</p>

<p align="center">
  <a href="https://swift.org"><img src="https://img.shields.io/badge/Swift-5.9+-F05138?style=flat&logo=swift&logoColor=white" alt="Swift"></a>
  <a href="https://developer.apple.com/ios/"><img src="https://img.shields.io/badge/iOS-15.0+-000000?style=flat&logo=apple&logoColor=white" alt="iOS"></a>
  <a href="LICENSE"><img src="https://img.shields.io/badge/License-MIT-green.svg" alt="License"></a>
</p>

<p align="center">
  <b>Modern networking library for iOS with async/await, type-safe requests, and automatic retry.</b>
</p>

---

## Features

- **Async/Await** — Modern Swift concurrency
- **Type-Safe** — Codable request/response handling
- **Retry Logic** — Automatic retry with exponential backoff
- **Request Interceptors** — Authentication, logging, caching
- **WebSocket** — Real-time communication support
- **Certificate Pinning** — Enhanced security

## Installation

```swift
dependencies: [
    .package(url: "https://github.com/muhittincamdali/SwiftNetworkPro.git", from: "1.0.0")
]
```

## Quick Start

### Basic GET Request

```swift
import SwiftNetworkPro

struct User: Codable {
    let id: Int
    let name: String
    let email: String
}

let client = NetworkClient()

// Simple GET
let user: User = try await client.get("https://api.example.com/users/1")
print(user.name)
```

### POST Request with Body

```swift
struct CreateUserRequest: Codable {
    let name: String
    let email: String
}

struct CreateUserResponse: Codable {
    let id: Int
    let name: String
}

let request = CreateUserRequest(name: "John", email: "john@example.com")
let response: CreateUserResponse = try await client.post(
    "https://api.example.com/users",
    body: request
)
```

### Define API Endpoints

```swift
enum UserAPI: Endpoint {
    case getUser(id: Int)
    case createUser(name: String, email: String)
    case updateUser(id: Int, name: String)
    case deleteUser(id: Int)
    
    var baseURL: URL { URL(string: "https://api.example.com")! }
    
    var path: String {
        switch self {
        case .getUser(let id): return "/users/\(id)"
        case .createUser: return "/users"
        case .updateUser(let id, _): return "/users/\(id)"
        case .deleteUser(let id): return "/users/\(id)"
        }
    }
    
    var method: HTTPMethod {
        switch self {
        case .getUser: return .get
        case .createUser: return .post
        case .updateUser: return .put
        case .deleteUser: return .delete
        }
    }
    
    var body: Encodable? {
        switch self {
        case .createUser(let name, let email):
            return ["name": name, "email": email]
        case .updateUser(_, let name):
            return ["name": name]
        default:
            return nil
        }
    }
}

// Usage
let user: User = try await client.request(UserAPI.getUser(id: 1))
```

### Request with Headers

```swift
let response: User = try await client.request(
    UserAPI.getUser(id: 1),
    headers: [
        "Authorization": "Bearer \(token)",
        "Accept-Language": "en-US"
    ]
)
```

### Automatic Retry

```swift
let config = NetworkConfig(
    retryPolicy: RetryPolicy(
        maxRetries: 3,
        delay: 1.0,
        multiplier: 2.0, // Exponential backoff
        retryableStatusCodes: [408, 500, 502, 503, 504]
    )
)

let client = NetworkClient(config: config)
```

### Request Interceptors

```swift
// Authentication interceptor
class AuthInterceptor: RequestInterceptor {
    func intercept(_ request: inout URLRequest) async {
        let token = await TokenManager.shared.getToken()
        request.setValue("Bearer \(token)", forHTTPHeaderField: "Authorization")
    }
}

// Logging interceptor
class LoggingInterceptor: RequestInterceptor {
    func intercept(_ request: inout URLRequest) async {
        print("➡️ \(request.httpMethod ?? "") \(request.url?.absoluteString ?? "")")
    }
    
    func intercept(_ response: HTTPURLResponse, data: Data) async {
        print("⬅️ \(response.statusCode)")
    }
}

let client = NetworkClient(
    interceptors: [AuthInterceptor(), LoggingInterceptor()]
)
```

### Error Handling

```swift
do {
    let user: User = try await client.get("https://api.example.com/users/1")
} catch NetworkError.httpError(let statusCode, let data) {
    print("HTTP Error: \(statusCode)")
    if let errorResponse = try? JSONDecoder().decode(APIError.self, from: data) {
        print("Message: \(errorResponse.message)")
    }
} catch NetworkError.noConnection {
    print("No internet connection")
} catch NetworkError.timeout {
    print("Request timed out")
} catch {
    print("Unknown error: \(error)")
}
```

### WebSocket

```swift
let socket = WebSocketClient(url: URL(string: "wss://api.example.com/ws")!)

// Connect
try await socket.connect()

// Send message
try await socket.send("Hello, server!")

// Send JSON
let message = ChatMessage(text: "Hi!", userId: 123)
try await socket.send(message)

// Receive messages
for await message in socket.messages {
    switch message {
    case .text(let string):
        print("Received: \(string)")
    case .data(let data):
        print("Received data: \(data.count) bytes")
    }
}

// Disconnect
await socket.disconnect()
```

### Download with Progress

```swift
let progress = Progress()
let fileURL = try await client.download(
    "https://example.com/file.zip",
    to: documentsDirectory.appendingPathComponent("file.zip"),
    progress: progress
)

// Observe progress
progress.publisher(for: \.fractionCompleted)
    .sink { fraction in
        print("Downloaded: \(Int(fraction * 100))%")
    }
```

### Upload with Multipart

```swift
let imageData = UIImage(named: "photo")!.jpegData(compressionQuality: 0.8)!

let response: UploadResponse = try await client.upload(
    "https://api.example.com/upload",
    multipart: [
        .data(imageData, name: "file", fileName: "photo.jpg", mimeType: "image/jpeg"),
        .text("Photo description", name: "description")
    ]
)
```

## Project Structure

```
SwiftNetworkPro/
├── Sources/
│   ├── Core/
│   │   ├── NetworkClient.swift
│   │   ├── Endpoint.swift
│   │   └── HTTPMethod.swift
│   ├── Interceptors/
│   │   ├── RequestInterceptor.swift
│   │   └── AuthInterceptor.swift
│   ├── WebSocket/
│   │   └── WebSocketClient.swift
│   └── Utils/
│       ├── RetryPolicy.swift
│       └── MultipartFormData.swift
├── Examples/
└── Tests/
```

## Requirements

- iOS 15.0+ / macOS 12.0+
- Xcode 15.0+
- Swift 5.9+

## Documentation

- [Getting Started](Documentation/GettingStarted.md)
- [Endpoints](Documentation/Endpoints.md)
- [Interceptors](Documentation/Interceptors.md)
- [WebSocket](Documentation/WebSocket.md)

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md).

## License

MIT License. See [LICENSE](LICENSE).

## Author

**Muhittin Camdali** — [@muhittincamdali](https://github.com/muhittincamdali)

---

<p align="center">
  <sub>Networking made simple ❤️</sub>
</p>
