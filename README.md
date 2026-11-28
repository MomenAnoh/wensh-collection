# Wensh API collection

This collection contains the HTTP API routes registered under `/api/v1`.
Choose a Bruno environment to set `userUrl` and `driverUrl`; both URLs should end at the corresponding API prefix. Requests accept JSON and return JSON. Send `Accept-Language: {{locale}}` to select `en` or `ar` where supported.

## Authentication

User and driver OTP flows are separate. Send an OTP, then verify the six-digit code. Successful verification stores the returned bearer token in the runtime variable `user_token` or `driver_token` for subsequent requests in that flow. Send it as `Authorization: Bearer <token>` on protected routes. `user_token` and `driver_token` are independent.

OTP send and verify routes are limited to 10 requests per minute. Local/testing environments may include the OTP in the send response; production delivery may omit it.

Driver registration requires a verified phone. OTP verification returns a short-lived `registration_token` for new drivers; submit that token with the registration form instead of entering the phone number again. Registration uses `multipart/form-data`; new drivers start with `verification_status: pending`. Driver offer routes require an authenticated, approved driver. Registration status is available to authenticated drivers before approval.

## User API

| Method | Path | Auth | Description / input |
|---|---|---|---|
| POST | `/auth/otp/send` | Public | JSON `{ "phone": "+201000000000" }`; phone must be E.164-like (`+` and 8–15 digits). |
| POST | `/auth/otp/verify` | Public | JSON `{ "phone": "+201000000000", "code": "123456" }`; returns bearer token and user. |
| GET | `/auth/me` | User | Current user profile. |
| POST | `/auth/logout` | User | Revoke current user token. |
| DELETE | `/auth/account` | User | Soft-delete the account and invalidate the current token. Verifying the same phone by OTP later restores the existing account. |
| GET | `/chat/messages` | User | Paginated chat history with the system. |
| POST | `/chat/messages` | User | JSON `{ "message": "..." }` (maximum 4000 characters); Laravel persists it and emits `chat_message_created` to Socket.IO. |
| GET | `/chat/socket-ticket` | User | Get a five-minute signed Socket.IO ticket; connect with `auth.ticket` and listen for `chat_message_created`. |
| POST | `/auth/refresh` | User | Refresh current user token. |
| GET | `/content/terms-and-conditions` | Public | Localized terms content. |
| GET | `/content/privacy-policy` | Public | Localized privacy content. |
| GET | `/content/about-us` | Public | Localized about content. |
| GET | `/content/faqs` | Public | Ordered FAQ list localized using `Accept-Language` (`ar` or `en`). |
| GET | `/complaint-types` | Public | Active, localized complaint types for the user audience. |
| POST | `/complaints` | User | Multipart `complaint_type_id`, `description` (max 5000 characters), and optional `image` (JPG/JPEG/PNG/WEBP, max 5 MB). Only active user complaint types are accepted; limited to 10 requests/minute. |
| GET | `/car-types` | Public | Paginated car types. Optional `page` and `per_page`; `per_page` is clamped to 1–50 (default 15). |
| POST | `/order/calc-suggested-price` | User | JSON `from_latitude`, `from_longitude`, `to_latitude`, `to_longitude`; valid coordinate ranges are −90..90 and −180..180. |
| POST | `/order` | User | Create an order. Required `car_type_id`, `from_address`, `to_address`, four coordinates, `type` (`normal` or `scheduled`), `payment_methods` (`cash` or `online`), and non-negative `price`. For `scheduled`, also provide `scheduled_day` (today or later) and `scheduled_time` (`HH:mm`). For `online`, the response contains only three signed links: `payment_url` (GET), `success_url` (POST), and `failure_url` (POST), valid for 15 minutes; it omits the order resource. Success sets order/offer to `accepted`; failure returns them to `pending` and broadcasts the order to drivers. |
| POST | `/order/offers` | User | Counter offer JSON `{ "order_id": 22, "price": 140 }`; price must be greater than zero. |
| GET | `/profile` | User | Get profile and notification preferences. |
| PUT | `/profile` | User | Update `name` (required), optional `email`, optional E.164-like `emergency_contact_phone`, and optional `profile_photo` (JPG/PNG/WEBP, max 5 MB). Use multipart form data when uploading the photo. |
| POST | `/profile/skip` | User | Mark profile setup as skipped. |
| PATCH | `/notifications/requests` | User | JSON `{ "enabled": true }` to set request notifications. |
| PATCH | `/notifications/offers` | User | JSON `{ "enabled": true }` to set offer notifications. |

## Driver API

| Method | Path | Auth | Description / input |
|---|---|---|---|
| POST | `/auth/otp/send` | Public | JSON `{ "phone_code": "+20", "phone": "1000000000" }`; phone code starts with `+`, phone contains 7–15 digits. |
| POST | `/auth/otp/verify` | Public | JSON `phone_code`, `phone`, and six-digit `code`. Existing drivers receive a token; new numbers receive `registration_required: true`. |
| POST | `/auth/register` | Public after OTP verification | Multipart form: `registration_token` (returned by OTP verification), `full_name`, optional `email`, `photo` (JPG/PNG, max 5 MB), `license_image` (JPG/PNG/PDF, max 5 MB), `truck_plate_number`, `truck_image` (JPG/PNG, max 5 MB), and `coverage_radius_km` (integer 1–400). Token expires with OTP verification (5 minutes) and is single-use. |
| GET | `/auth/status` | Driver | Return verification status and optional rejection reason. |
| GET | `/content/terms-and-conditions` | Public | Localized terms content for Wensh providers. |
| GET | `/content/privacy-policy` | Public | Localized privacy content for Wensh providers. |
| GET | `/content/about-us` | Public | Localized about content for Wensh providers. |
| GET | `/content/faqs` | Public | Ordered Wensh FAQ list localized using `Accept-Language` (`ar` or `en`). |
| GET | `/complaint-types` | Public | Active, localized complaint types for Wensh providers. |
| POST | `/complaints` | Driver | Multipart `complaint_type_id`, `description` (max 5000 characters), and optional `image` (JPG/JPEG/PNG/WEBP, max 5 MB). Only active Wensh complaint types are accepted; limited to 10 requests/minute. |
| GET | `/offers` | Approved driver | Paginated offers assigned to the authenticated Wensh. Optional `filter=completed` or `filter=cancelled` filters by the related order's status; omit `filter` for all order statuses. Optional `per_page` is clamped to 1–50 (default 15). |
| GET | `/offers/all` | Approved driver | Paginated list of offers across Wenshes. Optional `per_page` is clamped to 1–50 (default 15). |
| POST | `/offers` | Approved driver | JSON `{ "order_id": 22, "price": 120 }`; price must be greater than zero. |
| PATCH | `/offers/{offer}/accept` | Approved driver | Accept the specified pending user offer. Set the path offer ID before sending. |

## Responses and errors

Successful responses use `{ "message": "...", "data": ... }`. Validation failures use Laravel's JSON validation response (HTTP 422). Invalid state transitions for order/offer operations also return HTTP 422. Protected routes require a valid bearer token; driver offer creation and acceptance additionally require driver approval.
