## 🛡️ Repository Governance & Development Workflows

Repository standards are enforced through **GitHub Actions workflows** to maintain code quality, consistent project structure, automated reviews, and controlled deployment processes.

### 2.1 Automated CI/CD Workflows

| Workflow | Trigger | Actions / Capabilities |
|---|---|---|
| **Auto Format & Commit** | PR opened, synchronized, or reopened targeting `main` or `develop` | Runs `sqlfluff` (Databricks), `black` (120-character limit), `isort`, `nbqa`, and `nbstripout`. Automatically commits formatting changes and posts a PR summary. |
| **Auto Assign Reviewer** | PR opened or marked ready for review | Automatically assigns a reviewer based on the PR author's reviewer mapping. |
| **PR Checks** | PR targeting `main` | Validates the required repository structure (`src/`, `docs/`) and confirms the presence of SQL files. |

### 2.2 Reviewer Assignment Matrix

Peer-review assignments are automatically managed through workflow configuration.

```text
jess-christine
      ↓
catweyine
      ↓
jg0901
      ↓
go-viaaa
      ↓
JoanMaquinano
      ↓
jess-christine
PR Author	Assigned Reviewer
jess-christine	catweyine
catweyine	jg0901
jg0901	go-viaaa
go-viaaa	JoanMaquinano
JoanMaquinano	jess-christine

2.3 Code Standards & Environment Control
Databricks Asset Bundles (DAB): Configuration fixes are maintained under version control to improve deployment stability.
Branch Protection & Gating: PR branch gating is enabled to control changes before merging, with CODEOWNERS assigned to critical paths.
Repository Cleanup: Connection references and operational tracking artifacts such as .gitkeep are maintained and cleaned as needed.
Development Standards: Automated formatting and validation help maintain consistent code and repository structure across contributors.

2.4 Governance Summary

The repository uses automated workflows to establish a consistent development process:

Pull Request
     ↓
Auto Formatting
     ↓
PR Checks
     ↓
Reviewer Assignment
     ↓
Code Review
     ↓
Branch Gating
     ↓
Merge

Goal: Automate repetitive checks and formatting while enforcing consistent review, repository structure, and deployment practices.
