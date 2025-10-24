# Promptfoo Integration with CI/CD Pipelines

A comprehensive guide for integrating [Promptfoo](https://www.promptfoo.dev/) LLM evaluation framework with CI/CD pipelines including GitHub Actions, Azure Pipelines, and n8n workflow automation.

## 📋 Table of Contents

- [Overview](#overview)
- [Prerequisites](#prerequisites)
- [Project Structure](#project-structure)
- [Setup Instructions](#setup-instructions)
  - [1. GitHub Actions Setup](#1-github-actions-setup)
  - [2. Azure Pipelines Setup](#2-azure-pipelines-setup)
  - [3. n8n Integration Setup](#3-n8n-integration-setup)
- [Configuration](#configuration)
- [Usage](#usage)
- [Monitoring & Results](#monitoring--results)
- [Troubleshooting](#troubleshooting)
- [Best Practices](#best-practices)
- [Contributing](#contributing)

---

## 🎯 Overview

This project automates LLM prompt evaluation using Promptfoo across multiple CI/CD platforms:

- **GitHub Actions**: Automated testing on code push/PR
- **Azure Pipelines**: Enterprise-grade CI/CD integration
- **n8n**: Flexible workflow automation with custom triggers

### Why This Integration?

- ✅ Automated prompt quality assurance
- ✅ Consistent evaluation across environments
- ✅ Early detection of prompt regressions
- ✅ Scheduled or event-driven evaluations
- ✅ Multi-platform support for different workflows

---

## 📦 Prerequisites

### Required Tools

- **Git** (v2.0+)
- **Node.js** (v18+ recommended)
- **npm** or **yarn**
- **Docker** (for n8n setup)
- **Promptfoo CLI**: `npm install -g promptfoo`

### Required Accounts/Access

- GitHub account with repository access
- OpenAI API key (or other LLM provider)
- Azure DevOps account (for Azure Pipelines)
- Docker installed (for n8n)

### API Keys Needed

- `OPENAI_API_KEY` - Your OpenAI API key
- `ANTHROPIC_API_KEY` - (Optional) For Claude models
- Other provider keys as needed

---

## 📁 Project Structure

```
your-repo/
├── .github/
│   └── workflows/
│       └── promptfoo.yml          # GitHub Actions workflow
├── azure-pipelines/
│   └── promptfoo-pipeline.yml     # Azure Pipelines config
├── n8n/
│   ├── docker-compose.yml         # n8n Docker setup
│   └── .env.example               # Environment variables template
├── evaluations/
│   └── promptfoo/
│       ├── promptfooconfig.yaml   # Main Promptfoo configuration
│       ├── prompts/               # Prompt files
│       ├── test-data/             # Test datasets
│       └── output/                # Evaluation results
└── README.md                      # This file
```

---

## 🚀 Setup Instructions

### 1. GitHub Actions Setup

#### Step 1.1: Create Workflow File

Create `.github/workflows/promptfoo.yml`:

```yaml
name: Promptfoo Evaluation

on:
  workflow_dispatch:  # Manual trigger
  push:
    branches: [ main, develop ]
    paths:
      - 'evaluations/promptfoo/**'
  pull_request:
    branches: [ main ]

jobs:
  evaluate:
    runs-on: ubuntu-latest
    
    steps:
    - uses: actions/checkout@v3
    
    - name: Setup Node.js
      uses: actions/setup-node@v3
      with:
        node-version: '18'
        
    - name: Install promptfoo
      run: npm install -g promptfoo
      
    - name: Run promptfoo evaluation
      working-directory: ./evaluations/promptfoo
      env:
        OPENAI_API_KEY: ${{ secrets.OPENAI_API_KEY }}
        ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
      run: promptfoo eval
      
    - name: Upload results
      uses: actions/upload-artifact@v3
      if: always()
      with:
        name: promptfoo-results
        path: evaluations/promptfoo/output/
```

#### Step 1.2: Add API Keys to GitHub Secrets

1. Go to your repository on GitHub
2. Navigate to: **Settings** → **Secrets and variables** → **Actions**
3. Click **"New repository secret"**
4. Add secrets:
   - Name: `OPENAI_API_KEY`, Value: `sk-proj-...`
   - Name: `ANTHROPIC_API_KEY`, Value: `sk-ant-...`

#### Step 1.3: Update promptfooconfig.yaml

Ensure your config uses environment variables:

```yaml
providers:
  - id: openai:gpt-4
    config:
      apiKey: ${OPENAI_API_KEY}

prompts:
  - file://./prompts/my-prompt.txt

tests:
  - vars:
      question: "What is machine learning?"
    assert:
      - type: contains
        value: "algorithm"
```

#### Step 1.4: Test GitHub Actions

```bash
# Make a change and push
git add .
git commit -m "Add promptfoo GitHub Actions"
git push origin main

# Or trigger manually:
# Go to Actions tab → Select workflow → Run workflow
```

---

### 2. Azure Pipelines Setup

#### Step 2.1: Create Pipeline File

Create `azure-pipelines/promptfoo-pipeline.yml`:

```yaml
trigger:
  branches:
    include:
    - main
    - develop
  paths:
    include:
    - evaluations/promptfoo/**

pool:
  vmImage: 'ubuntu-latest'

variables:
  - group: promptfoo-secrets  # Variable group containing API keys

steps:
- task: NodeTool@0
  inputs:
    versionSpec: '18.x'
  displayName: 'Install Node.js'

- script: |
    npm install -g promptfoo
  displayName: 'Install Promptfoo'

- script: |
    cd evaluations/promptfoo
    promptfoo eval
  env:
    OPENAI_API_KEY: $(OPENAI_API_KEY)
    ANTHROPIC_API_KEY: $(ANTHROPIC_API_KEY)
  displayName: 'Run Promptfoo Evaluation'

- task: PublishBuildArtifacts@1
  inputs:
    PathtoPublish: 'evaluations/promptfoo/output'
    ArtifactName: 'promptfoo-results'
  displayName: 'Upload Results'
  condition: always()
```

#### Step 2.2: Configure Azure DevOps

1. Go to your Azure DevOps project
2. Navigate to **Pipelines** → **Library**
3. Create a **Variable Group** named `promptfoo-secrets`
4. Add variables:
   - `OPENAI_API_KEY` (mark as secret)
   - `ANTHROPIC_API_KEY` (mark as secret)

#### Step 2.3: Create Pipeline

1. Go to **Pipelines** → **New Pipeline**
2. Select your repository
3. Choose "Existing Azure Pipelines YAML file"
4. Select `azure-pipelines/promptfoo-pipeline.yml`
5. Click **Run**

---

### 3. n8n Integration Setup

#### Step 3.1: Create Docker Setup

Create `n8n/docker-compose.yml`:

```yaml
version: '3.8'

services:
  n8n:
    image: docker.n8n.io/n8nio/n8n
    container_name: n8n-promptfoo
    restart: unless-stopped
    ports:
      - "5678:5678"
    environment:
      - N8N_BASIC_AUTH_ACTIVE=true
      - N8N_BASIC_AUTH_USER=${N8N_USER}
      - N8N_BASIC_AUTH_PASSWORD=${N8N_PASSWORD}
      - OPENAI_API_KEY=${OPENAI_API_KEY}
      - ANTHROPIC_API_KEY=${ANTHROPIC_API_KEY}
      - WEBHOOK_URL=http://localhost:5678/
    volumes:
      - n8n_data:/home/node/.n8n
      - /var/run/docker.sock:/var/run/docker.sock
      - ../evaluations/promptfoo:/data/promptfoo:rw
    command: /bin/sh -c "apk add --no-cache git nodejs npm && npm install -g promptfoo && n8n start"

volumes:
  n8n_data:
```

#### Step 3.2: Create Environment File

Create `n8n/.env`:

```env
# n8n Authentication
N8N_USER=admin
N8N_PASSWORD=your_secure_password_here

# API Keys
OPENAI_API_KEY=sk-proj-your-key-here
ANTHROPIC_API_KEY=sk-ant-your-key-here

# Optional: Other provider keys
COHERE_API_KEY=
HUGGINGFACE_API_KEY=
```

⚠️ **Important**: Add `.env` to `.gitignore` to keep secrets safe!

#### Step 3.3: Start n8n

```bash
cd n8n
docker-compose up -d

# Verify it's running
docker ps

# View logs
docker-compose logs -f
```

#### Step 3.4: Access n8n

1. Open browser: `http://localhost:5678`
2. Login with credentials from `.env`

#### Step 3.5: Create n8n Workflow

**Method A: Import Pre-built Workflow** (Recommended)

1. Download the workflow JSON (see `n8n/workflows/` folder)
2. In n8n, click **Import from File**
3. Select the workflow JSON
4. Activate the workflow

**Method B: Create Manually**

1. Create new workflow: **"Promptfoo Evaluation"**
2. Add **Manual Trigger** or **Schedule Trigger** node
3. Add **Execute Command** node:
   - **Command**: `bash`
   - **Arguments**: 
     ```
     -c
     cd /data/promptfoo && export OPENAI_API_KEY="${OPENAI_API_KEY}" && promptfoo eval
     ```
4. (Optional) Add **HTTP Request** or **Slack** node to send notifications
5. Save and activate workflow

#### Step 3.6: Set Up Triggers

**Option A: Schedule Trigger**
```
Every day at 9:00 AM
Cron: 0 9 * * *
```

**Option B: Webhook Trigger**
1. Add **Webhook** node
2. Copy webhook URL
3. Use it in GitHub webhook settings or external tools

**Option C: GitHub Integration**
1. Add **GitHub Trigger** node
2. Authenticate with GitHub
3. Select repository and events (push, pull_request)

---

## ⚙️ Configuration

### promptfooconfig.yaml

Example configuration:

```yaml
# LLM Providers
providers:
  - id: openai:gpt-4
    config:
      apiKey: ${OPENAI_API_KEY}
      temperature: 0.7
  
  - id: anthropic:claude-3-sonnet-20240229
    config:
      apiKey: ${ANTHROPIC_API_KEY}

# Prompts to evaluate
prompts:
  - file://./prompts/customer-support.txt
  - file://./prompts/data-analysis.txt
  - |
    You are a helpful AI assistant.
    User query: {{query}}

# Test cases
tests:
  - vars:
      query: "Explain machine learning in simple terms"
    assert:
      - type: contains
        value: "algorithm"
      - type: llm-rubric
        value: "The response should be beginner-friendly"
  
  - vars:
      query: "What is the capital of France?"
    assert:
      - type: icontains
        value: "paris"
      - type: cost
        threshold: 0.01

# Output configuration
outputPath: ./output/results.json

# Evaluation options
evaluateOptions:
  maxConcurrency: 4
  cache: false
  showProgressBar: true
```

### Environment Variables

All platforms support these environment variables:

| Variable | Description | Required |
|----------|-------------|----------|
| `OPENAI_API_KEY` | OpenAI API key | Yes (if using OpenAI) |
| `ANTHROPIC_API_KEY` | Anthropic API key | Yes (if using Claude) |
| `COHERE_API_KEY` | Cohere API key | Optional |
| `HUGGINGFACE_API_KEY` | HuggingFace API key | Optional |

---

## 🎮 Usage

### Running Locally

```bash
# Navigate to promptfoo directory
cd evaluations/promptfoo

# Set environment variables
export OPENAI_API_KEY="your-key"

# Run evaluation
promptfoo eval

# View results
promptfoo view
```

### GitHub Actions

**Trigger manually:**
1. Go to repository **Actions** tab
2. Select "Promptfoo Evaluation" workflow
3. Click **"Run workflow"**

**Automatic triggers:**
- Pushes to `main` or `develop` branches
- Pull requests to `main`
- Only when files in `evaluations/promptfoo/` change

### Azure Pipelines

**Trigger manually:**
1. Go to **Pipelines**
2. Select your promptfoo pipeline
3. Click **"Run pipeline"**

**Automatic triggers:**
- Commits to `main` or `develop`
- Changes in `evaluations/promptfoo/` path

### n8n

**Manual execution:**
1. Open n8n interface
2. Go to your workflow
3. Click **"Execute Workflow"**

**Scheduled execution:**
- Configured via Schedule Trigger node
- Default: Daily at 9:00 AM

**Webhook trigger:**
- Use webhook URL to trigger from external systems
- Integrate with GitHub webhooks for automatic runs

---

## 📊 Monitoring & Results

### GitHub Actions

**View results:**
1. Go to **Actions** tab
2. Click on workflow run
3. Click on job name to see logs
4. Download artifacts from bottom of page

**Download results:**
```bash
# Results are saved as artifacts
# Download from Actions UI or use GitHub CLI:
gh run download <run-id> -n promptfoo-results
```

### Azure Pipelines

**View results:**
1. Go to **Pipelines** → **Runs**
2. Select your run
3. View logs in real-time
4. Download artifacts from **Summary** tab

### n8n

**View execution:**
1. Go to **Executions** tab in n8n
2. Click on execution to see details
3. View output of each node

**Access results:**
```bash
# SSH into container
docker exec -it n8n-promptfoo /bin/sh

# View results
cd /data/promptfoo/output
cat results.json
```

### Interpreting Results

Promptfoo generates:
- **results.json**: Detailed evaluation results
- **results.html**: Visual report (run `promptfoo view`)
- Console output with pass/fail status

Example output:
```
✓ Test 1: Simple question [PASS]
  - Provider: openai:gpt-4
  - Assertions: 2/2 passed
  - Cost: $0.003

✗ Test 2: Complex reasoning [FAIL]
  - Provider: openai:gpt-4
  - Assertions: 1/2 passed
  - Failed: Response too short
  - Cost: $0.005

Summary: 1/2 tests passed (50%)
Total cost: $0.008
```

---

## 🔧 Troubleshooting

### Common Issues

#### 1. "OPENAI_API_KEY not found"

**Cause**: API key not properly set in environment

**Solution**:
- **GitHub**: Check Secrets in repository settings
- **Azure**: Check Variable Group in Library
- **n8n**: Check `.env` file and ensure container has access

```bash
# Test locally
echo $OPENAI_API_KEY

# If empty, set it:
export OPENAI_API_KEY="sk-proj-..."
```

#### 2. "promptfooconfig.yaml not found"

**Cause**: Wrong working directory or file path

**Solution**:
```yaml
# GitHub Actions - add working-directory
working-directory: ./evaluations/promptfoo

# n8n - verify path in Execute Command
cd /data/promptfoo  # Adjust to your mounted path
```

#### 3. "Module not found" or "Command not found"

**Cause**: Promptfoo not installed

**Solution**:
```bash
# Install globally
npm install -g promptfoo

# Or install in project
npm install promptfoo --save-dev
```

#### 4. n8n Container Won't Start

**Cause**: Port conflict or volume mount issue

**Solution**:
```bash
# Check if port 5678 is already in use
lsof -i :5678

# Use different port in docker-compose.yml
ports:
  - "8080:5678"

# Check volume mounts
docker-compose config
```

#### 5. High API Costs

**Cause**: Running too many evaluations

**Solution**:
```yaml
# Limit test runs in promptfooconfig.yaml
evaluateOptions:
  maxConcurrency: 2  # Reduce parallelism
  
# Use caching
evaluateOptions:
  cache: true
  
# Filter tests
tests:
  # Only run critical tests in CI
```

#### 6. Tests Failing in CI but Passing Locally

**Cause**: Environment differences or API rate limits

**Solution**:
- Check environment variables are identical
- Add delays between API calls
- Verify API key quotas
- Check for network restrictions

### Debug Mode

Enable verbose logging:

```bash
# Local
promptfoo eval --verbose

# GitHub Actions - add to workflow
run: promptfoo eval --verbose

# n8n - add to Execute Command
promptfoo eval --verbose 2>&1 | tee /tmp/promptfoo.log
```

### Getting Help

1. Check [Promptfoo documentation](https://www.promptfoo.dev/docs)
2. Review workflow logs carefully
3. Test commands locally first
4. Check API provider status pages
5. Open an issue in this repository

---

## ✨ Best Practices

### Security

1. **Never commit API keys**
   ```bash
   # Add to .gitignore
   .env
   *.key
   secrets/
   ```

2. **Use environment variables**
   ```yaml
   # Good
   apiKey: ${OPENAI_API_KEY}
   
   # Bad
   apiKey: sk-proj-abc123...
   ```

3. **Rotate keys regularly**
   - Update secrets in all platforms
   - Use separate keys for dev/prod

4. **Limit key permissions**
   - Use read-only keys when possible
   - Set spending limits on API keys

### Cost Management

1. **Use caching**
   ```yaml
   evaluateOptions:
     cache: true
   ```

2. **Limit concurrent requests**
   ```yaml
   evaluateOptions:
     maxConcurrency: 2
   ```

3. **Monitor usage**
   - Set up billing alerts
   - Track costs per evaluation
   - Use cheaper models for testing

4. **Smart triggering**
   ```yaml
   # Only run on specific paths
   paths:
     - 'evaluations/promptfoo/**'
   ```

### Testing Strategy

1. **Start small**: Test with few providers and tests
2. **Use assertions**: Define clear pass/fail criteria
3. **Version control**: Track prompt changes over time
4. **Regular runs**: Schedule daily or weekly evaluations
5. **Regression testing**: Compare results across versions

### Workflow Optimization

1. **Parallel execution**: Use concurrent evaluations when possible
2. **Fail fast**: Stop on first failure in CI to save costs
3. **Selective testing**: Run full suite weekly, quick tests on every commit
4. **Artifact retention**: Keep results for historical comparison

---

## 🤝 Contributing

### Adding New Providers

1. Update `promptfooconfig.yaml`:
   ```yaml
   providers:
     - id: your-provider:model-name
       config:
         apiKey: ${YOUR_PROVIDER_KEY}
   ```

2. Add API key to all platforms:
   - GitHub Secrets
   - Azure Variable Group
   - n8n `.env` file

3. Test locally before pushing

### Adding New Tests

1. Create test case in `promptfooconfig.yaml`
2. Add test data to `test-data/` folder
3. Define assertions for expected behavior
4. Run locally to verify
5. Commit and push to trigger CI

### Improving Workflows

1. Fork this repository
2. Create a feature branch
3. Make your changes
4. Test thoroughly
5. Submit a pull request

---

## 📚 Additional Resources

- [Promptfoo Documentation](https://www.promptfoo.dev/docs)
- [GitHub Actions Documentation](https://docs.github.com/en/actions)
- [Azure Pipelines Documentation](https://learn.microsoft.com/en-us/azure/devops/pipelines/)
- [n8n Documentation](https://docs.n8n.io/)
- [OpenAI API Documentation](https://platform.openai.com/docs)
- [Anthropic API Documentation](https://docs.anthropic.com/)

---

## 📝 License

[Your License Here - e.g., MIT]

---

## 👥 Authors & Maintainers

- Name - https://github.com/avi350751


---

**Last Updated**: 10/23/2025
**Version**: 1.0.0