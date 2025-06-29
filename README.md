# 🛠️ Github Issues Tracker – n8n Workflow

This workflow fetches new GitHub issues from a repository on a scheduled basis, processes them through LLMs for enrichment, assigns responsibility based on content, and posts alerts to relevant Slack channels. 
Enriched data is stored into Supabase for persistent access.

---

## 📌 Features

- **Scheduled execution** every 10 minutes
- **Issue fetching** from GitHub via timestamp stored in Supabase
- **Text summarization**, **sentiment analysis**, and **tag generation** via OpenAI (GPT-3.5-Turbo)
- **Role classification** based on extracted context
- **Routing** of issues to:
  - `#engineering`
  - `#sales`
  - `#support`
- **Storage** of processed issues in Supabase
- **Slack notifications** sent via webhook to appropriate channels

---

## 🧩 Workflow Breakdown

### 1. Trigger

- The workflow is triggered every 10 minutes using `Schedule Trigger`.

---

### 2. Metadata Handling

- `Get new last_run_time`: Generates the ISO timestamp for the current run - this timestamp is generated at the beginning of the workflow, and used to updated the last_run_time value stored in Supabase ONLY at the end of a successful run. Since the workflow involves potentially lengthy operations (ie. llms calls, including recursive summarization chains), this guarantees that issues posted during the runtime of a workflow are not lost.
- `Get Last Run Time from Supabase`: Fetches previous run's timestamp
- `Fetch Github Issues since last run time`: Pulls GitHub issues created since that timestamp

---

### 3. Preprocessing

- `Parse relevant info from issues`: Extracts `title`, `body`, `url`, `user login`, and `labels`
- `Get body`: Extracts issue body (used for summarization)

---

### 4. AI Enrichment

- `Summarization Chain`: Produces a one-sentence summary using n8n's built-in OpenAI Summarization Chain (recursive summarization is used for text longer than 1000 characters).
- `Sentiment Analysis`: Determines sentiment (`positive`, `negative`, `neutral`)
- `Generate Dynamic Tags`: Adds contextual tags based on content (these are different from the actual 'labels' set by the creator of the Github issue)
- `Extract Category and Assigned Role`: Categorizes the issue (possible values: `bug`, `feature request`, `question`) and assigns a team to handle it (possible values: `engineering`, `sales`, `support`)

All AI logic is done using GPT-3.5-Turbo for personal cost-efficiency, feel free to switch up to a bigger model for better output quality.
All custom AI chains are equipped with an 'Output Parser' to enforce output data typing for robustness.

---

### 5. Slack Routing (via `Switch`)

- A `Switch` node branches the flow into `sales`, `support`, or `engineering` based on the assigned role
- Each branch sets a Slack webhook URL using a `Set` node
- All branches are merged back using `Merge` node
- A single `Send Slack message` node uses expressions and `[$itemIndex]` to ensure Slack posts match the original issue data

---

### 6. Storage & Metadata Update

- `Parse final data for Supabase`: Aggregates all enriched info and prepares data to be posted into Supabase
- `Add Issues to Supabase`: Inserts records into the `enriched_issues` Supabase table
- `Update last_run_time in Supabase`: Updates workflow metadata with the new `last_run_time` to be used for the next workflow run

---

## ⚠️ Error Handling

Error handling was not implemented in this version. In a production system, this would ideally include:

- An **Error Trigger** node (set-up in another linked workflow) which sends Slack notifications to an internal `#errors` channel
- The error payload would include:
  - Node name
  - Error message
- Future improvements could add retry logic and better failover strategies

---

## 🔐 Secrets / Credentials

This workflow assumes the following credentials are set up in the n8n UI:

- GitHub API
- Supabase API
- OpenAI API

---

## ⚙️ Setup

This workflow assumes the following tables and schemas exist in your Supabase project:

```sql
-- Table: enriched_issues
CREATE TABLE enriched_issues (
  id              BIGINT       PRIMARY KEY GENERATED ALWAYS AS IDENTITY,
  created_at      TIMESTAMPTZ  DEFAULT NOW(),
  title           TEXT,
  creator_login   TEXT,
  category        TEXT,         -- e.g., 'bug', 'feature request', 'question'
  summary         TEXT,         -- AI-generated summary
  assigned_role   TEXT,         -- 'engineering', 'sales', 'support'
  sentiment       TEXT,         -- 'positive', 'negative', 'neutral'
  tags            TEXT[],       -- Array of dynamic tags like ['ui', 'api', 'critical']
  issue_url       TEXT UNIQUE   -- GitHub issue URL
);

-- Table: workflow_metadata
CREATE TABLE workflow_metadata (
  id     BIGINT  PRIMARY KEY GENERATED ALWAYS AS IDENTITY,
  key    TEXT    UNIQUE,        -- e.g., 'last_run_time'
  value  TEXT                   -- e.g., ISO timestamp string
);
```

The workflow also assumes that the workflow_metadata table is prepopulated with a starting time for last_run_time. An example query for your convenience:

```sql
INSERT INTO workflow_metadata (key, value)
VALUES ('last_run_time', '2025-01-01T00:00:00Z');
```

This workflow assumes a Slack webhook bot is setup for your organization and has access to post messages.
More info: https://api.slack.com/messaging/webhooks


---

## 📝 Notes

- This workflow intentionally avoids using n8n's built-in Slack node due to OAuth constraints. Instead, direct webhook calls are made via `HTTP Request` nodes.
- The Slack webhook URLs are stored in `Set` nodes rather than hardcoded into code blocks for better visual organization.
- Slack routing avoids duplication by merging streams before the message is sent.

---

## 🚧 Limitations / TODOs

- No catch-all error handler is currently set up
- Message formatting could be richer (Markdown blocks, attachments, etc.)
- The workflow is linear (ie. async i/o nodes wait for each other). This is not ideal as some nodes are independent and do not rely and each other's outputs. In the future, as I get the hang of data merging in n8n, some nodes can be parallelized for faster performance.

---

## 📂 File Structure

This `README.md` assumes the accompanying `workflow.json` is imported directly into an n8n instance via the UI.

