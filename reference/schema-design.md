# Schema Design Guide

Best practices for designing Sanity schemas and content models.

## Table of Contents
- Document type design
- Field types and validation
- References and relationships
- Portable Text patterns
- Localization strategies
- Performance considerations

## Document Type Design

### Naming Conventions

**Document types:**
- Use camelCase: `blogPost`, `landingPage`, `productCategory`
- Be specific: `newsArticle` not `article`
- Avoid abbreviations: `product` not `prod`

**Field names:**
- Use camelCase: `publishedAt`, `featuredImage`
- Be descriptive: `shortDescription` not `desc`
- Boolean fields: prefix with `is`, `has`, `should`: `isPublished`, `hasCover`

### Document Structure

```typescript
// GOOD - Clear, specific document type
{
  name: 'blogPost',
  type: 'document',
  title: 'Blog Post',
  fields: [
    {
      name: 'title',
      type: 'string',
      title: 'Title',
      validation: Rule => Rule.required()
    },
    {
      name: 'slug',
      type: 'slug',
      title: 'Slug',
      options: {
        source: 'title',
        maxLength: 96
      },
      validation: Rule => Rule.required()
    },
    {
      name: 'publishedAt',
      type: 'datetime',
      title: 'Published At'
    }
  ]
}

// AVOID - Too generic
{
  name: 'content',
  type: 'document'
}
```

### Organizing Documents

**By content type:**
```
blog/
  post.ts
  category.ts
  author.ts
pages/
  landing.ts
  about.ts
  contact.ts
settings/
  site.ts
  navigation.ts
```

**Singleton documents:**
```typescript
{
  name: 'siteSettings',
  type: 'document',
  title: 'Site Settings',
  __experimental_singleton: true, // Only one instance allowed
  fields: [...]
}
```

## Field Types and Validation

### Required Fields

```typescript
// Always validate critical fields
{
  name: 'title',
  type: 'string',
  validation: Rule => Rule.required().min(10).max(200)
}

{
  name: 'slug',
  type: 'slug',
  validation: Rule => Rule.required()
}

{
  name: 'email',
  type: 'string',
  validation: Rule => Rule.required().email()
}
```

### Custom Validation

```typescript
{
  name: 'price',
  type: 'number',
  validation: Rule => Rule.required().min(0).positive()
}

{
  name: 'url',
  type: 'url',
  validation: Rule => Rule.uri({
    scheme: ['http', 'https']
  })
}

// Complex validation
{
  name: 'discount',
  type: 'number',
  validation: Rule => Rule.custom((discount, context) => {
    const price = context.document?.price
    if (discount && price && discount > price) {
      return 'Discount cannot exceed price'
    }
    return true
  })
}
```

### Field Organization

```typescript
// GOOD - Grouped related fields
{
  name: 'seo',
  type: 'object',
  title: 'SEO',
  fields: [
    { name: 'title', type: 'string' },
    { name: 'description', type: 'text' },
    { name: 'keywords', type: 'array', of: [{ type: 'string' }] },
    { name: 'image', type: 'image' }
  ]
}

// AVOID - Flat structure for related fields
{
  fields: [
    { name: 'seoTitle', type: 'string' },
    { name: 'seoDescription', type: 'text' },
    { name: 'seoKeywords', type: 'array' },
    { name: 'seoImage', type: 'image' }
  ]
}
```

## References and Relationships

### Reference Types

**Strong references (enforced):**
```typescript
{
  name: 'author',
  type: 'reference',
  title: 'Author',
  to: [{ type: 'person' }],
  validation: Rule => Rule.required()
}
```

**Weak references (optional):**
```typescript
{
  name: 'relatedPosts',
  type: 'array',
  title: 'Related Posts',
  of: [{
    type: 'reference',
    to: [{ type: 'blogPost' }]
  }],
  validation: Rule => Rule.max(3)
}
```

**Multiple types:**
```typescript
{
  name: 'content',
  type: 'array',
  of: [
    { type: 'reference', to: [{ type: 'blogPost' }] },
    { type: 'reference', to: [{ type: 'video' }] },
    { type: 'reference', to: [{ type: 'gallery' }] }
  ]
}
```

### Bidirectional Relationships

**One-to-many:**
```typescript
// Author (one)
{
  name: 'author',
  type: 'document',
  fields: [
    { name: 'name', type: 'string' }
  ]
}

// Post (many) - references author
{
  name: 'blogPost',
  type: 'document',
  fields: [
    {
      name: 'author',
      type: 'reference',
      to: [{ type: 'author' }]
    }
  ]
}

// Query: Find posts by author
*[_type == "blogPost" && author._ref == "author-123"]

// Query: Count author's posts
*[_type == "author"]{
  name,
  "postCount": count(*[_type == "blogPost" && references(^._id)])
}
```

**Many-to-many:**
```typescript
// Post with multiple categories
{
  name: 'blogPost',
  type: 'document',
  fields: [
    {
      name: 'categories',
      type: 'array',
      of: [{ type: 'reference', to: [{ type: 'category' }] }]
    }
  ]
}

// Query: Posts in category
*[_type == "blogPost" && "category-123" in categories[]._ref]

// Query: Categories with post count
*[_type == "category"]{
  title,
  "postCount": count(*[_type == "blogPost" && ^._id in categories[]._ref])
}
```

### Reference Prevention

```typescript
// Prevent circular references
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

## Portable Text Patterns

### Basic Configuration

```typescript
{
  name: 'body',
  type: 'array',
  title: 'Body',
  of: [
    {
      type: 'block',
      // Styles
      styles: [
        { title: 'Normal', value: 'normal' },
        { title: 'H2', value: 'h2' },
        { title: 'H3', value: 'h3' },
        { title: 'Quote', value: 'blockquote' }
      ],
      // Lists
      lists: [
        { title: 'Bullet', value: 'bullet' },
        { title: 'Numbered', value: 'number' }
      ],
      // Marks
      marks: {
        decorators: [
          { title: 'Strong', value: 'strong' },
          { title: 'Emphasis', value: 'em' },
          { title: 'Code', value: 'code' }
        ],
        annotations: [
          {
            name: 'link',
            type: 'object',
            title: 'Link',
            fields: [
              { name: 'href', type: 'url', title: 'URL' }
            ]
          }
        ]
      }
    }
  ]
}
```

### Custom Blocks

```typescript
{
  name: 'content',
  type: 'array',
  of: [
    { type: 'block' }, // Regular text
    {
      // Code block
      type: 'code',
      options: {
        language: 'javascript'
      }
    },
    {
      // Image with caption
      type: 'image',
      fields: [
        { name: 'caption', type: 'string' },
        { name: 'alt', type: 'string', title: 'Alternative text' }
      ]
    },
    {
      // Custom callout block
      name: 'callout',
      type: 'object',
      title: 'Callout',
      fields: [
        {
          name: 'type',
          type: 'string',
          options: {
            list: ['info', 'warning', 'error', 'success']
          }
        },
        { name: 'text', type: 'text' }
      ]
    }
  ]
}
```

### Internal References in Portable Text

```typescript
{
  name: 'body',
  type: 'array',
  of: [
    { type: 'block' },
    {
      type: 'reference',
      name: 'postReference',
      title: 'Referenced Post',
      to: [{ type: 'blogPost' }]
    }
  ]
}
```

## Localization Strategies

### Document-Level Localization

**Translation documents:**
```typescript
{
  name: 'blogPost',
  type: 'document',
  fields: [
    {
      name: 'language',
      type: 'string',
      options: {
        list: ['en', 'no', 'se']
      }
    },
    {
      name: 'translations',
      type: 'array',
      of: [{
        type: 'reference',
        to: [{ type: 'blogPost' }]
      }]
    },
    { name: 'title', type: 'string' },
    { name: 'body', type: 'array', of: [{ type: 'block' }] }
  ]
}

// Query: Get post with translations
*[_type == "blogPost" && _id == $id]{
  title,
  body,
  language,
  "translations": translations[]->{
    _id,
    language,
    title
  }
}
```

### Field-Level Localization

**Localized strings:**
```typescript
{
  name: 'title',
  type: 'object',
  fields: [
    { name: 'en', type: 'string', title: 'English' },
    { name: 'no', type: 'string', title: 'Norwegian' },
    { name: 'se', type: 'string', title: 'Swedish' }
  ]
}

// Query: Get localized field
*[_type == "blogPost"]{
  "title": title[$language],
  "body": body
}
```

### Translatable Plugin Pattern

```typescript
import { defineType } from 'sanity'

export default defineType({
  name: 'blogPost',
  type: 'document',
  i18n: {
    base: 'en',
    languages: ['en', 'no', 'se'],
    fieldNames: {
      lang: '__i18n_lang',
      references: '__i18n_refs'
    }
  },
  fields: [
    { name: 'title', type: 'string' },
    { name: 'slug', type: 'slug' }
  ]
})
```

## Performance Considerations

### Indexing Strategy

**Optimize for common queries:**
```typescript
// If you frequently query by slug
{
  name: 'slug',
  type: 'slug',
  options: {
    source: 'title',
    maxLength: 96
  }
}

// If you frequently filter by status
{
  name: 'status',
  type: 'string',
  options: {
    list: ['draft', 'published', 'archived']
  }
}
```

### Avoid Deep Nesting

```typescript
// AVOID - Too many levels
{
  name: 'content',
  type: 'object',
  fields: [
    {
      name: 'sections',
      type: 'array',
      of: [{
        type: 'object',
        fields: [
          {
            name: 'blocks',
            type: 'array',
            of: [{
              type: 'object',
              fields: [
                {
                  name: 'items',
                  type: 'array'
                }
              ]
            }]
          }
        ]
      }]
    }
  ]
}

// BETTER - Flatter structure with references
{
  name: 'sections',
  type: 'array',
  of: [{ type: 'reference', to: [{ type: 'section' }] }]
}
```

### Asset Optimization

```typescript
{
  name: 'coverImage',
  type: 'image',
  options: {
    hotspot: true,
    metadata: ['blurhash', 'lqip', 'palette'] // Enable for optimizations
  },
  fields: [
    { name: 'alt', type: 'string', title: 'Alternative text' }
  ]
}
```

## Common Patterns

### Conditional Fields

```typescript
{
  name: 'linkType',
  type: 'string',
  options: {
    list: ['internal', 'external']
  }
}
{
  name: 'internalLink',
  type: 'reference',
  to: [{ type: 'page' }],
  hidden: ({ parent }) => parent?.linkType !== 'internal'
}
{
  name: 'externalUrl',
  type: 'url',
  hidden: ({ parent }) => parent?.linkType !== 'external'
}
```

### Timestamps

```typescript
// Auto-managed timestamps
{
  name: 'blogPost',
  type: 'document',
  fields: [
    // These are automatic: _createdAt, _updatedAt
    {
      name: 'publishedAt',
      type: 'datetime',
      title: 'Published At',
      initialValue: () => new Date().toISOString()
    }
  ]
}
```

### Computed Fields (in queries)

```typescript
// Don't store computed values in schema
// Calculate in queries instead
*[_type == "blogPost"]{
  title,
  "wordCount": length(pt::text(body)),
  "readingTime": round(length(pt::text(body)) / 200),
  "authorName": author->name
}
```

## Migration Considerations

### Version Your Schema

```typescript
{
  name: 'blogPost',
  type: 'document',
  fields: [
    {
      name: 'schemaVersion',
      type: 'number',
      readOnly: true,
      initialValue: 2
    }
  ]
}
```

### Deprecated Fields

```typescript
{
  name: 'oldFieldName',
  type: 'string',
  deprecated: {
    reason: 'Use newFieldName instead'
  },
  hidden: true
}
```

### Safe Renames

```typescript
// Step 1: Add new field
{ name: 'newFieldName', type: 'string' }

// Step 2: Migrate data
// Use patch_document or transform_document

// Step 3: Hide old field
{
  name: 'oldFieldName',
  type: 'string',
  hidden: true,
  deprecated: { reason: 'Renamed to newFieldName' }
}

// Step 4: Remove after migration complete
```
