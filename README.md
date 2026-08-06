# httpservercpplinux

A minimal multithreaded HTTP server written from scratch in C++ on Linux, using raw POSIX sockets — no external web framework. It includes a small Express-style router supporting path parameters (e.g. `/user/:id`) and `GET`/`POST` route handlers.

## How It Works

1. `HTTPServer` opens a TCP socket, binds it to a port, and listens for connections.
2. On each accepted connection, a new `std::thread` is spawned to handle that client independently.
3. The raw HTTP request is parsed for its method and path.
4. The request is dispatched to `Router`, which matches the path against registered routes (including `:param` segments converted to regex capture groups) and invokes the matching handler.
5. The handler's return value is wrapped in a bare HTTP response and written back to the client socket.

## Routes Registered (in `src/server.cpp`)

| Method | Path | Behavior |
|---|---|---|
| `GET` | `/hello` | Returns `Hello, World!` |
| `GET` | `/user/:id` | Returns the `id` path parameter |
| `POST` | `/login` | Returns `Login Successful` |

## Project Structure — File Index

```
httpservercpplinux/
├── LICENSE                  Apache License 2.0.
│
├── include/
│   ├── server.hpp            Declares HTTPServer: holds the listening socket, port, and worker
│   │                         thread pool; exposes the constructor and start().
│   └── router.hpp            Declares Router: registers GET/POST routes (with :param support via
│                             regex), matches incoming requests, and extracts path parameters.
│                             Defines the RouteHandler function type.
│
└── src/
    ├── main.cpp               Entry point — constructs an HTTPServer on port 8080 and starts it.
    │
    ├── server.cpp             Implements HTTPServer: socket creation/bind/listen in the
    │                          constructor, an accept loop in start() that spawns a detached thread
    │                          per client, and handle_client(), which reads the raw request, parses
    │                          the method/path, and dispatches to the global Router instance.
    │                          Also defines setupRoutes(), which registers the /hello, /user/:id,
    │                          and /login routes shown above.
    │
    └── router.cpp             Implements Router: get()/post() convert path patterns like
                               "/user/:id" into regexes and store the handler; has_route() checks
                               for a method+path match; handleRequest() matches the path, extracts
                               named parameters via extractParams(), and calls the handler, wrapping
                               its return value in a bare "HTTP/1.1 200 OK" response (or 404 if no
                               route matches).
```

## Building

No build system is checked in (no `Makefile`/`CMakeLists.txt` yet). To compile directly:

```bash
g++ -std=c++17 -Iinclude src/*.cpp -o httpserver -lpthread
./httpserver
```

Then, in another terminal:

```bash
curl http://localhost:8080/hello
curl http://localhost:8080/user/42
curl -X POST http://localhost:8080/login
```

## Known Limitations

- Requests are read into a fixed 1024-byte buffer (`char buffer[1024]`) — larger requests will be truncated.
- No handling of HTTP headers, query strings, or request bodies (e.g. `/login` doesn't read POST body data).
- No graceful shutdown, connection keep-alive, or error handling around malformed requests.
- `start()` logs `"Server started on port "` without actually printing the port number.

These are natural next steps rather than bugs in what's implemented — worth keeping in mind if you extend this for a portfolio piece.
