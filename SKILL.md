---
name: sanity-development
description: Work with Sanity Content Operating System - query documents with GROQ, manage schemas, create and update content, handle releases and versioning. Use when working with Sanity projects, content operations, GROQ queries, or schema design.
---

# Sanity Development

Opinionated patterns for working with Sanity's Content Operating System.

## Tool Selection Guide

Choose tools based on the operation type:

**Content Discovery:**

- `query_documents` - Search and retrieve content with GROQ
- `semantic_search` - Find content by meaning (requires embeddings index)
- `get_schema` - Understand document types and field structure (always check first)

**Content Creation:**

- `create_document` - Generate new documents with AI assistance
- `create_version` - Create document versions for releases

**Content Modification:**

- `update_document` - AI-powered content rewrites and improvements
- `patch_document` - Precise, direct field modifications (one operation at a time)
- `transform_document` - Format-preserving transformations (preserves rich text)
- `translate_document` - Language translation with style guides

**Document Lifecycle:**

- `publish_document` - Make draft documents live
- `unpublish_document` - Return published documents to draft state
- `delete_document` - Permanently remove documents
- `version_replace_document` - Replace version content with another document
- `version_discard_document` - Remove document from release
- `version_unpublish_document` - Mark for unpublishing when release runs

**Images:**

- `transform_image` - AI-powered image generation and transformation

**Project Management:**

- `list_projects` - View all Sanity projects
- `get_project_studios` - Get studio applications for a project
- `list_datasets` - View datasets in a project
- `list_workspace_schemas` - See available workspace names

**Releases:**

- `list_releases` - View content release packages
- `create_release` - Create new release
- `edit_release` - Update release metadata
- `schedule_release` - Schedule publication time
- `publish_release` - Publish immediately
- `archive_release` - Archive inactive release
- `unarchive_release` - Restore archived release

## Schema-First Principle

**ALWAYS check schema before querying content.** This prevents failed queries and reveals correct document types.

Workflow:

1. User asks about content: "Where can I edit the pricing page?"
2. Check schema first: `get_schema` to discover document types
3. Find relevant type (e.g., `pricingPage`)
4. Query using correct type and fields

```bash
# Check available workspaces first if needed
list_workspace_schemas

# Get full schema for a workspace
get_schema --workspaceName=default

# Get specific type details
get_schema --workspaceName=default --type=post
```

## GROQ Query Patterns

### Basic Queries

```groq
# All documents of a type
*[_type == "post"]

# With field selection
*[_type == "post"]{ _id, title, publishedAt }

# With filters
*[_type == "post" && publishedAt > "2024-01-01"]

# Ordering and limiting
*[_type == "post"] | order(publishedAt desc)[0...10]
```

### Projection Syntax

**CRITICAL:** Computed field names MUST be quoted strings.

```groq
# CORRECT - quoted field names for computed values
*[_type == "author"]{
  _id,
  "title": name,
  "postCount": count(*[_type == "post" && references(^._id)])
}

# WRONG - will cause "string literal expected" error
*[_type == "author"]{
  _id,
  title: name  # Missing quotes!
}

# Regular fields don't need quotes
*[_type == "post"]{ _id, title, slug }
```

### Reference Queries

**Check schema to determine array vs single reference:**

```groq
# Single reference field (e.g., "author")
*[_type == "post" && author._ref == $authorId]

# Array reference field (e.g., "authors")
*[_type == "post" && $authorId in authors[]._ref]

# Dereferencing
*[_type == "post"]{
  title,
  "authorName": author->name,
  "categoryTitles": categories[]->title
}
```

### Text and Semantic Search

```groq
# Text search (new syntax)
*[_type == "post" && body match text::query("foo bar")]

# Exact phrase
*[_type == "post" && body match text::query("\"exact phrase\"")]

# Semantic search requires embeddings index
semantic_search --indexName=posts --query="content about AI"
```

### Multi-Step Queries

For queries involving relationships:

1. Query referenced entity first
2. Extract ID(s)
3. Query primary content using found ID(s)

```groq
# Step 1: Find author(s) named "Magnus"
*[_type == "author" && name match "Magnus"]{ _id, name }

# Step 2: Use found ID to query posts
# If single reference:
*[_type == "post" && author._ref == "author-123"]

# If array reference:
*[_type == "post" && "author-123" in authors[]._ref]
```

## Document Operations

### Creating Documents

Use `create_document` for AI-generated content:

```typescript
create_document({
  resource: { projectId, dataset },
  workspaceName: "default",
  type: "post",
  instruction: ["Create a blog post about Sanity Studio features"],
});
```

For multiple documents, use `async: true` for better performance.

### Updating Documents

**update_document** - AI rewrites/improvements:

```typescript
update_document({
  operations: [
    {
      documentId: "post-123",
      instruction: "Make the tone more conversational",
    },
  ],
  paths: ["body"], // Target specific fields
  resource: { projectId, dataset },
  workspaceName: "default",
});
```

**patch_document** - Precise, direct changes (no AI):

```typescript
// Set a field
patch_document({
  documentId: "post-123",
  operation: { op: "set", path: "title", value: "New Title" },
  resource: { projectId, dataset },
  workspaceName: "default"
})

// Unset a field
operation: { op: "unset", path: "featured" }

// Append to array
operation: {
  op: "append",
  path: "tags",
  value: ["new-tag"]
}
```

**transform_document** - Preserves rich text formatting:

```typescript
transform_document({
  documentId: "post-123",
  instruction: "Replace all instances of 'React' with 'Next.js'",
  paths: ["body"], // Optional - target specific fields
  operation: "edit", // or "create" for new documents
  resource: { projectId, dataset },
  workspaceName: "default",
});
```

### Publishing Workflow

```typescript
// Publish draft
publish_document({
  id: "post-123",
  resource: { projectId, dataset },
});

// Unpublish
unpublish_document({
  id: "post-123",
  resource: { projectId, dataset },
});

// Delete permanently
delete_document({
  id: "post-123",
  resource: { projectId, dataset },
});
```

## Working with Releases

Releases coordinate staged content updates:

```typescript
// Create release
create_release({
  resource: { projectId, dataset },
  title: "Q4 2025 Product Launch",
  description: "New features and pricing",
  releaseType: "scheduled", // or "asap", "undecided"
  intendedPublishAt: "2025-12-01T00:00:00.000Z",
});

// Create document version for release
create_version({
  documentIds: ["post-123", "page-456"],
  releaseId: "release-abc",
  instruction: "Update pricing to reflect new tiers", // Optional
  resource: { projectId, dataset },
  workspaceName: "default",
});

// Schedule release
schedule_release({
  releaseId: "release-abc",
  publishAt: "in two weeks", // Natural language or ISO format
  resource: { projectId, dataset },
});

// Publish immediately
publish_release({
  releaseId: "release-abc",
  resource: { projectId, dataset },
});
```

## Common Workflows

### Finding Content by Relationship

```
1. Check schema to understand document structure
2. Query referenced entity (e.g., author)
3. If multiple matches, show all to user
4. Query primary content with found ID(s)
5. Verify array vs single reference in schema
```

### Batch Content Updates

```
1. Query documents needing updates
2. For 1-5 documents: use update_document or patch_document
3. For >5 documents: suggest using releases
4. Set async=true for multiple documents
5. Verify results after completion
```

### Schema-Driven Content Creation

```
1. User requests content creation
2. Check schema for correct document type
3. Verify required fields
4. Suggest correct type if user's request doesn't match
5. Create document with proper structure
```

## Anti-Patterns

**DON'T assume document types:** Always check schema first

**DON'T guess array vs single reference:** Schema reveals correct query syntax

**DON'T forget quotes in projections:** Computed fields must be string literals

**DON'T batch >5 document operations:** Use releases or break into smaller batches

**DON'T apologize for errors:** Try alternative approach immediately

**DON'T query before checking schema:** Wastes time on failed queries

**DON'T mix old and new text search syntax:** Use `match text::query()` for text search

## Document ID Formats

- Draft: `drafts.{id}`
- Published: `{id}`
- Release version: `versions.{releaseId}.{id}`

## Resource Selection

When multiple resources available (projects/datasets), ALWAYS ask user which to use. Never assume.

## Field Path Syntax

Supports:

- Simple fields: `"title"`
- Nested objects: `"author.name"`
- Array items by key: `"items[_key==\"item-1\"]"`
- Nested in arrays: `"items[_key==\"item-1\"].title"`

## Advanced Topics

For deeper coverage of specific areas, see:

**[reference/groq-patterns.md](reference/groq-patterns.md)** - Advanced GROQ queries:

- Complex filtering and operators
- Performance optimization techniques
- Aggregations and computations
- Join patterns and dereferencing strategies

**[reference/schema-design.md](reference/schema-design.md)** - Schema design:

- Document type organization
- Field validation patterns
- Reference relationship strategies
- Localization approaches
- Performance considerations

**[reference/debugging.md](reference/debugging.md)** - Troubleshooting:

- Common GROQ errors and fixes
- Schema validation issues
- Reference problems
- Performance debugging
- Tool selection mistakes
- Step-by-step debugging workflows

## Best Practices

1. **Schema-first approach** - Check schema before content queries
2. **Action-first response** - Perform actions immediately, don't just suggest
3. **Verify before modify** - Confirm document exists before updates
4. **Use releases for coordination** - Stage related content updates together
5. **Respect limits** - Maximum 5 documents per operation
6. **Choose right tool** - Use patch for precision, update for AI assistance
7. **Handle errors gracefully** - Try alternatives immediately
8. **Natural language times** - "in two weeks" works for scheduling
9. **Preserve formatting** - Use transform_document for rich text operations
10. **Progressive refinement** - Query → Verify → Refine → Execute
