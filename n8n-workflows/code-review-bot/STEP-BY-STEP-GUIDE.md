# GitHub PR Code Review Automation - n8n Workflow

This workflow automatically reviews GitHub pull requests and posts code quality feedback as comments.

## Step 1: GitHub Trigger
- Drag **GitHub Trigger** node to canvas
- Configure:
  - Repository Owner: `your-github-username` (example: `code-to-innovation`)
  - Repository Name: `your-repo-name` (example: `tutorials`)
  - Events: Select `pull_request`
  - Connect your GitHub credentials

## Step 2: Build API URLs (Code Node)
- Drag **Code** node
- Connect from GitHub Trigger
- Paste this code:

```javascript
const json = $input.first().json;
const repo = json.repository || json.body?.repository;
const number = json.number || json.body?.number;
const action = json.action || json.body?.action;

if (!repo) {
  throw new Error('Repository not found in data');
}

return [{
  json: {
    filesUrl: `https://api.github.com/repos/${repo.owner.login}/${repo.name}/pulls/${number}/files`,
    commentsUrl: `https://api.github.com/repos/${repo.owner.login}/${repo.name}/issues/${number}/comments`,
    prNumber: number,
    repo: repo.name,
    owner: repo.owner.login,
    action: action
  }
}];
```

## Step 3: Filter PR Actions (IF Node)
- Drag **IF** node
- Connect from Code node
- Add conditions (OR logic):
  - Condition 1: `{{ $json.action }}` equals `opened`
  - Condition 2: `{{ $json.action }}` equals `synchronize`

## Step 4: Fetch PR Files (HTTP Request)
- Drag **HTTP Request** node
- Connect from IF node (true branch)
- Configure:
  - Method: `GET`
  - URL: `{{ $json.filesUrl }}`
  - Authentication: GitHub API credentials

## Step 5: Analyze Code Quality (Code Node)
- Drag **Code** node
- Connect from HTTP Request
- Paste this code:

```javascript
const files = $input.all();
const issues = [];
const suggestions = [];

files.forEach(file => {
  const filename = file.json.filename;
  const patch = file.json.patch || '';
  const additions = file.json.additions;
  
  if (patch.includes('console.log')) {
    issues.push(`**${filename}**: Remove console.log`);
  }
  
  if (patch.includes('debugger')) {
    issues.push(`**${filename}**: Remove debugger`);
  }
  
  if (patch.match(/TODO|FIXME/i)) {
    issues.push(`**${filename}**: TODO/FIXME found`);
  }
  
  if (additions > 500) {
    suggestions.push(`**${filename}**: Large file`);
  }
});

let comment = '## Code Review\n\n';

if (issues.length === 0 && suggestions.length === 0) {
  comment += 'No issues found!\n';
} else {
  if (issues.length > 0) {
    comment += '### Issues\n';
    issues.forEach(i => comment += `- ${i}\n`);
  }
  if (suggestions.length > 0) {
    comment += '\n### Suggestions\n';
    suggestions.forEach(s => comment += `- ${s}\n`);
  }
}

const urlData = $('Build API URLs').item.json;

return [{
  json: {
    comment: comment,
    prNumber: urlData.prNumber,
    repo: urlData.repo,
    owner: urlData.owner,
    commentsUrl: urlData.commentsUrl
  }
}];
```

## Step 6: Post Review Comment (HTTP Request)
- Drag **HTTP Request** node
- Connect from Analyze Code Quality
- Configure:
  - Method: `POST`
  - URL: `{{ $json.commentsUrl }}`
  - Authentication: GitHub API credentials
  - Send Body: Enable
  - Body Content Type: JSON
  - Specify Body: Using Fields Below
  - Add Body Parameter:
    - Name: `body`
    - Value: `{{ $json.comment }}`

## How to Run the Workflow

### Initial Setup
1. Click **Save** button (top right) to save the workflow
2. Click the **Active** toggle switch to activate the workflow
3. The workflow is now listening for GitHub PR events

### Verify Webhook Registration in GitHub

**Check if the webhook is registered:**
1. Go to your GitHub repository settings: `https://github.com/YOUR-USERNAME/YOUR-REPO/settings/hooks`
   - Example: `https://github.com/code-to-innovation/tutorials/settings/hooks`
2. You should see a webhook URL pointing to your n8n instance
3. Click on the webhook to see:
   - Recent deliveries
   - Which events it's subscribed to (should include `pull_request`)
   - Delivery status (green checkmark = successful)

**Webhook Details:**
- Payload URL: Your n8n webhook URL (e.g., `https://your-n8n-instance.com/webhook/...`)
- Content type: `application/json`
- Events: Pull requests
- Active: Should have a green checkmark

**Troubleshooting:**
- If webhook is missing: Deactivate and reactivate the workflow in n8n
- If webhook shows errors: Check the delivery details for error messages
- If webhook exists but not firing: Verify the events include `pull_request`

### Testing the Workflow

**Option 1: Create a Test PR**
1. Go to your GitHub repository
2. Create a new branch with test code containing:
   - `console.log()` statements
   - `debugger` statements
   - TODO/FIXME comments
3. Create a Pull Request from your branch to main
4. The workflow will automatically trigger and post a review comment

**Option 2: Manual Test with Pinned Data**
1. In n8n, click on the **GitHub Trigger** node
2. Click **Execute Node** button
3. Use the test data to simulate a PR event
4. Watch each node execute in sequence

**Option 3: Test Existing PR**
1. Update an existing open PR (push new commits)
2. The workflow triggers on the `synchronize` event
3. Check the PR for the automated review comment

### Verify It's Working
- Check your n8n workflow executions (left sidebar)
- Look for the review comment on your GitHub PR
- Review any error messages in failed executions
- Check webhook delivery history in GitHub Settings

## What This Workflow Does

When a PR is opened or updated, the workflow:
- Fetches all changed files
- Checks for `console.log` statements
- Checks for `debugger` statements
- Flags TODO/FIXME comments
- Warns about large files (>500 lines)
- Posts a review comment with findings

## Customization Ideas

Add more checks in Step 5:
- Hardcoded secrets or API keys
- Missing error handling
- Unused imports
- Code formatting issues
- Security vulnerabilities
