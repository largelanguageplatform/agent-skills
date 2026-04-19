---
name: lean-developer
description: Guiding principles and patterns for writing lean software. Covers architecture decisions, dependency management, abstraction timing, code clarity, and product scope - focused on building the smallest thing that solves the real problem well.
license: MIT
metadata:
    author: Large Language Platform, Inc.
    version: 0.1.0
---

# Lean Developer

A collection of guiding principles and patterns for writing software that is simple, clear, and focused on real problems - not imagined future ones.

These guidelines aim to help developers:

- Build the smallest architecture that solves the real problem well
- Resist adding complexity before pain is demonstrated
- Write code that is easy to understand, change, and delete
- Make deliberate decisions about dependencies, abstractions, and scope
- Distinguish genuine simplicity from underbuilding

## How To Use This Skill

When this skill is active, apply the following process before suggesting any solution:

1. **Identify the core task** - understand what the user is actually trying to achieve, not just what they asked for literally.
2. **Map the current solution end to end** - trace the full path: data in, logic applied, data out. Understand what already exists before proposing anything new.
3. **Look for unnecessary weight** - scan for redundant dependencies, speculative abstractions, unused configuration, duplicated layers, and moving parts that do not justify their presence.
4. **Prefer subtraction over addition** - present recommendations in this order: remove first, then simplify, then defer, then add. Addition is always the last resort.
5. **Protect the non-negotiables** - flag where simplicity would become underbuilding. The following must never be sacrificed for brevity: authentication and authorisation, data integrity and consistency, failure handling and recovery, security boundaries, observability, and backup/restore paths.
6. **Be honest about trade-offs** - when a simpler approach has a real cost, name it. Do not oversell minimalism where robustness genuinely requires more.

## The Three-Stage Mandate

Every piece of software should pass through three stages, in order:

1. **Make it work** - correctness first. Ship something that solves the real problem reliably. An elegant solution to the wrong problem, or one that crashes under real conditions, has no value.
2. **Make it clear** - once it works, make it obvious. Clean up the code so the next engineer (or future you) can read, understand, and safely change it. Clarity is not cosmetic; unclear code becomes unmaintainable code.
3. **Make it fast** - only after it is correct and clear, optimize for performance. And only where measurement shows it is actually needed. Premature optimization is complexity purchased before the invoice arrives.

The most common mistake is skipping to step 3 before finishing step 1, or spending all time on step 2 before step 1 is real.

## Code Is a Liability

Every line of code is a line that can contain a bug, must be read, must be tested, and must be maintained. The best code is the code you did not have to write.

> Write the least amount of code that correctly delivers the required features at the required quality and safety. No less - but never more.

This means:

- Delete code that no longer earns its place
- Reach for a built-in before writing a utility
- Solve the specific problem, not the general one
- A 10-line solution that handles the real cases beats a 100-line solution that handles imaginary ones

More code does not mean more value. It means more surface area for failure.

## Fewer Moving Parts

Every additional component - service, dependency, queue, database, abstraction layer - multiplies the surface area for bugs, coupling, and maintenance.

**Over-engineered:**

```
User Request
  → API Gateway
    → Auth Service
      → User Service
        → Notification Service
          → Kafka
            → Email Worker
```

**Lean:**

```
User Request
  → App Server (handles auth, user logic, sends email inline)
```

Start with the monolith. Extract only when a specific bottleneck is demonstrated.

**Dependency checklist before adding any package:**

- Does this solve a problem we actually have today?
- Could we write the 20 lines ourselves without meaningful risk?
- What happens when this package is abandoned or breaks?

## Delay Abstraction

Premature abstraction is as harmful as premature optimization. Duplicate code is cheaper than the wrong abstraction.

Build the concrete thing once. If a second real use case arrives and the shared shape is obvious, extract the abstraction then - not before. See the Go and TypeScript examples in the Language Examples section for concrete before/after illustrations.

## Optimize for Clarity

Lean code answers these questions immediately:

- What does this do?
- Where does this logic live?
- What happens on failure?
- Where does this data come from?
- How do I change this safely?

**Clever (hard to reason about):**

```typescript
const result = data
    .filter(Boolean)
    .reduce((acc, x) => ({ ...acc, [x.id]: (acc[x.id] ?? 0) + x.value }), {} as Record<string, number>)
```

**Clear:**

```typescript
const totals: Record<string, number> = {}
for (const item of data) {
    if (!item) continue
    totals[item.id] = (totals[item.id] ?? 0) + item.value
}
```

Fewer lines do not equal better code. A readable 10-line function beats an opaque 2-liner.

## Add Things Only When Pain Is Real

Do not solve scaling, extensibility, or edge cases that have not occurred.

| Premature Addition | Wait Until |
| --- | --- |
| Microservices | A single service cannot be deployed or scaled independently due to real traffic |
| Message queues | Synchronous calls measurably block user-facing requests |
| Caching layer | Database queries are a demonstrated bottleneck |
| Plugin architecture | A second real plugin use case exists |
| CQRS / event sourcing | Read/write contention is a real, measured problem |
| Agent orchestration framework | Simple scripted steps actually fail in production |

Ship. Observe real pain. Add the minimum structure needed to address it.

## Strong Opinions, Small Surface Area

Every configuration option, mode, and knob multiplies complexity for users and maintainers. Good systems have one obvious way to do the common thing.

**Too many options:**

```yaml
notifier:
  mode: async | sync | batch | hybrid
  retry_strategy: exponential | linear | none
  fallback_channel: email | sms | webhook | none
  timeout_ms: 1000
  max_retries: 3
```

**Lean:**

```yaml
notifier:
  channel: email   # the only supported channel
```

Expose configuration only when users demonstrably need to vary it.

## Prefer Convention Over Ceremony

Standard layouts, standard patterns, and clear defaults eliminate decision fatigue and reduce onboarding cost.

**Ceremony (custom framework internals a new engineer must learn):**

```
src/
  core/
    bus/
      EventBusFactory.ts
      AbstractEventHandler.ts
    registry/
      HandlerRegistry.ts
```

**Convention (familiar to any engineer in the ecosystem):**

```
src/
  handlers/
    user_created.ts
    order_placed.ts
```

Reach for the boring, well-understood structure before inventing a new one.

## Focus on the Core Loop

Identify the one action that defines the product's value and make it excellent before building anything else.

If the core loop is **user submits query → gets accurate answer**, then this comes before:

- admin dashboards
- export features
- collaboration tools
- advanced permissions
- usage analytics

Build the core loop end-to-end first. Everything else is a distraction until the core works.

## Language Examples

### Go

**Delay abstraction - one real use case does not need an interface:**

```go
// Too early: interface with a single implementation
type Notifier interface {
    Send(to, subject, body string) error
}

type EmailNotifier struct{ client *smtp.Client }

func (n *EmailNotifier) Send(to, subject, body string) error { ... }

func NotifyUser(n Notifier, user User) error {
    return n.Send(user.Email, "Welcome", "Hello "+user.Name)
}
```

```go
// Lean: just the function you need today
func sendWelcomeEmail(to, name string) error {
    return smtp.SendMail(addr, auth, from, []string{to}, buildMessage(name))
}
```

Introduce the interface when a second real implementation (e.g. SMS, push) actually exists.

---

**Reach for the stdlib before adding a dependency:**

```go
// Unnecessary dependency for a simple HTTP call
resp, err := resty.New().R().
    SetHeader("Authorization", "Bearer "+token).
    Get(url)
```

```go
// stdlib is sufficient
req, err := http.NewRequestWithContext(ctx, http.MethodGet, url, nil)
if err != nil {
    return err
}
req.Header.Set("Authorization", "Bearer "+token)
resp, err := http.DefaultClient.Do(req)
```

---

**Error handling: explicit and local, not wrapped in abstractions:**

```go
// Over-engineered: custom error hierarchy for a simple lookup
type NotFoundError struct{ ID string }
type ValidationError struct{ Field, Reason string }
type WrappedError struct{ Op string; Err error }

func GetUser(id string) (*User, error) {
    return nil, &WrappedError{Op: "GetUser", Err: &NotFoundError{ID: id}}
}
```

```go
// Lean: sentinel errors or fmt.Errorf are almost always enough
var ErrNotFound = errors.New("not found")

func GetUser(id string) (*User, error) {
    user, ok := db[id]
    if !ok {
        return nil, fmt.Errorf("user %s: %w", id, ErrNotFound)
    }
    return user, nil
}
```

---

**Write less code - use what the language gives you:**

```go
// Hand-rolled sort: more code, more bugs
func sortUsersByAge(users []User) {
    for i := 1; i < len(users); i++ {
        for j := i; j > 0 && users[j].Age < users[j-1].Age; j-- {
            users[j], users[j-1] = users[j-1], users[j]
        }
    }
}
```

```go
// One line with the stdlib
sort.Slice(users, func(i, j int) bool { return users[i].Age < users[j].Age })
```

### TypeScript

**Delay abstraction - a plain function before a class hierarchy:**

```typescript
// Too early: base class, factory, and strategy for one email type
abstract class NotificationSender {
    abstract send(to: string, payload: NotificationPayload): Promise<void>
}

class EmailSender extends NotificationSender {
    async send(to: string, payload: NotificationPayload): Promise<void> { ... }
}

class NotificationFactory {
    static create(type: string): NotificationSender { ... }
}
```

```typescript
// Lean: just the function
async function sendEmail(to: string, subject: string, body: string): Promise<void> {
    await mailer.send({ to, subject, body })
}
```

---

**Clarity over cleverness:**

```typescript
// Clever: hard to read, hard to debug
const result = data
    .filter(Boolean)
    .reduce((acc, x) => ({ ...acc, [x.id]: (acc[x.id] ?? 0) + x.value }), {} as Record<string, number>)
```

```typescript
// Clear: a reader can follow this without stopping to think
const totals: Record<string, number> = {}
for (const item of data) {
    if (!item) continue
    totals[item.id] = (totals[item.id] ?? 0) + item.value
}
```

---

**Reach for native APIs before adding a dependency:**

```typescript
// Unnecessary: axios adds 40 kB and an extra dependency for a simple request
import axios from "axios"
const { data } = await axios.get(url, { headers: { Authorization: `Bearer ${token}` } })
```

```typescript
// Native fetch is sufficient
const res = await fetch(url, { headers: { Authorization: `Bearer ${token}` } })
const data = await res.json()
```

---

**Types should clarify, not perform:**

```typescript
// Over-typed: generic machinery that adds noise without safety
type DeepPartial<T> = T extends object ? { [K in keyof T]?: DeepPartial<T[K]> } : T
type StrictExtract<T, U extends T> = U
function mergeConfig<T extends BaseConfig>(base: T, override: DeepPartial<T>): T { ... }
```

```typescript
// Lean: precise types for what actually varies
interface Config { timeout: number; retries: number }

function mergeConfig(base: Config, override: Partial<Config>): Config {
    return { ...base, ...override }
}
```

Use generics and advanced types when they remove real duplication or enforce a real invariant - not to demonstrate sophistication.

## Iron Laws

1. **NEVER** introduce a new service, queue, or database without first demonstrating that a single component cannot handle the load - complexity added for imaginary future scale always compounds into real present cost.
2. **ALWAYS** write the simplest direct implementation first; only abstract when a second real use case exists and the shared shape is obvious - the wrong abstraction is worse than duplication.
3. **Prefer inlining over adding a dependency** when the required functionality is small, well-understood, and carries no meaningful risk - but favour a well-maintained library when the problem is genuinely hard (cryptography, parsing, protocol handling) or when the implementation would need its own tests and maintenance.
4. **Aim for a solution a new engineer can follow without needing to jump across many files or layers** - if tracing the core user action requires significant archaeology, treat that as a signal to simplify, not a rule violation to fix by count.
5. **NEVER** add configuration options, modes, or feature flags speculatively - expose a knob only when a real user has a real need to vary the behavior.

### Elixir

**Delay abstraction - one storage backend does not need a behaviour:**

```elixir
# Too early: behaviour built for a swap that may never happen
defmodule ObjectStore do
  @callback upload(key :: String.t(), body :: binary()) :: :ok | {:error, term()}
  @callback download(key :: String.t()) :: {:ok, binary()} | {:error, term()}
  @callback delete(key :: String.t()) :: :ok | {:error, term()}
end

defmodule S3Store do
  @behaviour ObjectStore
  def upload(key, body), do: ExAws.S3.put_object(bucket(), key, body) |> ExAws.request()
  def download(key), do: ExAws.S3.get_object(bucket(), key) |> ExAws.request()
  def delete(key), do: ExAws.S3.delete_object(bucket(), key) |> ExAws.request()
end
```

```elixir
# Lean: just call S3 directly until a second backend is a real requirement
defmodule Storage do
  def upload(key, body), do: ExAws.S3.put_object(bucket(), key, body) |> ExAws.request()
  def download(key), do: ExAws.S3.get_object(bucket(), key) |> ExAws.request()
  def delete(key), do: ExAws.S3.delete_object(bucket(), key) |> ExAws.request()
end
```

Introduce the behaviour when local disk or GCS is a real requirement - not as insurance against a hypothetical swap.

---

**Use the standard library before reaching for a package:**

```elixir
# Unnecessary: pulling in a CSV parsing library for a simple two-column file
defp parse(path) do
  path |> File.read!() |> CSV.decode!(headers: true) |> Enum.to_list()
end
```

```elixir
# Lean: stdlib is sufficient for simple cases
defp parse(path) do
  path
  |> File.stream!()
  |> Stream.map(&String.trim/1)
  |> Stream.map(&String.split(&1, ","))
  |> Enum.to_list()
end
```

---

**Pattern matching over defensive conditionals:**

```elixir
# Over-engineered: manual type and nil checks
def process_response(response) do
  if response != nil do
    if Map.has_key?(response, :status) do
      if response.status == :ok do
        handle_success(response.data)
      else
        handle_error(response.reason)
      end
    end
  end
end
```

```elixir
# Lean: let pattern matching do the work
def process_response(%{status: :ok, data: data}), do: handle_success(data)
def process_response(%{status: :error, reason: reason}), do: handle_error(reason)
```

---

**Write less code - use `with` to flatten nested error handling:**

```elixir
# Nested case: grows with every step
def create_order(params) do
  case validate(params) do
    {:ok, validated} ->
      case fetch_user(validated.user_id) do
        {:ok, user} ->
          case charge(user, validated.amount) do
            {:ok, charge} -> {:ok, charge}
            {:error, reason} -> {:error, reason}
          end
        {:error, reason} -> {:error, reason}
      end
    {:error, reason} -> {:error, reason}
  end
end
```

```elixir
# Lean: with keeps the happy path flat
def create_order(params) do
  with {:ok, validated} <- validate(params),
       {:ok, user}      <- fetch_user(validated.user_id),
       {:ok, charge}    <- charge(user, validated.amount) do
    {:ok, charge}
  end
end
```

## Anti-Patterns

| Anti-Pattern | Why It Fails | Correct Approach |
| --- | --- | --- |
| Microservices from day one | Network overhead, distributed tracing, and deployment complexity before any scale justification | Start with a monolith; extract services only when independent scaling or deployment is a real need |
| Abstraction before the second use case | Abstractions built from one example embed the wrong shape; the second case always breaks the interface | Duplicate once; extract when the second case reveals the true shared contract |
| Adding Redis/Kafka/vector DB speculatively | Each store adds an operational dependency, a consistency boundary, and an onboarding burden | Add a data store only when Postgres (or equivalent) demonstrably cannot handle the access pattern |
| Clever one-liners over readable code | Saves the writer a minute; costs every future reader five minutes per encounter | Write the obvious four lines; compress only when the abstraction is universally understood |
| Calling missing features "focus" | Underbuilding critical paths (auth, error handling, data integrity) is technical debt, not minimalism | Distinguish the core loop - which must be robust - from peripheral features that can wait |
| Premature configurability | Every option doubles the test matrix and the mental model required to use the system | Hard-code sensible defaults; promote to config only when a real user needs to change it |
