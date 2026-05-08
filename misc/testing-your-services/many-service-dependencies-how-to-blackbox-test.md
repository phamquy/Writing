---
icon: vials
---

# Many service dependencies, how to blackbox test?

Testing your service from end to end is great, it makes you more confident in how your code behaves with other external dependencies. However, we often have too many external dependencies.

## Strategy for Black-Box Integration Testing with Multiple Service Dependencies

When your microservice depends on multiple external systems (Bigtable, Postgres, Pub/Sub, LaunchDarkly, other APIs), you want integration tests that:

1. **Run locally** without requiring real cloud dependencies.
2. **Treat your service as a black box**, verifying that it interacts with dependencies correctly.

***

### 1. Decide What You’re Actually Testing

You don’t need to validate that Bigtable, Pub/Sub, or LaunchDarkly work — those are already tested by their providers.\
✅ Your goal is to test that **your service integrates with them correctly** (calls APIs, writes messages, stores data, etc).

***

### 2. Classify Dependencies

For each dependency, ask: _Can I run a local emulator, or do I need a mock?_

* **Databases**
  * Postgres → Run via Docker/Testcontainers.
  * Bigtable → Use Google’s Bigtable emulator.
* **Messaging**
  * Pub/Sub → Use Google’s Pub/Sub emulator.
* **Feature Flags**
  * e.g LaunchDarkly → Use SDK test harness or run a lightweight mock/stub.
* **Other APIs**
  * Internal services → Mock with WireMock/Mountebank/Prism.
  * 3rd-party APIs → Stub them.

***

### 3. Local Integration Test Environment

Build a reproducible stack using Docker Compose or Testcontainers:

* Your service container.
* Emulators for Postgres, Pub/Sub, Bigtable.
* Mock servers for APIs that lack emulators.
* Optional: Local fake for LaunchDarkly.

Running `make integration-test` (or `docker compose up`) should spin up the whole environment.

***

### 4. Test Strategy

* **Unit tests** → No external deps.
* **Local integration tests** → Run against emulators/mocks.
* **Staging tests** → Run the same black-box suite against _real_ cloud services.

This gives you fast, isolated local feedback, plus confidence that staging matches production.

***

### 5. Verifying Results in Mocks/Emulators

How do you know your service actually wrote/produced something correctly?\
Different dependencies require different verification strategies:

#### Databases

* **Postgres, Bigtable emulator** → Query the DB/emulator after running your test action to confirm expected state.

#### Messaging

* **Pub/Sub emulator** → Attach a _test subscriber_ that consumes from the topic and assert the received message.

#### Feature Flags

* **LaunchDarkly mock** → Provide fake flag values (via SDK stub) and check that your service behaves correctly.

#### External APIs (Mock Containers)

* Use a tool like **WireMock**:
  * WireMock exposes an admin API (`/__requests`) that lists received requests.
  * Your test queries this endpoint to assert the correct calls were made.

#### General Options

1. **Expose Test API from Mock** → Query it for recorded requests.
2. **Observer Pattern** → Have mock log to file/DB, read back in test.
3. **Admin APIs in Emulators** → Many emulators support admin queries.
4. **Consumer-driven Contracts (PACT)** → Assert request formats rather than internal state.
5. **Event Capture** → Subscribe to async systems like Pub/Sub to verify side effects.

***

### 6. Example Flow

1. Start local stack: Postgres, Bigtable emulator, Pub/Sub emulator, LaunchDarkly mock.
2. Run your service inside the stack.
3. Test sends an API request to your service.
4. Service writes to Postgres → test queries DB.
5. Service publishes to Pub/Sub → test subscriber receives message.
6. Service calls mock API → test queries mock’s admin endpoint.
7. Assert results match expectations.

***

### ✅ End Result

* Local developers can run integration tests without cloud access.
* CI/CD can run the same suite against staging with real services.
* You verify **correctness of integration** while still treating your service as a black box.
