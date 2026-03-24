# Product Review Components

## Contents
- [Component Hierarchy](#component-hierarchy)
- [Query Keys Setup](#query-keys-setup)
- [ProductReviews (Gate Component)](#productreviews-gate-component)
- [StarRating](#starrating)
- [ReviewList](#reviewlist)
- [ReviewCard](#reviewcard)
- [ReviewForm](#reviewform)
- [Product Page Integration](#product-page-integration)

## Component Hierarchy

```
ProductReviews (gate)
├── ReviewList
│   └── ReviewCard (× N)
│       └── StarRating (read-only)
└── ReviewForm (modal)
    └── StarRating (interactive)
```

**File structure:**
```
src/components/product-reviews/
  index.tsx          — ProductReviews (gate component, default export)
  star-rating.tsx    — StarRating (shared by list and form)
  review-list.tsx    — ReviewList + ReviewCard
  review-form.tsx    — ReviewForm (modal dialog)
```

## Query Keys Setup

Before building components, add `reviews` to the project's query key utility. Locate the `queryKeys` object (typically in `src/lib/utils/query-keys.ts`) and add:

```typescript
reviews: {
  ...createDomainKeys("reviews"),
  forProduct: (productId: string) =>
    createDynamicKey("reviews", "forProduct", productId),
  globalSettings: () => ["reviews", "globalSettings"] as const,
},
```

These keys are used by:
- `queryKeys.reviews.globalSettings()` — feature gate query
- `queryKeys.reviews.forProduct(productId)` — review list infinite query
- Invalidation after submission uses predicate: `query.queryKey?.includes("reviews")`

## ProductReviews (Gate Component)

**File:** `src/components/product-reviews/index.tsx`
**Props:** `{ productId: string }`

The top-level wrapper. Checks the feature gate and renders nothing if reviews are disabled.

```typescript
import { useQuery } from "@tanstack/react-query"
import { sendRequest } from "@/lib/data/common"
import { queryKeys } from "@/lib/utils/query-keys"
import ReviewList from "./review-list"
import ReviewForm from "./review-form"

interface GlobalSettingsResponse {
  type: string
  data: {
    reviews_enabled?: boolean
    [key: string]: unknown
  }
}

interface ProductReviewsProps {
  productId: string
}

const ProductReviews = ({ productId }: ProductReviewsProps) => {
  const { data: settings, isLoading } = useQuery({
    queryKey: queryKeys.reviews.globalSettings(),
    queryFn: () =>
      sendRequest<GlobalSettingsResponse>("/store/global-settings", {
        method: "GET",
        query: { type: "common" },
      }),
    staleTime: 5 * 60 * 1000, // Cache for 5 minutes
  })

  if (isLoading) {
    return null
  }

  const reviewsEnabled = settings?.data?.reviews_enabled === true

  if (!reviewsEnabled) {
    return null
  }

  return (
    <div className="mt-16 border-t border-neutral-200 pt-12">
      <h2 className="text-2xl md:text-3xl font-display font-semibold text-neutral-900 mb-12 tracking-tight">
        Customer Reviews
      </h2>

      <ReviewList productId={productId} />

      <div className="mt-10">
        <ReviewForm productId={productId} />
      </div>
    </div>
  )
}

export default ProductReviews
```

**Key behaviors:**
- Returns `null` while loading (no flash of content)
- Returns `null` if disabled (silent disable)
- 5-minute stale time to avoid re-fetching settings on every navigation
- Section has top border separator and generous spacing

## StarRating

**File:** `src/components/product-reviews/star-rating.tsx`
**Props:** `{ value: number, onChange?: (value: number) => void, size?: "sm" | "md" | "lg", className?: string }`

Shared star rating component used in both read-only (ReviewCard) and interactive (ReviewForm) contexts.

```typescript
import { useState } from "react"

interface StarRatingProps {
  value: number
  onChange?: (value: number) => void
  size?: "sm" | "md" | "lg"
  className?: string
}

const sizeClasses = {
  sm: "text-base",
  md: "text-xl",
  lg: "text-2xl",
}

const StarRating = ({
  value,
  onChange,
  size = "md",
  className = "",
}: StarRatingProps) => {
  const [hovered, setHovered] = useState(0)
  const interactive = !!onChange

  const displayValue = hovered || value

  return (
    <div
      className={`inline-flex items-center gap-0.5 ${className}`}
      onMouseLeave={() => interactive && setHovered(0)}
    >
      {[1, 2, 3, 4, 5].map((star) => {
        const filled = star <= displayValue
        return (
          <span
            key={star}
            className={`${sizeClasses[size]} select-none ${
              filled ? "text-yellow-500" : "text-neutral-300"
            } ${interactive ? "cursor-pointer hover:scale-110 transition-transform" : ""}`}
            onClick={() => onChange?.(star)}
            onMouseEnter={() => interactive && setHovered(star)}
            role={interactive ? "button" : undefined}
            aria-label={interactive ? `Rate ${star} star${star > 1 ? "s" : ""}` : undefined}
          >
            {filled ? "★" : "☆"}
          </span>
        )
      })}
    </div>
  )
}

export default StarRating
```

**Key behaviors:**
- **Interactive mode** (when `onChange` provided): hover preview, click to select, cursor pointer, scale animation, ARIA button role
- **Read-only mode** (no `onChange`): static display, no hover/click, no ARIA role
- Uses `★` (filled) and `☆` (empty) unicode characters
- Colors: `text-yellow-500` (filled), `text-neutral-300` (empty)

## ReviewList

**File:** `src/components/product-reviews/review-list.tsx`
**Props:** `{ productId: string }`

Displays paginated reviews with average rating header and "Load more" button.

```typescript
import { useInfiniteQuery } from "@tanstack/react-query"
import { sendRequest } from "@/lib/data/common"
import { queryKeys } from "@/lib/utils/query-keys"
import StarRating from "./star-rating"

const REVIEWS_PER_PAGE = 10

const ReviewList = ({ productId }: ReviewListProps) => {
  const {
    data,
    isLoading,
    error,
    fetchNextPage,
    hasNextPage,
    isFetchingNextPage,
  } = useInfiniteQuery({
    queryKey: queryKeys.reviews.forProduct(productId),
    queryFn: ({ pageParam = 0 }) =>
      sendRequest<ReviewsResponse>("/store/reviews", {
        method: "GET",
        query: {
          product_id: productId,
          limit: REVIEWS_PER_PAGE,
          offset: pageParam,
        },
      }),
    initialPageParam: 0,
    getNextPageParam: (lastPage) => {
      const nextOffset = lastPage.offset + lastPage.limit
      return nextOffset < lastPage.count ? nextOffset : undefined
    },
    enabled: !!productId,
  })

  // ... render logic
}
```

**Pagination pattern:**
- `useInfiniteQuery` with offset-based pagination
- `getNextPageParam`: calculates next offset, returns `undefined` when all pages loaded
- `data.pages.flatMap(page => page.reviews)` to flatten all pages into one list
- `hasNextPage` controls "Load more" button visibility

**Display:**
- **Average rating header:** Star rating (rounded to nearest int for stars) + numeric average (1 decimal) + total count
- **Review cards:** Rendered via `ReviewCard` component
- **Empty state:** "No reviews yet. Be the first to review this product!"
- **Loading state:** "Loading reviews..." centered text
- **Error state:** Returns null (silent failure)

## ReviewCard

Nested in `review-list.tsx`. Displays a single review.

```typescript
const ReviewCard = ({ review }: { review: Review }) => {
  return (
    <div className="border-b border-neutral-200 py-6 last:border-b-0">
      <div className="flex items-center gap-3 mb-2">
        <StarRating value={review.rating} size="sm" />
        <span className="text-sm text-neutral-500">
          {formatDate(review.created_at)}
        </span>
      </div>

      {review.title && (
        <h4 className="text-base font-semibold text-neutral-900 mb-1">
          {review.title}
        </h4>
      )}

      <p className="text-sm text-neutral-700 leading-relaxed mb-2">
        {review.content}
      </p>

      {review.images?.length > 0 && (
        <div className="flex gap-2 mt-3 mb-2">
          {review.images
            .sort((a, b) => a.sort_order - b.sort_order)
            .map((image) => (
              <img
                key={image.id}
                src={image.url}
                alt="Review image"
                className="w-16 h-16 object-cover rounded-md border border-neutral-200"
              />
            ))}
        </div>
      )}

      <p className="text-sm text-neutral-500">
        {review.first_name}
        {review.last_name ? ` ${review.last_name.charAt(0)}.` : ""}
      </p>
    </div>
  )
}
```

**Date formatting:**
```typescript
function formatDate(dateStr: string): string {
  const date = new Date(dateStr)
  return date.toLocaleDateString("en-US", {
    year: "numeric",
    month: "long",
    day: "numeric",
  })
}
```

**Display details:**
- Star rating: `sm` size, read-only
- Date: "March 24, 2026" format (en-US locale)
- Title: Only shown if present (optional field)
- Images: 64×64px thumbnails (`w-16 h-16`), sorted by `sort_order`, object-cover
- Reviewer name: "John" or "John D." (first name + last initial with period)
- Bottom border between cards, no border on last card

## ReviewForm

**File:** `src/components/product-reviews/review-form.tsx`
**Props:** `{ productId: string }`

Modal dialog for submitting a new review. Uses Radix UI Dialog.

**Dependencies:**
- `@radix-ui/react-dialog` — modal dialog
- `@medusajs/icons` — `XMarkMini` for close button
- `@tanstack/react-query` — `useMutation`, `useQueryClient`
- Project's `useCustomer` hook — auto-fill name/email

**Form fields:**
| Field | Type | Required | Validation |
|-------|------|----------|------------|
| Rating | StarRating (interactive, lg) | Yes | Must be > 0 |
| Title | text input | No | — |
| Content | textarea | Yes | Min 10 characters |
| First Name | text input | Yes | Non-empty |
| Last Name | text input | No | — |
| Email | email input | Yes | Non-empty |
| Images | file input (hidden) | No | Max 5, JPEG/PNG/WebP/GIF |

**Auto-fill behavior:**
- On mount and when customer data loads, auto-fill `firstName`, `lastName`, `email` from `useCustomer()` hook
- Uses ref tracking to avoid overwriting user edits

**Image upload flow:**
1. Hidden `<input type="file">` triggered by "Choose Images" button
2. Selected files stored in state, previews generated via `FileReader`
3. Each preview shown as 56×56px (`w-14 h-14`) thumbnail with hover X button to remove
4. On form submit: upload all files to `/store/uploads` first, collect URLs
5. Include URLs in `images` field of review payload

**Submission flow:**
1. Client-side validation (rating, content length, first name, email)
2. Upload images if any (sets `isUploading` state)
3. Build `CreateReviewPayload` object
4. `useMutation` calls `sendRequest` POST to `/store/reviews`
5. On success: reset form, close modal, invalidate all review queries
6. On error: display error message in form footer (handles 409 duplicate)

**Query invalidation:**
```typescript
queryClient.invalidateQueries({
  predicate: (query) => query.queryKey?.includes("reviews"),
})
```
This invalidates both the review list and global settings queries.

**Modal behavior:**
- "Write a Review" button opens modal
- Sticky header with title + close button
- Scrollable form body
- Sticky footer with error message + submit/cancel buttons
- Cannot close while submission is pending
- Form resets on successful submission

## Product Page Integration

Import `ProductReviews` and add it to the product page template:

```typescript
import ProductReviews from "@/components/product-reviews"

// In the product page JSX, after product description/accordions,
// before related products:
<ProductReviews productId={product.id} />
```

The component handles its own feature gate check — it will render nothing if reviews are disabled, so no conditional logic is needed at the page level.
