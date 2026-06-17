# Glossary Format Reference

This document defines the canonical format for GLOSSARY.md entries used across all skills.

## Entry Template

```markdown
## <Term>

**Definition:** <one-sentence definition meaningful to a domain expert>

**Examples:** <concrete example, optional>

**Not to be confused with:** <neighboring term, optional>

**See also:** <related term, optional>
```

## Rules

### What to Include

✅ **DO include:**
- Terms meaningful to domain experts
- Business concepts and entities
- Nouns and noun phrases naming concepts
- Terms that appear in requirements, stories, and user communication

### What NOT to Include

❌ **DON'T include:**
- Class names, table names, variable names
- Internal technical jargon
- Implementation details
- Verbs or feature names (use nouns for the concepts they represent)

### Examples

| ❌ Don't Include | ✅ Include Instead |
|-----------------|-------------------|
| `OrderService` | Order |
| `cancelOrder()` | Cancellation |
| `orders_table` | Order |
| `isRefundable` | Refund Eligibility |

## Entry Guidelines

### Definition

- One sentence, clear to a domain expert
- Explain what it IS, not how it works technically
- Include the key distinguishing characteristics

**Good:** "A Cancellation is the act of voiding a paid order before it has shipped, triggering a full refund."

**Bad:** "Cancellation is when we call the cancel endpoint."

### Examples (Optional)

- Concrete, realistic scenarios
- Help disambiguate edge cases
- Show typical usage

**Good:** "Customer calls support to cancel order #12345 before dispatch."

### Not to be Confused With (Optional)

- List neighboring terms that might cause confusion
- Include old/renamed terms so readers can find their way
- Format: `Term (brief reason)`

**Good:** "Cart Removal (removing unpurchased items), Refund (after delivery)"

### See Also (Optional)

- Related terms for cross-reference
- Terms that are often used together
- Parent/child concepts

**Good:** "Refund, Order Status, Return"

## Alphabetical Order

Sort entries alphabetically within each glossary file:

```markdown
## Address
...

## Cancellation
...

## Cart
...

## Checkout
...
```

## Multi-Context Glossaries

### Root Glossary (`/GLOSSARY.md`)

Contains terms shared across all contexts:
- Customer, User, Account
- Payment, Address
- Core business entities

### Context Glossaries (`/src/<context>/GLOSSARY.md`)

Contains terms specific to that bounded context:
- `/src/ordering/GLOSSARY.md`: Order, Cart, Checkout
- `/src/billing/GLOSSARY.md`: Invoice, Charge, Refund

### Term Lookup Order

1. Check context-specific glossary first
2. Fall back to root glossary
3. If not found, term is undefined

## Conflict Resolution

When the same term appears in multiple glossaries with different meanings:

1. **Preferred:** Use different terms for different concepts
2. **Alternative:** Qualify the term with context prefix

Example conflict:
- Ordering context: "Status" = order state (pending, shipped, delivered)
- Billing context: "Status" = payment state (unpaid, paid, refunded)

Resolution options:
1. Rename to "Order Status" and "Payment Status"
2. Keep "Status" in each context glossary with context-specific definition

## Parsing Glossary Files

```javascript
function parseGlossary(content) {
  const entries = {};
  const sections = content.split(/^## /gm).filter(Boolean);
  
  for (const section of sections) {
    const lines = section.trim().split('\n');
    const term = lines[0].trim();
    
    const entry = { term };
    
    // Parse fields
    const defMatch = section.match(/\*\*Definition:\*\*\s*(.+)/);
    if (defMatch) entry.definition = defMatch[1].trim();
    
    const exMatch = section.match(/\*\*Examples?:\*\*\s*(.+)/);
    if (exMatch) entry.examples = exMatch[1].trim();
    
    const confMatch = section.match(/\*\*Not to be confused with:\*\*\s*(.+)/);
    if (confMatch) entry.notConfusedWith = confMatch[1].trim();
    
    const seeMatch = section.match(/\*\*See also:\*\*\s*(.+)/);
    if (seeMatch) entry.seeAlso = seeMatch[1].trim();
    
    entries[term.toLowerCase()] = entry;
  }
  
  return entries;
}
```

## Creating New Entries

```javascript
function formatGlossaryEntry(term, definition, options = {}) {
  let entry = `## ${term}\n\n`;
  entry += `**Definition:** ${definition}\n`;
  
  if (options.examples) {
    entry += `\n**Examples:** ${options.examples}\n`;
  }
  
  if (options.notConfusedWith) {
    entry += `\n**Not to be confused with:** ${options.notConfusedWith}\n`;
  }
  
  if (options.seeAlso) {
    entry += `\n**See also:** ${options.seeAlso}\n`;
  }
  
  return entry;
}
```

## Inserting in Alphabetical Order

```javascript
function insertTermAlphabetically(glossaryContent, newEntry, newTerm) {
  const sections = glossaryContent.split(/(?=^## )/gm);
  const header = sections[0].startsWith('## ') ? '' : sections.shift();
  
  // Find insertion point
  let insertIndex = sections.length;
  for (let i = 0; i < sections.length; i++) {
    const existingTerm = sections[i].match(/^## (.+)/)?.[1];
    if (existingTerm && newTerm.toLowerCase() < existingTerm.toLowerCase()) {
      insertIndex = i;
      break;
    }
  }
  
  sections.splice(insertIndex, 0, newEntry);
  return header + sections.join('\n');
}
```
