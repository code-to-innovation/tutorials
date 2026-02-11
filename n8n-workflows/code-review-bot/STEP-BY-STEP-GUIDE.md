# GitHub PR Code Review Automation - n8n Workflow

This workflow automatically reviews GitHub pull requests and posts code quality feedback as comments.

## Step 1: GitHub Trigger
- Drag **GitHub Trigger** node to canvas
- Configure:
  - Repository Owner: `code-to-innovation`
  - Repository Name: `tutorials`
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
    issues.push(`🔍 **${filename}**: Remove console.log`);
  }
  
  if (patch.includes('debugger')) {
    issues.push(`🐛 **${filename}**: Remove debugger`);
  }
  
  if (patch.match(/TODO|FIXME/i)) {
    issues.push(`📝 **${filename}**: TODO/FIXME found`);
  }
  
  if (additions > 500) {
    suggestions.push(`📦 **${filename}**: Large file`);
  }
});

let comment = '## 🤖 Code Review\n\n';

if (issues.length === 0 && suggestions.length === 0) {
  comment += '✅ No issues found!\n';
} else {
  if (issues.length > 0) {
    comment += '### ❌ Issues\n';
    issues.forEach(i => comment += `- ${i}\n`);
  }
  if (suggestions.length > 0) {
    comment += '\n### 💡 Suggestions\n';
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
  - Body: JSON
  - Add parameter: `body` = `{{ $json.comment }}`

## Final Steps
1. Save workflow
2. Activate workflow
3. Test by creating a PR in your repository

## What This Workflow Does

When a PR is opened or updated, the workflow:
- ✅ Fetches all changed files
- 🔍 Checks for `console.log` statements
- 🐛 Checks for `debugger` statements
- 📝 Flags TODO/FIXME comments
- 📦 Warns about large files (>500 lines)
- 💬 Posts a review comment with findings

## Customization Ideas

Add more checks in Step 5:
- Hardcoded secrets or API keys
- Missing error handling
- Unused imports
- Code formatting issues
- Security vulnerabilities
