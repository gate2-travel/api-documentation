# Gate2Travel Hotel Booking API — Integration Guide

**Version:** 1.0  
**Last updated:** 2026-04-14  
**Audience:** Third-party companies integrating hotel search and booking via Gate2Travel

---

## Table of Contents

1. [Overview](#1-overview)
2. [Base URL](#2-base-url)
3. [Authentication](#3-authentication)
4. [Standard Response Format](#4-standard-response-format)
5. [Booking Flow](#5-booking-flow)
6. [Hotel Endpoints](#6-hotel-endpoints)
   - 6.1 [Search Hotels by Name](#61-search-hotels-by-name)
   - 6.2 [Get Countries](#62-get-countries)
   - 6.3 [Check Availability](#63-check-availability)
   - 6.4 [PreBook](#64-prebook)
   - 6.5 [Book](#65-book)
   - 6.6 [Cancel](#66-cancel)
   - 6.7 [Booking Detail](#67-booking-detail)
7. [Status Codes](#7-status-codes)
8. [Enumerations](#8-enumerations)
9. [Timeout Settings](#9-timeout-settings)
10. [Key Notes](#10-key-notes)

---

## 1. Overview

Gate2Travel provides a hotel booking API that lets third-party partners search live hotel inventory, pre-book rooms, confirm bookings, and manage cancellations. Gate2Travel acts as an intermediary reseller on top of TBO Holidays' hotel inventory, which covers hundreds of thousands of properties worldwide.

The typical integration flow is:

```
Authenticate → Search by name → Check availability → PreBook → Book → (Cancel if needed)
```

All requests must be made over **HTTPS**. All request and response bodies use **JSON**.

---

## 2. Base URL

```
https://api.gate2.travel
```

All endpoints described in this guide are relative to this base URL.

---

## 3. Authentication

Authentication is required before calling the API. Credentials are issued by Gate2Travel on a per-partner basis.

### 3.1 Login

Obtain an access token by sending your credentials.

| | |
|---|---|
| **Method** | `POST` |
| **URL** | `https://api.gate2.travel/ms-main/api/v1/auth` |
| **Auth** | HTTP Basic Auth (username + password provided by Gate2Travel) |
| **Content-Type** | `application/json` |

#### Request

Pass credentials using the HTTP `Authorization` header with Basic Auth encoding:

```
Authorization: Basic <base64(username:password)>
```

#### Response

```json
{
  "success": true,
  "message": "Operation successful",
  "data": {
    "accessToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
    "refreshToken": "550e8400-e29b-41d4-a716-446655440000"
  }
}
```

| Field | Type | Description |
|---|---|---|
| `data.accessToken` | String | JWT token. Pass as `Bearer` token in subsequent requests. |
| `data.refreshToken` | String | UUID token. Use to obtain a new access token when it expires. |

### 3.2 Using the Token

Include the access token in the `Authorization` header for all subsequent requests:

```
Authorization: Bearer <accessToken>
```

### 3.3 Refresh Token

Use the refresh token to obtain a new access token without re-entering credentials.

| | |
|---|---|
| **Method** | `POST` |
| **URL** | `https://api.gate2.travel/ms-main/api/v1/auth/refresh-token` |

#### Request

```json
{
  "refreshToken": "550e8400-e29b-41d4-a716-446655440000"
}
```

#### Response

Same structure as the login response — returns a new `accessToken` and `refreshToken`.

> **Token lifetime:** Access tokens expire after a short period. Refresh tokens are valid for 30 days. When the refresh token itself expires, re-authenticate using your credentials.

---

## 4. Standard Response Format

All API responses are wrapped in a common envelope:

```json
{
  "success": true,
  "message": "Operation successful",
  "data": { ... }
}
```

| Field | Type | Description |
|---|---|---|
| `success` | Boolean | `true` if the request was processed successfully. |
| `message` | String | Human-readable status message. |
| `data` | Object / Array | The response payload. Structure varies per endpoint. |
| `fieldErrors` | Array | Present only on validation failures. |

On error:

```json
{
  "success": false,
  "message": "Hotel not found"
}
```

---

## 5. Booking Flow

```
┌─────────────────────────────────────────────────────────────────────┐
│                      Complete Booking Flow                          │
│                                                                     │
│  1. POST /auth          → Obtain accessToken                        │
│                                                                     │
│  2. GET  /hotels/search → Find hotel IDs by name                   │
│     GET  /hotels/countries → (optional) resolve country codes       │
│                                                                     │
│  3. POST /hotels        → Check live availability & pricing         │
│         ↓  Returns BookingCode per available room                   │
│                                                                     │
│  4. POST /hotels/prebook → Confirm up-to-date price & policies      │
│         ↓  Returns updated BookingCode + cancellation policies      │
│                                                                     │
│  5. POST /hotels/book    → Confirm and voucher the booking          │
│         ↓  Returns ConfirmationNumber                               │
│                                                                     │
│  6. POST /hotels/booking-detail  → Retrieve booking status         │
│                                                                     │
│  7. POST /hotels/cancel  → (if needed) Cancel the booking          │
└─────────────────────────────────────────────────────────────────────┘
```

> **Important:** The entire flow from Availability Check to Book must be completed within **30 minutes**. The `BookingCode` returned in Search / PreBook has a session lifetime — do not store it and reuse it across sessions.

---

## 6. Hotel Endpoints

All hotel endpoints share the following base path:

```
https://api.gate2.travel/ms-main/api/v1/hotels
```

---

### 6.1 Search Hotels by Name

Use this endpoint to search for hotels in Gate2Travel's database by name. Returns matching hotel identifiers to use in the availability check.

| | |
|---|---|
| **Method** | `GET` |
| **URL** | `/ms-main/api/v1/hotels/search` |

#### Query Parameters

| Parameter | Type | Required | Description |
|---|---|---|---|
| `name` | String | Yes | Full or partial hotel name to search for. |

#### Example Request

```
GET https://api.gate2.travel/ms-main/api/v1/hotels/search?name=Hilton+Dubai
Authorization: Bearer <accessToken>
```

#### Example Response

```json
{
  "success": true,
  "message": "Operation successful",
  "data": [
    "Hilton Dubai Creek",
    "Hilton Dubai Jumeirah"
  ]
}
```

---

### 6.2 Get Countries

Returns the list of countries with their ISO codes. Useful for building UI dropdowns and mapping country names to codes required in availability requests.

| | |
|---|---|
| **Method** | `GET` |
| **URL** | `/ms-main/api/v1/hotels/countries` |

#### Query Parameters

| Parameter | Type | Required | Description |
|---|---|---|---|
| `name` | String | No | Filter countries by name (case-insensitive partial match). |

#### Example Request

```
GET https://api.gate2.travel/ms-main/api/v1/hotels/countries?name=United
Authorization: Bearer <accessToken>
```

#### Example Response

```json
{
  "success": true,
  "message": "Operation successful",
  "data": [
    { "code": "AE", "name": "United Arab Emirates" },
    { "code": "GB", "name": "United Kingdom" },
    { "code": "US", "name": "United States" }
  ]
}
```

---

### 6.3 Check Availability

Searches live hotel room availability for given dates and occupancy. Returns bookable rooms with pricing, cancellation policies, and a `BookingCode` required in the next steps.

| | |
|---|---|
| **Method** | `POST` |
| **URL** | `/ms-main/api/v1/hotels` |
| **Content-Type** | `application/json` |

#### Request Body

| Parameter | Type | Required | Description |
|---|---|---|---|
| `name` | String | Yes | Hotel name (as returned by the search endpoint). |
| `checkIn` | String | Yes | Check-in date. Format: `YYYY-MM-DD`. |
| `checkOut` | String | Yes | Check-out date. Format: `YYYY-MM-DD`. |
| `guestNationality` | String | Yes | Lead guest nationality. ISO 3166-1 alpha-2 code (e.g., `AE`, `GB`, `US`). |
| `paxRooms` | Array | Yes | Occupancy per room. See table below. |
| `filters` | Object | No | Optional filters. See table below. |

**`paxRooms` object:**

| Field | Type | Required | Description |
|---|---|---|---|
| `adults` | Integer | Yes | Number of adults (1–8). |
| `children` | Integer | No | Number of children (0–4). |
| `childrenAges` | Array of Integer | If children > 0 | Age of each child (0–18). Length must equal `children`. |

**`filters` object:**

| Field | Type | Default | Description |
|---|---|---|---|
| `refundable` | Boolean | `false` | If `true`, returns only refundable rooms. |
| `noOfRooms` | Integer | `0` | Max number of room options to return. `0` = no limit. |
| `mealType` | Enum | `All` | Filter by meal plan. Values: `All`, `WithMeal`, `RoomOnly`. |

#### Example Request (Single Room)

```json
{
  "name": "Hilton Dubai Creek",
  "checkIn": "2026-06-15",
  "checkOut": "2026-06-17",
  "guestNationality": "AE",
  "paxRooms": [
    {
      "adults": 2,
      "children": 1,
      "childrenAges": [8]
    }
  ],
  "filters": {
    "refundable": false,
    "noOfRooms": 5,
    "mealType": "All"
  }
}
```

#### Example Request (Multiple Rooms)

```json
{
  "name": "Hilton Dubai Creek",
  "checkIn": "2026-06-15",
  "checkOut": "2026-06-17",
  "guestNationality": "AE",
  "paxRooms": [
    { "adults": 2, "children": 0 },
    { "adults": 1, "children": 1, "childrenAges": [5] }
  ],
  "filters": {
    "refundable": false,
    "noOfRooms": 0,
    "mealType": "All"
  }
}
```

#### Response

The `data` field contains a list of hotel results, each with bookable rooms.

```json
{
  "success": true,
  "message": "Operation successful",
  "data": [
    {
      "hotelCode": "1120548",
      "hotelName": "Hilton Dubai Creek",
      "image": "https://...",
      "currency": "USD",
      "rooms": [
        {
          "name": ["Deluxe Room, 1 King Bed"],
          "bookingCode": "1120548!TB!2!TB!4ee85bb9-9ca6-4c66-8a8a-524bfdd5ae2b",
          "inclusion": "Free WiFi",
          "totalFare": 320.00,
          "totalTax": 52.10,
          "extraGuestCharges": "17.22",
          "recommendedSellingRate": "336.00",
          "roomPromotion": ["Early Bird"],
          "mealType": "Room_Only",
          "isRefundable": true,
          "withTransfers": false,
          "dayRates": [
            [{ "basePrice": 155.50 }],
            [{ "basePrice": 112.50 }]
          ],
          "cancelPolicies": [
            {
              "fromDate": "2026-06-01 00:00:00",
              "chargeType": "Fixed",
              "cancellationCharge": 0.0
            },
            {
              "fromDate": "2026-06-14 00:00:00",
              "chargeType": "Percentage",
              "cancellationCharge": 100.0
            }
          ],
          "supplements": [
            [
              {
                "index": 1,
                "type": "AtProperty",
                "description": "City tax",
                "price": 20.00,
                "currency": "AED"
              }
            ]
          ]
        }
      ]
    }
  ]
}
```

**Key response fields:**

| Field | Type | Description |
|---|---|---|
| `hotelCode` | String | Internal TBO hotel code. |
| `hotelName` | String | Hotel name enriched from Gate2Travel's database. |
| `image` | String | Hotel thumbnail image URL. |
| `currency` | String | Currency of all pricing fields. |
| `rooms[].name` | Array | Room name(s). For multi-room bookings the array has one name per room. |
| `rooms[].bookingCode` | String | **Required for PreBook and Book steps.** Session-scoped unique ID. |
| `rooms[].totalFare` | Decimal | Total price inclusive of tax. |
| `rooms[].totalTax` | Decimal | Tax portion of the total fare. |
| `rooms[].extraGuestCharges` | String | Additional charges per guest if applicable. |
| `rooms[].recommendedSellingRate` | String | Minimum allowed reselling price. You **must not** sell below this rate. |
| `rooms[].isRefundable` | Boolean | Whether the room allows free cancellation. |
| `rooms[].cancelPolicies` | Array | Cancellation charge schedule (see below). |
| `rooms[].supplements` | Array | Charges payable at the property. **Must be displayed to the end customer.** |
| `rooms[].mealType` | Enum | Included meal plan. See [Enumerations](#8-enumerations). |
| `rooms[].dayRates` | Array | Day-by-day price breakdown. Outer array = rooms, inner array = days. |

**Cancellation policy fields:**

| Field | Description |
|---|---|
| `fromDate` | Date from which this policy applies. Format: `DD-MM-YYYY HH:mm:ss`. |
| `chargeType` | `Fixed` (absolute amount) or `Percentage` (% of total fare). |
| `cancellationCharge` | Charge amount or percentage. `0` = free cancellation. |

---

### 6.4 PreBook

Validates that the selected room is still available and confirms the up-to-date price and cancellation policies. Always call PreBook before Book to prevent price discrepancies.

| | |
|---|---|
| **Method** | `POST` |
| **URL** | `/ms-main/api/v1/hotels/prebook` |
| **Content-Type** | `application/json` |

#### Request Body

| Parameter | Type | Required | Description |
|---|---|---|---|
| `bookingCode` | String | Yes | The `bookingCode` from the availability response. |

#### Example Request

```json
{
  "bookingCode": "1120548!TB!2!TB!4ee85bb9-9ca6-4c66-8a8a-524bfdd5ae2b"
}
```

#### Response

Same structure as the availability response but with additional fields:

```json
{
  "success": true,
  "message": "Operation successful",
  "data": [
    {
      "hotelCode": "1120548",
      "currency": "USD",
      "rooms": [
        {
          "name": ["Deluxe Room, 1 King Bed"],
          "bookingCode": "1120548!TB!2!TB!4ee85bb9-9ca6-4c66-8a8a-524bfdd5ae2b",
          "inclusion": "Free WiFi",
          "dayRates": [[{ "basePrice": 155.50 }], [{ "basePrice": 112.50 }]],
          "totalFare": 320.00,
          "totalTax": 52.10,
          "roomPromotion": ["Early Bird"],
          "cancelPolicies": [
            { "fromDate": "2026-06-01 00:00:00", "chargeType": "Fixed", "cancellationCharge": 0.0 },
            { "fromDate": "2026-06-14 00:00:00", "chargeType": "Percentage", "cancellationCharge": 100.0 }
          ],
          "mealType": "Room_Only",
          "isRefundable": true,
          "withTransfers": false,
          "amenities": [
            "Free WiFi",
            "Air conditioning",
            "Flat-panel TV",
            "In-room safe",
            "Daily housekeeping"
          ]
        }
      ],
      "rateConditions": [
        "Early check-out will attract full cancellation charge unless otherwise specified.",
        "CheckIn Time-Begin: 3:00 PM",
        "CheckOut Time: 12:00 PM"
      ]
    }
  ]
}
```

**Additional fields in PreBook response:**

| Field | Type | Description |
|---|---|---|
| `rooms[].amenities` | Array of String | Room-level amenity list. |
| `rateConditions` | Array of String | Hotel policies and check-in/check-out instructions. Must be shown to the customer. |

> **Important:** The `cancelPolicies` and `rateConditions` returned in PreBook are **final and binding** for the booking. Always present these to the customer before proceeding to Book.

> **Rate unavailability:** If the room is no longer available, you will receive a `success: false` response with an appropriate message. In that case, restart from the availability check.

---

### 6.5 Book

Confirms and vouchers the booking. This is the final step that charges the account and creates the reservation.

| | |
|---|---|
| **Method** | `POST` |
| **URL** | `/ms-main/api/v1/hotels/book` |
| **Content-Type** | `application/json` |

#### Request Body

| Parameter | Type | Required | Description |
|---|---|---|---|
| `bookingCode` | String | Yes | `bookingCode` from the PreBook response. |
| `customerDetails` | Array | Yes | Guest details, one object per room. |
| `customerDetails[].customerNames` | Array | Yes | Guest name objects for that room. |
| `customerDetails[].customerNames[].title` | String | Yes | `Mr`, `Mrs`, or `Ms`. |
| `customerDetails[].customerNames[].firstName` | String | Yes | Guest first name. |
| `customerDetails[].customerNames[].lastName` | String | Yes | Guest last name. |
| `customerDetails[].customerNames[].type` | String | Yes | `Adult` or `Child`. |
| `clientReferenceId` | String | Yes | Your internal reference ID for this booking. |
| `bookingReferenceId` | String | No | Additional reference ID (e.g., your PNR or order ID). |
| `totalFare` | Decimal | Yes | Total fare as received in the PreBook response. |
| `emailId` | String | Yes | Guest email address for the voucher. |
| `phoneNumber` | String | Yes | Guest phone number (with country code, no spaces). |
| `bookingType` | String | No | Default: `Voucher`. |

#### Example Request (Single Room)

```json
{
  "bookingCode": "1120548!TB!2!TB!4ee85bb9-9ca6-4c66-8a8a-524bfdd5ae2b",
  "customerDetails": [
    {
      "customerNames": [
        {
          "title": "Mr",
          "firstName": "John",
          "lastName": "Smith",
          "type": "Adult"
        },
        {
          "title": "Ms",
          "firstName": "Emma",
          "lastName": "Smith",
          "type": "Child"
        }
      ]
    }
  ],
  "clientReferenceId": "ORDER-2026-001234",
  "bookingReferenceId": "PNR-XYZ123",
  "totalFare": 320.00,
  "emailId": "john.smith@example.com",
  "phoneNumber": "971501234567",
  "bookingType": "Voucher"
}
```

#### Example Request (Multiple Rooms)

```json
{
  "bookingCode": "1120548!TB!4!TB!8bd7a82e-439a-4b2d-869d-09de4456e482",
  "customerDetails": [
    {
      "customerNames": [
        { "title": "Mr", "firstName": "John", "lastName": "Smith", "type": "Adult" }
      ]
    },
    {
      "customerNames": [
        { "title": "Mrs", "firstName": "Sara", "lastName": "Jones", "type": "Adult" }
      ]
    }
  ],
  "clientReferenceId": "ORDER-2026-001235",
  "bookingReferenceId": "PNR-ABC456",
  "totalFare": 640.00,
  "emailId": "john.smith@example.com",
  "phoneNumber": "971501234567",
  "bookingType": "Voucher"
}
```

#### Response

```json
{
  "success": true,
  "message": "Operation successful",
  "data": {
    "status": {
      "code": 200,
      "description": "Successful"
    },
    "clientReferenceId": "ORDER-2026-001234",
    "confirmationNumber": "FL1IMA"
  }
}
```

| Field | Type | Description |
|---|---|---|
| `data.confirmationNumber` | String | **The booking's unique confirmation number.** Store this — it is required for cancellation and retrieving booking details. |
| `data.clientReferenceId` | String | Echo of your reference ID. |

---

### 6.6 Cancel

Cancels an existing confirmed booking.

| | |
|---|---|
| **Method** | `POST` |
| **URL** | `/ms-main/api/v1/hotels/cancel` |
| **Content-Type** | `application/json` |

#### Request Body

| Parameter | Type | Required | Description |
|---|---|---|---|
| `confirmationNumber` | String | Yes | The confirmation number returned by the Book endpoint. |

#### Example Request

```json
{
  "confirmationNumber": "FL1IMA"
}
```

#### Response

```json
{
  "success": true,
  "message": "Operation successful",
  "data": {
    "status": {
      "code": 200,
      "description": "Cancelled"
    },
    "confirmationNumber": "FL1IMA"
  }
}
```

> **Note:** Cancellation charges apply as per the `cancelPolicies` received in the PreBook response. A non-refundable booking will still be cancelled but the full fare may be charged.

---

### 6.7 Booking Detail

Retrieves the full details and current status of an existing booking.

| | |
|---|---|
| **Method** | `POST` |
| **URL** | `/ms-main/api/v1/hotels/booking-detail` |
| **Content-Type** | `application/json` |

#### Request Body

| Parameter | Type | Required | Description |
|---|---|---|---|
| `confirmationNumber` | String | Yes | The confirmation number returned by the Book endpoint. |

#### Example Request

```json
{
  "confirmationNumber": "FL1IMA"
}
```

#### Response

```json
{
  "success": true,
  "message": "Operation successful",
  "data": {
    "Status": { "Code": 200, "Description": "Successful" },
    "BookingDetail": {
      "BookingStatus": "Confirmed",
      "VoucherStatus": true,
      "ConfirmationNumber": "FL1IMA",
      "HotelConfirmationNumber": "HTL-987654",
      "InvoiceNumber": "INV-001234",
      "CheckIn": "2026-06-15T00:00:00",
      "CheckOut": "2026-06-17T00:00:00",
      "BookingDate": "2026-04-14T00:00:00",
      "NoOfRooms": 1,
      "HotelDetails": {
        "HotelName": "Hilton Dubai Creek",
        "Rating": "FiveStar",
        "Map": "25.2532|55.3286",
        "City": "Dubai"
      },
      "Rooms": [
        {
          "Currency": "USD",
          "Name": ["Deluxe Room, 1 King Bed"],
          "Inclusion": "Free WiFi",
          "TotalFare": 320.00,
          "TotalTax": 52.10,
          "MealType": "Room_Only",
          "IsRefundable": true,
          "CancelPolicies": [
            { "FromDate": "2026-06-14 00:00:00", "ChargeType": "Percentage", "CancellationCharge": 100.0 }
          ],
          "CustomerDetails": [
            {
              "CustomerNames": [
                { "Title": "Mr", "FirstName": "John", "LastName": "Smith", "Type": "Adult" }
              ]
            }
          ]
        }
      ]
    }
  }
}
```

**Booking status values:**

| Status | Description |
|---|---|
| `Confirmed` | Booking is confirmed. |
| `Cancelled` | Booking has been cancelled. |
| `CancellationInProgress` | Cancellation request is being processed. |
| `CancelPending` | Cancellation is pending. |
| `CxlRequestSentToHotel` | Cancellation request sent to the hotel. |
| `CancelledAndRefundAwaited` | Cancelled; refund is pending. |

---

## 7. Status Codes

These status codes may appear inside the `data` object returned by the underlying booking operations:

| Code | Name | Description |
|---|---|---|
| `200` | `SUCCESS` | Operation completed successfully. |
| `201` | `NO_AVAILABILITY` | No rooms available for the given criteria. |
| `207` | `RATE_UNAVAILABLE` | The selected rate is no longer available. |
| `300` | `INSUFFICIENT_BALANCE` | Partner account has insufficient funds. |
| `315` | `BOOKINGCODE_EXPIRED` | Session expired. The `BookingCode` is no longer valid. Restart from availability. |
| `400` | `INVALID_REQUEST` | A required parameter is missing or has an invalid value. |
| `401` | `UNAUTHORIZED` | Invalid or missing credentials / access token. |
| `402` | `AGENT_BLOCKED` | Partner account is blocked. Contact Gate2Travel support. |
| `405` | `BOOKING_FAIL` | Booking could not be created. |
| `429` | `LIMIT_EXCEEDED` | Request rate limit exceeded. Implement back-off and retry. |
| `479` | `CANCEL_FAIL` | Cancellation could not be completed. |
| `500` | `UNEXPECTED_ERROR` | Unexpected server error. Contact Gate2Travel support with the full request/response logs. |

---

## 8. Enumerations

### MealType

| Value | Description |
|---|---|
| `Room_Only` | No meals included. |
| `BreakFast` | Breakfast included. |
| `Breakfast_For_1` | Breakfast for one guest. |
| `Breakfast_For_2` | Breakfast for two guests. |
| `Lunch` | Lunch included. |
| `Dinner` | Dinner included. |
| `BreakFast_Lunch` | Breakfast and lunch included. |
| `Half_Board` | Breakfast and one main meal. |
| `Full_Board` | All main meals included. |
| `All_Inclusive_All_Meal` | All-inclusive. |

### StarRating

`OneStar`, `TwoStar`, `ThreeStar`, `FourStar`, `FiveStar`

### MealPlan filter (in availability request)

| Value | Description |
|---|---|
| `All` | Return all meal types. |
| `WithMeal` | Return only rooms with meals. |
| `RoomOnly` | Return only room-only rates. |

### Supplement Type

| Value | Description |
|---|---|
| `Included` | Charge is included in the total fare. |
| `AtProperty` | Charge must be paid directly at the hotel. **Must be shown to the customer.** |

---

## 9. Timeout Settings

| API | Recommended Client Timeout |
|---|---|
| Availability (Search) | 23 seconds |
| PreBook | 23 seconds |
| Book | 120 seconds |
| Cancel | 30 seconds |
| Booking Detail | 30 seconds |

The complete flow from Availability to Book must be completed within **30 minutes** of the initial search.

---

## 10. Key Notes

1. **Never hardcode guest nationality.** Always accept the actual guest's nationality. Incorrect nationality may lead to pricing or operational issues.

2. **Always call PreBook before Book.** Prices and policies can change between the availability search and the time of booking. PreBook confirms the current state.

3. **Cancellation policies are final from PreBook.** The policies returned in the PreBook response are the definitive terms for the booking.

4. **Display supplements marked `AtProperty` to the customer.** These are additional charges the guest must pay directly at the hotel. Failing to communicate them may lead to guest disputes.

5. **Display `rateConditions`.** These hotel-specific policies (check-in/check-out times, mandatory fees, etc.) must be presented to the customer before confirming the booking.

6. **`BookingCode` is session-scoped.** Do not cache or reuse a `BookingCode` across separate user sessions. If the session expires (status `315`), restart from availability.

7. **`recommendedSellingRate`.** The TBO-set minimum reselling price. You may not sell the room at a rate lower than this value.

8. **`clientReferenceId`** should be unique per booking and traceable in your own system. It is echoed back in the Book response and included in booking detail calls.

9. **Contact:** For API errors with status `500`, send the full JSON request and response to Gate2Travel support for investigation.

---

*This document covers the hotel booking module. For flight booking and eSIM product integration, refer to the respective module guides.*
