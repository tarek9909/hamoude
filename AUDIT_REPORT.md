# Hamoude (Skin Cella) App — Audit Report

**Date:** 2026-10-27
**Scope:** The Flutter app in `Hamoude/Skin-Cella`, including its uncommitted changes. Every API call was checked against the backend in `backend/src`.
**Method:**
- Ran `flutter analyze` and `flutter test`.
- Reviewed all screens, the app state, the API client and the models by hand.
- Re-read the source to confirm every critical finding.

## Summary

| Check | Result |
|---|---|
| `flutter analyze` | ✅ No issues |
| `flutter test` | ✅ 31/31 pass |
| Real end-to-end flows | ❌ Several core flows are broken |

The tests use a fake API (`FakeStorefrontApi`), so they never exercise the real backend contract. That is how the broken flows below got past a green test suite.

**What blocks release:**
- Account deletion, password change, profile edit and password recovery do not work.
- Bundles cannot be checked out.
- Items with a variant silently disappear from the cart.
- Prices shown in the app do not match what the backend charges.

---

## 🔴 Critical: blocks completion or release

### C1. Account deletion always fails, which blocks App Store review
- **Where:**
  - App: `lib/services/storefront_api.dart` ~L895 (`deleteAccount`) sends only `{customer_id}`.
  - Backend: `deleteMe` (`backend/src/controllers/storefront/storefront.controller.js:1118`) requires `verification_token`, an email OTP with purpose `account_delete`.
- **Why it can't work as built:** username accounts have no email, so an OTP can never be obtained.
- **Result:** Profile → Delete Account returns 422 every time. Apple guideline 5.1.1(v) requires working in-app deletion.
- **Fix:** Add a backend deletion path for username accounts (for example, confirm with the password) and connect the app to it.

### C2. Edit Profile and Change Password always fail
- **Where:**
  - App: `storefront_api.dart:828-840` sends `PATCH /me` with `{customer_id, full_name, password}`.
  - Backend: `updateCustomerProfile` (`storefront.model.js:1937-1940`) rejects any key except `full_name` and `phone` with 422 "Profile updates only accept full_name and phone."
- **Password change:** it must go through `POST /me/change-password`, which also needs an email OTP that username accounts can't get.
- **"Current password" check:** `change_password_screen.dart:37-45` compares against `appState.accountPassword`. That field is never set, so the check never runs.
- **Fix:**
  - Send only `{full_name}` to `PATCH /me`.
  - Add a backend change-password endpoint that takes `current_password` and `new_password`, and call it from the app.

### C3. Password recovery can't be reached and doesn't work
- **Not reachable:** nothing in the app opens `ForgotPasswordScreen`, and the login screen has no "Forgot password" link.
- **Doesn't match the backend:**
  - The screen asks for a phone number or account ID and tells the user an administrator will review the request.
  - The backend reset endpoint expects an email and sends an OTP, and the app has no screen for entering that OTP.
- **Result:** username accounts have no way to recover a password.
- **Stale test:** the test `app_state_test.dart:1179` locks in this admin-approval behaviour, which the backend doesn't have.
- **Fix:**
  - Pick one flow: an admin-approved reset keyed by username, or an optional email collected at signup.
  - Make the backend and the app match, and add the link to the login screen.

### C4. Bundles can never be checked out
- **Where:**
  - App: `lib/screens/bundle_detail_screen.dart:472-482` creates a fake `Product` with `id: 'bundle_<id>'` and `stockQuantity: 99`.
  - Backend: `pricing.service.js:147,165` runs `Number(product_id)`, gets NaN, and returns 422 "Each pricing item must include a valid product_id."
- **Result:**
  - Any order containing a bundle fails.
  - The bundle stays in the saved cart, so every later order keeps failing too.
- **Fix:** Send the bundle's real component products, or a `bundle_id` field that the backend supports.

### C5. Variant items silently disappear from the cart, about every 60 seconds *(caused by the uncommitted changes)*
- **What happens:** `_reconcileCartWithProducts` (`app_state.dart:1546-1557`) looks up the cart item's variant in `liveProduct.variants`. The product list endpoint (`attachPublicAttributes`, `storefront.model.js:482-490`) never returns a `variants` array, so the lookup always fails and the item is removed.
- **What triggers it:** the new 60-second refresh timer and the refresh that runs whenever the app returns to the foreground.
- **Result:** a customer picks a size or shade, adds it to the bag, and it vanishes within about a minute.
- **Fix:** Skip the variant check when the list data has no variants (or fetch product detail for those items). Alternatively, have the backend include `variants` in list responses.

---

## 🟠 High: workflow errors that break or corrupt orders

| # | Issue | Where | Fix |
|---|---|---|---|
| H1 | **The cart comes back after a successful order.** `checkout()` clears the cart in memory but never saves the empty cart. The same happens at L1367, L1406, L1430 and L1443. After a restart or a new login the ordered items are back, which invites duplicate orders. | `app_state.dart:983` | Call `clearCart()` (it saves) instead of `_cart.clear()`. |
| H2 | **Duplicate orders on retry.** A new idempotency key (`slug-<now ms>`) is generated on every tap, and checkout times out after 8 s. On a slow network the server creates the order, the app shows an error, the user taps again, and a second order is created. | `storefront_api.dart:344, 656` | Create the key once per checkout attempt and reuse it on retries. Give checkout a longer timeout. |
| H3 | **Shown prices ≠ charged prices.** Product pages and home promotions show discounted prices, but the cart uses base prices and `cartDiscount` is hard-coded to `0`. The backend then applies promotions and tax, so the total differs. `quoteCart` exists but is never called. A missing `discount_value` falls back to an invented 20%. | `app_state.dart:893-898`, `product_detail_screen.dart:881-888`, `home_screen.dart:125-176`, `offers_screen.dart:78` | Use the `cart/quote` endpoint for cart and checkout totals, and remove the `?? 20.0` fallbacks. |
| H4 | **Fake ratings.** The app reads `rating`/`reviewsCount` and falls back to 4.5 and 12. The backend sends `rating_average`/`rating_count`, so every product shows "4.5 (12 reviews)". This is misleading and a store-review risk. | `lib/models/product.dart:334-335` | Read the backend field names and default to 0. |
| H5 | **Fake address and fake success.** Deleting the last address inserts the placeholder "100 Market Street, Suite 12". With no saved address, checkout falls back to the branch name, and the backend rejects it with 422. If the cart empties during checkout, `checkout()` returns null and the screen shows "Order Placed!" with the hard-coded order id `SC-8921`. | `app_state.dart:2187`, `checkout_screen.dart:1276` | Remove the placeholders. Disable Place Order when there is no real address or the cart is empty, and treat a null result as an error. |
| H6 | **Expired or revoked token is never handled.** Tokens last 30 days and are revoked when the password changes. The resulting 401/403 errors are caught silently. The user looks signed in, but their orders are empty and checkout fails. | `app_state.dart:2089-2097` | On a 401/403 from a customer endpoint, sign the user out and ask them to log in again. |
| H7 | **Logout leaks the previous user's data.** `signOutCustomer` does not clear orders, order timelines or the cart (the saved cart is per store, not per user), and it doesn't reload products after leaving wholesale mode. | `app_state.dart:1894-1944` | Clear all user data, both in memory and saved, and reload products. |
| H8 | **Support ticket "created" before it is.** `createTicket` isn't awaited. The success toast always shows and the form is cleared even when the request fails, so the user's message is lost. | `live_support_screen.dart:240` | Await the call, show errors, and clear the form only on success. |
| H9 | **Ticket polling can rate-limit the user out of login and checkout.** It polls every 4 s with no guard against overlapping requests. The endpoint shares the 100-requests-per-5-minutes-per-IP limiter (`sensitiveActionRateLimiter`) with login and checkout. "VIEW" also opens a duplicate thread screen. | `profile_screen.dart:1506, 1534` | Poll every 15–30 s, skip while a request is in flight, and pause in the background. Exempt read-only requests from the sensitive limiter. |
| H10 | **Home feed jumps back to page 1 every 60 s** *(uncommitted changes)*. The background refresh calls `loadHomeProducts(reset: true)`. | `app_state.dart:1478, 542` | Update items in place without resetting. |
| H11 | **Made-up bundle prices on Home.** When a bundle has no price, Home shows `$110` and a crossed-out "original" price of 1.27 × that. The bundle detail page shows nothing in the same case, so the two screens disagree. | `home_screen.dart:778-790` | Use the bundle detail logic. |

---

## 🟡 Medium

1. **The delivery zone sent is always the first one,** whatever the address. With no zones the backend returns 422. → Let the customer choose a zone, or resolve it with `delivery-zones/resolve`. (`app_state.dart:938-959`)
2. **Disabled features still show.** `feature_is_active ?? store_is_active` ignores a store-level disable, so the app shows modules that then return 403. → Require both flags. (`app_state.dart:2685-2720`)
3. **Pre-order quantity can't go above 1 in the bag,** because the + button checks stock, and pre-orders have 0 stock. → Check against `preorderMaxQuantity` instead. (`cart_screen.dart:303-319`)
4. **Checkout summary shows the base product's price and name, not the variant's.** → Use `item.unitPrice` and `item.displayName`. (`checkout_screen.dart:399-431`)
5. **Editing an address selects the wrong one** (`savedAddresses.last`). (`checkout_screen.dart:1654`)
6. **The pickup slot can't be used or goes stale.** The default date can be today with no slots left, and a slot that has already passed isn't re-checked before submitting. (`checkout_screen.dart:37-54, 1247`)
7. **Category banners do nothing visible.** They set a filter but never switch to the catalog tab. (`home_screen.dart:1894-1944`)
8. **The cart isn't re-checked when the branch changes,** and a refresh resets the branch to the first one. Product loads for different branches can also race and overwrite each other. (`app_state.dart:584, 1327-1347`)
9. **Variants in the wholesale catalog are merged into one cart line with no `variant_id`.** (`storefront_api.dart:363, 475`)
10. **Android `usesCleartextTraffic="true"`,** and notification links open any URL in a WebView with JavaScript enabled. → Remove the cleartext flag and allow only https URLs on known hosts.
11. **Policies WebView calls `setState` after dispose** if the user presses back while the page loads, and a load error shows a blank page. (`policies_webview_screen.dart:33-46`)
12. **The web build breaks.** `webview_flutter` and `dart:io File` don't work on web, and the web title and manifest still say "Storefront Template".
13. **iOS and Android IDs differ:** `com.tarek.skincella` on iOS, `com.vendoor.skincella` on Android. Pick one before creating the store listings; it can't be changed later.
14. **"Mark all read" sends one request per notification** (which also counts against the rate limit), and it reports success before any of them finish.
15. **Uploads have no timeout or size check,** and the raw server response is shown in error toasts. (`storefront_api.dart:906-923, 1049-1072`)

## ⚪ Low / cleanup

- **Delivery note:** `notes: address` sends the branch or address name as the customer's note on every order. (`app_state.dart:960`)
- **Mixed carts:** `updateQuantity` skips the check that keeps pre-orders and regular items apart, so the backend rejects the order at checkout.
- **Fake size:** volume defaults to `'50ml'` (`product.dart:319`).
- **Order address:** it is shown as raw JSON, and pickup orders show the currently selected branch instead of the order's branch.
- **Error pages:** an HTML 502/504 response causes a `FormatException` instead of the app's normal API error.
- **Offline behaviour:**
  - Request storm: while offline, the home feed sends a new request on every scroll.
  - No recovery: after an offline start the app never reconnects on its own.
- **Promo quick-add:** there is no loading guard, so repeated taps add the item several times.
- **Cart share link:** it is hard-coded to `zeyy.app` and leaves out variants.
- **Sign-in redirect:** it is a dead end when the profile tab is disabled.
- **Broken text:** `orders_list_screen.dart:157` reads "sync your active s, order histories…".
- **Pre-orders screen:** it shows "No preorder items…" while products are still loading.
- **Bundle images:** they skip `resolveMediaUrl`, so relative image paths show a placeholder icon.
- **Login password length:** the client rejects passwords under 8 characters, which locks out older accounts with shorter passwords.
- **Signup screen:**
  - While loading, the button is `() {}` instead of `null`, so it still looks enabled.
  - The `TapGestureRecognizer` is never disposed.
- **Undisposed controllers:** some controllers in the profile screen are never disposed.
- **Template leftovers:**
  - `pubspec.yaml` still has `name: storefront_template`.
  - `rename` is listed as a runtime dependency.
  - The Android namespace is `com.example.storefront_template`.
- **Android camera:** the `CAMERA` permission without `uses-feature required=false` hides the app from devices without a camera.
- **App name:** "Skin Cella App" on iOS vs "Skin Cella" on Android.
- **Placeholder test:** `test/widget_test.dart` is `expect(true, true)`.
- **Missing contract tests:** no test checks request bodies against the backend.

## ✅ Checked and working

- Username registration and login: endpoints, payload and the username format rule all match the backend.
- Endpoint paths and methods, pagination, and the checkout response fields.
- The production base URL (`https://dashboard.zeyy.app/api/v1/storefront`).
- iOS camera and photo-library permission text.
- Android INTERNET permission.
- Android release signing setup; the key files are git-ignored.
- Mounted checks in the login, signup, edit-profile and address screens.
- No TODO, FIXME or lorem text left in the code.

## Recommended order of work

1. **C5, H1, H2:** cart loss and duplicate orders. These are quick fixes with the biggest impact on customers.
2. **C4, H3, H5, H11:** make checkout work for bundles and show correct prices.
3. **C1–C3:** a username-based account flow. This needs a small backend change: password-confirmed delete, a change-password endpoint, and a reset flow.
4. **H4, H6–H10:** fake ratings, session handling, support tickets and polling.
5. **Release config:** bundle IDs, cleartext traffic, template names, and contract tests against the real API.
