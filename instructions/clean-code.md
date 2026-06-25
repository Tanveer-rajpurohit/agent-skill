# Clean Code Standards — Always Apply

Distilled from PatrickJS/awesome-cursorrules clean-code.mdc + codequality.mdc.
These are baseline rules, not a skill — they apply to every file, every task.

## Names That Reveal Intent

```ts
// Bad
const d = new Date()
const arr = users.filter(u => u.a)
function proc(data: any) {}

// Good  
const createdAt = new Date()
const activeUsers = users.filter(u => u.isActive)
function processUserPayment(payment: Payment): Result<Receipt>
```

Rules:
- Variables/functions: say what they ARE or DO, not how they're stored
- No single-letter names except loop counters (i, j, k) and errors (err)
- No abbreviations unless universal: url, id, ctx, err, req, res are fine
- Booleans: is/has/can/should prefix — isLoading, hasPermission, canDelete
- Acronyms: consistent case — userID not userId, httpClient not HttpClient

## One Thing Per Function

```ts
// Bad — does 3 things
async function handleUserRegistration(data: RegisterInput) {
  const hashed = await bcrypt.hash(data.password, 10)
  const user = await db.user.create({ ...data, password: hashed })
  await emailService.sendWelcome(user.email)
  await analyticsService.track('user_registered', user.id)
  return user
}

// Good — each function does one thing
async function hashPassword(plain: string): Promise<string> {
  return bcrypt.hash(plain, 10)
}

async function createUser(data: CreateUserInput): Promise<User> {
  return db.user.create(data)
}

async function onUserRegistered(user: User): Promise<void> {
  await Promise.all([
    emailService.sendWelcome(user.email),
    analyticsService.track('user_registered', user.id)
  ])
}
```

Rule: if you need a comment to explain what a function does, split it.
Max 40 lines per function. If longer, extract.

## Constants Over Magic Numbers

```ts
// Bad
if (attempts > 5) lockAccount()
setTimeout(refresh, 900000)
const salt = await bcrypt.genSalt(10)

// Good
const MAX_LOGIN_ATTEMPTS = 5
const TOKEN_REFRESH_MS = 15 * 60 * 1000  // 15 minutes
const BCRYPT_ROUNDS = 10

if (attempts > MAX_LOGIN_ATTEMPTS) lockAccount()
setTimeout(refresh, TOKEN_REFRESH_MS)
const salt = await bcrypt.genSalt(BCRYPT_ROUNDS)
```

## Comments — Why, Not What

```ts
// Bad: explains what the code obviously does
// Loop through users and check if active
users.forEach(user => {
  if (user.isActive) { ... }
})

// Good: explains why a non-obvious choice was made
// bcrypt.compare is intentionally not short-circuited here
// to prevent timing attacks on the comparison
const matches = await bcrypt.compare(plain, hashed)

// Good: documents a constraint or gotcha
// MediaSoup requires transports to be connected before producing.
// Order matters: connect → produce → consume
await transport.connect({ dtlsParameters })
```

## DRY — One Source of Truth

```ts
// Bad: same validation logic in 3 route handlers
if (!email || !email.includes('@')) res.status(400).json(...)

// Good: one place
const emailSchema = z.string().email()
// Used everywhere through the same schema
```

## No Dead Code

```ts
// Remove, don't comment out
// const oldAuth = async (req) => { ... }   ← DELETE this, don't comment

// If you need history, that's what git is for
```

## Fail Fast at the Boundary

```ts
// Bad: validate deep inside business logic
async function createOrder(data: any) {
  // ... 50 lines of logic ...
  if (!data.userId) throw new Error('userId required')  // too late
}

// Good: validate at the entry point
async function createOrder(data: unknown): Promise<Order> {
  const validated = createOrderSchema.parse(data)  // fail here
  return orderService.create(validated)
}
```

## AI-Specific Rules (from codequality.mdc)

- Never invent changes beyond what was asked
- Never remove existing code unless specifically asked to
- Make one file change at a time — show it, verify, then continue
- If something is unclear, ask ONE question before proceeding
- Never ask the user to verify things visible in the provided context
- Never summarize changes in prose after making them — the diff speaks
