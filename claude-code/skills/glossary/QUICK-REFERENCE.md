# Glossary Skill - Quick Reference

## Commands

```text
/glossary list                  Show all terms
/glossary search "cancel"       Find related terms
/glossary add "Refund"          Add new term
/glossary define "Cancellation" Get full definition
/glossary check CARD-101        Check story for term issues
/glossary conflicts             Show unresolved conflicts
```

## Entry Format

```markdown
## <Term>

**Definition:** <one-sentence definition>

**Examples:** <optional>

**Not to be confused with:** <optional>

**See also:** <optional>
```

## Location

**Single-context:** `/GLOSSARY.md` at repo root

**Multi-context:**
```
/GLOSSARY.md                  ← shared
/src/ordering/GLOSSARY.md     ← context-specific
/src/billing/GLOSSARY.md      ← context-specific
```

## Rules

1. Domain terms only (not implementation details)
2. Nouns and noun phrases (concepts, not actions)
3. Alphabetical order
4. Track renames in "Not to be confused with"

## Challenge Conflicts

When user uses a term differently than glossary:

```
⚠️ Glossary defines "Cancellation" as "voiding a paid order."
   You used it to mean "removing from cart."
   
   Options:
   1. Use existing definition (rephrase)
   2. Add new term "Cart Removal"
   3. Update glossary definition
```

## New Term Flow

```
Skill: "Chargeback" isn't in glossary.
       Suggested: "Bank-initiated forced reversal of a charge."
       Accept?

User: Yes

Skill: ✅ Added to GLOSSARY.md
```

## Skills That Use Glossary

| Skill | Usage |
|-------|-------|
| grill-to-brd | Build glossary during BRD creation |
| grill-me | Build glossary during grilling |
| story-grooming | Enforce terms in stories |
| brd-to-issues | Use terms in Tracker issues |
