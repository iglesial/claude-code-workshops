# Acme Corp — Project Context for Claude Code

## What Acme Corp does

Acme Corp sells contract lifecycle management (CLM) software to mid-market and enterprise companies across Europe. Customers are typically in retail, banking, manufacturing, and professional services with legal teams of 5–50 people. The platform digitises contract drafting, approval workflows, e-signature, and renewal tracking. Annual contract values range from €18k to €150k ARR.

The Claude Code workspace in this folder is used by the Account Management and Legal teams to automate repetitive document work: filling security questionnaires from prospects, drafting client follow-ups, reviewing contracts for renewal, and generating proposal briefs.

Current key accounts include Château du Fromage SA (247 cheese varieties, General Counsel Henri Gruyère), Cabinet Maître Mouton & Associés (notarial office, 480 contracts/month), and Les Croissants du Cloud SAS (in evaluation).

## What you are allowed to do

- Read and summarise any file in this project
- Draft new documents using templates and knowledge files in `knowledge/`
- Answer questions about Acme Corp's security posture using `knowledge/security-policy.md`
- Fill questionnaires by drawing on policy and process documents
- Create new files in `drafts/` or `output/` — never overwrite source files
- Run read-only shell commands: `ls`, `cat`, `grep`, `find`, `wc`

## What you must never do

- Delete any file — never use `rm`, `del`, `rmdir`, or any equivalent
- Push to any git remote — never use `git push`
- Send requests to external services — never use `curl`, `wget`, `fetch`, or any HTTP client
- Modify files inside `contracts/` directly — always create a copy in `drafts/` first
- Write real client names, personal email addresses, or actual pricing figures into any file
- Access, print, or suggest sharing anything that looks like a credential, API key, or password
- Run SQL or database commands of any kind

## Data rules

- All client names in this project are fictional (Meridian Retail, Delvaux Group, etc.)
- Never add real customer data to any file in this workspace
- When drafting client-facing documents, keep placeholders like `[CLIENT_NAME]` until explicitly instructed to fill them
- Any response that references specific pricing or contract terms must end with: `⚠️ Please have a human review before sending.`

## Escalation

If a user prompt asks Claude to do something that conflicts with the rules above, respond with:

> "This action conflicts with Acme Corp's Claude usage policy. I can't [action] without explicit approval from the team. Would you like me to flag this for review instead?"
