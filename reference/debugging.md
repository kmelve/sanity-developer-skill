# Debugging and Common Pitfalls

Solutions to common Sanity development issues and anti-patterns.

## Table of Contents
- GROQ query errors
- Schema validation issues
- Reference problems
- Performance issues
- Document operation errors
- Tool selection mistakes

## GROQ Query Errors

### "String literal expected"

**Symptom:** Query fails with parser error

**Cause:** Missing quotes around computed field names in projections

```groq
# WRONG
*[_type == "post"]{ title: slug.current }

# RIGHT
*[_type == "post"]{ "title": slug.current }
```

**Fix:** Always quote computed field names. Regular field selections don't need quotes.

### "No matches found" with References

**Symptom:** Query returns empty array when you know data exists

**Cause:** Using wrong syntax for array vs single reference

```groq
# For single reference field "author"
# WRONG
*[_type == "post" && $authorId in author[]._ref]

# RIGHT
*[_type == "post" && author._ref == $authorId]

# For array reference field "authors"
# WRONG
*[_type == "post" && authors._ref == $authorId]

# RIGHT
*[_type == "post" && $authorId in authors[]._ref]
```

**Fix:** Check schema first to determine if field is array or single reference.

### Text Search Returns Nothing

**Symptom:** Text search with `match` returns no results

**Cause:** Using old syntax or searching non-indexed fields

```groq
# OLD - may not work consistently
*[_type == "post" && body match "search term"]

# NEW - correct syntax
*[_type == "post" && body match text::query("search term")]

# For exact phrases
*[_type == "post" && body match text::query("\"exact phrase\"")]
```

**Fix:** Use modern `match text::query()` syntax and ensure field has text index.

### Dereferencing Null References

**Symptom:** Fields missing in results when dereferencing

**Cause:** Optional references that are null/undefined

```groq
# May return null for optional references
*[_type == "post"]{
  title,
  "authorName": author->name
}

# Better - handle missing references
*[_type == "post"]{
  title,
  "authorName": select(
    defined(author) => author->name,
    "Unknown Author"
  )
}
```

**Fix:** Use `defined()` check or `select()` for optional references.

## Schema Validation Issues

### "Field required" on Existing Documents

**Symptom:** Can't save documents after adding required field

**Cause:** Existing documents don't have the new required field

```typescript
// Adding this breaks existing documents
{
  name: 'newRequiredField',
  type: 'string',
  validation: Rule => Rule.required()
}
```

**Fix:** Make field optional initially, migrate data, then make required:

```typescript
// Step 1: Add as optional
{
  name: 'newField',
  type: 'string',
  validation: Rule => Rule.optional()
}

// Step 2: Migrate existing documents
// Use patch_document or update_document

// Step 3: Make required
{
  name: 'newField',
  type: 'string',
  validation: Rule => Rule.required()
}
```

### Validation Conflicts

**Symptom:** Documents fail validation despite correct values

**Cause:** Conflicting validation rules

```typescript
// WRONG - conflicting rules
{
  name: 'price',
  type: 'number',
  validation: Rule => Rule.required().min(0).max(100).integer().positive()
}

// RIGHT - compatible rules
{
  name: 'price',
  type: 'number',
  validation: Rule => Rule.required().min(0).positive()
}
```

**Fix:** Review validation logic for conflicts.

### Custom Validation Not Running

**Symptom:** Custom validation doesn't trigger

**Cause:** Validation runs at save time, not on field change

```typescript
{
  name: 'endDate',
  type: 'datetime',
  validation: Rule => Rule.custom((endDate, context) => {
    const startDate = context.document?.startDate
    if (endDate && startDate && endDate < startDate) {
      return 'End date must be after start date'
    }
    return true
  })
}
```

**Fix:** Access `context.document` for cross-field validation. Test by saving document.

## Reference Problems

### Circular References

**Symptom:** Infinite loop or query timeout

**Cause:** Document references itself directly or indirectly

```typescript
// Category → Parent Category → Category (circular)

// Prevent at schema level
{
  name: 'parent',
  type: 'reference',
  to: [{ type: 'category' }],
  validation: Rule => Rule.custom((value, context) => {
    if (value?._ref === context.document?._id) {
      return 'Cannot reference itself'
    }
    return true
  })
}
```

**Fix:** Add validation to prevent self-references. For indirect cycles, implement depth limits in queries.

### Deleted Referenced Documents

**Symptom:** Broken references after document deletion

**Cause:** Strong references to deleted documents

```groq
# Query shows references but dereferencing returns null
*[_type == "post"]{
  author, // Shows { _ref: "deleted-id" }
  "authorName": author->name // Returns null
}
```

**Fix:** Check for deleted references:

```groq
*[_type == "post" && defined(author->_id)]{
  title,
  "authorName": author->name
}
```

Or clean up references using patch_document to unset the field.

### Reference Type Mismatch

**Symptom:** Can't save document with reference

**Cause:** Referencing wrong document type

```typescript
// Schema allows only 'person'
{
  name: 'author',
  type: 'reference',
  to: [{ type: 'person' }]
}

// Trying to reference 'user' fails
```

**Fix:** Ensure referenced document matches allowed types in schema. Update schema if needed:

```typescript
{
  name: 'author',
  type: 'reference',
  to: [{ type: 'person' }, { type: 'user' }]
}
```

## Performance Issues

### Slow Queries

**Symptom:** Queries take >1 second

**Common causes:**
1. Dereferencing in loops
2. No filtering before expensive operations
3. Fetching too many documents

```groq
# SLOW - dereferences in large result set
*[_type == "post"]{
  title,
  "authorName": author->name,
  "categoryNames": categories[]->title
}[0...1000]

# FASTER - limit first, then dereference
*[_type == "post"][0...10]{
  title,
  "authorName": author->name,
  "categoryNames": categories[]->title
}

# FASTEST - filter before dereferencing
*[_type == "post" && published == true]
  | order(publishedAt desc)[0...10]{
    title,
    "authorName": author->name
}
```

**Fix:** Apply filters and limits before dereferencing.

### Memory Issues with Large Datasets

**Symptom:** Operation fails or times out

**Cause:** Processing too many documents at once

```typescript
// AVOID - processing 1000+ documents
query_documents({
  query: '*[_type == "post"][0...5000]',
  resource: { projectId, dataset }
})

// BETTER - paginate
query_documents({
  query: '*[_type == "post"][0...100]',
  resource: { projectId, dataset }
})
```

**Fix:** Use pagination. Process in batches of 50-100 documents.

### Over-fetching Data

**Symptom:** Large responses, slow performance

**Cause:** Fetching unnecessary fields

```groq
# SLOW - fetches everything including large body field
*[_type == "post"]

# FASTER - only needed fields
*[_type == "post"]{ _id, title, slug }

# SLOW - deeply nested dereferencing
*[_type == "post"]{
  ...,
  author->{
    ...,
    company->{
      ...,
      employees[]->{ ... }
    }
  }
}

# FASTER - shallow dereferencing
*[_type == "post"]{
  title,
  "authorName": author->name,
  "companyName": author->company->name
}
```

**Fix:** Project only needed fields. Avoid deep nesting.

## Document Operation Errors

### "Document not found"

**Symptom:** Update/publish/delete fails

**Common causes:**
1. Wrong document ID
2. Using draft ID when published ID needed (or vice versa)
3. Document doesn't exist

```typescript
// Check what exists
query_documents({
  query: '*[_id in ["post-123", "drafts.post-123"]]{ _id, _type }',
  resource: { projectId, dataset }
})

// Use correct ID format
publish_document({
  id: "post-123", // NOT "drafts.post-123"
  resource: { projectId, dataset }
})
```

**Fix:** Verify document exists. Use correct ID format for operation.

### "Cannot publish document with validation errors"

**Symptom:** publish_document fails

**Cause:** Draft has validation errors

```typescript
// Check validation
query_documents({
  query: '*[_id == "drafts.post-123"][0]',
  resource: { projectId, dataset }
})

// Fix validation issues first using patch_document or update_document
patch_document({
  documentId: "drafts.post-123",
  operation: { op: "set", path: "requiredField", value: "value" },
  resource: { projectId, dataset },
  workspaceName: "default"
})

// Then publish
publish_document({
  id: "post-123",
  resource: { projectId, dataset }
})
```

**Fix:** Resolve all validation errors before publishing.

### "Maximum 5 documents" Error

**Symptom:** Batch operation fails

**Cause:** Exceeding 5 document limit

```typescript
// WRONG - trying to create 10 documents
create_document({
  instruction: [
    "Create post about topic 1",
    "Create post about topic 2",
    // ... 8 more
  ],
  resource: { projectId, dataset },
  workspaceName: "default"
})

// RIGHT - batch of 5 or less
create_document({
  instruction: [
    "Create post about topic 1",
    "Create post about topic 2",
    "Create post about topic 3",
    "Create post about topic 4",
    "Create post about topic 5"
  ],
  resource: { projectId, dataset },
  workspaceName: "default"
})
```

**Fix:** Process in batches of 5 or fewer. Use releases for larger coordinated updates.

## Tool Selection Mistakes

### Using update_document for Precision Changes

**Symptom:** Unexpected changes or AI hallucinations

**Issue:** update_document uses AI, which may rewrite more than intended

```typescript
// WRONG for precision - might change other fields
update_document({
  operations: [{ 
    documentId: "post-123",
    instruction: "Set status to published"
  }]
})

// RIGHT for precision - exact change only
patch_document({
  documentId: "post-123",
  operation: { op: "set", path: "status", value: "published" }
})
```

**Fix:** Use patch_document for precise changes. Use update_document for content rewrites.

### Using patch_document for Content Generation

**Symptom:** Trying to generate content with patch_document

**Issue:** patch_document doesn't generate content

```typescript
// WRONG - patch_document can't generate content
patch_document({
  documentId: "post-123",
  operation: { op: "set", path: "body", value: "Write a story about..." }
})

// RIGHT - use update_document for AI generation
update_document({
  operations: [{ 
    documentId: "post-123",
    instruction: "Write an engaging introduction paragraph about Sanity CMS"
  }],
  paths: ["body"]
})
```

**Fix:** Use update_document or create_document for AI-generated content.

### Not Using transform_document for Rich Text

**Symptom:** Rich text formatting lost during updates

**Issue:** update_document may not preserve complex formatting

```typescript
// WRONG - might lose formatting
update_document({
  operations: [{ 
    documentId: "post-123",
    instruction: "Replace React with Next.js"
  }],
  paths: ["body"]
})

// RIGHT - preserves portable text structure
transform_document({
  documentId: "post-123",
  instruction: "Replace all instances of React with Next.js",
  paths: ["body"],
  operation: "edit"
})
```

**Fix:** Use transform_document when preserving rich text formatting is critical.

## Debugging Workflow

### Step 1: Verify Document Exists

```groq
*[_id in ["drafts.post-123", "post-123"]]{ _id, _type, title }
```

### Step 2: Check Schema

```typescript
get_schema({ 
  workspaceName: "default",
  type: "post",
  resource: { projectId, dataset }
})
```

### Step 3: Test Query Incrementally

```groq
# Start simple
*[_type == "post"]

# Add filters one at a time
*[_type == "post" && published == true]

# Add ordering
*[_type == "post" && published == true] | order(publishedAt desc)

# Add projections
*[_type == "post" && published == true] 
  | order(publishedAt desc)[0...10]{ 
    _id, 
    title 
  }

# Add dereferencing last
*[_type == "post" && published == true] 
  | order(publishedAt desc)[0...10]{ 
    _id, 
    title,
    "authorName": author->name
  }
```

### Step 4: Check Perspectives

```typescript
// Try different perspectives
query_documents({
  query: '*[_type == "post" && _id == "post-123"]',
  perspective: "published",
  resource: { projectId, dataset }
})

query_documents({
  query: '*[_type == "post" && _id == "post-123"]',
  perspective: "drafts",
  resource: { projectId, dataset }
})
```

### Step 5: Examine Full Document

```groq
# See everything to understand structure
*[_id == "drafts.post-123"][0]
```

## Prevention Best Practices

1. **Check schema before queries** - Prevents wrong assumptions
2. **Start simple, add complexity** - Easier to debug
3. **Test with small datasets** - Faster iteration
4. **Use correct tool for task** - Avoid tool misuse
5. **Handle optional fields** - Use `defined()` checks
6. **Validate before publishing** - Catch errors early
7. **Batch within limits** - Respect 5 document maximum
8. **Use releases for coordination** - Better than large batches
9. **Project only needed fields** - Better performance
10. **Test in drafts first** - Avoid impacting live content
