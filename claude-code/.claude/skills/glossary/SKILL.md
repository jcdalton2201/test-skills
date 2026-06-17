---
name: glossary
description: |
  The `glossary` skill manages canonical domain terminology in GLOSSARY.md files. Use when: defining new terms, checking for term conflicts, searching existing definitions, or ensuring consistent language across BRDs, stories, and code. Supports single-context repos (one glossary) and multi-context repos (per-folder glossaries).
---

# glossary: Implementation Guide

## Overview

The `glossary` skill manages canonical domain terminology to ensure consistent language across:
- BRDs (Business Requirements Documents)
- User stories and tasks
- Code and documentation
- Team communication

**Key Principle:** Use terms meaningful to domain experts, not implementation details.

---

## Invocation

```text
/glossary list                     → Show all terms
/glossary search "cancel"          → Find related terms
/glossary add "Refund"             → Add new term interactively
/glossary check CARD-101           → Check Tracker story for term issues
/glossary conflicts                → Show unresolved conflicts
/glossary define "Cancellation"    → Get definition of a term
```

---

## GLOSSARY.md Location

### Single-Context Repos (Default)

One `GLOSSARY.md` at the repo root:

```
/
├── GLOSSARY.md           ← all domain terms
├── src/
└── docs/
```

### Multi-Context Repos

Per-folder glossaries with shared root glossary:

```
/
├── GLOSSARY.md                    ← terms shared across project
└── src/
    ├── ordering/
    │   └── GLOSSARY.md            ← ordering-specific terms
    ├── billing/
    │   └── GLOSSARY.md            ← billing-specific terms
    └── shipping/
        └── GLOSSARY.md            ← shipping-specific terms
```

### Detecting Multi-Context

Look for these signals:
- Domain folders: `src/ordering/`, `src/billing/`, `services/payments/`
- Existing `CONTEXT-MAP.md` file
- Multiple `package.json` or `pom.xml` (microservices)
- Clear bounded context structure

**Default:** Single context. Prompt if multi-context signals detected:

```
Skill: I see domain folders: ordering/, billing/, shipping/
       Should we use per-folder glossaries, or one shared glossary?
```

---

## Glossary Entry Format

```markdown
## <Term>

**Definition:** <one-sentence definition meaningful to a domain expert>

**Examples:** <concrete example, optional>

**Not to be confused with:** <neighboring term, optional>

**See also:** <related term, optional>
```

### Rules

1. **Domain terms only** - Include terms meaningful to domain experts
2. **No implementation details** - Skip class names, table names, internal jargon
3. **Nouns and noun phrases** - Terms name concepts, not actions or features
4. **Alphabetical order** - Sort entries within each glossary file
5. **Track renames** - When renaming a fuzzy term, add old name under "Not to be confused with"

### Example Entry

```markdown
## Cancellation

**Definition:** The act of voiding a paid order before it has shipped, triggering a full refund to the customer's original payment method.

**Examples:** Customer calls support to cancel order #12345 before dispatch.

**Not to be confused with:** Cart Removal (removing unpurchased items), Refund (partial return after delivery)

**See also:** Refund, Order Status
```

---

## Core Operations

### 1. List Terms

```text
User: /glossary list

Skill: 📖 **GLOSSARY.md** (12 terms)
       
       | Term | Definition (truncated) |
       |------|------------------------|
       | Cancellation | Voiding a paid order before shipment... |
       | Cart | Collection of items selected for purchase... |
       | Checkout | Process of converting cart to order... |
       | ...
       
       Use `/glossary define "Term"` for full definition.
```

### 2. Search Terms

```text
User: /glossary search "cancel"

Skill: 🔍 Found 2 matches for "cancel":
       
       **Cancellation** (exact match)
       Definition: Voiding a paid order before shipment...
       
       **Cart Removal** (related - see "Not to be confused with")
       Definition: Removing an unpurchased item from cart...
       
       Did you mean one of these, or is this a new term?
```

### 3. Add New Term

```text
User: /glossary add "Chargeback"

Skill: Let's define **Chargeback** for the glossary.
       
       What does "Chargeback" mean in your domain?

User: When a customer disputes a charge with their bank and the bank reverses it

Skill: Got it. Here's the proposed entry:
       
       ## Chargeback
       **Definition:** When a customer disputes a charge with their bank, resulting in a forced reversal of the transaction.
       
       Any examples, related terms, or terms it might be confused with?

User: Related to Refund. Not the same as Cancellation.

Skill: ✅ Added to GLOSSARY.md:
       
       ## Chargeback
       **Definition:** When a customer disputes a charge with their bank, resulting in a forced reversal of the transaction.
       **Not to be confused with:** Cancellation (voluntary), Refund (merchant-initiated)
       **See also:** Refund
```

### 4. Check Story/BRD for Term Issues

```text
User: /glossary check CARD-101

Skill: 📋 Checking CARD-101 against glossary...
       
       ✅ **Known terms used correctly:**
       - Cancellation (2 occurrences)
       - Refund (1 occurrence)
       
       ⚠️ **Potential new terms (not in glossary):**
       - "Void" (used 3x) - Should this be added?
       - "Credit" (used 1x) - Is this different from Refund?
       
       ❌ **Possible conflicts:**
       - "Cancellation" on line 15 seems to mean "removing from cart"
         but glossary defines it as "voiding a paid order"
       
       Want me to help resolve these?
```

### 5. Show Conflicts

```text
User: /glossary conflicts

Skill: ⚠️ **Unresolved Glossary Conflicts**
       
       | Term | Glossary Definition | Conflicting Usage | Source |
       |------|---------------------|-------------------|--------|
       | Cancellation | Voiding paid order | Removing from cart | CARD-101 |
       | Status | Order state | Payment state | BRD-005 |
       
       Use `/glossary resolve "Term"` to fix a conflict.
```

---

## Conflict Detection

### When Conflicts Occur

A conflict is detected when:
1. User uses a glossary term with a different meaning
2. Same term defined differently in multiple context glossaries
3. Term usage contradicts "Not to be confused with" entry

### Handling Conflicts (Soft Warning)

```text
Skill: ⚠️ **Glossary conflict: "Cancellation"**
       
       📖 Glossary says: "Voiding a paid order before shipment"
       📝 Your usage: "Removing an item from the cart"
       
       Options:
       1. **Use existing definition** - I'll rephrase to "Cart Removal"
       2. **Add new term** - Create "Cart Removal" for this meaning
       3. **Update glossary** - Change the definition of "Cancellation"
       4. **Ignore for now** - Continue without resolving
       
       Which approach?
```

---

## Building Glossary Incrementally

### First Term Creates the File

```text
Skill: You used "Cancellation" which isn't defined anywhere.
       
       I'll create GLOSSARY.md with this term.
       
       Suggested definition based on context:
       "Voiding a paid order before it ships, triggering a full refund"
       
       Accept, or provide your own definition?

User: Accept

Skill: ✅ Created GLOSSARY.md with first entry:
       
       ## Cancellation
       **Definition:** Voiding a paid order before it ships, triggering a full refund.
```

### Subsequent Terms Append

New terms are added in alphabetical order within the file.

### Splitting to Multi-Context

When a repo grows to need per-folder glossaries:

```text
User: /glossary split ordering billing

Skill: I'll split GLOSSARY.md into context-specific files:
       
       📁 /GLOSSARY.md (shared terms)
          - Customer, Payment, Address (used across contexts)
       
       📁 /src/ordering/GLOSSARY.md
          - Order, Cart, Checkout, Cancellation
       
       📁 /src/billing/GLOSSARY.md
          - Invoice, Charge, Refund, Chargeback
       
       Terms used in both contexts stay in root glossary.
       Proceed?
```

---

## Integration with Other Skills

This skill provides glossary functionality that other skills use:

| Skill | Glossary Usage |
|-------|----------------|
| `grill-to-brd` | Build glossary during grilling, use in BRD |
| `grill-me` | Build glossary during any grilling session |
| `story-grooming` | Enforce glossary terms in stories/tasks |
| `brd-to-issues` | Use glossary terms in Trakcer issues |
| `story-readiness` | Check for undefined/conflicting terms |

### Shared Functions

```javascript
// Find glossary file(s) for current context
function findGlossaryFiles(repoRoot, currentPath) {
  const files = [];
  
  // Always check root glossary
  const rootGlossary = path.join(repoRoot, 'GLOSSARY.md');
  if (exists(rootGlossary)) files.push(rootGlossary);
  
  // Check for context-specific glossary
  const contextGlossary = findNearestGlossary(currentPath);
  if (contextGlossary && contextGlossary !== rootGlossary) {
    files.push(contextGlossary);
  }
  
  return files;
}

// Parse glossary into term map
function parseGlossary(content) {
  const terms = {};
  const termRegex = /^## (.+)$/gm;
  // ... parse each entry
  return terms;
}

// Check text for glossary term usage
function checkTermUsage(text, glossary) {
  const issues = { conflicts: [], undefined: [], correct: [] };
  // ... analyze text against glossary
  return issues;
}

// Suggest definition based on context
function suggestDefinition(term, contextText) {
  // Use surrounding text to infer meaning
  // Return suggested definition
}
```

---

## Error Handling

| Situation | Response |
|-----------|----------|
| No GLOSSARY.md exists | Offer to create on first term |
| Term not found | Suggest similar terms or offer to add |
| Multiple definitions | Show all, ask which context applies |
| Circular "See also" | Warn and suggest fix |

---

## Testing Checklist

- [ ] Creates GLOSSARY.md on first term
- [ ] Adds terms in alphabetical order
- [ ] Detects multi-context repos correctly
- [ ] Finds conflicts when term usage differs
- [ ] Suggests definitions from context
- [ ] Handles "Not to be confused with" correctly
- [ ] Lists all terms with truncated definitions
- [ ] Searches partial matches and related terms
- [ ] Splits glossary into per-context files
