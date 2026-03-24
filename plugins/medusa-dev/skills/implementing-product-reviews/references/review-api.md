# Product Review API Reference

## Contents
- [Feature Gate](#feature-gate)
- [List Reviews](#list-reviews)
- [Submit Review](#submit-review)
- [Image Upload](#image-upload)
- [TypeScript Types](#typescript-types)
- [Usage Examples](#usage-examples)

## Feature Gate

Before rendering any review UI, check if reviews are enabled via global settings.

**Endpoint:** `GET /store/global-settings?type=common`

**Response:**
```typescript
{
  type: string
  data: {
    reviews_enabled?: boolean
    [key: string]: unknown
  }
}
```

**Check:** `settings.data.reviews_enabled === true`

If `false`, `undefined`, or the request fails — render nothing. Do not show an empty review section.

```typescript
import { sendRequest } from "@/lib/data/common"

const settings = await sendRequest<GlobalSettingsResponse>("/store/global-settings", {
  method: "GET",
  query: { type: "common" },
})

const reviewsEnabled = settings?.data?.reviews_enabled === true
```

## List Reviews

**Endpoint:** `GET /store/reviews`

**Query Parameters:**
| Parameter | Type | Required | Default | Constraints |
|-----------|------|----------|---------|-------------|
| `product_id` | string | Yes | — | Product ID to fetch reviews for |
| `limit` | number | No | 10 | Min 1, max 100 |
| `offset` | number | No | 0 | Min 0 |

**Response (200):**
```typescript
{
  reviews: Review[]       // Reviews for current page
  count: number           // Total approved reviews for this product
  limit: number           // Limit used
  offset: number          // Offset used
  average_rating: number  // Average rating across ALL approved reviews (0 if none)
  total_count: number     // Total approved reviews (same as count)
}
```

**Behavior:**
- Only returns reviews with `status: "approved"`
- Ordered by `created_at` DESC (newest first)
- `average_rating` and `total_count` span all pages, not just the current page
- Returns empty reviews array with `count: 0` and `average_rating: 0` if reviews are disabled (HTTP 200, not an error)

## Submit Review

**Endpoint:** `POST /store/reviews`

**Request Body:**
```typescript
{
  product_id: string       // Required — product to review
  rating: number           // Required — integer 1-5
  content: string          // Required — min 10 characters
  first_name: string       // Required — min 1 character
  email: string            // Required — valid email
  title?: string           // Optional — review headline
  last_name?: string       // Optional
  images?: string[]        // Optional — max 5 image URLs (upload first, see Image Upload)
}
```

**Response (201 — success):**
```typescript
{ review: Review }
```

**Response (403 — reviews disabled):**
```typescript
{ error: "Reviews are disabled" }
```

**Response (409 — duplicate review):**
```typescript
{ error: "You have already submitted a review for this product" }
```

**Business Rules:**
- All new reviews are auto-approved (no moderation queue)
- If user is authenticated, `customer_id` is captured automatically
- Duplicate prevention: one review per product per customer (by customer_id if logged in, by email if guest)
- Images must be uploaded separately first (see Image Upload), then pass the returned URLs

## Image Upload

**Endpoint:** `POST /store/uploads`

**⚠️ EXCEPTION: This is the ONE endpoint that requires raw `fetch()` instead of `sendRequest`.**

The SDK's `sendRequest` / `sdk.client.fetch` serializes the body as JSON. Image uploads require `FormData`, which is incompatible with JSON serialization. You must use raw `fetch()` with the publishable API key header.

**Upload Flow:**
1. User selects image files
2. On form submit, upload each file to `/store/uploads`
3. Collect returned URLs
4. Include URLs in the review creation payload

**Request:**
- Method: `POST`
- Body: `FormData` with field `files` containing the image file
- Headers: `x-publishable-api-key` (required)

**Response:**
```typescript
{ files: [{ url: string }] }
```

**Constraints:**
- Maximum 5 images per review
- Accepted types: JPEG, PNG, WebP, GIF

**Example:**
```typescript
const uploadImages = async (files: File[]): Promise<string[]> => {
  const urls: string[] = []
  const backendUrl = import.meta.env.VITE_MEDUSA_BACKEND_URL || "http://localhost:9000"
  const publishableKey = import.meta.env.VITE_MEDUSA_PUBLISHABLE_KEY || ""

  for (const file of files) {
    const formData = new FormData()
    formData.append("files", file)
    const response = await fetch(`${backendUrl}/store/uploads`, {
      method: "POST",
      body: formData,
      headers: {
        "x-publishable-api-key": publishableKey,
      },
    })
    if (!response.ok) {
      throw new Error("Failed to upload image")
    }
    const data = await response.json()
    if (data.files?.[0]?.url) {
      urls.push(data.files[0].url)
    }
  }
  return urls
}
```

## TypeScript Types

```typescript
interface ReviewImage {
  id: string
  url: string
  sort_order: number
}

interface Review {
  id: string
  product_id: string
  title: string | null
  content: string
  rating: number
  first_name: string
  last_name: string | null
  images: ReviewImage[]
  created_at: string
}

interface ReviewsResponse {
  reviews: Review[]
  count: number
  limit: number
  offset: number
  average_rating: number
  total_count: number
}

interface CreateReviewPayload {
  product_id: string
  rating: number
  content: string
  first_name: string
  email: string
  title?: string
  last_name?: string
  images?: string[]
}

interface GlobalSettingsResponse {
  type: string
  data: {
    reviews_enabled?: boolean
    [key: string]: unknown
  }
}
```

## Usage Examples

### Fetching Reviews

```typescript
import { sendRequest } from "@/lib/data/common"

// Fetch first page of reviews
const data = await sendRequest<ReviewsResponse>("/store/reviews", {
  method: "GET",
  query: {
    product_id: productId,
    limit: 10,
    offset: 0,
  },
})

console.log(data.reviews)        // Review[]
console.log(data.average_rating) // e.g. 4.3
console.log(data.total_count)    // e.g. 47
```

### Submitting a Review

```typescript
import { sendRequest } from "@/lib/data/common"

// ✅ CORRECT — use sendRequest with plain object
const result = await sendRequest<{ review: Review }>("/store/reviews", {
  method: "POST",
  body: {
    product_id: "prod_123",
    rating: 5,
    content: "Great product, highly recommend!",
    first_name: "John",
    email: "john@example.com",
    title: "Excellent quality",
    images: ["https://uploaded-url-1.jpg", "https://uploaded-url-2.jpg"],
  },
})

// ❌ WRONG — never use raw fetch() for review API calls
const result = await fetch("/store/reviews", { ... })

// ❌ WRONG — never use JSON.stringify on body
const result = await sendRequest("/store/reviews", {
  method: "POST",
  body: JSON.stringify({ ... }), // DON'T DO THIS
})
```
