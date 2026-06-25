# File Operations — Built-in Tools Only

## The Rule
**Never use shell commands to read, write, or edit files.**
Use the built-in file tools the AI has natively. Always.

## Forbidden — Never Use These for File Work
```bash
# These are BANNED for file reading/writing
cat file.ts
echo "content" > file.ts
sed -i 's/old/new/g' file.ts
awk '{print}' file.ts
head / tail on source files
tee, truncate, dd
```

Why: Shell commands bypass the AI's native edit tracking, produce
terminal noise Tanveer has to read through, and can't be undone
with /rewind or diff tools.

## Required — Always Use These Instead

| Task | Use |
|---|---|
| Read a file | read_file / View tool |
| Edit a file | edit_file / str_replace / write_file |
| Create a new file | create_file / write_file |
| Find a file | find . -name "pattern" (search only, OK) |
| Check if file exists | Bash stat/test is OK for existence check only |

## What Bash IS Allowed For
These are fine in terminal — they don't touch file content:

```bash
# Running tools — OK
npm run lint
npm run build
go build ./...
go test ./...

# Finding files — OK  
find . -name "*.go" -not -path "*/vendor/*"
ls -la src/

# Git operations — OK
git status
git diff
git log --oneline -10
git add .
git commit -m "feat: ..."

# Checking process/ports — OK
lsof -i :3001
ps aux | grep node
```

## Pattern: Reading Before Editing
Always read the file first, then edit:

```
1. read_file("src/server.ts")          ← see current content
2. edit_file("src/server.ts", ...)     ← make the change
3. read_file("src/server.ts")          ← confirm change is correct
```

Never assume the current state of a file. Read it first.

## Pattern: New Files
For new files, use create_file — not echo or heredoc:
```
create_file("src/handlers/user.go", content)
```

## Why This Matters for Tanveer's Workflow
- Claude Code's /rewind only works on native file operations
- /diff only shows changes made through native tools
- Terminal output clutters the conversation context
- Native edits are atomic — no partial write corruption
