# 001. Documentation Index Architecture

**Status:** Accepted

**Date:** 2025-10-17

**Deciders:** Architecture Team, Documentation Team

---

## Context

As the Escher V2 project grew to 60+ files and 48,000+ lines of documentation across 8 main sections, we identified several challenges:

1. **No single entry point**: Developers and stakeholders struggled to find relevant documentation quickly
2. **Unclear audience**: Documentation mixed product specifications with developer implementation guides
3. **Inconsistent organization**: Some sections had detailed overviews, others had minimal navigation
4. **AI code generator needs**: Teams using Claude Code, Cursor, and GitHub Copilot needed structured prompts for code generation
5. **Documentation sprawl**: Multiple similar documents (PROJECT-SUMMARY.md, STRUCTURE.md, 02-client/INDEX.md) with overlapping purposes but different audiences

We needed a clear, well-organized entry point that serves multiple audiences (product managers, architects, developers, AI code generators) while maintaining a specification-first focus.

---

## Decision

We created `docs/INDEX.md` as the primary entry point for product and architecture specification documentation, with the following architectural decisions:

### Decision 1: Target Length (400-500 lines)
- **Selected**: 400-500 lines (moderate detail)
- **Rationale**: Sweet spot between comprehensive context and concise navigation
- **Result**: 479 lines in final implementation

### Decision 2: Detail Level (Summary + Links)
- **Selected**: Summary in INDEX, full details in section READMEs
- **Approach**: Entry point link + high-level "what is this section" + 3-5 key topics
- **Rationale**: Avoids duplication while providing enough context to decide where to dive deeper

### Decision 3: Audience Balance (70% / 15% / 15%)
- **Structure**:
  - 70% Product & Architecture Specification
  - 15% AI & Code Generators
  - 15% Developer Navigation
- **Rationale**: Primary purpose is specification, not developer docs (that's 02-client/INDEX.md)

### Decision 4: Key Architecture Concepts (6 concepts)
- **Selected Concepts**:
  1. Client-Server Separation (moved to #1 - most fundamental)
  2. Privacy-First Design
  3. Multi-Cloud Architecture (AWS/Azure/GCP)
  4. 6 RAG Collections (Local Qdrant)
  5. Server-Driven UI
  6. Playbook Intelligence (LLM + RAG)
- **Rationale**: "Client-Server Separation" is the most critical principle (no credentials leave client, execution happens locally)

### Decision 5: Tables vs Prose (70% / 30%)
- **Selected**: Table-heavy format (70% tables, 30% prose)
- **Rationale**: Specification index needs to be scannable; tables provide structure

### Decision 6: ASCII Diagrams (Flexible)
- **Selected**: Add diagrams where helpful, no hard rule
- **Result**: 4-6 diagrams total across the document
- **Rationale**: Some concepts benefit from visual representation, others don't need it

### Decision 7: Status Indicators (Emojis)
- **Selected**: Emojis (✅ Complete, 🔄 In Progress, 📋 TBD)
- **Rationale**: Quick visual scanning, consistent with existing docs, markdown-friendly

---

## Rationale

### Why These Decisions Matter

**1. Specification-First Approach (70%)**
- Escher is a complex system with client-server separation, multi-cloud support, and privacy-first architecture
- Stakeholders and architects need to understand "what we're building" before "how to implement it"
- Developer-specific guides already exist in 02-client/INDEX.md

**2. AI Code Generator Section (15%)**
- Modern development teams use AI assistants (Claude Code, Cursor, GitHub Copilot)
- Different modules require different prompts (Frontend React vs Backend Rust vs Server Go)
- Copy-paste-ready prompts enable rapid, standards-compliant code generation
- Strategic decision to support AI-augmented development

**3. Client-Server Separation as #1 Concept**
- This is THE defining principle of Escher's architecture
- Most common mistakes stem from violating this principle (sending credentials, executing on server, etc.)
- Making it #1 ensures it's never overlooked

**4. Multiple Module-Specific AI Prompts**
- Initially considered one universal prompt
- Realized different teams have different needs:
  - Frontend: React, TypeScript, MVC, Zustand
  - Backend Rust: Tauri, async, traits, IPC
  - Server Go: Stateless, LLM integration, agents
- Module-specific prompts provide better context and standards

---

## Consequences

### Positive

1. **Single Entry Point**: Clear navigation hub for all documentation
2. **Audience Clarity**: Different paths for different audiences (PM vs architect vs developer vs AI)
3. **Reduced Duplication**: Summary + links approach avoids repeating content from other docs
4. **AI Support**: Teams can use AI code generators effectively with module-specific prompts
5. **Scalability**: Easy to add new sections (09-xxx, 10-xxx) following the same pattern
6. **Scannability**: Table-heavy format makes it easy to find information quickly
7. **Status Visibility**: Clear indicators show what's complete vs in progress

### Negative

1. **Maintenance Burden**: docs/INDEX.md must be updated when section structure changes
2. **Length Trade-off**: 479 lines is substantial (though within target 400-500)
3. **Duplication Risk**: Need discipline to keep summaries in INDEX, details in section READMEs
4. **AI Prompt Creation**: 6 prompts needed (~3,000-4,000 lines total) - significant effort
5. **Multiple Entry Points**: Now have 4 different "index" documents (INDEX.md, PROJECT-SUMMARY.md, STRUCTURE.md, 02-client/INDEX.md) - could be confusing

### Neutral

1. **Emoji Usage**: Some prefer text indicators, but emojis are established pattern in this project
2. **70/15/15 Split**: Could shift to 60/20/20 or 80/10/10 - chosen balance works but isn't objectively "correct"
3. **Table-Heavy Format**: Trades narrative flow for scannability - neither approach is universally better

---

## Alternatives Considered

### Alternative 1: Developer-First Index
- **Description**: Organize by developer role (Frontend, Backend, Server, etc.) like 02-client/INDEX.md
- **Pros**:
  - Easier for developers to find implementation guides
  - Matches mental model of "I'm a frontend dev, where do I start?"
- **Cons**:
  - Mixes specification with implementation
  - Doesn't serve product managers and architects well
  - We already have 02-client/INDEX.md for this purpose
- **Why Rejected**: Duplicate of 02-client/INDEX.md, doesn't solve the spec documentation gap

### Alternative 2: Ultra-Concise Index (200-300 lines)
- **Description**: Minimal summaries, mostly just links
- **Pros**:
  - Quick to scan
  - Low maintenance burden
  - Forces users to dive into section docs
- **Cons**:
  - Insufficient context to decide where to go
  - Users waste time exploring wrong sections
  - Doesn't establish critical architecture concepts
- **Why Rejected**: Too minimal - users need enough context to navigate effectively

### Alternative 3: Comprehensive Narrative (800-1000 lines)
- **Description**: Detailed explanations like PROJECT-SUMMARY.md, covering all aspects
- **Pros**:
  - Self-contained - users don't need to read other docs
  - Complete picture in one place
- **Cons**:
  - Massive duplication with PROJECT-SUMMARY.md
  - Too long to be an "index"
  - Hard to maintain (changes must be replicated)
- **Why Rejected**: We already have PROJECT-SUMMARY.md (998 lines) for this purpose

### Alternative 4: Single Universal AI Prompt
- **Description**: One comprehensive prompt that works for all modules
- **Pros**:
  - Single source of truth
  - Easier to maintain
  - Consistent standards across modules
- **Cons**:
  - 1,500-2,000 lines long (overwhelming)
  - Generic guidance doesn't help with module-specific patterns
  - Frontend React patterns irrelevant to Server Go developers
- **Why Rejected**: Different teams need different context - module-specific prompts provide better value

### Alternative 5: No AI Section at All
- **Description**: Focus purely on specification and developer navigation
- **Pros**:
  - Simpler structure (70% spec / 30% dev)
  - Less maintenance burden
  - No need to create 6 AI prompts
- **Cons**:
  - Teams already using AI code generators extensively
  - Missing strategic opportunity to enable AI-augmented development
  - Would need to document AI prompts elsewhere anyway
- **Why Rejected**: AI code generation is a reality of modern development - better to support it explicitly

---

## Implementation Notes

### File Structure Created

```
/docs/INDEX.md (479 lines)
├─ Introduction (5%)
├─ System Architecture (10%)
├─ Product Specifications (55%)
│  ├─ 01-overview/
│  ├─ 02-client/
│  ├─ 03-server/
│  ├─ 04-services/
│  ├─ 05-flows/
│  ├─ 06-security/
│  ├─ 07-data/
│  └─ 08-operations/
├─ Key Architecture Concepts (15%)
│  ├─ 1. Client-Server Separation
│  ├─ 2. Privacy-First Design
│  ├─ 3. Multi-Cloud Architecture
│  ├─ 4. 6 RAG Collections
│  ├─ 5. Server-Driven UI
│  └─ 6. Playbook Intelligence
├─ Documentation Status (5%)
├─ For AI & Code Generators (15%)
│  └─ Links to /working-docs/ai-prompts/ (6 prompts planned)
└─ For Developers (10%)
```

### Key Architectural Principles Emphasized

**Client-Server Separation** (Most Critical):
```
CLIENT                          SERVER
├─ Owns ALL data                ├─ 100% stateless
├─ Credentials (NEVER sent)     ├─ Stores NOTHING
├─ Executes operations          ├─ Provides intelligence
└─ State + Executor             └─ Brain (no memory)
```

**Rules Enforced**:
- ❌ NO credentials ever leave the client
- ❌ NO estate data sent to server (only minimal summaries)
- ✅ ALL cloud operations execute on client
- ✅ Server provides intelligence, client executes

### Documentation Standards Established

1. **Status Indicators**: ✅ Complete, 🔄 In Progress, 📋 TBD
2. **Multi-Cloud Terminology**: "cloud (AWS/Azure/GCP)", NOT "AWS-only"
3. **6 RAG Collections**: Explicitly documented (not "dual collection")
4. **Table Format**: Use tables for structured data (file listings, component specs, status)
5. **ASCII Diagrams**: Add where helpful for visualizing architecture concepts

### Future Maintenance

**When adding new sections** (09-xxx, 10-xxx):
1. Create section folder with README.md
2. Add entry in docs/INDEX.md Product Specifications section
3. Follow table format: | Entry Point | Status | Key Topics |
4. Keep summary to 3-5 key topics maximum
5. Link to section README for full details

**When updating existing sections**:
1. Update section README first (source of truth)
2. Update summary in docs/INDEX.md if high-level structure changed
3. Keep status indicator current (✅ 🔄 📋)

---

## References

- **Implementation**: [docs/INDEX.md](../docs/INDEX.md) - The actual index (479 lines)
- **Planning Document**: [working-docs/DOCS-INDEX-PLAN.md](../working-docs/DOCS-INDEX-PLAN.md) - Complete planning process (740 lines)
- **Related Documents**:
  - [PROJECT-SUMMARY.md](../PROJECT-SUMMARY.md) - Comprehensive narrative (998 lines)
  - [STRUCTURE.md](../STRUCTURE.md) - File organization reference
  - [docs/02-client/INDEX.md](../docs/02-client/INDEX.md) - Developer-first navigation
  - [CLAUDE.md](../CLAUDE.md) - Development guidance and coding standards
- **Commit**: `8f2c2a1` - "docs: add main documentation index and planning document"
- **Related ADRs**: None (this is ADR 001)

---

## Future Work

### Phase 2: AI Implementation Prompts (Planned)

**Location**: `/working-docs/ai-prompts/`

**6 Module-Specific Prompts Needed**:

| Prompt | Audience | Lines | Priority |
|--------|----------|-------|----------|
| AI-PROMPT-FRONTEND.md | Frontend (React/TypeScript) | ~500-700 | High |
| AI-PROMPT-BACKEND-RUST.md | Backend Rust (Tauri) | ~500-700 | High |
| AI-PROMPT-SERVER-GO.md | Server Go (AI Agents) | ~500-700 | High |
| AI-PROMPT-STORAGE-SERVICE.md | Storage (Qdrant/RAG) | ~400-600 | Medium |
| AI-PROMPT-EXECUTION-ENGINE.md | Execution (Tokio/Streaming) | ~400-600 | Medium |
| AI-PROMPT-ESTATE-SCANNER.md | Scanner (Multi-cloud) | ~400-600 | Medium |

**Each Prompt Contains**:
- Project context and architecture principles (10%)
- Module-specific context (10%)
- Code standards for that language/framework (25%)
- Testing requirements (unit + integration) (15%)
- Security checklist (10%)
- Documentation requirements (10%)
- Working examples and patterns (15%)
- Pre-implementation checklist (5%)

**Total Effort**: ~3,000-4,000 lines across 6 prompts

---

**Last Updated**: 2025-10-17
**Status**: Implemented and Accepted