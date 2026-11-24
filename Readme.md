# App Release Tag Pipeline

## Overview
This Harness pipeline automates the creation of production release tags from Release Candidate (RC) tags in GitHub. It creates a clean release version tag and automatically cleans up old RC tags.

## Pipeline Details

| Property | Value |
|----------|-------|
| **Pipeline Name** | App Release Tag |
| **Pipeline Identifier** | `cicd_app_release_tag` |
| **Workflow Stage** | App Release Tag |
| **Repository** | cicd-app-release-tag |
| **Step Type** | Shell Script (Bash) |
| **Container Image** | alpine/git:latest |

## Functionality

### What It Does:
1. **Finds Latest RC Tag** - Identifies the most recent RC tag (e.g., `v1.1.20-rc1`)
2. **Creates Release Tag** - Generates production tag by removing RC suffix (e.g., `v1.1.20`)
3. **Tags Correct Commit** - Creates tag on the same commit SHA as the RC tag
4. **Pushes to Remote** - Pushes the release tag to GitHub repository
5. **Cleans Up RC Tags** - Automatically deletes all RC tags for the released version

### Example Flow:
```
RC Tag: v1.1.20-rc1  →  Release Tag: v1.1.20
RC Tag: v2.0.3-rc2   →  Release Tag: v2.0.3
```

After creating `v1.1.20`, the pipeline deletes `v1.1.20-rc1`, `v1.1.20-rc2`, etc.

---

## Prerequisites & Setup

### 1. GitHub Personal Access Token (Required)

**Create GitHub Token:**
1. Go to GitHub → **Settings** → **Developer settings** → **Personal access tokens** → **Tokens (classic)**
2. Click **Generate new token (classic)**
3. Select scopes:
   - ✅ `repo` - Full control of private repositories
   - ✅ `delete_repo` - Delete repositories (for tag deletion)
4. Generate and copy the token

**Add to Harness:**
1. Go to Harness → **Project Settings** → **Secrets**
2. Click **+ New Secret** → **Text**
3. Name: `github_token`
4. Value: Paste your GitHub token
5. Click **Save**

### 2. GitHub Connector (Required)

**Create Connector:**
1. Go to Harness → **Project Settings** → **Connectors**
2. Click **+ New Connector** → Select **GitHub**
3. Enter connector name (e.g., `github_connector`)
4. Authentication: **Personal Access Token**
5. Use the token created above
6. **Important**: Ensure it has **WRITE** access to repository
7. Test and save

### 3. Docker Registry Connector (Required)

**Create Connector:**
1. Go to Harness → **Project Settings** → **Connectors**
2. Click **+ New Connector** → Select **Docker Registry**
3. Enter connector name (e.g., `dockerhub_connector`)
4. Registry URL: `https://index.docker.io/v2/`
5. Authentication: **Anonymous** (for public images)
6. Test and save

### 4. Repository Tag Format (Required)

Your repository must have RC tags in this format:
- **RC Tags**: `v{major}.{minor}.{patch}-rc{number}`
  - Examples: `v1.1.20-rc1`, `v2.0.3-rc2`, `v1.5.0-rc3`
- **Release Tags**: `v{major}.{minor}.{patch}`
  - Examples: `v1.1.20`, `v2.0.3`, `v1.5.0`

---

## How to Use

### Step 1: Import Pipeline to Harness
1. Go to Harness → **Pipelines**
2. Click **+ Create a Pipeline**
3. Select **Import YAML**
4. Copy contents from `pipelines/cicd-app-release-tag.yaml`
5. Paste and save

### Step 2: Create RC Tag in Your Repository
```bash
git tag -a v1.0.0-rc1 -m "Release candidate 1"
git push origin v1.0.0-rc1
```

### Step 3: Run the Pipeline
1. Click **Run** on the pipeline
2. Provide runtime inputs:
   - **Codebase Connector**: Select your GitHub connector
   - **Repository Name**: `your-org/your-repo`
   - **Branch**: `main` or `master`
   - **Container Registry**: Select Docker Hub connector
3. Click **Run Pipeline**

### Step 4: Verify Results
Check your GitHub repository:
```bash
git fetch --tags
git tag -l
```

You should see:
- ✅ New release tag created (e.g., `v1.0.0`)
- ✅ Old RC tags removed (e.g., `v1.0.0-rc1` deleted)

---

## Expected Output

### Successful Execution:
```
=========================================
Creating Release Tag from RC Version
=========================================
Git authentication configured using token
Latest RC Tag found: v1.1.20-rc1
Creating Release Tag: v1.1.20
RC Tag commit SHA: a1b2c3d4e5f6
Pushing release tag to remote repository...
Release tag v1.1.20 created successfully!
=========================================

Cleaning up old RC tags...
Found RC tags to clean up:
v1.1.20-rc1
Deleting RC tag: v1.1.20-rc1
RC tags cleaned up successfully!

=========================================
Process completed successfully!
Release Tag: v1.1.20
=========================================
```

---

## Troubleshooting

### Error: "No RC tag found"
**Solution**: Create an RC tag first
```bash
git tag -a v1.0.0-rc1 -m "Release candidate"
git push origin v1.0.0-rc1
```

### Error: "Permission denied" or "Authentication failed"
**Solution**: 
1. Verify GitHub token has `repo` and `delete_repo` scopes
2. Check if secret `github_token` exists in Harness
3. Ensure GitHub connector has write access
4. Test the connector in Harness

### Error: "Failed to pull image alpine/git:latest"
**Solution**: 
1. Verify Docker Hub connector is configured
2. Select correct connector in pipeline runtime inputs
3. Test connector connection

### Error: "Failed to push tag to remote"
**Solution**:
1. Check if `GITHUB_TOKEN` environment variable is set in pipeline
2. Verify token is valid (not expired)
3. Ensure repository exists and you have write access
4. Check Harness pipeline logs for detailed error

### Tags Not Visible in GitHub
**Solution**:
1. Check Harness pipeline logs for errors
2. Verify GitHub token has write permissions
3. Manually fetch tags:
   ```bash
   git fetch --tags
   git tag -l
   ```

---

## Pipeline Configuration

### Runtime Inputs:
- **Connector Reference** (`<+input>`): GitHub connector with write access
- **Repository Name** (`<+input>`): Target repository
- **Build/Branch** (`<+input>`): Source branch (usually `main`)
- **Container Registry** (`<+input>`): Docker connector for alpine/git

### Environment Variables:
- `GITHUB_TOKEN`: Loaded from Harness secret `github_token`

### Failure Strategy:
- **Action**: Abort on all errors
- **Errors**: All errors trigger abort

---

## Integration with CI/CD

### Typical Workflow:
1. **App Build** → Package application
2. **App Containerize** → Build Docker image
3. **App Release Tag** → **(This Pipeline)** Create release tag
4. **Release Promotion** → Deploy to environments

### Automatic Triggering (Optional):

**Trigger on RC Tag Creation:**
1. Go to Pipeline → **Triggers**
2. Click **+ New Trigger** → **GitHub**
3. Configure:
   - Event: **Tag**
   - Tag Pattern: `v*-rc*`
   - Action: Run this pipeline

---

## Best Practices

1. ✅ **Always create RC tags before running** this pipeline
2. ✅ **Verify RC tag is on correct commit** before releasing
3. ✅ **Use semantic versioning** (v{major}.{minor}.{patch})
4. ✅ **Monitor pipeline logs** for successful execution
5. ✅ **Coordinate with team** before creating release tags
6. ✅ **Rotate GitHub tokens** every 90 days for security

---

## Security Notes

- Store GitHub token as a Harness secret (never in code)
- Use project-level secrets (not account-level)
- Limit token scope to only necessary permissions
- Regularly audit who has access to secrets
- Consider using GitHub Apps for better security

---

## Support & Maintenance

**Pipeline File**: `pipelines/cicd-app-release-tag.yaml`  
**Documentation**: This README  
**Last Updated**: 2026-11-19  
**Maintained By**: DevOps Team

For issues or questions:
1. Review Harness pipeline execution logs
2. Check GitHub repository permissions
3. Verify connector configurations
4. Contact DevOps team for assistance

---

## Quick Reference

### Tag Format:
- RC: `v1.2.3-rc1` → Release: `v1.2.3`

### Required Secrets:
- `github_token` (GitHub Personal Access Token)

### Required Connectors:
- GitHub Connector (with write access)
- Docker Hub Connector (for alpine/git)

### Key Commands in Pipeline:
- `git fetch --tags` - Fetch all tags
- `git tag -l "v*-rc*"` - List RC tags
- `git tag -a v1.0.0 <commit> -m "msg"` - Create tag
- `git push origin v1.0.0` - Push tag
- `git push origin --delete v1.0.0-rc1` - Delete RC tag
