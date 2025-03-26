---
title: "Advanced Testing in Deno"
description: "Master advanced testing techniques in Deno including snapshot testing, parameterized tests, test fixtures, custom reporters, resource sanitization, and CI integration. Take your testing strategy beyond the basics with practical examples for experienced developers."
---

# Advanced Testing Techniques in Deno

This guide explores advanced testing techniques in Deno, building on the
concepts covered in our [basic testing tutorial](/examples/testing_tutorial/)
and [mocking guide](/examples/mocking_tutorial/). It's designed for experienced
developers who already understand testing fundamentals and want to leverage
Deno's more powerful testing capabilities.

## Table of Contents

- [Snapshot testing](#snapshot-testing)
- [Parameterized tests](#parameterized-tests)
- [Test fixtures](#test-fixtures)
- [Custom test reporters](#custom-test-reporters)
- [Resource sanitization](#resource-sanitization)
- [Testing in CI environments](#testing-in-ci-environments)
- [Optimizing test performance](#optimizing-test-performance)
- [Advanced testing patterns](#advanced-testing-patterns)
- [Test debugging techniques](#test-debugging-techniques)
- [Capturing console output](#capturing-console-output)
- [Testing web components and DOM manipulation](#testing-web-components-and-dom-manipulation)
- [Performance testing](#performance-testing)
- [Testing with external dependencies](#testing-with-external-dependencies)
- [AI-assisted testing](#ai-assisted-testing)
- [Conclusion and next steps](#conclusion-and-next-steps)

## Snapshot testing

Snapshot testing captures the output of a component or function and compares it
to a previously saved "snapshot" to detect unexpected changes. Instead of
manually writing assertions, you let the test runner record the expected output
automatically.

This approach is particularly useful for testing complex, structured output that
would be tedious to verify manually or catching unintended changes in function
output or UI components.

When a snapshot test runs for the first time, it creates a reference file with
the expected output. On subsequent runs, the test compares the current output
with this reference. If they match, the test passes; if they differ, the test
fails, alerting you to review the changes.

Here's how to use snapshot testing in Deno:

```ts
import { assertSnapshot } from "jsr:@std/testing/snapshot";

// Function that generates some complex output
function generateUserReport(userId) {
  return {
    id: userId,
    name: "User " + userId,
    permissions: ["read", "write"],
    metadata: {
      created: "2023-01-15T12:00:00Z",
      lastActive: "2023-04-22T15:30:45Z",
      preferences: {
        theme: "dark",
        notifications: true,
      },
    },
    statistics: {
      logins: 27,
      actions: 142,
    },
  };
}

Deno.test("user report snapshot", async (t) => {
  const report = generateUserReport("user123");

  // Compare with stored snapshot
  await assertSnapshot(t, report);

  // You can also provide a custom name for the snapshot
  // await assertSnapshot(t, report, "custom_name");
});
```

When this test runs:

1. If no snapshot exists, it creates one in a `__snapshots__` directory
2. On subsequent runs, it compares the output to the saved snapshot
3. If the output changes, the test fails with a diff showing what changed

To update snapshots when changes are intentional, run tests with the `--update`
flag:

```bash
deno test --allow-read --allow-write -- --update
```

When using snapshots, try keep them focused on a particular part, use
well-defined outputs rather than large objects with many properties and give
them meaningful names that describe what they test. And when updaint your
snapshots, be sure to verify that the changes you see are what is expected.

Shapshots are best for stable outputs that change infrequently, you can commit
them to your version control so that other developers can use them to test.

## Parameterized tests

Parameterized testing allows you to run the same test logic with different
inputs, increasing test coverage without duplicating code. This technique is
especially valuable for testing edge cases and boundary conditions.

In the below example we create an array of test cases as data and iterate
through each case, dynamically creating a test for each.

```ts
import { assertEquals } from "jsr:@std/assert";

function sum(a: number, b: number): number {
  return a + b;
}

// Define test cases as an array of input/expected pairs
const testCases = [
  { input: [1, 2], expected: 3, name: "positive numbers" },
  { input: [5, -3], expected: 2, name: "mixed signs" },
  { input: [0, 0], expected: 0, name: "zeros" },
  { input: [-5, -5], expected: -10, name: "negative numbers" },
  { input: [0.1, 0.2], expected: 0.3, name: "floating point", delta: 0.0001 },
];

// Run each test case with dynamic test names
for (const { input, expected, name, delta } of testCases) {
  Deno.test(`sum: ${name} - (${input[0]}, ${input[1]}) = ${expected}`, () => {
    const result = sum(...input);
    if (delta !== undefined) {
      // For floating point comparisons, check within delta
      assertEquals(Math.abs(result - expected) < delta, true);
    } else {
      assertEquals(result, expected);
    }
  });
}
```

The code loops through the test cases, creates descriptive test names and uses
assertions to verify that the function works correctly.

### Advanced Parameterization with Test Steps

For more complex scenarios, you can combine parameterized tests with test steps.

In the example code below we set up a `validateUsername` function to test. This
will valididate usernames against three rules, a minimum length of 3 characters,
a maximum length of 20 characters and only containing letters, numbers, and
underscores:

```ts
import { assertEquals } from "jsr:@std/assert";

// Function to validate a username
function validateUsername(
  username: string,
): { valid: boolean; errors: string[] } {
  const errors: string[] = [];

  if (username.length < 3) {
    errors.push("Username must be at least 3 characters");
  }

  if (username.length > 20) {
    errors.push("Username must be at most 20 characters");
  }

  if (!/^[a-zA-Z0-9_]+$/.test(username)) {
    errors.push("Username can only contain letters, numbers, and underscores");
  }

  return { valid: errors.length === 0, errors };
}

// Combined test cases for validation
Deno.test("username validation", async (t) => {
  const testCases = [
    {
      scenario: "valid usernames",
      usernames: ["user123", "john_doe", "ADMIN", "a_1_b_2"],
      expectValid: true,
    },
    {
      scenario: "too short",
      usernames: ["a", "xy"],
      expectValid: false,
      expectedError: "Username must be at least 3 characters",
    },
    {
      scenario: "too long",
      usernames: ["thisusernameiswaytoolongforoursystem"],
      expectValid: false,
      expectedError: "Username must be at most 20 characters",
    },
    {
      scenario: "invalid characters",
      usernames: ["user@name", "john.doe", "user-name", "user name"],
      expectValid: false,
      expectedError:
        "Username can only contain letters, numbers, and underscores",
    },
  ];

  // Iterate through each test scenario
  for (const { scenario, usernames, expectValid, expectedError } of testCases) {
    await t.step(scenario, async (t) => {
      // Test each username in the scenario
      for (const username of usernames) {
        await t.step(`username: "${username}"`, () => {
          const result = validateUsername(username);
          assertEquals(result.valid, expectValid);

          if (expectedError) {
            assertEquals(result.errors.includes(expectedError), true);
          }
        });
      }
    });
  }
});
```

The code sets up a structured array of test cases, organised into logical
groups. Each group has a descriptive scenario, and contains multiple usernames
to test against. They each define an expected outcome and an expected error
message.

The `for` loop iterates over each scenario and uses a nested structure with
three levels:

- Top level: The main test "username validation"
- Middle level: Scenario-based steps (e.g., "valid usernames", "too short")
- Bottom level: Individual username tests (e.g., "username: user123")

When executed it will produce a heirachical test report:

```sh
username validation
  ✓ valid usernames
    ✓ username: "user123"
    ✓ username: "john_doe"
    ✓ username: "ADMIN" 
    ✓ username: "a_1_b_2"
  ✓ too short
    ✓ username: "a"
    ✓ username: "xy"
  ✓ too long
    ✓ username: "thisusernameiswaytoolongforoursystem"
  ✓ invalid characters
    ✓ username: "user@name"
    ✓ username: "john.doe"
    ✓ username: "user-name"
    ✓ username: "user name"
```

This pattern not only helps organise your testing to make for easier
readability, but also outputs nice clear reporting. The tests are self
documenting and comprehensive.

## Test fixtures

Test fixtures provide reusable setup and teardown logic for tests, ensuring a
consistent environment for each test case. Deno simplifies this with a
step-based approach.

In the below example we set up an interface to definte what a test database
should be able to do, in this case:

- `connect()`: Establish a database connection
- `disconnect()`: Close the connection
- `clear()`: Remove all test data
- `addUser()`: Add a user record
- `getUser()`: Retrieve a user by ID

Then we create an in-memory implementation of the database which uses objects to
store user records and implements the methods defined in the interface.

We set up a test database instance, connect to it and return it for testing,
remembering to disconnect after the tests complete.

```ts
import { assertEquals } from "jsr:@std/assert";

interface TestDatabase {
  connect(): Promise<void>;
  disconnect(): Promise<void>;
  clear(): Promise<void>;
  addUser(
    user: { name: string; email: string },
  ): Promise<{ id: string; name: string; email: string }>;
  getUser(
    id: string,
  ): Promise<{ id: string; name: string; email: string } | null>;
}

// A test implementation of the database
const createTestDatabase = (): TestDatabase => {
  const users: Record<string, { id: string; name: string; email: string }> = {};
  let nextId = 1;

  return {
    connect: async () => {
      console.log("Database connected");
    },
    disconnect: async () => {
      console.log("Database disconnected");
    },
    clear: async () => {
      Object.keys(users).forEach((key) => delete users[key]);
    },
    addUser: async (user) => {
      const id = String(nextId++);
      const newUser = { id, ...user };
      users[id] = newUser;
      return newUser;
    },
    getUser: async (id) => {
      return users[id] || null;
    },
  };
};

// Setup function that creates and initializes the test database
async function setupTestDatabase() {
  const db = createTestDatabase();
  await db.connect();
  return db;
}

// Main test that uses the fixture
Deno.test("user database operations", async (t) => {
  // Setup fixture
  const db = await setupTestDatabase();

  // Register teardown to run after all tests
  t.teardown(async () => {
    await db.disconnect();
  });

  await t.step("adding a user", async () => {
    await db.clear(); // Ensure clean state

    const user = await db.addUser({
      name: "Alice",
      email: "alice@example.com",
    });

    assertEquals(user.id, "1");
    assertEquals(user.name, "Alice");
    assertEquals(user.email, "alice@example.com");
  });

  await t.step("retrieving a user", async () => {
    await db.clear();

    const added = await db.addUser({
      name: "Bob",
      email: "bob@example.com",
    });

    const retrieved = await db.getUser(added.id);
    assertEquals(retrieved, added);
  });

  await t.step("user not found", async () => {
    await db.clear();

    const user = await db.getUser("999");
    assertEquals(user, null);
  });
});
```

We use `t.step` to set up test steps. Each step clears the database to start
with a known state, performs an operation and then asserts the expected
outcome(s).

This pattern ensures that:

- Each test starts with a fresh database
- The database is properly connected before tests run
- The database is disconnected after all tests complete

Test fixtures can be adapted to many types of external dependencies (databases,
APIs, file systems) to create reliable, reproducible tests.

## Test reporters

Deno allows you to configure how test results are reported, which is
particularly useful for CI environments. Deno supports four built-in test
reporters:

- **pretty** - The default reporter that displays a detailed, human-readable
  output with test names, status, and duration
- **dot** - A minimal reporter that shows a dot for each passing test and
  letters for other statuses (F for failures, etc.)
- **junit** - Generates XML output in [JUnit format](https://junit.org/junit5/),
  commonly used in CI systems for integrated reporting
- **tap** - Produces output in the
  [Test Anything Protocol format](https://testanything.org/), compatible with
  TAP consumers

To specify a reporter when running tests:

```bash
deno test --reporter=dot
```

The JUnit reporter is especially useful for CI integration, as many CI platforms
can parse JUnit XML to display test results:

```bash
deno test --reporter=junit --junit-path=test-results.xml
```

For TAP output that can be piped to other tools:

```bash
deno test --reporter=tap
```

## Resource sanitization

Deno provides powerful sanitization mechanisms that automatically verify your
tests clean up after themselves. These built-in sanitizers help prevent resource
leaks and ensure tests don't interfere with each other:

- **Resource sanitizer** - Checks that all resources (files, network
  connections, etc.) opened during a test are properly closed before the test
  completes
- **Op sanitizer** - Ensures that all asynchronous operations started during a
  test are completed by the end of the test
- **Exit sanitizer** - Prevents tests from calling `Deno.exit()`, which would
  prematurely terminate the test process and potentially mask failures

These sanitizers are enabled by default for all tests. While this helps maintain
clean test environments, there are legitimate cases where you might need to
disable them:

```ts
Deno.test({
  name: "test without sanitization",
  fn: async () => {
    // Test code that intentionally leaves resources open
    // or has background operations that continue after the test
  },
  sanitizeResources: false, // Don't check for unclosed resources
  sanitizeOps: false, // Don't check for incomplete async operations
  sanitizeExit: false, // Allow Deno.exit() calls (use with caution)
});
```

Consider disabling sanitizers when:

- Testing error handling for unclosed resources
- Working with third-party libraries that manage their own resource lifecycle
- Testing long-running background processes that intentionally outlive the test
- Debugging specific test issues where sanitizers might obscure the root problem
- Running performance benchmarks where sanitization overhead could affect
  results

In most cases, however, keeping sanitizers enabled helps maintain a clean
testing environment and catch potential issues early.

## Testing in CI environments

Deno tests work seamlessly in CI environments. Here's an example GitHub Actions
configuration with advanced features:

```yaml
name: Advanced Deno Testing

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3

      - name: Set up Deno
        uses: denoland/setup-deno@v1
        with:
          deno-version: v1.x

      - name: Cache dependencies
        uses: actions/cache@v3
        with:
          path: ~/.cache/deno
          key: ${{ runner.os }}-deno-${{ hashFiles('**/deps.ts') }}
          restore-keys: ${{ runner.os }}-deno-

      - name: Run unit tests
        run: deno test --allow-read --allow-env src/

      - name: Run integration tests
        run: deno test --allow-all tests/integration/

      - name: Generate coverage report
        run: |
          deno test --coverage=coverage --allow-all
          deno coverage --lcov coverage > coverage.lcov

      - name: Upload coverage to Codecov
        uses: codecov/codecov-action@v3
        with:
          file: ./coverage.lcov
          fail_ci_if_error: true

      - name: Check lint
        run: deno lint

      - name: Check formatting
        run: deno fmt --check
```

### Matrix Testing Across Deno Versions

For libraries that need to support multiple Deno versions:

```yaml
jobs:
  test:
    runs-on: ${{ matrix.os }}
    strategy:
      matrix:
        os: [ubuntu-latest, windows-latest, macos-latest]
        deno-version: [1.32.x, 1.33.x, 1.34.x, 1.35.x]

    steps:
      - uses: actions/checkout@v3

      - name: Set up Deno ${{ matrix.deno-version }}
        uses: denoland/setup-deno@v1
        with:
          deno-version: ${{ matrix.deno-version }}

      - name: Run tests
        run: deno test --allow-all
```

## Optimizing test performance

As test suites grow, performance becomes increasingly important. Here are some
advanced techniques to optimize test performance:

### Test Filtering and Tagging

Use test filtering to run only relevant tests during development:

```ts
// Tag tests by category
Deno.test({
  name: "fast database query test",
  fn: () => {/* test logic */},
  tags: ["database", "fast"],
});

Deno.test({
  name: "slow integration test",
  fn: async () => {/* test logic */},
  tags: ["integration", "slow"],
});

// Run with:
// deno test --filter "database" // Run only database tests
// deno test --filter "fast" // Run only fast tests
```

### Test Sharding

For very large test suites, sharding allows distributing tests across multiple
processes:

```bash
# Run tests in 4 shards
deno test --jobs=4
```

### In-Memory Databases for Testing

Replace real databases with in-memory alternatives for testing:

```ts
import { DB } from "https://deno.land/x/sqlite@v3.4.0/mod.ts";

// Create an in-memory database for testing
const db = new DB(":memory:");

// Set up test schema
db.execute(`
  CREATE TABLE users (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    username TEXT UNIQUE,
    email TEXT UNIQUE
  )
`);

// Test with the in-memory database
Deno.test("database operations", async () => {
  db.query("INSERT INTO users (username, email) VALUES (?, ?)", [
    "testuser",
    "test@example.com",
  ]);

  const [user] = db.query("SELECT * FROM users WHERE username = ?", [
    "testuser",
  ]);
  assertEquals(user[1], "testuser");
  assertEquals(user[2], "test@example.com");
});
```

## Advanced testing patterns

### Contract Testing

Contract testing ensures that different parts of your system interact correctly
through defined interfaces:

```ts
import { assertEquals } from "jsr:@std/assert";

// Define the contract/interface
interface UserRepository {
  findById(id: string): Promise<User | null>;
  save(user: User): Promise<User>;
  delete(id: string): Promise<boolean>;
}

interface User {
  id: string;
  name: string;
  email: string;
}

// Contract test suite that can be reused for any implementation
function testUserRepositoryContract(createRepo: () => UserRepository) {
  return async (t: Deno.TestContext) => {
    await t.step("findById returns null for non-existent user", async () => {
      const repo = createRepo();
      const user = await repo.findById("nonexistent");
      assertEquals(user, null);
    });

    await t.step("can save and retrieve a user", async () => {
      const repo = createRepo();
      const user: User = {
        id: "1",
        name: "Test User",
        email: "test@example.com",
      };

      await repo.save(user);
      const retrieved = await repo.findById("1");

      assertEquals(retrieved, user);
    });

    await t.step("can delete a user", async () => {
      const repo = createRepo();
      const user: User = {
        id: "2",
        name: "Delete Me",
        email: "delete@example.com",
      };

      await repo.save(user);
      const deleteResult = await repo.delete("2");

      assertEquals(deleteResult, true);
      assertEquals(await repo.findById("2"), null);
    });
  };
}

// Memory implementation of the repository
class InMemoryUserRepository implements UserRepository {
  private users: Map<string, User> = new Map();

  async findById(id: string): Promise<User | null> {
    return this.users.get(id) || null;
  }

  async save(user: User): Promise<User> {
    this.users.set(user.id, { ...user });
    return user;
  }

  async delete(id: string): Promise<boolean> {
    return this.users.delete(id);
  }
}

// Run the contract tests against the in-memory implementation
Deno.test(
  "InMemoryUserRepository implementation",
  testUserRepositoryContract(() => new InMemoryUserRepository()),
);

// Later, you can test other implementations with the same contract
// Deno.test("SQLiteUserRepository implementation", testUserRepositoryContract(() => new SQLiteUserRepository()));
```

### Property-Based Testing

Property-based testing generates random inputs to validate that general
properties of your functions hold true:

```ts
import { assertEquals } from "jsr:@std/assert";

// Function to test
function reverse<T>(arr: T[]): T[] {
  return [...arr].reverse();
}

Deno.test("reverse - property testing", () => {
  // Generate random array lengths
  for (let len = 0; len < 100; len++) {
    // Create an array of that length with unique values
    const arr = Array.from({ length: len }, (_, i) => `item${i}`);

    // Test properties that should always be true for any input
    const reversed = reverse(arr);

    // Property 1: Length is preserved
    assertEquals(reversed.length, arr.length);

    // Property 2: Reversing twice gives the original array
    assertEquals(reverse(reversed), arr);

    // Property 3: First element becomes last and last becomes first
    if (len > 0) {
      assertEquals(reversed[0], arr[len - 1]);
      assertEquals(reversed[len - 1], arr[0]);
    }

    // Property 4: Original array is not modified
    const arrCopy = [...arr];
    reverse(arr);
    assertEquals(arr, arrCopy);
  }
});
```

## Test debugging techniques

Debugging tests can be challenging, especially for complex scenarios. Here are
advanced techniques to help identify and fix test issues:

```ts
import { assertEquals } from "jsr:@std/assert";

// Function with a subtle bug
function complexCalculation(a: number, b: number, c: number): number {
  // Bug: should be (a * b) + c
  return a * (b + c);
}

Deno.test("debugging complex tests", async () => {
  // Using console.log for simple visibility
  const result = complexCalculation(2, 3, 4);
  console.log("Result:", result); // Output visible in test logs

  // Advanced: use Deno.inspect for better object visualization
  const complexObject = {
    nested: {
      data: [1, 2, { key: "value" }],
    },
    circular: {},
  };
  (complexObject.circular as any).self = complexObject; // Circular reference

  console.log(Deno.inspect(
    complexObject,
    { depth: 4, colors: true, circular: true },
  ));

  // Conditional breakpoints with --inspect-brk
  // When running with: deno test --inspect-brk
  if (result !== 10) {
    // You can set a breakpoint here or use debugger statement
    debugger; // This will pause execution when running with --inspect-brk
  }

  assertEquals(result, 10); // Will fail, helping identify the bug
});
```

To debug tests with Chrome DevTools:

1. Run your test with `deno test --inspect-brk your_test.ts`
2. Open Chrome and navigate to `chrome://inspect`
3. Click "Inspect" on the Deno process
4. Use the debugger to step through your test

## Capturing console output

Sometimes you need to test functions that write to the console:

```ts
import { assertEquals } from "jsr:@std/assert";
import { restore, spy } from "jsr:@std/testing/mock";

function logWarning(message: string): void {
  console.warn(`WARNING: ${message}`);
}

Deno.test("capturing console output", () => {
  // Create a spy on console.warn
  const warnSpy = spy(console, "warn");

  try {
    // Execute the function
    logWarning("Disk space low");

    // Assert console.warn was called with the right message
    assertEquals(warnSpy.calls.length, 1);
    assertEquals(warnSpy.calls[0].args[0], "WARNING: Disk space low");
  } finally {
    // Always restore original functionality after test
    warnSpy.restore();
    // Alternative: restore all spies at once
    // restore();
  }
});
```

## Testing web components and DOM manipulation

For applications that interact with the DOM, you can use Deno's built-in DOM API
or third-party libraries:

```ts
import { assertEquals } from "jsr:@std/assert";

// Component or function that manipulates the DOM
function createUserCard(user: { name: string; email: string }): HTMLElement {
  const card = document.createElement("div");
  card.className = "user-card";

  const name = document.createElement("h2");
  name.textContent = user.name;
  card.appendChild(name);

  const email = document.createElement("p");
  email.textContent = user.email;
  email.className = "email";
  card.appendChild(email);

  return card;
}

Deno.test("DOM manipulation test", () => {
  // Create a test user
  const testUser = { name: "Test User", email: "test@example.com" };

  // Call the function
  const card = createUserCard(testUser);

  // Assert the DOM structure is correct
  assertEquals(card.className, "user-card");
  assertEquals(card.children.length, 2);
  assertEquals(card.querySelector("h2")?.textContent, "Test User");
  assertEquals(card.querySelector(".email")?.textContent, "test@example.com");
});
```

## Performance testing

In addition to correctness, you might want to measure performance:

```ts
import { assertEquals } from "jsr:@std/assert";

// Function to test for performance
function findPrimes(max: number): number[] {
  const primes: number[] = [];
  outer: for (let i = 2; i <= max; i++) {
    for (let j = 2; j < i; j++) {
      if (i % j === 0) continue outer;
    }
    primes.push(i);
  }
  return primes;
}

// Basic correctness test
Deno.test("findPrimes correctness", () => {
  assertEquals(findPrimes(10), [2, 3, 5, 7]);
});

// Performance test
Deno.test("findPrimes performance", () => {
  const start = performance.now();
  findPrimes(1000); // Test with a larger input
  const end = performance.now();
  const duration = end - start;

  console.log(`findPrimes(1000) took ${duration.toFixed(2)}ms`);

  // Optional: assert performance is within acceptable bounds
  // This helps catch performance regressions
  assertTrue(
    duration < 50,
    `Expected execution to take less than 50ms but took ${duration}ms`,
  );
});
```

For more formal performance testing, use Deno's built-in benchmarking
capabilities:

```ts
// Save this in bench.ts
import { findPrimes } from "./your_module.ts";

Deno.bench("findPrimes(100)", () => {
  findPrimes(100);
});

Deno.bench("findPrimes(1000)", () => {
  findPrimes(1000);
});

// Run with: deno bench
```

## Testing with external dependencies

Complex applications often depend on external services. Here's how to test them
effectively:

```ts
import { assertEquals } from "jsr:@std/assert";
import { delay } from "jsr:@std/async";

// Simplified database connection function
async function connectToDatabase(
  url: string,
): Promise<{ query: (sql: string) => Promise<unknown[]> }> {
  // In a real application, this would connect to a real database
  console.log(`Connecting to database at ${url}...`);
  await delay(100); // Simulate connection time

  return {
    query: async (sql: string) => {
      console.log(`Executing query: ${sql}`);
      if (sql.includes("SELECT")) {
        return [{ id: 1, name: "Test" }];
      }
      return [];
    },
  };
}

// Function that uses the database
async function getUserById(
  id: number,
  dbUrl: string,
): Promise<{ id: number; name: string } | null> {
  const db = await connectToDatabase(dbUrl);
  const results = await db.query(`SELECT * FROM users WHERE id = ${id}`);
  return results[0] as { id: number; name: string } || null;
}

// Test approaches:

// 1. Using test doubles (recommended)
Deno.test("getUserById with fake database", async () => {
  // Replace real database with test double
  const originalConnectToDatabase = globalThis.connectToDatabase;
  globalThis.connectToDatabase = async () => ({
    query: async (sql: string) => {
      if (sql.includes("WHERE id = 1")) {
        return [{ id: 1, name: "Test User" }];
      }
      return [];
    },
  });

  try {
    const user = await getUserById(1, "fake://db");
    assertEquals(user, { id: 1, name: "Test User" });

    const nonExistentUser = await getUserById(999, "fake://db");
    assertEquals(nonExistentUser, null);
  } finally {
    globalThis.connectToDatabase = originalConnectToDatabase;
  }
});

// 2. Using actual test database (integration test)
Deno.test({
  name: "getUserById with test database",
  ignore: !Deno.env.get("RUN_INTEGRATION_TESTS"), // Only run when explicitly enabled
  async fn() {
    const testDbUrl = Deno.env.get("TEST_DB_URL") ||
      "postgres://localhost:5432/test_db";

    // This test will only pass if the test database is properly set up
    const user = await getUserById(1, testDbUrl);
    assertEquals(user?.id, 1);
  },
});
```

## AI-assisted testing

Leverage AI tools to help generate test cases and identify potential edge cases:

```ts
// Using property-based testing patterns to test a wider range of inputs
import { assertEquals } from "jsr:@std/assert";

// Function to test
function formatPhoneNumber(input: string): string {
  // Remove non-numeric characters
  const cleaned = input.replace(/\D/g, "");

  // Check if it's a valid US phone number
  if (cleaned.length !== 10) {
    throw new Error("Phone number must be exactly 10 digits");
  }

  // Format as (XXX) XXX-XXXX
  return `(${cleaned.substring(0, 3)}) ${cleaned.substring(3, 6)}-${
    cleaned.substring(6, 10)
  }`;
}

Deno.test("phone number formatter", async (t) => {
  // Standard case
  await t.step("formats properly with numeric input", () => {
    assertEquals(formatPhoneNumber("1234567890"), "(123) 456-7890");
  });

  // AI-suggested edge cases:
  await t.step("handles input with existing formatting", () => {
    assertEquals(formatPhoneNumber("(123) 456-7890"), "(123) 456-7890");
  });

  await t.step("handles input with spaces", () => {
    assertEquals(formatPhoneNumber("123 456 7890"), "(123) 456-7890");
  });

  await t.step("handles input with hyphens", () => {
    assertEquals(formatPhoneNumber("123-456-7890"), "(123) 456-7890");
  });

  await t.step("handles input with mixed characters", () => {
    assertEquals(formatPhoneNumber("(123) abc-7890"), "(123) ABC-7890");
  });

  await t.step("rejects too short input", () => {
    try {
      formatPhoneNumber("12345");
      throw new Error("Should have thrown an error for short input");
    } catch (error) {
      assertEquals(error.message, "Phone number must be exactly 10 digits");
    }
  });

  await t.step("rejects too long input", () => {
    try {
      formatPhoneNumber("12345678901234");
      throw new Error("Should have thrown an error for long input");
    } catch (error) {
      assertEquals(error.message, "Phone number must be exactly 10 digits");
    }
  });
});
```

🦕 Mastering these advanced testing techniques will help you build more
reliable, maintainable Deno applications. By combining snapshot testing,
parameterized tests, test fixtures, and custom reporting, you can create a test
suite that not only catches bugs but also serves as living documentation for
your codebase.

For more detailed information about testing in Deno, check out:

- [Deno's Testing Documentation](/runtime/fundamentals/testing)
- [Basic Testing Tutorial](/examples/testing_tutorial/)
- [Mocking in Deno](/examples/mocking_tutorial/)
- [Standard Library Testing Tools](https://jsr.io/@std/testing)
