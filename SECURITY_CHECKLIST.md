# Public Repository Security Checklist

This document provides a security checklist for maintaining the NOFX public repository.

## ⚠️ CRITICAL: Never Commit These Files

The following files contain sensitive information and **must not** be committed to version control:

### 1. `.env` - Environment Variables
**Contains:** API keys, encryption keys, JWT secrets, database credentials
**Location:** Repository root
**Status:** Ignored by .gitignore (line 31)

### 2. `config.db` - SQLite Database
**Contains:** Exchange API keys, trader configurations, encrypted data
**Location:** Repository root
**Status:** Ignored by .gitignore (line 33)

### 3. `beta_codes.txt` - Beta Invite Codes
**Contains:** Invitation codes (even if empty)
**Location:** Repository root
**Status:** Ignored by .gitignore (line 134, added in recent commit)

### 4. `secrets/` - RSA Encryption Keys
**Contains:** Private key (rsa_key) and public key (rsa_key.pub)
**Location:** Repository root
**Status:** Ignored by .gitignore (line 59-64)

## ✅ Safe to Commit

These files are safe to commit and should be included:

### Configuration Templates
- `.env.example` - Template with placeholder values (no real secrets)
- `config.json.example` - Example configuration file
- `.gitignore` - Ensure sensitive files are excluded

### Documentation
- `README.md` - Project documentation
- `*.md` files - All documentation is safe
- `.merge_docs/` - Documentation directory

### Source Code
- All Go source files (`*.go`)
- All TypeScript/JavaScript files (`*.ts`, `*.tsx`, `*.js`)
- All configuration files (not containing secrets)
- Docker files (`Dockerfile*`, `docker-compose.yml`)

### Test Files
- All test files (`*_test.go`)
- Test utilities and mocks

## 🛡️ Security Verification Commands

### Check if sensitive files are tracked
```bash
# Check if .env is tracked
git ls-files | grep "^\.env$"

# Check if config.db is tracked
git ls-files | grep "^config\.db$"

# Check if secrets directory is tracked
git ls-files | grep "^secrets/"

# Check all sensitive files at once
git ls-files | grep -E "^\.env$|^config\.db$|^beta_codes\.txt$|^secrets/"
```

**If any output appears, the sensitive file is tracked - IMMEDIATELY STOP and fix!**

### Check if files are properly ignored
```bash
# Test .env
git check-ignore .env

# Test config.db
git check-ignore config.db

# Test multiple files
git check-ignore .env config.db secrets/rsa_key
```

**If no output, the file is NOT ignored - fix .gitignore!**

### Verify before committing
```bash
# Check what's being committed
git status

# Check staged files
git diff --staged --name-only

# Check for any sensitive files in staging
git diff --staged --name-only | grep -E "^\.env$|^config\.db$|^beta_codes\.txt$|^secrets/"
```

## 🔧 How to Fix if Sensitive Files Were Committed

### Option 1: If just committed (easy)
```bash
# If you just committed sensitive files:
git reset HEAD~1

# Add to .gitignore if not already there
echo ".env" >> .gitignore
echo "config.db" >> .gitignore
echo "beta_codes.txt" >> .gitignore
echo "secrets/" >> .gitignore

# Verify files are ignored
git check-ignore .env config.db secrets/

# Commit again (without sensitive files)
git add .gitignore
git commit -m "security: remove sensitive files"
git push --force
```

### Option 2: If committed in past (harder)
```bash
# Use git filter-branch or BFG Repo-Cleaner
# This rewrites history - use with caution!

# With BFG Repo-Cleaner:
java -jar bfg.jar --delete-files .env --delete-files config.db --delete-files beta_codes.txt

# Then force push (affects all collaborators):
git push --force
```

### Option 3: Rotate compromised secrets
If secrets were exposed in public repository:
1. **Immediately rotate all API keys**
2. Generate new encryption keys
3. Change database credentials
4. Update JWT secret
5. Revoke and regenerate all tokens

## 📋 Pre-Commit Checklist

Before every commit, verify:

- [ ] `git status` shows no untracked sensitive files
- [ ] `git diff --staged` contains no sensitive files
- [ ] `.env` is in .gitignore
- [ ] `config.db` is in .gitignore
- [ ] `beta_codes.txt` is in .gitignore
- [ ] `secrets/` is in .gitignore
- [ ] `.env.example` exists with placeholders
- [ ] No API keys in source code
- [ ] No private keys in source code

## 🚨 What to Do if You Accidentally Commit Secrets

1. **PANIC IS NORMAL** - But act fast!
2. **Don't just remove in next commit** - It's still in history
3. **Use git reset (if just committed)** or git filter-branch/BFG
4. **Rotate ALL affected credentials immediately**
5. **Force push to overwrite history**
6. **Notify collaborators to rebase their branches**
7. **Monitor API usage for suspicious activity**

## 🔍 Continuous Monitoring

Set up pre-commit hooks to catch secrets before they leak:

```bash
# Install pre-commit hook
cat > .git/hooks/pre-commit << 'EOF'
#!/bin/bash
# Pre-commit hook to check for secrets

# Detect secrets in staged files
if git diff --cached --name-only | grep -E "^\.env$|^config\.db$|^beta_codes\.txt$|^secrets/"; then
  echo "ERROR: Sensitive files detected in commit!"
  echo "Remove these files from staging before committing."
  exit 1
fi

# Detect common secret patterns
if git diff --cached | grep -iE "api[_-]?key|secret|password|private[_-]?key"; then
  echo "WARNING: Potential secrets detected in changes!"
  echo "Review your changes carefully."
  exit 1
fi
EOF

chmod +x .git/hooks/pre-commit
```

Remember: **When in doubt, don't commit!** It's easier to add files later than to remove secrets from Git history.
