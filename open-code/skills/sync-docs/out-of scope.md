# sync-docs: Out of Scope

---

## What the Skill Does NOT Do

### ❌ Not a Writing Tool
This skill **automates documentation updates based on code changes**. It does NOT write documentation from scratch.

Use case: Skill updates existing docs when code changes
Not use case: Skill doesn't create docs before code is written

**Team responsibility**: Write initial documentation; skill keeps it updated.

---

### ❌ Not a Design Tool
Skill updates architecture diagrams when code structure changes. It does NOT design architecture.

**Team responsibility**: Architect and design; skill documents the result.

---

### ❌ Not a Manual Editor
Skill auto-updates content between markers. Outside markers, content is preserved as-is.

If you want to manually edit sections:
1. Remove markers (`<!-- AUTO-GENERATED START/END -->`)
2. Skill won't auto-update that section
3. You maintain it manually

**Team responsibility**: Decide which sections are auto-updated vs manual.

---

### ❌ Not Marketing Documentation
Skill focuses on **technical documentation** (README, API docs, deployment, onboarding).

It does NOT create:
- User-facing marketing docs
- Product manuals
- Business documentation
- Customer-facing guides

**Team responsibility**: Create marketing docs separately.

---

### ❌ Not Code Documentation
Skill updates project-level docs (README, architecture, deployment).

It does NOT:
- Generate JavaDoc
- Generate inline code comments
- Update code comments
- Refactor code based on docs

**Team responsibility**: Write code comments; use tools like Javadoc for API documentation.

---

### ❌ Not Decision Logging
Skill updates CHANGELOG with features/fixes.

It does NOT:
- Create Architecture Decision Records (ADRs)
- Log WHY decisions were made
- Document business context
- Record meeting notes

**Team responsibility**: Write ADRs for architectural decisions; use CHANGELOG for release notes.

---

## Team Responsibilities

### Developers
- ✅ Write clear commit messages (conventional commits)
- ✅ Review generated documentation
- ✅ Approve or modify content before merging
- ✅ Maintain manual documentation sections
- ✅ Ensure README has all 7 mandatory sections

### Tech Lead / Architect
- ✅ Define documentation standards
- ✅ Review architecture documentation
- ✅ Ensure API documentation is accurate
- ✅ Approve deployment documentation changes

### CI/CD / DevOps
- ✅ Configure README validation gate (all 7 sections required)
- ✅ Run documentation validation in pipeline
- ✅ Block deployment if README is incomplete

### Product
- ✅ Define which sections are auto-generated vs manual
- ✅ Review CHANGELOG for accuracy
- ✅ Approve release notes before publication

---

## Key Assumptions

### Assumption 1: Git History is Clean
**Assumed**: Commits follow conventional commits (feat:, fix:, docs:, etc.)
**If not true**: Skill may generate incorrect CHANGELOG

**Team responsibility**: Enforce conventional commits in pre-commit hooks or code review.

---

### Assumption 2: Documentation Files Exist
**Assumed**: README.md, application.yml, pom.xml, etc. exist
**If not true**: Skill can create them from templates

**Team responsibility**: Ensure base documentation files are in place.

---

### Assumption 3: Code Structure Follows Standards
**Assumed**: Code organized by layers (Controllers, Processors, Services, DAOs)
**If not true**: Skill may not detect all changes accurately

**Team responsibility**: Follow LANGUAGE-SPRING-REACTIVE.md architecture standards.

---

### Assumption 4: Users Review Before Merging
**Assumed**: Generated docs are reviewed and approved by humans
**If not true**: Bad documentation can slip through

**Team responsibility**: Always review generated content; don't auto-merge.

---

### Assumption 5: CI/CD Enforces README Validation
**Assumed**: Pipeline checks that README has all 7 mandatory sections
**If not true**: Incomplete README can be deployed

**Team responsibility**: Add README validation gate to CI/CD pipeline.

---

## Common Scenarios: Who Decides?

### Q: What if auto-generated content is wrong?
**Answer**: Edit in preview before approving. Or remove markers and maintain manually.

### Q: Can I have sections that are never auto-updated?
**Answer**: Yes. Remove the `<!-- AUTO-GENERATED START/END -->` markers. Skill won't touch them.

### Q: What if a section needs to be 100% manual?
**Answer**: Don't include AUTO-GENERATED markers. Skill will ignore it.

### Q: Does this replace actual writing?
**Answer**: No. Skill keeps docs in sync with code. You still need to write initial documentation.

### Q: What if README validation fails in CI/CD?
**Answer**: Add the missing section or content. Skill can help generate, but CI/CD won't pass until all 7 sections are present and non-empty.

---

## Summary

**What the skill does**: 
- Automatically update documentation when code changes
- Validate README has all 7 mandatory sections
- Preserve user-edited content
- Generate CHANGELOG from commits
- Update API, architecture, deployment, onboarding docs

**What the skill doesn't do**:
- Everything else (write initial docs, design architecture, make decisions, generate code comments)

**The skill is one tool** in documentation maintenance. Use it to keep docs in sync, but don't rely on it for initial documentation creation or major rewrites.
