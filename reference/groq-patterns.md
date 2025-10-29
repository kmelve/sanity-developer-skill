# GROQ Query Reference

Advanced patterns and examples for GROQ queries.

## Table of Contents
- Filtering and operators
- Joins and dereferencing
- Aggregations and computations
- Ordering and slicing
- Conditional logic
- Performance optimization

## Filtering and Operators

### Comparison Operators

```groq
# Equality
*[_type == "post" && status == "published"]

# Inequality
*[_type == "post" && status != "draft"]

# Greater/less than
*[_type == "post" && publishedAt > "2024-01-01"]
*[views < 1000]

# In array
*[_type == "post" && status in ["published", "featured"]]

# Not in array
*[_type == "post" && !(status in ["draft", "archived"])]
```

### Pattern Matching

```groq
# Text contains (case-insensitive by default)
*[_type == "post" && title match "sanity"]

# Case-sensitive match
*[_type == "post" && title match "Sanity"]

# Text search with modern syntax
*[_type == "post" && body match text::query("content management")]

# Exact phrase search (escape quotes)
*[_type == "post" && body match text::query("\"Sanity Studio\"")]
```

### Boolean Logic

```groq
# AND
*[_type == "post" && published == true && featured == true]

# OR
*[_type == "post" && (category == "tech" || category == "design")]

# NOT
*[_type == "post" && !(draft == true)]

# Complex combinations
*[_type == "post" && published == true && (featured == true || views > 1000)]
```

## Joins and Dereferencing

### Single Reference

```groq
# Dereference inline
*[_type == "post"]{
  title,
  "authorName": author->name,
  "authorBio": author->bio
}

# Filter by referenced document
*[_type == "post" && author->slug.current == "jane-doe"]

# Nested dereferencing
*[_type == "post"]{
  title,
  "categoryName": category->title,
  "parentCategory": category->parent->title
}
```

### Array References

```groq
# Dereference array
*[_type == "post"]{
  title,
  "authorNames": authors[]->name,
  "categoryTitles": categories[]->title
}

# Filter by reference in array
*[_type == "post" && "author-123" in authors[]._ref]

# Array of specific fields from references
*[_type == "post"]{
  title,
  "authors": authors[]->{
    name,
    slug,
    image
  }
}
```

### References() Function

```groq
# Find documents that reference a specific document
*[references("author-123")]

# Find documents of specific type that reference something
*[_type == "post" && references("author-123")]

# Count references
*[_type == "author"]{
  name,
  "postCount": count(*[_type == "post" && references(^._id)])
}
```

## Aggregations and Computations

### Counting

```groq
# Count all documents
count(*[_type == "post"])

# Count with filter
count(*[_type == "post" && published == true])

# Count references
*[_type == "author"]{
  name,
  "publishedPosts": count(*[_type == "post" && references(^._id) && published == true])
}
```

### Array Operations

```groq
# Array length
*[_type == "post"]{
  title,
  "tagCount": length(tags)
}

# Array filtering
*[_type == "post"]{
  title,
  "publishedAuthors": authors[published == true]->name
}

# Array slicing
*[_type == "post"]{
  title,
  "firstThreeTags": tags[0...3]
}

# Array contains
*[_type == "post" && "react" in tags]
```

### String Operations

```groq
# String concatenation
*[_type == "author"]{
  "fullName": firstName + " " + lastName
}

# Conditional string
*[_type == "post"]{
  "status": select(
    published == true => "Published",
    featured == true => "Featured",
    "Draft"
  )
}
```

## Ordering and Slicing

### Basic Ordering

```groq
# Ascending
*[_type == "post"] | order(publishedAt asc)

# Descending
*[_type == "post"] | order(publishedAt desc)

# Multiple fields
*[_type == "post"] | order(featured desc, publishedAt desc)

# Order by referenced field
*[_type == "post"] | order(author->name asc)
```

### Slicing Results

```groq
# First 10 results
*[_type == "post"][0...10]

# Skip first 10, get next 10
*[_type == "post"][10...20]

# Get single document
*[_type == "post"][0]

# Last document
*[_type == "post"] | order(_createdAt desc)[0]

# Combined with ordering
*[_type == "post"] | order(publishedAt desc)[0...5]
```

## Conditional Logic

### Select Expression

```groq
# Basic select
*[_type == "post"]{
  title,
  "badge": select(
    featured == true => "Featured",
    popular == true => "Popular",
    "Regular"
  )
}

# Select with conditions
*[_type == "post"]{
  title,
  "ageCategory": select(
    publishedAt > "2024-01-01" => "New",
    publishedAt > "2023-01-01" => "Recent",
    "Archive"
  )
}
```

### Conditional Fields

```groq
# Include field conditionally
*[_type == "post"]{
  title,
  featured == true => {
    "featuredImage": image,
    "featuredUntil": featuredUntil
  }
}

# Alternative syntax
*[_type == "post"]{
  title,
  ...(featured == true && {
    featuredImage: image,
    featuredUntil
  })
}
```

## Performance Optimization

### Efficient Filtering

```groq
# GOOD - Filter early
*[_type == "post" && published == true] | order(publishedAt desc)[0...10]

# AVOID - Filter after ordering/slicing
*[_type == "post"] | order(publishedAt desc)[0...100]{
  @[published == true][0...10]
}
```

### Projection Optimization

```groq
# GOOD - Project only needed fields
*[_type == "post"]{ _id, title, slug }

# AVOID - Fetch everything then project
*[_type == "post"]{ ..., image { asset->{ ... } } }
```

### Index Utilization

```groq
# Indexed filters (faster)
*[_type == "post" && _id == "post-123"]
*[_type == "post" && slug.current == "my-post"]

# Non-indexed filters (slower)
*[_type == "post" && customField == "value"]
```

## Common Patterns

### Pagination

```groq
# Define variables
$page = 0
$pageSize = 10

# Query with offset
*[_type == "post"] | order(publishedAt desc)[$page * $pageSize...($page + 1) * $pageSize]
```

### Search with Filters

```groq
# Combine text search with filters
*[
  _type == "post" 
  && body match text::query($searchTerm)
  && published == true
  && category->slug.current in $categories
] | order(_score desc)[0...20]
```

### Related Content

```groq
# Find posts with same categories
*[_type == "post" && _id == $currentPostId]{
  "related": *[
    _type == "post" 
    && _id != $currentPostId
    && count((categories[]._ref)[@ in ^.^.categories[]._ref]) > 0
  ] | order(publishedAt desc)[0...3]
}
```

### Unique Values

```groq
# Get all unique categories
array::unique(*[_type == "post"].category->title)

# Get all unique tags
array::unique(*[_type == "post"].tags[])
```

## Debugging Queries

### Add Score for Search

```groq
# See relevance scores
*[_type == "post" && title match "sanity"]{
  title,
  _score
} | order(_score desc)
```

### Count Before Fetching

```groq
# Check how many results before expensive operations
{
  "total": count(*[_type == "post" && published == true]),
  "results": *[_type == "post" && published == true][0...10]
}
```

### Explain Dereferencing

```groq
# Show reference structure
*[_type == "post"][0]{
  "authorRef": author._ref,
  "authorResolved": author->{name, email}
}
```

## Error Prevention

### Always Quote Computed Field Names

```groq
# WRONG - causes "string literal expected" error
*[_type == "post"]{ title: slug.current }

# RIGHT - quoted field names
*[_type == "post"]{ "title": slug.current }
```

### Check Array vs Single Reference

```groq
# For single reference field "author"
*[_type == "post" && author._ref == $id]

# For array reference field "authors"
*[_type == "post" && $id in authors[]._ref]

# NOT this - wrong for single reference
*[_type == "post" && $id in author[]._ref] # ERROR!
```

### Escape Special Characters in Text Search

```groq
# For exact phrases with quotes
*[_type == "post" && body match text::query("\"quoted text\"")]

# For searches with special characters
*[_type == "post" && title match text::query("C++ programming")]
```
