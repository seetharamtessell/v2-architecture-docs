# Contributing to Escher Documentation

**Welcome!** This guide helps you contribute to Escher's documentation effectively.

---

## Table of Contents

1. [Documentation Organization](#documentation-organization)
2. [Adding New Sections](#adding-new-sections)
3. [Maintaining docs/INDEX.md](#maintaining-docsindexmd)
4. [Documentation Standards](#documentation-standards)
5. [Style Guide](#style-guide)
6. [Review Process](#review-process)

---

## Documentation Organization

### Overview

Escher documentation is organized into **8 main sections** (01-08) plus supporting files:

```
/docs/
├── INDEX.md                    # Main entry point (product specification)
├── 01-overview/                # Product vision & architecture
├── 02-client/                  # Client application specification
│   └── INDEX.md                # Client-specific navigation (developer-first)
├── 03-server/                  # Server system specification
├── 04-services/                # Shared Rust crates specification
├── 05-flows/                   # System flows & integration
├── 06-security/                # Security & privacy model
├── 07-data/                    # Data architecture & models
├── 08-operations/              # Deployment & operations
└── meta/                       # Documentation metadata
    └── CONTRIBUTING.md         # This file
```

### Key Documents

| Document | Purpose | Audience |
|----------|---------|----------|
| [INDEX.md](../INDEX.md) | Product specification index (70% spec / 15% AI / 15% dev) | PM, architects, developers, AI |
| [PROJECT-SUMMARY.md](../../PROJECT-SUMMARY.md) | Comprehensive narrative (998 lines) | All audiences (deep dive) |
| [STRUCTURE.md](../../STRUCTURE.md) | File organization reference | Contributors |
| [CLAUDE.md](../../CLAUDE.md) | Development guidance & coding standards | Developers, AI assistants |
| [02-client/INDEX.md](../02-client/INDEX.md) | Developer-first navigation | Frontend/backend developers |

**Key Principle**: Each document has a distinct purpose - avoid duplication!

---

## Adding New Sections

### When to Add a New Section

Add a new section (09-xxx, 10-xxx) when:
- Content doesn't fit in existing sections (01-08)
- Topic warrants its own folder with multiple files
- Content is substantial (3+ files, 1,000+ lines)

**Don't create a new section** for:
- Single-file documentation (add to existing section)
- Temporary working docs (use `/working-docs/`)
- Implementation details (those go in code repos)

### Steps to Add a New Section

#### 1. Create Section Folder

```bash
mkdir docs/09-your-section-name
cd docs/09-your-section-name
```

#### 2. Create README.md (Entry Point)

Every section MUST have a README.md that serves as its entry point.

**Template**:

```markdown
# [Section Name]

**Purpose**: Brief description (1-2 sentences)

**Status**: ✅ Complete | 🔄 In Progress | 📋 TBD

---

## Overview

2-3 paragraphs explaining what this section covers.

## Key Topics

- Topic 1
- Topic 2
- Topic 3

## Files in This Section

| File | Purpose | Status |
|------|---------|--------|
| [file1.md](./file1.md) | Description | ✅ Complete |
| [file2.md](./file2.md) | Description | 🔄 In Progress |

## Related Sections

- [Related Section 1](../01-overview/)
- [Related Section 2](../02-client/)

---

**Last Updated**: YYYY-MM-DD
```

#### 3. Update docs/INDEX.md

Add your section to the **Product Specifications** section:

```markdown
### 09-your-section-name/ - Brief Title

**Entry Point**: [09-your-section-name/README.md](./09-your-section-name/README.md)

**Key Topics**:
- Topic 1 (keep to 3-5 bullet points)
- Topic 2
- Topic 3

**Status**: 🔄 In Progress
```

**Format Rules**:
- Use table format if showing file listings
- Keep to 3-5 key topics maximum
- Include entry point link
- Add status indicator (✅ 🔄 📋)

#### 4. Update STRUCTURE.md

Add your section to `/STRUCTURE.md`:

```markdown
### 09-your-section-name/
Brief description

**Key Files**:
- [README.md](docs/09-your-section-name/README.md) - Entry point
- [file1.md](docs/09-your-section-name/file1.md) - Description
```

#### 5. Create ADR (If Significant)

If your section represents a significant architectural decision, create an ADR:

```bash
# Follow the template
cp adr/template.md adr/00X-your-decision-name.md
```

See [ADR README](../../adr/README.md) for guidance.

---

## Maintaining docs/INDEX.md

### When to Update docs/INDEX.md

Update `docs/INDEX.md` when:
- ✅ Adding or removing sections (01-08, 09-xxx, etc.)
- ✅ Section status changes (🔄 → ✅ or ✅ → 🔄)
- ✅ High-level structure of a section changes
- ✅ Key topics list changes significantly
- ❌ NOT for minor edits within a section (update section README instead)

### How to Update

#### Updating Section Status

```markdown
**Status**: ✅ 90% Complete (~25,000+ lines)
```

Change to:

```markdown
**Status**: ✅ Complete (~27,000+ lines)
```

#### Adding/Updating Key Topics

Keep to **3-5 topics maximum**:

```markdown
**Key Topics**:
- Two deployment models (Laptop-only, Extend to Cloud)
- Target users (DevOps engineers, SREs, platform teams)
- Multi-cloud support strategy (AWS/Azure/GCP)
```

#### Updating Documentation Status Section

When sections reach milestones:

```markdown
### ✅ Complete (90%)
- Product Vision (3 files, ~1,500 lines)
- Client Specification (40+ files, ~25,000 lines)
- Shared Services (28 files, ~13,650 lines)
- Server: Playbook Agent & UI Agent (~6,800 lines)
- PROJECT-SUMMARY (~998 lines)

**Total**: 60+ files, 48,000+ lines documented
```

Update file counts and line counts as they change.

---

## Documentation Standards

### Status Indicators

Use emojis for status (consistent with existing docs):

- ✅ **Complete** - Section is finished, reviewed, and accurate
- 🔄 **In Progress** - Section is being actively developed
- 📋 **TBD** / **Planned** - Section is planned but not started

**Usage**:
```markdown
**Status**: ✅ Complete
**Status**: 🔄 In Progress (60% complete)
**Status**: 📋 TBD (planned for Q2)
```

### Multi-Cloud Terminology

**Always use cloud-agnostic terminology**:

| ❌ Wrong | ✅ Correct |
|---------|-----------|
| "AWS estate" | "cloud estate (AWS/Azure/GCP)" |
| "AWS resources" | "cloud resources (AWS/Azure/GCP)" |
| "AWS CLI" | "cloud CLI (AWS/Azure/GCP)" |
| "AWS-only" | "multi-cloud (AWS/Azure/GCP)" |

**Why**: Escher supports AWS, Azure, and GCP - never imply AWS-only.

### Architecture Principles (Always Emphasize)

When documenting, always emphasize these **6 core principles**:

1. **Client-Server Separation** (Most Critical)
   - No credentials leave client (ever)
   - Execution happens on client
   - Server is brain, client is executor

2. **Privacy-First Design**
   - Client owns ALL data
   - Server stores NOTHING

3. **Multi-Cloud Architecture**
   - AWS/Azure/GCP unified experience

4. **6 RAG Collections**
   - NOT "dual collection" (that's wrong)
   - 2 with real 384D vectors, 4 with dummy 1D vectors

5. **Server-Driven UI**
   - Dynamic component rendering

6. **Playbook Intelligence**
   - LLM + RAG 4-step flow

### Tables vs Prose

**Use tables for** (70% of content):
- File listings
- Component specifications
- Status tracking
- Comparison of options
- Structured data

**Use prose for** (30% of content):
- Introductions and context
- Explanations and rationale
- Transition between sections
- Narratives and flows

**Example**:

```markdown
## Overview

The Storage Service handles all local storage operations (prose - context).

## Collections

| Collection | Vectors | Purpose |
|-----------|---------|---------|
| Cloud Estate | 384D | Semantic search + IAM permissions |
| Chat History | 1D | Filter-based access |

(Table - structured data)
```

### ASCII Diagrams

Use ASCII diagrams where they clarify architecture concepts:

```markdown
┌─────────────────┐
│  Client (Tauri) │
│  ├─ React UI    │
│  ├─ Rust Engine │
│  └─ Local DB    │
└─────────────────┘
       ↕
┌─────────────────┐
│  Server (Go)    │
│  ├─ AI Agents   │
│  └─ Playbooks   │
└─────────────────┘
```

**Guidelines**:
- Use for component relationships
- Use for data flows
- Keep diagrams simple (< 20 lines)
- Don't over-use (4-6 per document max)

### Links

**Relative Links**:
- Use relative paths: `[Client](./02-client/INDEX.md)`
- NOT absolute paths: `[Client](/docs/02-client/INDEX.md)`

**Markdown Format**:
- Use markdown links: `[text](link)`
- NOT HTML: `<a href="link">text</a>`
- NOT backticks for links: `` `link` ``

**External Links**:
- Include full URL: `[GitHub](https://github.com/user/repo)`

---

## Style Guide

### Tone

- **Professional** but accessible
- **Specification-focused** (what/how, not detailed implementation)
- **Architecture-first** (explain design, then point to details)
- **Clear and concise** (avoid unnecessary jargon)

### Voice

- Use **active voice**: "The client executes operations locally"
- NOT passive voice: "Operations are executed locally by the client"

### Paragraphs

- Keep paragraphs short (2-4 sentences max)
- One idea per paragraph
- Use bullet points for lists

### Examples

**Good Example** (concise, clear, active voice):

```markdown
## Storage Service

The Storage Service manages all local data using an embedded Qdrant vector database. It provides 6 specialized collections for different data types. All data is encrypted at rest using AES-256-GCM.
```

**Bad Example** (verbose, passive voice, too much detail):

```markdown
## Storage Service

The storage service that is being used in our system is responsible for managing all of the various types of data that need to be stored locally. This is accomplished through the utilization of an embedded instance of the Qdrant vector database system. There are six different collections that have been created, each of which serves a specific purpose in terms of the type of data that is stored within them. The encryption that is applied to the data is performed using the AES-256-GCM algorithm.
```

### Headings

- Use **sentence case** for headings: "Adding new sections"
- NOT title case: "Adding New Sections"
- Exception: Proper nouns and acronyms: "Using Qdrant for RAG"

### Code Blocks

Always specify language for syntax highlighting:

```typescript
// ✅ Good
const example: string = "hello";
```

```
// ❌ Bad (no language specified)
const example: string = "hello";
```

### Lists

**Unordered lists**: Use `-` (not `*` or `+`)

```markdown
- Item 1
- Item 2
- Item 3
```

**Ordered lists**: Use `1.` for all items (Markdown auto-numbers)

```markdown
1. Step 1
1. Step 2
1. Step 3
```

### Emphasis

- **Bold** for emphasis: `**important**`
- *Italic* for terms: `*term*`
- `Code` for technical terms: `` `function_name` ``

---

## Review Process

### Before Submitting

**Self-Review Checklist**:

- [ ] Followed documentation standards (status indicators, multi-cloud terminology, etc.)
- [ ] Updated docs/INDEX.md if adding/changing sections
- [ ] Updated STRUCTURE.md if adding new files
- [ ] Checked all links work (use relative paths)
- [ ] Used tables for structured data (70% of content)
- [ ] Used consistent status indicators (✅ 🔄 📋)
- [ ] Emphasized architecture principles where relevant
- [ ] Kept paragraphs short (2-4 sentences)
- [ ] Specified language for code blocks
- [ ] No spelling or grammar errors

### Creating Pull Requests

**Title Format**:
```
docs: [section] brief description

Examples:
docs: add 09-monitoring section
docs(client): update MVC architecture details
docs: fix broken links in 03-server/
```

**Description**:
```markdown
## What Changed
- Added 09-monitoring section with 3 files
- Updated docs/INDEX.md with new section entry
- Updated STRUCTURE.md

## Why
Monitoring documentation was missing and needed for ops team.

## Checklist
- [x] Updated docs/INDEX.md
- [x] Updated STRUCTURE.md
- [x] All links checked and working
- [x] Follows documentation standards
```

### Review Criteria

Reviewers will check:

1. **Accuracy**: Technical information is correct
2. **Completeness**: All necessary context provided
3. **Consistency**: Follows existing patterns and standards
4. **Clarity**: Easy to understand for target audience
5. **Links**: All links work correctly (relative paths)
6. **Standards**: Follows style guide and documentation standards

---

## Common Mistakes to Avoid

### ❌ Mistake 1: AWS-Only Terminology
```markdown
❌ "Store AWS resources in Qdrant"
✅ "Store cloud resources (AWS/Azure/GCP) in Qdrant"
```

### ❌ Mistake 2: Wrong Collection Count
```markdown
❌ "dual collection strategy"
✅ "6 RAG collections (2 with 384D vectors, 4 with 1D vectors)"
```

### ❌ Mistake 3: Violating Client-Server Separation
```markdown
❌ "Server executes cloud operations"
✅ "Client executes cloud operations locally; server provides intelligence"
```

### ❌ Mistake 4: Absolute Paths
```markdown
❌ [Client](/docs/02-client/INDEX.md)
✅ [Client](../02-client/INDEX.md)
```

### ❌ Mistake 5: Missing Status Indicators
```markdown
❌ Status: Complete
✅ **Status**: ✅ Complete
```

### ❌ Mistake 6: Too Much Detail in INDEX.md
```markdown
❌ Full file listings in docs/INDEX.md
✅ "See [section README](./0X-section/README.md) for file listings"
```

---

## Quick Reference

### Add New Section
```bash
# 1. Create folder
mkdir docs/09-new-section

# 2. Create README.md (use template above)
# 3. Update docs/INDEX.md (add section entry)
# 4. Update STRUCTURE.md (add section reference)
# 5. Create ADR if significant decision (optional)
```

### Update Existing Section
```bash
# 1. Update section files
# 2. Update section README.md if structure changed
# 3. Update docs/INDEX.md if high-level topics changed
# 4. Update STRUCTURE.md if files added/removed
```

### Fix Broken Links
```bash
# Use relative paths from current file location
../other-section/file.md      # Go up one level
./current-section/file.md     # Stay in current level
../../top-level-file.md       # Go up two levels
```

---

## Need Help?

- **General Questions**: See [PROJECT-SUMMARY.md](../../PROJECT-SUMMARY.md)
- **Documentation Organization**: See [STRUCTURE.md](../../STRUCTURE.md)
- **Development Standards**: See [CLAUDE.md](../../CLAUDE.md)
- **Architecture Decisions**: See [/adr/](../../adr/)
- **Recent Changes**: Review [/working-docs/](../../working-docs/)

---

**Last Updated**: 2025-10-17
**Maintained By**: Documentation Team