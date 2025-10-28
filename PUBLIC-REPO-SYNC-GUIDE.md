# Public Repository Sync Guide

## Overview

The SUPERVISOR documentation is published in two places:

1. **Private Main Repo**: `docs/SUPERVISOR/` in `encore-loyalty-new-system`
2. **Public Repo**: https://github.com/PREDICTif/encore-loyalty-supervisor

This guide explains how to keep them synchronized.

---

## Quick Reference

```bash
# Push local changes to public repo (after committing to main repo)
./sync-supervisor.sh push

# Pull changes from public repo to local
./sync-supervisor.sh pull

# Check sync status
./sync-supervisor.sh status
```

---

## Typical Workflow

### When You Update SUPERVISOR Documents

1. **Make your changes** to files in `docs/SUPERVISOR/`
2. **Test** that documentation is accurate
3. **Commit to main repo**:
   ```bash
   git add docs/SUPERVISOR/
   git commit -m "Update SUPERVISOR documentation"
   git push origin main
   ```
4. **Sync to public repo**:
   ```bash
   ./sync-supervisor.sh push
   ```

### When Client Suggests Changes via Public Repo

If someone creates an issue or PR on the public repo:

1. **Pull their changes**:
   ```bash
   ./sync-supervisor.sh pull
   ```
2. **Review the changes** in `docs/SUPERVISOR/`
3. **Commit to main repo** if you accept them:
   ```bash
   git add docs/SUPERVISOR/
   git commit -m "Apply client suggestions from public repo"
   git push origin main
   ```

---

## README Management

The SUPERVISOR folder has **two README files**:

- **`README.md`** - Internal guide specific to Encore Loyalty project
- **`README-PUBLIC.md`** - Public-facing framework documentation

### How It Works

- In the **main repo**: `README.md` is the primary file (internal version)
- In the **public repo**: `README.md` is the public version (from `README-PUBLIC.md`)
- The sync script automatically swaps them during push/pull

### Editing READMEs

- To update **internal README**: Edit `docs/SUPERVISOR/README.md`
- To update **public README**: Edit `docs/SUPERVISOR/README-PUBLIC.md` or `PUBLIC-README.md`

After editing either, run:
```bash
git add docs/SUPERVISOR/
git commit -m "Update SUPERVISOR README"
./sync-supervisor.sh push
```

---

## What Gets Published

All files in `docs/SUPERVISOR/` are published to the public repo:

✅ **Included**:
- SUPERVISOR-FRAMEWORK.md
- CURRENT-STATUS.md
- THINKING-LOG.md
- DELEGATION-TRACKER.md
- DECISIONS-LOG.md
- INDEX.md
- QUICK-REFERENCE.md
- WORKER-PROMPTS/ (directory)
- README.md (public version)

❌ **Excluded**:
- Nothing is excluded - all SUPERVISOR content is shareable

⚠️ **Sensitive Info**: Review commits before pushing to ensure no secrets, keys, or private client data are included.

---

## Technical Details

### Git Configuration

- **Remote Name**: `supervisor-public`
- **Remote URL**: `https://github.com/PREDICTif/encore-loyalty-supervisor.git`
- **Branch**: `main`
- **Method**: Clone → Copy → Commit → Push (not using git subtree due to availability)

### How the Sync Script Works

**Push Process**:
1. Checks for uncommitted changes (fails if found)
2. Clones public repo to `/tmp/supervisor-sync-$$`
3. Removes old files from clone
4. Copies all files from `docs/SUPERVISOR/`
5. Swaps `README-PUBLIC.md` → `README.md`
6. Commits and pushes to public repo
7. Cleans up temp directory

**Pull Process**:
1. Checks for uncommitted changes (fails if found)
2. Clones public repo to temp directory
3. Backs up internal `README.md`
4. Removes files from `docs/SUPERVISOR/`
5. Copies all files from public repo
6. Restores internal `README.md` and saves public version as `README-PUBLIC.md`
7. User must review and commit changes

---

## Troubleshooting

### Error: "You have uncommitted changes"

**Cause**: You have modified files in `docs/SUPERVISOR/` that aren't committed.

**Solution**:
```bash
git add docs/SUPERVISOR/
git commit -m "Update SUPERVISOR docs"
./sync-supervisor.sh push
```

### Error: "spawn /bin/bash ENOENT"

**Cause**: Bash interpreter issue (rare).

**Solution**: Run directly with bash:
```bash
bash sync-supervisor.sh push
```

### Public Repo Out of Sync

**Check status**:
```bash
./sync-supervisor.sh status
```

**Force sync**:
```bash
# Commit any local changes first
git add docs/SUPERVISOR/
git commit -m "Update SUPERVISOR"

# Push to public repo
./sync-supervisor.sh push
```

### Need to Undo a Public Push

Public repo changes are visible immediately. To revert:

1. **In main repo**, revert the commit:
   ```bash
   git revert <commit-hash>
   git push origin main
   ```
2. **Sync to public repo**:
   ```bash
   ./sync-supervisor.sh push
   ```

---

## Security Considerations

### Before Pushing to Public Repo

Review the diff to ensure no sensitive data:

```bash
# Check what changed in SUPERVISOR folder
git diff HEAD~1 -- docs/SUPERVISOR/

# Review specific file
git diff HEAD~1 -- docs/SUPERVISOR/CURRENT-STATUS.md
```

### Sensitive Information Checklist

❌ **Never include**:
- AWS credentials or API keys
- Database passwords or connection strings
- Private client contact information
- Internal server IP addresses (unless public-facing)
- Proprietary business logic details
- Financial information

✅ **Safe to include**:
- General project status (without sensitive details)
- Strategic thinking and methodology
- Delegation patterns and frameworks
- Decision-making processes
- Generic examples and templates

---

## Maintenance

### Weekly

- Review `CURRENT-STATUS.md` and `THINKING-LOG.md` for sensitive info
- Push updates after major milestones
- Verify public repo is rendering correctly

### Monthly

- Check for issues or discussions on public repo
- Review analytics (if enabled) to see who's viewing
- Update documentation based on lessons learned

### As Needed

- Respond to questions or PRs on public repo
- Update framework based on project evolution
- Share link with clients and stakeholders

---

## Repository URLs

- **Private Main**: https://github.com/PREDICTif/encore-loyalty-new-system (assumed)
- **Public SUPERVISOR**: https://github.com/PREDICTif/encore-loyalty-supervisor

---

## Getting Help

If you encounter issues with the sync script:

1. Check this guide
2. Review `sync-supervisor.sh` script comments
3. Test commands manually:
   ```bash
   git clone https://github.com/PREDICTif/encore-loyalty-supervisor.git /tmp/test
   cd /tmp/test
   # Make changes
   git add .
   git commit -m "Test"
   git push origin main
   ```

---

**Last Updated**: October 28, 2025
**Script Version**: 1.0
**Status**: ✅ Operational


