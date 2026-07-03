# AI Secure Coding Assistant

An n8n-based autonomous AI agent that accepts source code, identifies software vulnerabilities, explains vulnerable lines of code, generates a secure patched version, and produces a professional Markdown report.

This project is built for the Secure Coding Assistant assessment. It uses n8n built-in nodes for orchestration, OpenRouter for LLM reasoning, Pinecone for RAG-based secure coding context, and optional Google Sheets audit logging.

## Deliverables

- `n8n-secure-coding-assistant.workflow.json` - importable n8n workflow
- `AI_Secure_Coding_Assistant_System_Document.pdf` - Word system PDF

## What The Workflow Does

1. Accepts vulnerable source code through either a JSON webhook or an n8n file upload form.
2. Normalizes one or multiple source files into a common internal structure.
3. Adds line numbers so the AI can identify vulnerable LOC accurately.
4. Uses an AI Agent with OpenRouter and Pinecone RAG context to analyze the code.
5. Generates structured vulnerability findings, risk, impact, patched source code, and unified diff.
6. Verifies whether required patch artifacts were produced.
7. Generates a Markdown security report.
8. Optionally stores scan history in Google Sheets.

## Requirements Covered

- Accept a vulnerable source code file in Python, PHP, JavaScript, TypeScript, Java, Go, Ruby, C#, C/C++, or HTML.
- Identify the vulnerability.
- Identify vulnerable line numbers.
- Explain risk and impact.
- Generate a patched version of the code.
- Preserve original functionality while removing the vulnerable implementation.
- Generate a professional Markdown report.
- Scan multiple source files through the `files[]` JSON input.
- Use RAG with Pinecone for secure coding context.
- Verify whether the generated patch output is complete.
- Store previous scan history through the optional Google Sheets audit node.

## Prerequisites

Install or have access to:

- n8n, self-hosted or desktop
- OpenRouter API key
- Pinecone API key
- Pinecone index for secure coding knowledge, for example `secure-coding-kb`
- Google Gemini API key for embeddings used by the Pinecone retrieval node
- Optional Google Sheets OAuth credential if audit logging is enabled

The workflow is configured to use the OpenRouter model:

```text
openai/gpt-oss-20b:free
```

You can replace this with another OpenRouter-supported model inside the `OpenRouter Chat Model - Free GPT` node.

## Setup Instructions

### 1. Import The Workflow

1. Open n8n.
2. Go to `Workflows`.
3. Select `Import from File`.
4. Upload `n8n-secure-coding-assistant.workflow.json`.
5. Save the imported workflow.

### 2. Configure OpenRouter

1. Open the node named `OpenRouter Chat Model - Free GPT`.
2. Create or select an OpenRouter credential.
3. Add your OpenRouter API key.
4. Keep the model as `openai/gpt-oss-20b:free` or choose another available model.

### 3. Configure Pinecone

1. Open the node named `Pinecone - Secure Coding Knowledge Tool`.
2. Create or select a Pinecone credential.
3. Replace the placeholder index name if needed.
4. The default index name in the workflow is:

```text
secure-coding-kb
```

The Pinecone index should contain secure coding knowledge such as OWASP Top 10 notes, CWE references, remediation examples, secure coding guidelines, or previous vulnerability reports.

### 4. Configure Gemini Embeddings

1. Open the node named `Gemini Embeddings - Retrieve`.
2. Create or select a Google Gemini or PaLM API credential.
3. Save the credential.

This embeddings node is used by Pinecone during retrieval.

### 5. Configure Optional Google Sheets Audit Logging

The workflow includes an optional audit log node named `Google Sheets - Audit Log`.

To use it:

1. Create a Google Sheet.
2. Add a sheet tab named:

```text
Secure Code Scan History
```

3. Add these headers:

```text
Timestamp
Scan ID
File Path
Original Hash
Findings
Verified
Report
```

4. Open the `Google Sheets - Audit Log` node.
5. Replace `REPLACE_WITH_SPREADSHEET_ID` with your spreadsheet ID.
6. Select or create a Google Sheets OAuth credential.

If you do not want audit logging, disable the `Google Sheets - Audit Log` node.

### 6. Activate Or Test The Workflow

For webhook testing, use n8n test mode first:

1. Open the workflow.
2. Click `Execute Workflow`.
3. Send a POST request to the test webhook URL shown by n8n.

For production use:

1. Activate the workflow.
2. Use the production webhook URL from the `Webhook - JSON Source Input` node.

The webhook path is:

```text
secure-coding-assistant
```

## Input Method 1: JSON Webhook

Send a `POST` request with one source file:

```json
{
  "scan_id": "demo-scan-001",
  "file_path": "server.js",
  "language": "javascript",
  "source_code": "const express = require('express');\nconst { exec } = require('child_process');\nconst app = express();\napp.get('/ping', (req, res) => {\n  const host = req.query.host;\n  exec('ping -c 1 ' + host, (error, stdout, stderr) => {\n    res.send(stdout || stderr || error?.message);\n  });\n});\napp.listen(3000);"
}
```

You can also scan multiple files in one request:

```json
{
  "scan_id": "multi-file-demo-001",
  "files": [
    {
      "path": "server.js",
      "language": "javascript",
      "content": "const { exec } = require('child_process');\nfunction ping(host) {\n  exec('ping -c 1 ' + host);\n}"
    },
    {
      "path": "app.py",
      "language": "python",
      "content": "import os\nfrom flask import request\ncmd = request.args.get('cmd')\nos.system(cmd)"
    }
  ]
}
```

The workflow processes each item independently after normalization, so multi-file input is handled as parallel n8n items through the same AI review path.

## Input Method 2: File Upload Form

Use the n8n form trigger for a browser-based demo:

1. Open the node named `Form Trigger - File Upload`.
2. Copy the form test URL or production URL.
3. Open it in a browser.
4. Upload a source code file.
5. Optionally enter `language` and `scan_id`.
6. Submit the form.
7. Open the n8n execution result to view the generated report.

The form responds immediately with:

```text
Secure code scan started. Check the n8n execution output for the generated report.
```

## Good JavaScript Test File

Create a file named `vulnerable-app.js` with this content and upload it through the form:

```javascript
const express = require('express');
const { exec } = require('child_process');

const app = express();

app.get('/ping', (req, res) => {
  const host = req.query.host;
  exec('ping -c 1 ' + host, (error, stdout, stderr) => {
    if (error) {
      return res.send(error.message);
    }
    res.send(stdout || stderr);
  });
});

app.get('/search', (req, res) => {
  const q = req.query.q || '';
  res.send('<h1>Search results for: ' + q + '</h1>');
});

app.listen(3000, () => {
  console.log('Demo app running on port 3000');
});
```

Expected findings include command injection in the `/ping` route and reflected XSS in the `/search` route.

## Expected Output

The final result contains:

- `scan_id`
- `file_path`
- `source_type`
- `file_hash`
- `analysis_json`
- `patched_source`
- `unified_diff`
- `finding_count`
- `verified`
- `verification_status`
- `markdown_report`

The Markdown report includes:

- Scan summary
- Vulnerability findings
- Vulnerable LOC
- Risk and impact
- Functionality preservation notes
- Unified diff
- Patched source
- Verification status
- References

## How Patch Verification Works

The workflow verifies that the AI output includes the expected patch artifacts:

- Findings exist when vulnerabilities are detected.
- Vulnerable line numbers are present.
- Patched source is present when a vulnerability is reported.
- A unified diff is generated when a patch is created.
- The final result is marked as passed or requiring review.

For a production-grade version, add sandboxed execution nodes to run language-specific checks such as:

```text
node --check server.js
npm audit
python -m py_compile app.py
bandit -r .
semgrep scan
```

## Notes For Evaluators

This solution intentionally avoids hard-coded vulnerability rules or predefined vulnerability responses. The LLM receives the numbered source code, retrieves security context from Pinecone, reasons over the code, and returns structured findings and patches.

The workflow uses n8n built-in nodes for the main orchestration:

- Webhook
- Form Trigger
- Set
- IF
- AI Agent
- OpenRouter Chat Model
- Pinecone Vector Store
- Gemini Embeddings
- Google Sheets

Only one Code node is used for file normalization, line numbering, and lightweight hashing because n8n uploaded files and JSON files need to be converted into the same internal format before AI analysis.

## Troubleshooting

### Form Trigger Error

If the n8n Form Trigger fails to start, ensure there is no `Respond to Webhook` node connected to the form path. This workflow uses the Form Trigger's built-in `onReceived` response mode.

### Empty Or Corrupted Source Code

If uploaded content appears as unreadable characters, upload a plain-text source file such as `.js`, `.py`, or `.php`. Avoid uploading compiled files, compressed files, screenshots, or rich text documents.

### Pinecone Retrieval Fails

Check that:

- The Pinecone credential is valid.
- The index exists.
- The index name in the node matches your Pinecone project.
- The Gemini embeddings credential is configured.

### Google Sheets Fails

Either configure the Google Sheets credential and spreadsheet ID or disable the `Google Sheets - Audit Log` node.

