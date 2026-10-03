What is FastAPI and why would you choose it over Flask or Django?
What are the main features of FastAPI?
How does FastAPI achieve high performance?
What is ASGI?
What is the difference between ASGI and WSGI?
What is Uvicorn?
What is Starlette and how is it related to FastAPI?
How do you create a basic FastAPI application?
How do you start a FastAPI application using Uvicorn?
What is an API path operation?
What is the difference between GET, POST, PUT, PATCH, and DELETE?
How do you define path parameters?
How do you define query parameters?
How do you define request body parameters?
What is Pydantic?
Why does FastAPI use Pydantic?
What is request validation?
How does FastAPI automatically generate API documentation?
What is OpenAPI?
What are Swagger UI and ReDoc?
"Explain how FastAPI processes a request from the time it reaches the server until the response is returned."

How do you define path parameters in FastAPI?
How do you validate path parameters?
How do you define optional query parameters?
What is the difference between path and query parameters?
How do you define request bodies?
How do you accept multiple body parameters?
How do you handle nested Pydantic models?
How do you define default values?
How do you validate string length?
How do you validate numeric ranges?
How do you validate email addresses?
How do you create reusable Pydantic models?
How do you handle list and dictionary request bodies?
How do you upload files using FastAPI?
How do you return a custom HTTP status code?

What problem does Pydantic solve?
What is the difference between Pydantic models and Python dataclasses?
How does Pydantic perform validation?
How do you create custom validators?
How do you validate nested objects?
How do you make a field optional?
How do you handle extra/unknown fields?
How do you serialize a Pydantic model?
How do you convert a Pydantic model to a dictionary?
What is the difference between request models and response models?
Why should you define separate request and response schemas?
How do you prevent sensitive fields from appearing in API responses?
How do you handle backward compatibility when changing schemas?
How would you validate an enum field?
How would you handle different API versions with different schemas?

What is dependency injection in FastAPI?
What is Depends()?
Why would you use dependencies?
How do you create a reusable dependency?
Can dependencies depend on other dependencies?
How do you use dependencies for authentication?
How do you inject a database session?
How do you share configuration through dependencies?
What is the lifecycle of a dependency?
What is the difference between dependency injection and middleware?
How do you override dependencies during testing?
How do you create class-based dependencies?
Can dependencies be asynchronous?
How would you implement role-based authorization using dependencies?
What problems can occur if dependencies contain expensive operations?


"You have 100 APIs and every API needs the authenticated user. How would you design this?"

What is asynchronous programming?
What is asyncio?
What does async def mean?
What does await do?
What is an event loop?
What is the difference between synchronous and asynchronous code?
What is I/O-bound work?
What is CPU-bound work?
When should you use async def in FastAPI?
When should you use normal def?
What happens if you call blocking code inside an async endpoint?
Why is time.sleep() problematic inside an async endpoint?
What should you use instead?
Can you use synchronous libraries inside FastAPI?
How does FastAPI handle synchronous endpoints?
What is an async database driver?
How do you make HTTP calls asynchronously?
How do you handle concurrent external API calls?
What is asyncio.gather()?
How would you troubleshoot an API whose latency suddenly increased?

What is middleware?
How do you create middleware in FastAPI?
What is the execution order of middleware?
How do you implement request logging?
How do you measure request latency?
How do you add correlation IDs?
How do you implement CORS?
What is CORS?
How do you add security headers?
How do you handle exceptions globally?
Middleware vs dependency — when would you use each?
How can middleware affect application performance?
How do you exclude specific routes from middleware logic?
How do you implement request/response logging without logging sensitive information?
How would you implement distributed tracing middleware?


How do you implement authentication in FastAPI?
What is JWT?
How does JWT authentication work?
What is the difference between authentication and authorization?
Where should JWT tokens be stored?
What are access tokens and refresh tokens?
How do you implement OAuth2 with FastAPI?
What is OAuth2 Password Flow?
What is OpenID Connect?
How do you implement role-based access control?
How do you implement permission-based authorization?
How do you expire JWT tokens?
How do you revoke a JWT?
What are the security risks of JWT?
What is token rotation?
How would you secure an internal FastAPI service?
How do you validate tokens issued by an external identity provider?
How do you integrate FastAPI with AWS Cognito?
How would you protect admin endpoints?
How would you prevent privilege escalation?

How do you connect FastAPI to PostgreSQL?
What ORM have you used?
What is SQLAlchemy?
What is SQLModel?
How do you create a database session?
Why shouldn't you create a new database connection for every request?
What is connection pooling?
How do you configure connection pool size?
What happens when the database connection pool is exhausted?
How do you handle database transactions?
What is commit vs rollback?
How do you handle database exceptions?
How do you perform migrations?
What is Alembic?
How do you handle database schema changes without downtime?
What is the N+1 query problem?
How do you optimize slow database queries?
How do you implement pagination?
Offset pagination vs cursor pagination?
How do you prevent SQL injection?
How do you implement database health checks?
How would you handle read replicas?
How would you handle database failover?
How would you optimize an API returning 1 million records?
How would you design a repository/service layer?


How would you structure a large FastAPI project?
Why shouldn't all business logic be inside route handlers?
What is the service layer?
What is the repository pattern?
How would you separate API, business logic, and database logic?
How do you organize routers?
How do you organize Pydantic schemas?
How do you organize configuration?
How do you manage environment variables?
How do you manage secrets?
How would you implement API versioning?
URI versioning vs header versioning?
How do you maintain backward compatibility?
How do you design reusable components?
How would you prevent circular imports?
How do you handle shared utilities?
How do you design a modular FastAPI application?
Monolith vs microservices — when would you choose each?
How would you split a large FastAPI monolith?
How would you design FastAPI services for independent deployment?


Why is FastAPI considered fast?
How would you benchmark a FastAPI application?
What metrics would you monitor?
How do you reduce API latency?
How do you identify slow endpoints?
How would you handle 10,000 requests per second?
How does Uvicorn handle concurrency?
What are Uvicorn workers?
When should you increase workers?
Worker processes vs threads?
How does Kubernetes scale FastAPI?
How would you configure HPA?
CPU-based vs memory-based autoscaling?
What is connection pooling?
How would Redis improve API performance?
When should you introduce caching?
What data should not be cached?
How do you prevent cache stampede?
What is rate limiting?
How would you implement rate limiting?
How would you handle sudden traffic spikes?
How would you optimize JSON serialization?
How would you handle large file downloads?
How would you stream large responses?
How would you reduce memory consumption?

What are FastAPI BackgroundTasks?
When should you use BackgroundTasks?
When should you NOT use BackgroundTasks?
BackgroundTasks vs Celery?
BackgroundTasks vs AWS SQS?
How would you process a long-running job?
How would you implement asynchronous job processing?
How would you design a job status API?
How do you retry failed jobs?
What is exponential backoff?
How do you prevent duplicate job execution?
What is idempotency?
How would you implement an idempotent POST API?
How would you handle failed messages?
What is a dead-letter queue?


How do you test FastAPI applications?
What is pytest?
What is TestClient?
How do you test GET endpoints?
How do you test POST endpoints?
How do you test validation failures?
How do you test authentication?
How do you mock external APIs?
How do you mock database dependencies?
How do you override Depends() during testing?
Unit tests vs integration tests?
What should be mocked?
What shouldn't be mocked?
How do you test asynchronous endpoints?
How do you test exception handlers?
How do you test middleware?
How do you test database transactions?
How do you test race conditions?
How do you measure test coverage?
How would you integrate FastAPI tests into GitLab CI/CD?

How do you monitor a FastAPI application?
What metrics would you expose?
What is Prometheus?
How do you expose /metrics?
What is OpenTelemetry?
What is distributed tracing?
What is a trace?
What is a span?
What is a correlation ID?
Correlation ID vs trace ID?
How do you propagate trace context between microservices?
How would you instrument FastAPI with OpenTelemetry?
How would you monitor external API latency?
How would you create useful Grafana dashboards?
What alerts would you create?
How would you detect memory leaks?
How would you troubleshoot high CPU?
How would you troubleshoot increasing latency?
How would you troubleshoot HTTP 5xx spikes?
What are the four golden signals?

How do you Dockerize a FastAPI application?
What should your Dockerfile look like?
Why use a non-root user?
How do you reduce FastAPI image size?
What is a multi-stage Docker build?
How do you pass environment variables to FastAPI?
How do you manage secrets in Kubernetes?
How do you deploy FastAPI on Kubernetes?
Deployment vs StatefulSet?
How do you configure readiness probes?
How do you configure liveness probes?
What should /health return?
What is the difference between liveness and readiness?
How do you perform zero-downtime deployments?
How do you configure resource requests and limits?
How does HPA work?
How would you troubleshoot CrashLoopBackOff?
How would you troubleshoot OOMKilled?
How would you troubleshoot slow Kubernetes pods?
How would you perform a rollback?

ow would you build a CI/CD pipeline for FastAPI?
What checks should happen before deployment?
Lint
 ↓
Unit Tests
 ↓
Security Scan
 ↓
Build Docker Image
 ↓
Image Scan
 ↓
Push Registry
 ↓
Deploy
 ↓
Smoke Tests
How do you manage environment-specific configuration?
How do you perform database migrations during deployment?
How do you rollback application code?
How do you rollback database changes?
How would you implement blue-green deployment?
What is canary deployment?
How would you deploy FastAPI using ArgoCD?
How would you manage secrets using AWS Secrets Manager?
How would you handle configuration changes without rebuilding the image?
How would you secure the CI/CD pipeline?
How do you scan Python dependencies?
How do you scan Docker images?
How do you prevent vulnerable dependencies from reaching production?


Design a scalable FastAPI microservice architecture.
Design a FastAPI API handling 100K requests/minute.
Design an API gateway using FastAPI.
Design a multi-tenant FastAPI application.
Design a FastAPI authentication service.
Design an asynchronous document-processing API.
Design a FastAPI service using PostgreSQL + Redis + SQS.
Design a distributed FastAPI system with OpenTelemetry.
Design an API that must maintain 99.99% availability.
Design an API with idempotent payment/order creation.
Design an API supporting millions of users.
Design a file-upload API for multi-GB files.
Design an API for long-running ML jobs.
Design an API with rate limiting.
Design an API supporting API versioning.
Design a zero-downtime FastAPI deployment.
Design a highly available FastAPI service across multiple AZs.
Design FastAPI services that communicate through Kafka/SQS.
Design a FastAPI service capable of surviving downstream service failures.
Design a production-grade FastAPI platform on AWS EKS.