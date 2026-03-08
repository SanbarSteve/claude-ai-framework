# Claude AI Framework — Entity File Interview Tool

> An AI-powered interview tool that builds personalised Claude Entity Files — reusable context documents that make every Claude session dramatically more effective.
>
> **Owner:** SanbarSteve
> **Version:** 1.0
> **Status:** Active Development
> **Last Updated:** March 8, 2026
>
> ---
>
> ## What This Is
>
> The Entity File Interview Tool is a single-file HTML application that conducts a structured AI interview and generates a Claude Entity File — a markdown document you upload to a Claude Project so Claude always knows who you are, how you work, and what you need.
>
> No server. No installation. No build process. Open the HTML file in Chrome, paste your Anthropic API key, and start answering questions.
>
> ---
>
> ## Current Use Cases
>
> | Use Case | Topics | Status |
> |---|---|---|
> | 🧠 Personal Co-Pilot | Identity, Goals, Projects, Work Style, Tools, Knowledge, Constraints, Integrations | ✅ Complete |
> | 📈 Sales & Business | Company, Products, ICP, Pipeline, Process, Competitors, CRM, Revenue Goals | ⏳ In Progress |
> | ⚡ Consumer Apps | Portfolio, Tech Stack, Standards, Active Project, Users, Agents, Memory, APIs | ⏳ In Progress |
> | ➕ Custom Roles | AI-generated topic sets from plain-language role description | 🗺️ Planned |
>
> ---
>
> ## Key Features
>
> - **3-pass interview system** — Initial answers, refinement, polish
> - - **Auto-save** — Progress saved to localStorage, restored on reopen
>   - - **Live preview** — Entity file builds in real time as you answer
>     - - **Direct Anthropic API** — Calls claude-sonnet-4-5 directly from the browser
>       - - **One-click download** — Generates a `.md` file ready to upload to Claude Projects
>         - - **Enter = new line** — Blue arrow button submits answers
>          
>           - ---
>
> ## Files in This Repository
>
> ```
> claude-ai-framework/
> ├── app/
> │   └── entity-interview-v2.html      # The complete application
> ├── docs/
> │   ├── user-instructions.docx        # How to use the tool
> │   ├── technical-documentation.docx  # Architecture & API reference
> │   └── maintenance-log.docx          # Version history & roadmap
> └── README.md
> ```
>
> ---
>
> ## Quick Start
>
> 1. Get an Anthropic API key at [console.anthropic.com](https://console.anthropic.com)
> 2. 2. Download `entity-interview-v2.html`
>    3. 3. Open it in Chrome (double-click)
>       4. 4. Paste your API key in the yellow banner
>          5. 5. Select a use case and start answering
>             6. 6. Click **Generate Entity File** when ready
>                7. 7. Download your `.md` file and upload it to a Claude Project
>                  
>                   8. ---
>                  
>                   9. ## Roadmap
>                  
>                   10. ### Short Term
> - Encrypted localStorage for API key persistence
> - - Session export as `.json`
>   - - Edit mode for collected answers before generation
>    
>     - ### Medium Term — Custom Role Builder
>     - - **+ New Role tab** — define any role in plain language
>       - - **AI-generated topic sets** — Claude creates relevant questions for your role
>         - - Role persistence and export/import as `.json`
>          
>           - ### Long Term
>           - - **Role Library** — curated community-contributed role templates
>             - - Role versioning and diff
>               - - Claude Projects direct upload via API
>                 - - Voice input
>                  
>                   - ---
>
> ## Architecture
>
> Single-file HTML application (~1,200 lines). HTML + CSS + JavaScript in one file.
>
> - **API:** `POST https://api.anthropic.com/v1/messages`
> - - **Model:** `claude-sonnet-4-5`
>   - - **Auth:** User-provided API key, stored in session memory only
>     - - **State:** JavaScript object + localStorage auto-save
>       - - **DATA block protocol:** Claude appends structured JSON to every reply for reliable state parsing
>        
>         - See `docs/technical-documentation.docx` for full architecture details.
>        
>         - ---
>
> ## For Reviewers
>
> This repository is currently **read-only for collaborators**. To submit feedback:
>
> 1. Open an **Issue** using the Issues tab above
> 2. 2. Label it: `bug`, `enhancement`, `question`, or `review-feedback`
>    3. 3. Be as specific as possible — include the use case, what you expected, and what happened
>      
>       4. Pull requests are not being accepted at this stage.
>      
>       5. ---
>      
>       6. ## Related Projects
>
> This tool is part of a broader Claude AI Framework being developed under the Personal Co-Pilot project. Additional tools and entity files for Sales & Business and Consumer Apps contexts are in progress.
>
> ---
>
> *Built with Claude Sonnet 4.5 · Personal Co-Pilot Project · SanbarSteve*
