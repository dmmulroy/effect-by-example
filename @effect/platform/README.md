# Effect Platform Guides

This directory contains comprehensive guides for all major @effect/platform modules, following the standardized template and best practices defined in the project's CLAUDE.md documentation standards.

## Available Guides

### Tier 1: Core Platform Modules
**Essential modules for most Effect platform applications**

- **[HttpClient-Guide.md](./HttpClient-Guide.md)** - HTTP client requests, authentication, retries, and caching
- **[HttpServer-Guide.md](./HttpServer-Guide.md)** - HTTP server creation, routing, middleware, and static files  
- **[FileSystem-Guide.md](./FileSystem-Guide.md)** - File system operations, streaming, and cross-platform compatibility
- **[KeyValueStore-Guide.md](./KeyValueStore-Guide.md)** - Key-value storage abstractions with multiple backends
- **[Path-Guide.md](./Path-Guide.md)** - Cross-platform file path operations and manipulation

### Tier 2: System Integration Modules
**Important for system-level operations and infrastructure**

- **[Terminal-Guide.md](./Terminal-Guide.md)** - Terminal input/output, CLI tools, and interactive applications
- **[Command-Guide.md](./Command-Guide.md)** - Command execution, build automation, and system operations
- **[Socket-Guide.md](./Socket-Guide.md)** - TCP/UDP networking, real-time communication, and inter-process communication
- **[Worker-Guide.md](./Worker-Guide.md)** - Parallel processing, CPU-intensive tasks, and background jobs
- **[Runtime-Guide.md](./Runtime-Guide.md)** - Application lifecycle, graceful shutdown, and error handling

### Tier 3: Specialized Modules
**Advanced modules for specific use cases**

- **[HttpRouter-Guide.md](./HttpRouter-Guide.md)** - HTTP routing, middleware composition, and REST API patterns
- **[HttpApi-Guide.md](./HttpApi-Guide.md)** - Declarative API definition, OpenAPI generation, and type-safe clients
- **[PlatformLogger-Guide.md](./PlatformLogger-Guide.md)** - File-based logging, structured logging, and log rotation

## Guide Standards

All guides in this directory follow the established standards:

### Template Structure
Each guide includes these sections:
1. **Introduction & Core Concepts** - Problem/solution pattern with concrete examples
2. **Basic Usage Patterns** - Progressive learning from simple to intermediate
3. **Real-World Examples** - 3+ complete, practical scenarios
4. **Advanced Features Deep Dive** - Power-user techniques and optimization
5. **Practical Patterns & Best Practices** - Reusable helpers and common patterns
6. **Integration Examples** - Popular library integrations and testing strategies

### Code Quality Standards
- **Effect.gen + yield*** for business logic and sequential operations
- **.pipe** for post-processing, error handling, and composition
- Complete, runnable examples with all necessary imports
- Comprehensive error handling patterns
- Type safety demonstrations
- Cross-platform compatibility considerations
- Integration with popular frameworks and libraries

### Real-World Focus
- Every example solves actual developer problems
- Production-ready patterns and best practices
- Performance considerations and optimization techniques
- Testing strategies with practical examples
- Integration patterns with existing codebases

## Getting Started

1. **New to Effect Platform?** Start with [HttpClient-Guide.md](./HttpClient-Guide.md) or [FileSystem-Guide.md](./FileSystem-Guide.md)
2. **Building APIs?** Check out [HttpServer-Guide.md](./HttpServer-Guide.md) and [HttpRouter-Guide.md](./HttpRouter-Guide.md)
3. **System Integration?** Explore [Command-Guide.md](./Command-Guide.md) and [Terminal-Guide.md](./Terminal-Guide.md)
4. **Performance & Scaling?** See [Worker-Guide.md](./Worker-Guide.md) and [Socket-Guide.md](./Socket-Guide.md)

## Module Dependencies

Many platform modules work together. Common integration patterns:

- **HttpServer + HttpRouter + HttpApi** - Complete web application stack
- **FileSystem + Path** - File operations with safe path handling
- **Command + Terminal** - CLI tool development
- **HttpClient + Socket** - Network communication patterns
- **Worker + Runtime** - Background processing systems
- **KeyValueStore + PlatformLogger** - Data persistence and monitoring

## Contributing

When adding new guides or updating existing ones:

1. Follow the CLAUDE.md template structure exactly
2. Use the established Effect-TS syntax patterns (Effect.gen + yield* for business logic)
3. Include 3+ real-world examples per guide
4. Ensure all code examples are complete and runnable
5. Add integration examples with popular libraries
6. Include comprehensive testing strategies

For more details, see the project's [CLAUDE.md](../../CLAUDE.md) file.

## Documentation Coverage

This collection provides comprehensive documentation for **13 major @effect/platform modules**, covering:

- ✅ All core platform abstractions (HTTP, FileSystem, Path, Storage)
- ✅ System integration modules (Terminal, Command, Socket, Worker, Runtime)  
- ✅ Specialized modules (Router, API, Logger)
- ✅ Real-world integration patterns
- ✅ Production-ready examples and best practices
- ✅ Cross-platform compatibility guidance
- ✅ Comprehensive testing strategies

Each guide is designed to be immediately practical while teaching Effect's fundamental patterns and benefits.