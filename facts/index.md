---
facts:
  - slug: api-silent-empty-response
    category: Backend
    question: >-
      # API endpoint returns 200 OK but the response body is empty


      The database has data, the endpoint is accessible, but the response body
      is completely empty. What are the most common causes of silent empty
      responses?
    tags:
      - debugging
      - api
      - backend
      - experience
    date: '2026-04-06'
  - slug: api-versioning-strategies
    category: Backend
    question: >-
      # API Versioning Strategies


      You deploy a new API version (v2) but mobile apps on app stores still use
      v1. You need to support both versions without duplicating code. How do you
      manage API versioning without breaking existing clients?
    tags:
      - api-design
      - versioning
      - backend
      - interview
    date: '2026-04-06'
  - slug: async-job-processing-at-scale
    category: Backend
    question: >-
      # Async Job Processing at Scale


      You need to send 100,000 emails after a marketing campaign triggers. Doing
      it synchronously blocks your API for hours. How do you design async job
      processing that scales reliably?
    tags:
      - queue
      - background-jobs
      - scalability
      - interview
    date: '2026-04-06'
  - slug: canary-deployment-progressive-rollout
    category: DevOps
    question: >-
      You deploy a new feature to 100% of users and it crashes for 10% of them.
      How do you implement canary deployments and progressive rollouts?
    tags:
      - deployment
      - canary
      - reliability
      - interview
    date: '2026-04-06'
  - slug: centralized-logging-elk-loki
    category: DevOps
    question: >-
      # Centralized Logging with ELK and Loki


      Your production containers log to stdout but nobody monitors them. An
      error loop generates 1GB of logs per hour. How do you set up centralized
      logging?
    tags:
      - logging
      - elk
      - loki
      - observability
      - experience
    date: '2026-04-06'
  - slug: change-data-capture-debezium
    category: Data
    question: >-
      # Change Data Capture with Debezium


      You need to sync data between PostgreSQL and Elasticsearch in real-time.
      Manual sync scripts often miss updates. How does Change Data Capture (CDC)
      with Debezium work?
    tags:
      - cdc
      - debezium
      - kafka
      - data-engineering
      - interview
    date: '2026-04-06'
  - slug: circuit-breaker-microservices
    category: Architecture
    question: >-
      Your `Order Service` calls the `Payment Service`, which calls the `Fraud
      Detection Service`. The Fraud service goes down. Now every order request
      hangs for 30 seconds waiting for a timeout, your thread pool fills up, and
      the entire checkout system cascades into failure. How do you prevent one
      failing microservice from taking down the whole system?
    tags:
      - microservices
      - resilience
      - senior
      - interview
    date: '2026-04-06'
  - slug: client-vs-server-state
    category: Frontend
    question: >-
      Your team debates between client-side state (Zustand/Jotai) and server
      state (TanStack Query). What's the difference and when should you use
      each?
    tags:
      - react
      - state-management
      - tanstack-query
      - interview
    date: '2026-04-06'
  - slug: complex-form-management-react
    category: Frontend
    question: >-
      # Complex Form Management in React


      You build a form with 20 fields, complex validation, and conditional
      logic. useState becomes a nightmare. How do you manage complex forms in
      React?
    tags:
      - react
      - forms
      - validation
      - experience
    date: '2026-04-06'
  - slug: container-image-tagging-strategy
    category: DevOps
    question: >-
      Your container image uses latest tag. Production mysteriously breaks even
      though nobody deployed. How do you implement proper container image
      tagging and immutable deployments?
    tags:
      - docker
      - deployment
      - ci-cd
      - experience
    date: '2026-04-06'
  - slug: cors-configuration-properly
    category: Security
    question: >-
      # CORS Configuration Done Properly


      Your application sets Access-Control-Allow-Origin to * for convenience.
      Why is this dangerous and how do you configure CORS properly?
    tags:
      - cors
      - api
      - frontend
      - security
      - interview
    date: '2026-04-06'
  - slug: css-architecture-large-project
    category: Frontend
    question: >-
      Your team writes CSS in 5 different ways — global CSS, CSS modules,
      styled-components, Tailwind, inline styles. How do you choose a CSS
      architecture for a large project?
    tags:
      - css
      - architecture
      - tailwind
      - experience
    date: '2026-04-06'
  - slug: cursor-based-pagination
    category: Backend
    question: >-
      # Implement Efficient Cursor-Based Pagination


      Your REST API uses OFFSET/LIMIT pagination. At page 10,000 the query takes
      30 seconds because it scans 10M rows. How do you implement efficient
      cursor-based pagination that stays fast at any offset?
    tags:
      - api-design
      - performance
      - database
      - interview
    date: '2026-04-06'
  - slug: database-index-anti-patterns
    category: Database
    question: >-
      You added an index on `users.email` to speed up login queries. It worked.
      So you added indexes on every column "just in case." Now your INSERT
      throughput dropped 60%, your disk usage tripled, and the query planner is
      ignoring half your indexes anyway. When do indexes actually hurt
      performance, and what are the most common indexing mistakes?
    tags:
      - sql
      - performance
      - experience
      - optimization
    date: '2026-04-06'
  - slug: database-table-partitioning
    category: Database
    question: >-
      Your orders table has 500M rows and queries are slow. You need to split
      it. How do you partition a table and what strategy do you choose?
    tags:
      - postgresql
      - partitioning
      - performance
      - senior
    date: '2026-04-06'
  - slug: design-image-processing-pipeline
    category: Architecture
    question: >-
      # Image Processing Pipeline Architecture


      Your image upload service receives 1,000 images per minute. Each needs to
      be resized into 5 formats. How do you design an image processing pipeline?
    tags:
      - image-processing
      - queue
      - scalability
      - senior
    date: '2026-04-06'
  - slug: design-notification-system
    category: Architecture
    question: >-
      # Multi-Channel Notification System Architecture


      Your notification system needs to send push, email, SMS, and in-app
      notifications. Each channel has different delivery guarantees. How do you
      design a multi-channel notification system?
    tags:
      - notifications
      - system-design
      - scalability
      - senior
    date: '2026-04-06'
  - slug: design-realtime-leaderboard
    category: Architecture
    question: >-
      # Real-Time Leaderboard with Redis Sorted Sets


      You need to build a leaderboard that updates in real-time for 1M users.
      SQL ORDER BY with LIMIT is too slow. How do you design a real-time
      leaderboard with Redis Sorted Sets?
    tags:
      - redis
      - leaderboard
      - system-design
      - interview
    date: '2026-04-06'
  - slug: distributed-transactions-saga-pattern
    category: Architecture
    question: >-
      In a Monolith, if an Order fails to submit, a single SQL `ROLLBACK` undoes
      the database. But in Microservices, the `Inventory DB` deducts the stock
      upon checkout, but the `Payment API` network completely fails. How do you
      rollback an inventory deduction on a separate database when there is no
      single SQL transaction spanning across the microservices?
    tags:
      - microservices
      - database
      - senior
      - system-design
    date: '2026-04-06'
  - slug: docker-container-debugging
    category: DevOps
    question: >-
      It's 2 AM. You deploy a new Docker image to production. `docker ps` shows
      the container restarting every 10 seconds. `docker logs` shows nothing
      useful — just "Killed." You run `docker inspect` and see `OOMKilled:
      true`. What is your systematic approach to debug a container that keeps
      crashing, and what are the most common causes beyond OOM?
    tags:
      - docker
      - debugging
      - experience
      - deployment
    date: '2026-04-06'
  - slug: environment-config-management
    category: DevOps
    question: >-
      # Environment Configuration Management


      You need to deploy to 3 environments (dev, staging, prod) with different
      configs. How do you manage environment-specific configuration without
      hardcoding?
    tags:
      - configuration
      - deployment
      - twelve-factor
      - interview
    date: '2026-04-06'
  - slug: error-boundaries-fault-isolation
    category: Frontend
    question: >-
      # Error Boundaries & Fault Isolation


      A third-party script crashes and takes your entire React app down. How do
      you implement Error Boundaries and fault isolation?
    tags:
      - react
      - error-handling
      - reliability
      - interview
    date: '2026-04-06'
  - slug: event-driven-vs-request-response
    category: Architecture
    question: >-
      # Mixing Event-Driven and Request-Response Patterns


      Your event-driven system processes events asynchronously. But sometimes
      you need a synchronous response. How do you mix request/response with
      event-driven patterns?
    tags:
      - event-driven
      - messaging
      - architecture
      - interview
    date: '2026-04-06'
  - slug: event-sourcing-when-to-use
    category: Architecture
    question: >-
      # Event Sourcing: When to Use It


      Your system processes user actions and needs to know exactly what happened
      and when — for compliance, debugging, and analytics. What is Event
      Sourcing and when should you use it?
    tags:
      - event-sourcing
      - cqrs
      - architecture
      - interview
    date: '2026-04-06'
  - slug: explain-analyze-query-plans
    category: Database
    question: >-
      Your PostgreSQL query was fast with 1M rows but takes 30 seconds with 100M
      rows. EXPLAIN shows a sequential scan. How do you read and optimize query
      execution plans?
    tags:
      - postgresql
      - performance
      - optimization
      - interview
    date: '2026-04-06'
  - slug: feature-flag-system-design
    category: Backend
    question: >-
      Feature flags are scattered across if/else blocks everywhere. Removing old
      flags is terrifying. How do you implement a clean feature flag system?
    tags:
      - feature-flags
      - deployment
      - backend
      - experience
    date: '2026-04-06'
  - slug: fix-unnecessary-react-rerenders
    category: Frontend
    question: >-
      # Fix Unnecessary React Re-renders


      Your React component re-renders 47 times when you type one character in an
      input. How do you diagnose and fix unnecessary re-renders?
    tags:
      - react
      - performance
      - debugging
      - interview
    date: '2026-04-06'
  - slug: flatten-nested-conditionals
    category: Patterns
    question: >-
      # Flattening Nested Conditionals


      Your code has deeply nested if/else blocks — 6 levels deep. It's
      unreadable. How do you flatten nested conditionals with early returns and
      guard clauses?
    tags:
      - clean-code
      - refactoring
      - readability
      - experience
    date: '2026-04-06'
  - slug: frontend-monorepo-turborepo
    category: Frontend
    question: >-
      Your monorepo has 3 apps and 20 shared packages. Changing a shared
      component means rebuilding everything. How do you optimize builds in a
      monorepo with Turborepo?
    tags:
      - monorepo
      - turborepo
      - build
      - experience
    date: '2026-04-06'
  - slug: fulltext-search-postgresql
    category: Database
    question: >-
      You need to search products by name, description, and tags with typo
      tolerance. PostgreSQL LIKE is too slow. How do you implement full-text
      search — pg_trgm vs tsvector vs Elasticsearch?
    tags:
      - postgresql
      - search
      - performance
      - interview
    date: '2026-04-06'
  - slug: gitops-with-argocd
    category: DevOps
    question: >-
      # GitOps with ArgoCD


      Your team pushes to main and it auto-deploys. But nobody reviews the
      infrastructure changes. How do you implement GitOps with ArgoCD?
    tags:
      - gitops
      - argocd
      - kubernetes
      - interview
    date: '2026-04-06'
  - slug: handle-replication-lag
    category: Database
    question: >-
      Your application writes to a primary and reads from a replica. But
      read-after-write shows stale data because replication lag is 500ms. How do
      you handle replication lag?
    tags:
      - replication
      - consistency
      - postgresql
      - interview
    date: '2026-04-06'
  - slug: high-availability-database-setup
    category: Database
    question: >-
      Your primary database goes down. Failover to the replica takes 5 minutes
      and you lose 30 seconds of writes. How do you design a high-availability
      database setup?
    tags:
      - postgresql
      - ha
      - failover
      - senior
    date: '2026-04-06'
  - slug: http2-multiplexing-performance
    category: Networking
    question: >-
      # HTTP/2 Multiplexing for High-Concurrency APIs


      Your API uses HTTP/1.1 with keep-alive. Under high concurrency,
      connections queue up. How does HTTP/2 multiplexing solve head-of-line
      blocking?
    tags:
      - http2
      - networking
      - performance
      - interview
    date: '2026-04-06'
  - slug: idempotency-keys-in-apis
    category: Backend
    question: >-
      A mobile user on a flaky 3G connection hits the "Pay $100" button twice
      because nothing loaded. Your API receives two identical HTTP POST requests
      at the exact same millisecond. How do you prevent charging their credit
      card $200 while maintaining a graceful user experience?
    tags:
      - architecture
      - senior
      - api
      - stripe
    date: '2026-04-06'
  - slug: image-optimization-core-web-vitals
    category: Frontend
    question: >-
      # Image Optimization for Core Web Vitals


      Your image-heavy page loads 50 high-res images at once. LCP is 8 seconds.
      How do you optimize image loading for Core Web Vitals?
    tags:
      - performance
      - images
      - core-web-vitals
      - experience
    date: '2026-04-06'
  - slug: instant-rollback-strategies
    category: DevOps
    question: >-
      # Instant Rollback Strategies


      You deploy to production and the new version has a critical bug. Rolling
      back takes 20 minutes. How do you implement instant rollback strategies?
    tags:
      - deployment
      - rollback
      - reliability
      - interview
    date: '2026-04-06'
  - slug: jsonb-indexing-querying
    category: Database
    question: >-
      You store JSON in a PostgreSQL JSONB column for flexibility. Queries that
      filter on JSON fields are slow. How do you index and query JSONB
      efficiently?
    tags:
      - postgresql
      - jsonb
      - performance
      - experience
    date: '2026-04-06'
  - slug: junior-to-senior-developer
    category: Career
    question: >-
      # Becoming a Senior Developer


      You're a mid-level developer aiming for senior. What concrete skills and
      behaviors differentiate a senior engineer from a mid-level one?
    tags:
      - career
      - growth
      - senior
      - experience
    date: '2026-04-06'
  - slug: jwt-in-localstorage-xss
    category: Security
    question: >-
      Every tutorial tells you to build your React app, get the JWT token from
      the login API, and do `localStorage.setItem('token', jwt)`. 

      However, Senior Security Engineers will instantly fail your PR if you do
      this. Why is putting JWTs in LocalStorage considered a critical security
      vulnerability, and how should you actually store them?
    tags:
      - jwt
      - auth
      - security
      - interview
      - frontend
    date: '2026-04-06'
  - slug: kubernetes-hpa-autoscaling-pitfalls
    category: DevOps
    question: >-
      Your service already has HPA (Horizontal Pod Autoscaler) with CPU-based
      scaling and it works fine under normal load. But when traffic spikes
      suddenly — latency shoots up, requests start timing out, and users see
      errors. The pods ARE scaling... just not fast enough. What's wrong with
      CPU-only autoscaling and how do you design an autoscale strategy that
      actually handles traffic bursts?
    tags:
      - kubernetes
      - autoscaling
      - hpa
      - interview
    date: '2026-04-06'
  - slug: kubernetes-ingress-service-exposure
    category: DevOps
    question: >-
      # Kubernetes Service Exposure Strategies


      You need to expose a Kubernetes service to the internet. Should you use
      NodePort, LoadBalancer, or Ingress? What are the trade-offs?
    tags:
      - kubernetes
      - networking
      - ingress
      - interview
    date: '2026-04-06'
  - slug: kubernetes-resource-tuning
    category: DevOps
    question: >-
      # Kubernetes Resource Tuning


      Your Kubernetes pod gets OOMKilled randomly. Memory limits are set to
      512Mi but the app sometimes spikes to 600Mi. How do you tune resource
      requests and limits?
    tags:
      - kubernetes
      - resources
      - oom
      - experience
    date: '2026-04-06'
  - slug: linux-oom-killer-protection
    category: OS
    question: >-
      # Linux OOM Killer and Process Protection


      Your server's OOM killer randomly kills your database process. How does
      Linux OOM killer work and how do you protect critical processes?
    tags:
      - linux
      - memory
      - oom
      - devops
      - experience
    date: '2026-04-06'
  - slug: load-balancing-algorithms
    category: Networking
    question: >-
      # Load Balancing Algorithms


      Your load balancer uses round-robin but some servers have 10x more load
      than others. Why does this happen and what's a better load balancing
      algorithm?
    tags:
      - load-balancing
      - networking
      - scalability
      - interview
    date: '2026-04-06'
  - slug: loading-states-skeleton-screens
    category: Frontend
    question: >-
      # Loading States & Skeleton Screens


      You fetch data in useEffect, show a loading spinner, then render. But
      users see content flash and layout shift. How do you implement proper
      loading states and skeleton screens?
    tags:
      - ux
      - performance
      - react
      - experience
    date: '2026-04-06'
  - slug: meaningful-health-checks
    category: Backend
    question: >-
      # Design Meaningful Health Checks


      Your health check endpoint returns 200 OK but the service is actually
      broken. It can't reach the database, can't connect to external APIs, but
      the liveness probe says everything is fine. How do you design health
      checks that actually reflect service health?
    tags:
      - monitoring
      - devops
      - reliability
      - experience
    date: '2026-04-06'
  - slug: mobile-offline-first-sync
    category: Mobile
    question: >-
      # Offline-First Data Sync with Conflict Resolution


      Your app needs to work offline and sync data when connectivity returns.
      Conflicts arise when two users edit the same record offline. How do you
      implement offline-first with conflict resolution?
    tags:
      - mobile
      - offline
      - sync
      - interview
    date: '2026-04-06'
  - slug: mocks-vs-real-dependencies-testing
    category: Testing
    question: >-
      # Mocks vs Real Dependencies: Testing Strategy


      Your test suite mocks everything. Tests pass but bugs still reach
      production. When should you use real dependencies vs mocks?
    tags:
      - testing
      - mocks
      - integration
      - interview
    date: '2026-04-06'
  - slug: monolith-vs-microservices-scaling
    category: Architecture
    question: >-
      # Monolith vs Microservices: When to Scale Out


      Your monolith handles 10,000 users fine. CEO says we're growing to 1M
      users. Do you really need microservices or can the monolith scale?
    tags:
      - microservices
      - monolith
      - scaling
      - interview
    date: '2026-04-06'
  - slug: multi-tenant-data-isolation
    category: Backend
    question: >-
      Your multi-tenant SaaS accidentally shows Company A's data to Company B.
      How do you implement proper tenant isolation?
    tags:
      - saas
      - multi-tenant
      - security
      - senior
    date: '2026-04-06'
  - slug: n-plus-one-query-problem
    category: Database
    question: >-
      "What is the N+1 Query Problem?"

      This is one of the most common Backend/Database interview questions. When
      you test locally, your API responds in 50ms. As soon as you deploy to
      production with real data, the API takes 5 seconds and crashes the
      database. Why did your ORM betray you, and how do you fix it?
    tags:
      - sql
      - orm
      - interview
      - performance
    date: '2026-04-06'
  - slug: never-use-float-for-money
    category: Database
    question: >-
      Your database stores prices as FLOAT and customers see charges like
      $19.990000000001. Why should you never use floating point for money?
    tags:
      - sql
      - money
      - precision
      - experience
    date: '2026-04-06'
  - slug: nextjs-rendering-strategies
    category: Frontend
    question: >-
      # Next.js Rendering Strategies


      Your Next.js app has SSR, SSG, ISR, and client-side rendering mixed
      together. Pages sometimes show stale data. How do you choose the right
      rendering strategy?
    tags:
      - nextjs
      - ssr
      - ssg
      - performance
      - interview
    date: '2026-04-06'
  - slug: node-stream-large-file-processing
    category: Backend
    question: >-
      Your Node.js server processes a huge CSV file and runs out of memory. How
      do you handle large file processing with streams?
    tags:
      - nodejs
      - streams
      - performance
      - interview
    date: '2026-04-06'
  - slug: npm-supply-chain-attacks
    category: Security
    question: >-
      # NPM Supply Chain Attack Prevention


      Your Node.js app uses 200 npm packages. One of them gets compromised
      (event-stream incident). How do you protect against supply chain attacks?
    tags:
      - npm
      - dependencies
      - supply-chain
      - security
      - interview
    date: '2026-04-06'
  - slug: optimistic-updates-pattern
    category: Frontend
    question: >-
      # Optimistic Updates Pattern


      Clicking a button dispatches an action, but sometimes the UI updates
      before the API confirms, sometimes after. How do you implement optimistic
      updates correctly?
    tags:
      - react
      - ux
      - data-fetching
      - interview
    date: '2026-04-06'
  - slug: optimize-ci-cd-pipeline-speed
    category: DevOps
    question: >-
      # Optimizing CI/CD Pipeline Speed


      Your CI/CD pipeline takes 45 minutes. Developers push code and wait. How
      do you optimize build and test times in CI pipelines?
    tags:
      - ci-cd
      - performance
      - github-actions
      - experience
    date: '2026-04-06'
  - slug: optimize-docker-image-size
    category: DevOps
    question: >-
      # Optimizing Docker Image Size


      Your Docker image is 2.3GB. Pulling it takes 5 minutes on every deploy.
      How do you optimize Docker image size?
    tags:
      - docker
      - optimization
      - deployment
      - interview
    date: '2026-04-06'
  - slug: optimize-llm-api-costs
    category: AI
    question: >-
      # Optimizing LLM API Costs


      Your GPT-based feature costs $5,000/month in API calls for 10,000 users.
      How do you optimize LLM API costs with caching, prompt compression, and
      model routing?
    tags:
      - llm
      - cost-optimization
      - api
      - experience
    date: '2026-04-06'
  - slug: over-engineering-when-harmful
    category: Architecture
    question: >-
      # When Over-Engineering is Actually Harmful


      Your startup has 3 developers. The CTO wants to use Kubernetes,
      microservices, event sourcing, and CQRS from day one. When is
      over-engineering actually harmful?
    tags:
      - architecture
      - startup
      - pragmatism
      - career
    date: '2026-04-06'
  - slug: parallelize-sequential-api-calls
    category: Backend
    question: >-
      # Parallelize Sequential API Calls Without Breaking Dependencies


      Your REST API takes 3 seconds to respond because it calls 5 internal
      services sequentially. Each call waits for the previous one to finish. How
      do you parallelize these calls without breaking dependencies between
      services?
    tags:
      - performance
      - async
      - backend
      - interview
    date: '2026-04-06'
  - slug: password-hashing-algorithms
    category: Security
    question: >-
      # Password Hashing Algorithms


      You implement password hashing with MD5 because it's fast. A security
      audit flags this as critical. Why is MD5 dangerous and what hashing
      algorithm should you use?
    tags:
      - authentication
      - hashing
      - bcrypt
      - security
      - interview
    date: '2026-04-06'
  - slug: postgresql-vacuum-autovacuum
    category: Database
    question: >-
      Your PostgreSQL database runs out of disk space because dead tuples from
      UPDATE/DELETE aren't cleaned up. What is VACUUM and how do you configure
      autovacuum properly?
    tags:
      - postgresql
      - maintenance
      - performance
      - experience
    date: '2026-04-06'
  - slug: prevent-automated-account-abuse
    category: Security
    question: >-
      # Prevent Automated Account Abuse


      Your user registration endpoint allows unlimited attempts. An attacker
      creates 1 million fake accounts overnight. How do you prevent automated
      abuse without blocking real users?
    tags:
      - rate-limiting
      - bot-detection
      - security
      - experience
    date: '2026-04-06'
  - slug: prevent-cache-stampede
    category: Database
    question: >-
      Your Redis cache has a 95% hit rate but during peak traffic, a popular
      cache key expires and 10,000 requests simultaneously hit the database. How
      do you prevent cache stampede?
    tags:
      - redis
      - caching
      - performance
      - interview
    date: '2026-04-06'
  - slug: prevent-inventory-overselling-race-condition
    category: Database
    question: >-
      During a Flash Sale, User A and User B both view a product page. The
      database says `stock = 1`. They click "Buy" at the exact same millisecond.


      User A reads stock: 1.

      User B reads stock: 1.

      User A bypasses the `if (stock > 0)` check and updates stock to 0. 

      User B also bypasses the `if (stock > 0)` check (because they read 1
      earlier) and updates stock to 0.


      You just sold 2 iPhones when you only had 1. How do you prevent this
      classic Database Race Condition (Overselling)?
    tags:
      - ecommerce
      - concurrency
      - locking
      - interview
    date: '2026-04-06'
  - slug: prevent-overlapping-cron-jobs
    category: Backend
    question: >-
      # Prevent Overlapping Cron Jobs


      Your cron job runs at midnight but sometimes takes 2 hours to complete.
      The next execution starts before the first one finishes, causing duplicate
      work and data corruption. How do you prevent overlapping cron job
      executions?
    tags:
      - cron
      - scheduling
      - reliability
      - experience
    date: '2026-04-06'
  - slug: prevent-secret-leaks-in-code
    category: Security
    question: >-
      # Prevent Secret Leaks in Code


      A developer hardcodes an API key in the source code. It gets pushed to
      GitHub. The key is used to access production data. How do you prevent and
      detect secret leaks?
    tags:
      - secrets
      - git
      - ci-cd
      - security
      - experience
    date: '2026-04-06'
  - slug: prevent-zombie-queries-on-client-disconnect
    category: Architecture
    question: >-
      A user clicks to download a heavy report. Tired of waiting, they angrily
      hit "Cancel" or "F5" to refresh. The frontend instantly drops the
      connection. However, the backend is completely unaware! It blindly
      continues to burn CPU and hammer the database for 10 more seconds to run a
      giant "zombie" query. If 1,000 users mash F5, the server crashes. How do
      we make the backend "listen" and instantly slam the brakes on the DB
      query?
    tags:
      - backend
      - performance
      - database
      - interview
    date: '2026-04-06'
  - slug: rag-grounded-llm-responses
    category: AI
    question: >-
      # RAG: Grounding LLM Responses with Retrieved Context


      Your chatbot hallucinates facts about your product. Users get wrong
      information. How do you implement RAG (Retrieval-Augmented Generation) to
      ground LLM responses?
    tags:
      - llm
      - rag
      - ai
      - interview
    date: '2026-04-06'
  - slug: rate-limiting-strategies
    category: Backend
    question: >-
      Your public API suddenly gets 50,000 requests/second from a single IP.
      Your server is choking. The interviewer asks: "Explain the difference
      between Token Bucket, Sliding Window, and Fixed Window rate limiting.
      Which one would you pick for a payment API vs a search API, and why?"
    tags:
      - api
      - security
      - interview
      - scalability
    date: '2026-04-06'
  - slug: rbac-implementation-patterns
    category: Security
    question: >-
      # RBAC Implementation Patterns


      Your admin panel is accessible to anyone who knows the URL. There's no
      role-based access. An intern accidentally deletes production data. How do
      you implement RBAC properly?
    tags:
      - authorization
      - rbac
      - access-control
      - security
      - interview
    date: '2026-04-06'
  - slug: react-code-splitting-lazy-loading
    category: Frontend
    question: >-
      # React Code Splitting & Lazy Loading


      Your React app bundle is 2.5MB. First load takes 8 seconds on 3G. How do
      you implement code splitting and lazy loading effectively?
    tags:
      - react
      - performance
      - optimization
      - interview
    date: '2026-04-06'
  - slug: react-native-new-architecture
    category: Mobile
    question: >-
      # React Native New Architecture: Fabric & TurboModules


      Your React Native app is slow on Android but smooth on iOS. Profiling
      shows the JS bridge is the bottleneck. How does the New Architecture
      (Fabric + TurboModules) solve this?
    tags:
      - react-native
      - mobile
      - performance
      - interview
    date: '2026-04-06'
  - slug: react-state-management-choosing
    category: Frontend
    question: >-
      # Choosing the Right State Management


      Your team has 200 React components. Some use Redux, some useState, some
      Context. State management is chaos. How do you choose the right state
      management approach?
    tags:
      - react
      - state-management
      - architecture
      - interview
    date: '2026-04-06'
  - slug: react-useeffect-infinite-loop
    category: Frontend
    question: >-
      The Infamous React Infinite Loop: "Why did my component re-render 10,000
      times a second and crash the browser?"

      This is a rite of passage for every React developer. You add a `useEffect`
      to fetch some data, update the state, and suddenly your CPU fans sound
      like a jet engine taking off.
    tags:
      - react
      - hooks
      - interview
      - bugs
    date: '2026-04-06'
  - slug: redis-memory-optimization
    category: Database
    question: >-
      Your Redis instance uses 32GB of memory for 10M keys. Most keys are rarely
      accessed. How do you optimize Redis memory usage?
    tags:
      - redis
      - memory
      - optimization
      - experience
    date: '2026-04-06'
  - slug: reduce-logging-cost-at-scale
    category: Architecture
    question: >-
      Your centralized logging system ingests 1TB of logs per day. The
      Elasticsearch/Datadog/Splunk bill is skyrocketing. Management says "cut
      logging costs by 70%." But engineering says "we need every log line to
      trace business flows end-to-end." How do you drastically reduce logging
      volume and cost while still preserving full traceability of
      business-critical flows?
    tags:
      - observability
      - logging
      - cost-optimization
      - senior
    date: '2026-04-06'
  - slug: reliable-webhook-consumer
    category: Backend
    question: >-
      # Building a Reliable Webhook Consumer


      Your webhook endpoint receives duplicate events from Stripe or PayPal.
      Some events arrive out of order. How do you build a reliable webhook
      consumer that handles these real-world challenges?
    tags:
      - webhooks
      - stripe
      - reliability
      - interview
    date: '2026-04-06'
  - slug: restart-nginx-no-downtime
    category: DevOps
    question: >-
      There is heavy incoming traffic. If I modify the config, is there a way to
      restart Nginx without dropping connections?
    tags:
      - nginx
      - devops
      - interview
    date: '2026-04-06'
  - slug: safe-file-upload-handling
    category: Backend
    question: >-
      # Safely Handle File Uploads


      Your file upload endpoint accepts any file up to 10GB. Someone uploads a
      zip bomb that expands to 5TB on disk. How do you safely handle file
      uploads without becoming vulnerable to DoS attacks, malware, or path
      traversal?
    tags:
      - security
      - file-upload
      - backend
      - experience
    date: '2026-04-06'
  - slug: search-system-elasticsearch
    category: Architecture
    question: >-
      # Search System Architecture with Elasticsearch


      Your e-commerce search needs to be fast, support filters, facets, and
      fuzzy matching. The database can't handle it. How do you architect a
      search system with Elasticsearch?
    tags:
      - elasticsearch
      - search
      - scalability
      - interview
    date: '2026-04-06'
  - slug: secure-file-serving-signed-urls
    category: Security
    question: >-
      # Secure File Serving with Signed URLs


      Your S3 bucket is public because the frontend needs to read files. An
      attacker enumerates all files. How do you serve private files securely
      with signed URLs?
    tags:
      - s3
      - cloud
      - file-access
      - security
      - experience
    date: '2026-04-06'
  - slug: secure-secrets-management
    category: DevOps
    question: >-
      # Secure Secrets Management


      Your secrets are stored in environment variables that anyone with server
      access can read. How do you manage secrets securely with Vault or Sealed
      Secrets?
    tags:
      - security
      - secrets
      - vault
      - experience
    date: '2026-04-06'
  - slug: soft-delete-performance
    category: Database
    question: >-
      Your application uses soft deletes (deleted_at column). After 2 years, 80%
      of rows are "deleted" but still scanned by every query. How do you manage
      soft deletes efficiently?
    tags:
      - sql
      - patterns
      - performance
      - experience
    date: '2026-04-06'
  - slug: software-estimation-techniques
    category: Career
    question: >-
      # Software Estimation Techniques


      Your team lead asks you to estimate a feature. You say "2 weeks." It takes
      2 months. Why are software estimates always wrong and how do you estimate
      better?
    tags:
      - estimation
      - project-management
      - career
      - experience
    date: '2026-04-06'
  - slug: spa-memory-leak-debugging
    category: Frontend
    question: >-
      Your React SPA has been running fine for months. Then users start
      complaining: "The app gets slower the longer I use it." You open Chrome
      DevTools Memory tab and see heap usage climbing from 50MB to 400MB over 30
      minutes without ever dropping. Where do you start hunting for memory leaks
      in a Single Page Application?
    tags:
      - react
      - performance
      - debugging
      - experience
    date: '2026-04-06'
  - slug: sql-injection-beyond-basics
    category: Security
    question: >-
      # SQL Injection Beyond the Basics


      Your API accepts user input and passes it to a SQL query. Bobby Tables
      drops your entire database. How do you prevent SQL injection beyond just
      parameterized queries?
    tags:
      - sql-injection
      - database
      - security
      - interview
    date: '2026-04-06'
  - slug: sse-vs-websocket-realtime
    category: Backend
    question: >-
      # Server-Sent Events vs WebSockets for Real-Time Updates


      You need to notify 10,000 connected clients when a price changes. Polling
      wastes resources. How do Server-Sent Events (SSE) compare to WebSockets
      for real-time communication?
    tags:
      - realtime
      - websocket
      - sse
      - interview
    date: '2026-04-06'
  - slug: standardize-api-error-handling
    category: Backend
    question: >-
      Your API returns different error formats — sometimes {error}, sometimes
      {message}, sometimes plain text. How do you standardize API error
      handling?
    tags:
      - api-design
      - error-handling
      - backend
      - experience
    date: '2026-04-06'
  - slug: strategy-pattern-payment-providers
    category: Patterns
    question: >-
      # Strategy Pattern for Payment Providers


      Your payment module supports Stripe, PayPal, and MoMo. Adding a new
      provider requires changing 10 files. How does the Strategy pattern
      simplify payment provider integration?
    tags:
      - design-patterns
      - strategy
      - clean-code
      - interview
    date: '2026-04-06'
  - slug: terraform-state-management
    category: DevOps
    question: >-
      # Terraform State Management


      Your Terraform state file gets corrupted when two people run terraform
      apply at the same time. How do you manage Terraform state safely in a
      team?
    tags:
      - terraform
      - iac
      - state
      - experience
    date: '2026-04-06'
  - slug: transaction-isolation-levels
    category: Database
    question: >-
      Your transaction isolation level is READ COMMITTED but you're seeing
      phantom reads in a report query. What are the 4 isolation levels and when
      do you need SERIALIZABLE?
    tags:
      - sql
      - transactions
      - concurrency
      - interview
    date: '2026-04-06'
  - slug: transactional-outbox-pattern
    category: Backend
    question: >-
      Your service publishes events to Kafka but sometimes the database write
      succeeds and the event publish fails. How does the Transactional Outbox
      pattern solve this?
    tags:
      - microservices
      - messaging
      - consistency
      - senior
    date: '2026-04-06'
  - slug: trunk-based-development
    category: Git
    question: >-
      # Trunk-Based Development


      Your team uses long-lived feature branches that diverge for weeks. Merge
      conflicts are constant. How does trunk-based development reduce
      integration pain?
    tags:
      - git
      - branching
      - ci-cd
      - interview
    date: '2026-04-06'
  - slug: virtual-list-large-data
    category: Frontend
    question: >-
      # Virtual List for Large Data


      Your table component renders 10,000 rows and the browser freezes. How do
      you implement virtualization for large lists and tables?
    tags:
      - performance
      - react
      - virtualization
      - interview
    date: '2026-04-06'
  - slug: web-accessibility-react
    category: Frontend
    question: >-
      Accessibility audit flags 50 issues — missing alt text, no keyboard
      navigation, screen readers can't read your custom dropdown. How do you
      make React components accessible?
    tags:
      - accessibility
      - a11y
      - react
      - experience
    date: '2026-04-06'
  - slug: write-maintainable-e2e-tests
    category: Testing
    question: >-
      # Maintainable E2E Tests with Playwright


      Your E2E tests with Playwright break whenever the UI changes slightly.
      Tests are brittle and always red. How do you write maintainable E2E tests?
    tags:
      - testing
      - playwright
      - e2e
      - experience
    date: '2026-04-06'
  - slug: zero-downtime-schema-migration
    category: Database
    question: >-
      Your users table has 50M rows. Adding a NOT NULL column with a default
      value locks the table for 10 minutes. How do you run schema migrations on
      large tables without downtime?
    tags:
      - migration
      - postgresql
      - deployment
      - senior
    date: '2026-04-06'
---

