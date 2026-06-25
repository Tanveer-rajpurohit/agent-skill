---
name: code-review
description: >
  Staff-engineer-level code review. Trigger for: "review my code",
  "check this", "what's wrong here", "is this good", "any issues",
  "code review", "review this PR", "give me feedback", "audit this".
  Finds bugs that pass CI but break in production. Based on
  garrytan/review patterns + PatrickJS clean-code rules.
---

# Code Review Skill — Staff Engineer Level

Find bugs that pass CI but blow up in production.
Review order: Bugs → Security → Logic → Performance → Readability.
Never bury a bug at the bottom.

## Output Format

```
## 🔴 Bugs (Must Fix Before Merge)
[line number + what breaks + why]

## 🟠 Security (Fix This Sprint)
[injection, auth bypass, data exposure]

## 🟡 Logic Issues (Should Fix)
[wrong behavior, edge cases missed, race conditions]

## 🔵 Performance (Fix If This Is Hot Path)
[N+1, unbounded queries, unnecessary re-renders]

## ⚪ Readability (Nice to Have)
[naming, structure, comments]

## ✅ What's Done Well
[always include — acknowledge good patterns]

## Summary
[2-3 line verdict]
```

---

## Bug Patterns to Check Every Review

### TypeScript / Node.js
```ts
// Missing await — most common TS bug
const user = db.user.findById(id)  // ← no await, user is Promise
if (!user) throw new NotFoundError() // ← always throws

// Type assertion hiding a bug
const data = (res as any).data       // ← hides null check
const id = req.params.id as string   // ← could be undefined

// Unhandled promise rejection
someAsyncFn()  // ← no await, no .catch — silent failure

// forEach with async (doesn't await)
users.forEach(async (user) => {
  await sendEmail(user)  // ← these run but errors are swallowed
})
// Fix: await Promise.all(users.map(async u => sendEmail(u)))

// Object mutation in React state
const newList = items  // ← same reference, React won't re-render
newList.push(item)
setItems(newList)
// Fix: setItems([...items, item])

// useEffect dependency array lie
useEffect(() => {
  fetchData(userId)  // ← userId used but not in deps
}, [])  // stale closure — never re-fetches on userId change

// Missing error boundary
const data = JSON.parse(untrustedInput)  // ← throws on invalid JSON
```

### Go
```go
// Ignoring error return
result, _ := db.Query(...)  // ← error silently dropped

// Goroutine leak — no exit condition
go func() {
  for {
    // ← runs forever, no done channel, no context
    process()
  }
}()

// Race condition on shared map
var cache = map[string]string{}
go func() { cache["key"] = "val" }()   // ← concurrent write — panic

// Context not passed — can't cancel
func fetchUser(id string) (*User, error) {
  // ← should be fetchUser(ctx context.Context, id string)
  return db.QueryRow("SELECT ...")
}

// Defer in loop — resources not freed until function returns
for _, file := range files {
  f, _ := os.Open(file)
  defer f.Close()  // ← defers stack up, not freed each iteration
  // Fix: extract to function or close explicitly
}
```

### React
```tsx
// Key using array index — breaks reconciliation on reorder/delete
{items.map((item, i) => <Card key={i} />)}
// Fix: key={item.id}

// State update in render — infinite loop
function Component() {
  const [count, setCount] = useState(0)
  setCount(count + 1)  // ← called every render → infinite loop
}

// Memory leak — event listener not cleaned up
useEffect(() => {
  window.addEventListener('resize', handler)
  // ← missing return () => window.removeEventListener(...)
}, [])

// Prop drilling flag — smell for refactoring
<A userPermissions={p}>
  <B userPermissions={p}>
    <C userPermissions={p} />  // ← extract to context
```

---

## Security Patterns to Check

```ts
// SQL injection
db.query(`SELECT * FROM users WHERE id = '${req.params.id}'`)
// Fix: db.query('SELECT * FROM users WHERE id = $1', [req.params.id])

// Missing auth check
router.delete('/users/:id', async (req, res) => {
  // ← where is authenticate middleware?
  // ← is this user allowed to delete THIS id?
  await db.user.delete(req.params.id)
})

// Secret in code
const secret = "hardcoded-jwt-secret-123"
// Fix: process.env.JWT_SECRET (and validate it exists at startup)

// Path traversal
const file = req.query.filename
fs.readFile(`./uploads/${file}`)  // ← ../../etc/passwd

// XSS
element.innerHTML = userContent  // ← use textContent or DOMPurify

// SSRF — fetching user-supplied URL
const data = await fetch(req.body.webhookUrl)  // ← allow-list URLs
```

---

## Performance Patterns to Check

```ts
// N+1 query — most common backend performance bug
const users = await db.user.findAll()
for (const user of users) {
  user.orders = await db.order.findByUser(user.id)  // ← 1 query per user!
}
// Fix: JOIN or include in the initial query

// Unbounded query — will kill DB at scale
const allLogs = await db.log.findAll()  // ← no LIMIT
// Fix: always paginate: LIMIT $1 OFFSET $2

// Missing index — slow at 100k rows
WHERE created_at > $1  // ← is created_at indexed?

// Unnecessary serialization
JSON.stringify(bigObject)  // ← in a hot loop?

// React: creating objects/functions in render (breaks memoization)
<Component onClick={() => doSomething(id)} />  // ← new fn every render
// Fix: useCallback or move outside component
```

---

## What Good Code Looks Like (Acknowledge These)

- Error handling at every async boundary
- Types that make illegal states unrepresentable
- Functions that do one thing
- Tests that test behavior not implementation
- Parameterized queries
- Context passed through for cancellation (Go)
- Cleanup in useEffect return (React)
- Structured logging with context (not console.log("user:", user))
- Pagination on all list endpoints
- Input validation at the boundary with a schema (Zod, validator)
