---
name: implementing-product-reviews
description: Load when adding product reviews to a storefront — covers review display, submission form, star ratings, image uploads, feature gate, and pagination. Self-contained skill; does not require building-storefronts.
---

# Implementing Product Reviews

Guide for adding product reviews to a Medusa storefront. Covers the review API, React components (list, form, star ratings), image uploads, feature gate, and pagination.

## When to Apply

**Load this skill when:**
- Adding product reviews to a storefront
- Implementing review submission forms
- Displaying star ratings or review lists
- Integrating review API endpoints
- Adding image upload to reviews

## CRITICAL: Load Reference Files When Needed

**The quick reference below is NOT sufficient for implementation.** You MUST load the reference files before writing code.

**Load these references when implementing review features:**

- **Calling review API endpoints?** → MUST load `references/review-api.md` first
- **Checking feature gate?** → MUST load `references/review-api.md` first
- **Building React components?** → MUST load `references/review-components.md` first
- **Adding image upload?** → MUST load `references/review-api.md` AND `references/review-components.md`

## Rule Categories by Priority

| Priority | Category | Impact | Prefix |
|----------|----------|--------|--------|
| 1 | Feature Gate | CRITICAL | `gate-` |
| 2 | API Integration | HIGH | `api-` |
| 3 | Component Structure | HIGH | `comp-` |
| 4 | Image Uploads | MEDIUM | `img-` |

## Quick Reference

### 1. Feature Gate (CRITICAL)

- `gate-check-first` - **ALWAYS check `reviews_enabled` via global settings before rendering any review components**
- `gate-silent-disable` - Return null (render nothing) if reviews are disabled or settings are loading — never show broken/empty review UI

### 2. API Integration (HIGH)

- `api-use-send-request` - Use the project's `sendRequest` helper (wraps `sdk.client.fetch`) for all review API calls — NEVER use raw fetch()
- `api-store-reviews` - List reviews: `GET /store/reviews?product_id={id}&limit=10&offset=0`
- `api-post-review` - Submit review: `POST /store/reviews` with product_id, rating, content, first_name, email
- `api-approved-only` - Store endpoint returns only approved reviews; no status filtering needed on frontend
- `api-average-in-response` - `average_rating` and `total_count` are returned in the list response — no separate endpoint needed

### 3. Component Structure (HIGH)

- `comp-hierarchy` - Component tree: `ProductReviews` (gate) → `ReviewList` + `ReviewForm`, `ReviewList` → `ReviewCard` (uses `StarRating`)
- `comp-product-page` - Place `<ProductReviews productId={product.id} />` on product page after description, before related products
- `comp-infinite-query` - Use `useInfiniteQuery` for review list pagination (10 per page, "Load more" button)
- `comp-invalidate` - Invalidate review queries after successful submission
- `comp-query-keys` - Add `reviews` domain to the project's `queryKeys` utility

### 4. Image Uploads (MEDIUM)

- `img-upload-first` - Upload images to `/store/uploads` FIRST, then pass returned URLs in the review payload
- `img-max-five` - Maximum 5 images per review
- `img-use-fetch` - Image upload is the ONE exception to the no-fetch rule — use `fetch()` with FormData and `x-publishable-api-key` header (SDK cannot handle FormData)

## Common Mistakes Checklist

Before implementing, verify you're NOT doing these:

**Feature Gate:**
- [ ] Not checking `reviews_enabled` before rendering review components
- [ ] Showing empty review UI instead of returning null when disabled

**API Integration:**
- [ ] Using raw `fetch()` instead of `sendRequest` for review API calls
- [ ] Not handling 409 duplicate review error on submission

**Components:**
- [ ] Using `useQuery` with manual offset instead of `useInfiniteQuery` for pagination
- [ ] Forgetting to invalidate review queries after successful submission
- [ ] Not auto-filling customer name/email when user is logged in

**Image Uploads:**
- [ ] Using `sendRequest` for image uploads (needs FormData, which requires raw `fetch()`)
- [ ] Passing image File objects directly in review payload instead of uploading first to get URLs
- [ ] Not including `x-publishable-api-key` header in upload fetch call

## How to Use

**For detailed patterns and examples, load reference files:**

```
references/review-api.md      - API endpoints, request/response shapes, TypeScript types
references/review-components.md - React component implementation, query patterns, integration
```
