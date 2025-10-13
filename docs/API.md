---
layout: default
title: API Documentation
nav_order: 2
---

# API Documentation

## Overview

This document provides a comprehensive reference for all API endpoints available in the UPSC Exam Preparation Backend.

### Base URLs

- **Local Development**: `http://localhost:3000/api/v1`
- **Production**: `http://upsc-alb-590268270.ap-south-1.elb.amazonaws.com/api/v1`

### Authentication

Most endpoints require JWT Bearer token authentication obtained through AWS Cognito OAuth providers (Google/Apple).

```http
Authorization: Bearer <your_jwt_token>
```

### Swagger Documentation

Interactive API documentation is available at `/api/docs`

- **Username**: `admin` (or value from `SWAGGER_USERNAME` env variable)
- **Password**: Set via `SWAGGER_PASSWORD` environment variable

### Response Format

All paginated responses follow this standard format:

```json
{
  "data": [...],
  "meta": {
    "total": 150,
    "page": 1,
    "limit": 10,
    "totalPages": 15,
    "hasNext": true,
    "hasPrev": false
  }
}
```

### Error Responses

Standard error response format:

```json
{
  "statusCode": 400,
  "message": "Error description",
  "error": "Bad Request",
  "timestamp": "2025-10-10T10:30:00.000Z",
  "path": "/api/v1/endpoint"
}
```

---

## 1. Authentication Module

Base path: `/auth`

### 1.1 Exchange Cognito Tokens

Exchange OAuth provider tokens from AWS Cognito for custom backend JWT tokens.

**Endpoint**: `POST /auth/cognito/exchange`
**Authentication**: Public (No token required)
**Rate Limit**: 5 requests per minute

#### Request Body

```json
{
  "idToken": "eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9...",
  "accessToken": "eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9...",
  "refreshToken": "eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9...",
  "provider": "google",
  "deviceInfo": {
    "userAgent": "Mozilla/5.0...",
    "ipAddress": "192.168.1.1",
    "deviceId": "device-uuid-123"
  }
}
```

**Field Descriptions:**

- `idToken` (required): Cognito ID token from OAuth provider
- `accessToken` (required): Cognito access token from OAuth provider
- `refreshToken` (required): Cognito refresh token from OAuth provider
- `provider` (required): OAuth provider used - `"google"` or `"apple"`
- `deviceInfo` (optional): Device information object

#### Response (200 OK)

```json
{
  "success": true,
  "accessToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "refreshToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "expiresIn": 3600,
  "user": {
    "id": "user-uuid-123",
    "email": "user@example.com",
    "name": "John Doe",
    "isVerified": true,
    "provider": "google",
    "profilePicture": "https://example.com/profile.jpg",
    "profileCompletionPercentage": 75,
    "onboardingCompleted": true,
    "status": "ACTIVE"
  }
}
```

#### Error Responses

- `400 Bad Request` - Invalid token or provider
- `401 Unauthorized` - Token validation failed
- `429 Too Many Requests` - Rate limit exceeded

---

### 1.2 Refresh Access Token

Refresh an expired access token using a refresh token.

**Endpoint**: `POST /auth/refresh`
**Authentication**: Public (No token required)

#### Request Body

```json
{
  "refreshToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
}
```

#### Response (200 OK)

```json
{
  "success": true,
  "accessToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "refreshToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "expiresIn": 3600
}
```

#### Error Responses

- `401 Unauthorized` - Invalid refresh token

---

### 1.3 Get User Profile

Get current authenticated user's profile information.

**Endpoint**: `GET /auth/profile`
**Authentication**: Required (JWT Bearer Token)

#### Response (200 OK)

```json
{
  "id": "user-uuid-123",
  "email": "user@example.com",
  "name": "John Doe",
  "isVerified": true,
  "provider": "google",
  "profilePicture": "https://example.com/profile.jpg",
  "profileCompletionPercentage": 75,
  "onboardingCompleted": true,
  "status": "ACTIVE",
  "createdAt": "2025-01-15T10:30:00.000Z"
}
```

#### Error Responses

- `401 Unauthorized` - Invalid or expired token

---

### 1.4 Sign Out

Sign out from the current session.

**Endpoint**: `POST /auth/signout`
**Authentication**: Required (JWT Bearer Token)

#### Response (200 OK)

```json
{
  "success": true,
  "message": "Signed out successfully"
}
```

---

### 1.5 Revoke All Sessions

Sign out from all devices/sessions.

**Endpoint**: `POST /auth/revoke-all`
**Authentication**: Required (JWT Bearer Token)

#### Response (200 OK)

```json
{
  "success": true,
  "message": "All sessions revoked successfully"
}
```

---

### 1.6 Get Active Sessions

Get all active sessions for the current user.

**Endpoint**: `GET /auth/sessions`
**Authentication**: Required (JWT Bearer Token)

#### Response (200 OK)

```json
{
  "success": true,
  "sessions": [
    {
      "id": "session-uuid-123",
      "deviceInfo": {
        "userAgent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64)...",
        "ipAddress": "192.168.1.100",
        "deviceId": "device-uuid-456"
      },
      "createdAt": "2025-10-10T08:00:00.000Z",
      "lastAccessedAt": "2025-10-10T10:30:00.000Z",
      "expiresAt": "2025-10-11T08:00:00.000Z",
      "isCurrent": true
    }
  ]
}
```

---

## 2. User Profile Module

Base path: `/user`

### 2.1 Get Current User Info

Get basic current user information from JWT token.

**Endpoint**: `GET /user/me`
**Authentication**: Required (JWT Bearer Token)

#### Response (200 OK)

```json
{
  "id": "user-uuid-123",
  "email": "user@example.com",
  "name": "John Doe",
  "isVerified": true,
  "provider": "google"
}
```

---

### 2.2 Get Current User Profile

Get the full profile of the current user.

**Endpoint**: `GET /user/profile/me`
**Authentication**: Required (JWT Bearer Token)

#### Response (200 OK)

```json
{
  "id": "profile-uuid-123",
  "userId": "user-uuid-123",
  "phoneNumber": "+919876543210",
  "targetExamYear": 2026,
  "preferredLanguage": "ENGLISH",
  "onboardingCompleted": true,
  "psychometricTestCompleted": true,
  "profileCompletionPercentage": 85,
  "createdAt": "2025-01-15T10:30:00.000Z",
  "updatedAt": "2025-10-10T10:30:00.000Z"
}
```

#### Error Responses

- `404 Not Found` - Profile not found

---

### 2.3 Create User Profile

Create a new user profile. Can only be called once per user.

**Endpoint**: `POST /user/profile`
**Authentication**: Required (JWT Bearer Token)

#### Request Body

```json
{
  "phoneNumber": "+919876543210",
  "targetExamYear": 2026,
  "preferredLanguage": "ENGLISH"
}
```

#### Response (201 Created)

```json
{
  "id": "profile-uuid-123",
  "userId": "user-uuid-123",
  "phoneNumber": "+919876543210",
  "targetExamYear": 2026,
  "preferredLanguage": "ENGLISH",
  "onboardingCompleted": false,
  "psychometricTestCompleted": false,
  "profileCompletionPercentage": 30,
  "createdAt": "2025-10-10T10:30:00.000Z"
}
```

#### Error Responses

- `400 Bad Request` - Validation failed
- `409 Conflict` - Profile already exists or unique constraint violation

---

### 2.4 Update Current User Profile

Update the current user's profile.

**Endpoint**: `PATCH /user/profile/me`
**Authentication**: Required (JWT Bearer Token)

#### Request Body

```json
{
  "phoneNumber": "+919876543211",
  "targetExamYear": 2027,
  "preferredLanguage": "HINDI"
}
```

#### Response (200 OK)

```json
{
  "id": "profile-uuid-123",
  "userId": "user-uuid-123",
  "phoneNumber": "+919876543211",
  "targetExamYear": 2027,
  "preferredLanguage": "HINDI",
  "profileCompletionPercentage": 85,
  "updatedAt": "2025-10-10T11:00:00.000Z"
}
```

#### Error Responses

- `404 Not Found` - Profile not found
- `409 Conflict` - Unique constraint violation

---

### 2.5 Complete Onboarding

Mark user onboarding as complete.

**Endpoint**: `POST /user/profile/onboarding/complete`
**Authentication**: Required (JWT Bearer Token)

#### Response (200 OK)

```json
{
  "id": "profile-uuid-123",
  "onboardingCompleted": true,
  "profileCompletionPercentage": 90,
  "updatedAt": "2025-10-10T11:00:00.000Z"
}
```

---

### 2.6 Complete Psychometric Test

Mark psychometric test as complete.

**Endpoint**: `POST /user/profile/psychometric/complete`
**Authentication**: Required (JWT Bearer Token)

#### Response (200 OK)

```json
{
  "id": "profile-uuid-123",
  "psychometricTestCompleted": true,
  "profileCompletionPercentage": 100,
  "updatedAt": "2025-10-10T11:00:00.000Z"
}
```

---

### 2.7 Get All User Profiles (Admin Only)

Get paginated list of all user profiles. Requires ADMIN or SUPER_ADMIN role.

**Endpoint**: `GET /user/profiles`
**Authentication**: Required (JWT Bearer Token + Admin Role)

#### Query Parameters

- `page` (optional, default: 1): Page number
- `limit` (optional, default: 10): Items per page

#### Request Example

```
GET /user/profiles?page=1&limit=20
```

#### Response (200 OK)

```json
{
  "data": [
    {
      "id": "profile-uuid-123",
      "userId": "user-uuid-123",
      "phoneNumber": "+919876543210",
      "targetExamYear": 2026,
      "profileCompletionPercentage": 85
    }
  ],
  "meta": {
    "total": 150,
    "page": 1,
    "limit": 20,
    "totalPages": 8,
    "hasNext": true,
    "hasPrev": false
  }
}
```

---

### 2.8 Get User Profile by ID (Admin Only)

Get a specific user profile by ID. Requires ADMIN or SUPER_ADMIN role.

**Endpoint**: `GET /user/profile/:id`
**Authentication**: Required (JWT Bearer Token + Admin Role)

#### Response (200 OK)

```json
{
  "id": "profile-uuid-123",
  "userId": "user-uuid-123",
  "phoneNumber": "+919876543210",
  "targetExamYear": 2026,
  "preferredLanguage": "ENGLISH",
  "onboardingCompleted": true,
  "psychometricTestCompleted": true,
  "profileCompletionPercentage": 85
}
```

#### Error Responses

- `404 Not Found` - Profile not found

---

### 2.9 Update User Profile by ID (Admin Only)

Update a user profile by ID. Requires ADMIN or SUPER_ADMIN role.

**Endpoint**: `PATCH /user/profile/:id`
**Authentication**: Required (JWT Bearer Token + Admin Role)

#### Request Body

```json
{
  "targetExamYear": 2027,
  "onboardingCompleted": true
}
```

#### Response (200 OK)

```json
{
  "id": "profile-uuid-123",
  "targetExamYear": 2027,
  "onboardingCompleted": true,
  "updatedAt": "2025-10-10T11:00:00.000Z"
}
```

---

### 2.10 Delete User Profile by ID (Super Admin Only)

Delete a user profile by ID. Requires SUPER_ADMIN role.

**Endpoint**: `DELETE /user/profile/:id`
**Authentication**: Required (JWT Bearer Token + Super Admin Role)

#### Response (200 OK)

```json
{
  "success": true,
  "message": "Profile deleted successfully"
}
```

#### Error Responses

- `404 Not Found` - Profile not found

---

## 3. Previous Year Questions (PYQ) Module

Base path: `/pyq`

### 3.1 Get All PYQ Papers

Get paginated list of PYQ papers with optional filtering.

**Endpoint**: `GET /pyq`
**Authentication**: Required (JWT Bearer Token)

#### Query Parameters

- `page` (optional, default: 1): Page number
- `limit` (optional, default: 10): Items per page
- `subject` (optional): Filter by subject (e.g., "History", "Geography")
- `year` (optional): Filter by year (e.g., 2023)
- `difficulty` (optional): Filter by difficulty - `EASY`, `MEDIUM`, `HARD`, `EXPERT`

#### Request Example

```
GET /pyq?page=1&limit=10&subject=History&year=2023&difficulty=MEDIUM
```

#### Response (200 OK)

```json
{
  "data": [
    {
      "id": "pyq-uuid-123",
      "title": "UPSC Prelims 2023 - History",
      "subject": "History",
      "year": 2023,
      "difficulty": "MEDIUM",
      "totalQuestions": 100,
      "duration": 7200,
      "createdAt": "2023-06-01T00:00:00.000Z",
      "updatedAt": "2023-06-01T00:00:00.000Z"
    }
  ],
  "meta": {
    "total": 45,
    "page": 1,
    "limit": 10,
    "totalPages": 5,
    "hasNext": true,
    "hasPrev": false
  }
}
```

---

### 3.2 Get PYQ Paper by ID

Get a specific PYQ paper by its ID.

**Endpoint**: `GET /pyq/:id`
**Authentication**: Required (JWT Bearer Token)

#### Path Parameters

- `id` (required): PYQ paper ID (UUID)

#### Response (200 OK)

```json
{
  "id": "pyq-uuid-123",
  "title": "UPSC Prelims 2023 - History",
  "subject": "History",
  "year": 2023,
  "difficulty": "MEDIUM",
  "totalQuestions": 100,
  "duration": 7200,
  "instructions": "Read all questions carefully...",
  "questions": [
    {
      "id": "question-uuid-456",
      "questionText": "Who was the founder of Maurya Empire?",
      "options": ["Chandragupta Maurya", "Ashoka", "Bindusara", "Brihadratha"],
      "correctAnswer": 0,
      "explanation": "Chandragupta Maurya founded the Maurya Empire...",
      "marks": 2
    }
  ],
  "createdAt": "2023-06-01T00:00:00.000Z",
  "updatedAt": "2023-06-01T00:00:00.000Z"
}
```

#### Error Responses

- `400 Bad Request` - Invalid UUID format
- `404 Not Found` - PYQ paper not found

---

### 3.3 Create PYQ Paper (Admin Only)

Create a new PYQ paper.

**Endpoint**: `POST /pyq`
**Authentication**: Required (JWT Bearer Token + Admin Role)

#### Request Body

```json
{
  "title": "UPSC Prelims 2024 - History",
  "subject": "History",
  "year": 2024,
  "difficulty": "MEDIUM",
  "totalQuestions": 100,
  "duration": 7200,
  "instructions": "Read all questions carefully..."
}
```

#### Response (201 Created)

```json
{
  "id": "pyq-uuid-789",
  "title": "UPSC Prelims 2024 - History",
  "subject": "History",
  "year": 2024,
  "difficulty": "MEDIUM",
  "totalQuestions": 100,
  "duration": 7200,
  "createdAt": "2025-10-10T10:30:00.000Z"
}
```

#### Error Responses

- `400 Bad Request` - Invalid input data
- `403 Forbidden` - Insufficient permissions

---

### 3.4 Update PYQ Paper

Update an existing PYQ paper.

**Endpoint**: `PATCH /pyq/:id`
**Authentication**: Required (JWT Bearer Token + Admin Role)

#### Path Parameters

- `id` (required): PYQ paper ID (UUID)

#### Request Body

```json
{
  "title": "UPSC Prelims 2024 - History (Updated)",
  "difficulty": "HARD"
}
```

#### Response (200 OK)

```json
{
  "id": "pyq-uuid-789",
  "title": "UPSC Prelims 2024 - History (Updated)",
  "difficulty": "HARD",
  "updatedAt": "2025-10-10T11:00:00.000Z"
}
```

---

### 3.5 Delete PYQ Paper

Delete a PYQ paper.

**Endpoint**: `DELETE /pyq/:id`
**Authentication**: Required (JWT Bearer Token + Admin Role)

#### Path Parameters

- `id` (required): PYQ paper ID (UUID)

#### Response (200 OK)

```json
{
  "success": true,
  "message": "PYQ paper deleted successfully"
}
```

#### Error Responses

- `404 Not Found` - PYQ paper not found

---

## 4. Mains Papers Module

Base path: `/mains`

### 4.1 Get All Mains Papers

Get paginated list of Mains papers with optional filtering.

**Endpoint**: `GET /mains`
**Authentication**: Required (JWT Bearer Token)

#### Query Parameters

- `page` (optional, default: 1): Page number
- `limit` (optional, default: 10): Items per page
- `subject` (optional): Filter by subject (e.g., "History", "Geography")
- `year` (optional): Filter by year (e.g., 2023)
- `difficulty` (optional): Filter by difficulty - `EASY`, `MEDIUM`, `HARD`, `EXPERT`

#### Request Example

```
GET /mains?page=1&limit=10&subject=Essay&year=2023&difficulty=MEDIUM
```

#### Response (200 OK)

```json
{
  "data": [
    {
      "id": "mains-uuid-123",
      "title": "UPSC Mains 2023 - Essay",
      "subject": "Essay",
      "year": 2023,
      "difficulty": "MEDIUM",
      "totalQuestions": 8,
      "duration": 10800,
      "createdAt": "2023-09-01T00:00:00.000Z",
      "updatedAt": "2023-09-01T00:00:00.000Z"
    }
  ],
  "meta": {
    "total": 35,
    "page": 1,
    "limit": 10,
    "totalPages": 4,
    "hasNext": true,
    "hasPrev": false
  }
}
```

---

### 4.2 Get Mains Paper by ID

Get a specific Mains paper by its ID.

**Endpoint**: `GET /mains/:id`
**Authentication**: Required (JWT Bearer Token)

#### Path Parameters

- `id` (required): Mains paper ID (UUID)

#### Response (200 OK)

```json
{
  "id": "mains-uuid-123",
  "title": "UPSC Mains 2023 - Essay",
  "subject": "Essay",
  "year": 2023,
  "difficulty": "MEDIUM",
  "totalQuestions": 8,
  "duration": 10800,
  "instructions": "Write essays on any four topics...",
  "questions": [
    {
      "id": "question-uuid-789",
      "questionText": "Is the Colonial mentality hindering India's success?",
      "wordLimit": 1000,
      "marks": 125
    }
  ],
  "createdAt": "2023-09-01T00:00:00.000Z"
}
```

#### Error Responses

- `400 Bad Request` - Invalid UUID format
- `404 Not Found` - Mains paper not found

---

### 4.3 Create Mains Paper (Admin Only)

Create a new Mains paper.

**Endpoint**: `POST /mains`
**Authentication**: Required (JWT Bearer Token + Admin Role)

#### Request Body

```json
{
  "title": "UPSC Mains 2024 - Essay",
  "subject": "Essay",
  "year": 2024,
  "difficulty": "MEDIUM",
  "totalQuestions": 8,
  "duration": 10800,
  "instructions": "Write essays on any four topics..."
}
```

#### Response (201 Created)

```json
{
  "id": "mains-uuid-456",
  "title": "UPSC Mains 2024 - Essay",
  "subject": "Essay",
  "year": 2024,
  "difficulty": "MEDIUM",
  "createdAt": "2025-10-10T10:30:00.000Z"
}
```

#### Error Responses

- `400 Bad Request` - Invalid input data
- `403 Forbidden` - Insufficient permissions

---

### 4.4 Update Mains Paper

Update an existing Mains paper.

**Endpoint**: `PATCH /mains/:id`
**Authentication**: Required (JWT Bearer Token + Admin Role)

#### Path Parameters

- `id` (required): Mains paper ID (UUID)

#### Request Body

```json
{
  "title": "UPSC Mains 2024 - Essay (Updated)",
  "difficulty": "HARD"
}
```

#### Response (200 OK)

```json
{
  "id": "mains-uuid-456",
  "title": "UPSC Mains 2024 - Essay (Updated)",
  "difficulty": "HARD",
  "updatedAt": "2025-10-10T11:00:00.000Z"
}
```

---

### 4.5 Delete Mains Paper

Delete a Mains paper.

**Endpoint**: `DELETE /mains/:id`
**Authentication**: Required (JWT Bearer Token + Admin Role)

#### Path Parameters

- `id` (required): Mains paper ID (UUID)

#### Response (200 OK)

```json
{
  "success": true,
  "message": "Mains paper deleted successfully"
}
```

#### Error Responses

- `404 Not Found` - Mains paper not found

---

## 5. Videos & Reels Module

Base path: `/reels` and `/videos`

### 5.1 Get All Reels

Get paginated list of reels with optional filtering.

**Endpoint**: `GET /reels`
**Authentication**: Public (No token required)

#### Query Parameters

- `page` (optional, default: 1): Page number
- `limit` (optional, default: 10): Items per page
- `title` (optional): Filter by video title (partial match)
- `status` (optional): Filter by video status - `UPLOADING`, `PROCESSING`, `READY`, `ERROR`

#### Request Example

```
GET /reels?page=1&limit=10&status=READY&title=UPSC
```

#### Response (200 OK)

```json
{
  "data": [
    {
      "id": "reel-uuid-123",
      "title": "UPSC History Quick Review",
      "description": "Quick review of important historical events",
      "assetId": "mux-asset-123456",
      "playbackId": "mux-playback-789012",
      "status": "READY",
      "likes": 45,
      "date": "2025-10-05T00:00:00.000Z",
      "createdAt": "2025-10-05T10:30:00.000Z",
      "updatedAt": "2025-10-05T11:00:00.000Z"
    }
  ],
  "meta": {
    "total": 120,
    "page": 1,
    "limit": 10,
    "totalPages": 12,
    "hasNext": true,
    "hasPrev": false
  }
}
```

#### Error Responses

- `400 Bad Request` - Invalid query parameters (e.g., "Page must be greater than 0")
- `500 Internal Server Error` - Failed to retrieve reels

---

### 5.2 Upload Video

Upload a video file to Mux for processing.

**Endpoint**: `POST /videos/upload`
**Authentication**: Public (No token required)
**Content-Type**: `multipart/form-data`

#### Request Body (Form Data)

- `video` (required): Video file (binary)
- `videoId` (required): UUID generated by frontend
- `title` (required): Initial title from filename

#### Request Example

```http
POST /videos/upload
Content-Type: multipart/form-data

------WebKitFormBoundary
Content-Disposition: form-data; name="videoId"

clrx1234567890abcdef1234
------WebKitFormBoundary
Content-Disposition: form-data; name="title"

UPSC Preparation Video
------WebKitFormBoundary
Content-Disposition: form-data; name="video"; filename="upsc-prep.mp4"
Content-Type: video/mp4

[binary video data]
------WebKitFormBoundary--
```

#### Response (200 OK)

```json
{
  "success": true,
  "data": {
    "videoId": "clrx1234567890abcdef1234",
    "message": "Upload started successfully"
  }
}
```

#### Error Responses

- `400 Bad Request` - Video file is required
- `413 Payload Too Large` - File too large
- `422 Unprocessable Entity` - Invalid file type (only video files allowed)

---

### 5.3 Get Video Status

Get the current processing status of a video from Mux.

**Endpoint**: `GET /videos/:videoId/status`
**Authentication**: Public (No token required)

#### Path Parameters

- `videoId` (required): Video ID (UUID)

#### Response (200 OK)

```json
{
  "success": true,
  "data": {
    "videoId": "clrx1234567890abcdef1234",
    "status": "READY",
    "assetId": "mux-asset-123456",
    "playbackId": "mux-playback-789012",
    "duration": 300.5,
    "aspectRatio": "16:9"
  }
}
```

**Status Values:**

- `UPLOADING` - Video is being uploaded
- `PROCESSING` - Video is being processed by Mux
- `READY` - Video is ready for playback
- `ERROR` - An error occurred during processing

#### Error Responses

- `404 Not Found` - Video with ID not found

---

### 5.4 Bulk Create/Update Reels

Bulk create or update reel metadata. Used after video processing is complete.

**Endpoint**: `PUT /reels/bulk`
**Authentication**: Public (No token required)

#### Request Body

```json
{
  "videos": [
    {
      "id": "clrx1234567890abcdef1234",
      "title": "UPSC History Quick Review",
      "description": "Quick review of important historical events",
      "assetId": "mux-asset-123456",
      "playbackId": "mux-playback-789012",
      "status": "READY",
      "date": "2025-10-10T00:00:00.000Z"
    },
    {
      "id": "clrx0987654321fedcba4321",
      "title": "Geography Tips for UPSC",
      "description": "Important geography concepts",
      "assetId": "mux-asset-654321",
      "playbackId": "mux-playback-210987",
      "status": "PROCESSING"
    }
  ]
}
```

#### Response (200 OK)

```json
{
  "success": true,
  "data": {
    "created": 1,
    "updated": 1,
    "failed": 0,
    "total": 2
  }
}
```

#### Error Responses

- `400 Bad Request` - Videos array cannot be empty or validation errors
- `500 Internal Server Error` - Failed to perform bulk upsert operation

---

### 5.5 Increment Reel Likes

Increment the likes count for a specific reel by 1.

**Endpoint**: `POST /reels/:videoId/like`
**Authentication**: Public (No token required)

#### Path Parameters

- `videoId` (required): Video ID (UUID)

#### Response (200 OK)

```json
{
  "success": true,
  "data": {
    "likes": 46
  }
}
```

#### Error Responses

- `404 Not Found` - Video with ID not found

---

## 6. Progress Tracking Module (Graph Database)

Base path: `/progress`

This module uses Neo4j graph database for tracking user learning progress, streaks, and assessments.

### 6.1 Create User in Graph

Create a new user node in the graph database.

**Endpoint**: `POST /progress/users`
**Authentication**: Required (JWT Bearer Token)

#### Request Body

```json
{
  "userId": "user-uuid-123",
  "name": "John Doe",
  "email": "john@example.com"
}
```

#### Response (201 Created)

```json
{
  "success": true,
  "data": {
    "userId": "user-uuid-123",
    "name": "John Doe",
    "email": "john@example.com",
    "createdAt": "2025-10-10T10:30:00.000Z"
  }
}
```

---

### 6.2 Get User Profile

Get user profile from graph database.

**Endpoint**: `GET /progress/users/:id`
**Authentication**: Required (JWT Bearer Token)

#### Path Parameters

- `id` (required): User ID

#### Response (200 OK)

```json
{
  "success": true,
  "data": {
    "userId": "user-uuid-123",
    "name": "John Doe",
    "email": "john@example.com",
    "totalAttempts": 150,
    "correctAnswers": 120,
    "accuracy": 80
  }
}
```

---

### 6.3 Get User with Tier

Get user with tier subscription information.

**Endpoint**: `GET /progress/users/:id/tier`
**Authentication**: Required (JWT Bearer Token)

#### Path Parameters

- `id` (required): User ID

#### Response (200 OK)

```json
{
  "success": true,
  "data": {
    "userId": "user-uuid-123",
    "name": "John Doe",
    "tier": {
      "tierId": "tier-uuid-789",
      "tierName": "Premium",
      "subscriptionStart": "2025-01-01T00:00:00.000Z",
      "subscriptionEnd": "2026-01-01T00:00:00.000Z",
      "isActive": true
    }
  }
}
```

---

### 6.4 Create Question in Graph

Create a new question node in the graph database.

**Endpoint**: `POST /progress/questions`
**Authentication**: Required (JWT Bearer Token + Admin Role)

#### Request Body

```json
{
  "questionId": "question-uuid-456",
  "questionText": "Who was the founder of Maurya Empire?",
  "category": "PRELIMS",
  "difficulty": "MEDIUM",
  "subject": "History",
  "topic": "Ancient India"
}
```

#### Response (201 Created)

```json
{
  "success": true,
  "data": {
    "questionId": "question-uuid-456",
    "questionText": "Who was the founder of Maurya Empire?",
    "category": "PRELIMS",
    "createdAt": "2025-10-10T10:30:00.000Z"
  }
}
```

---

### 6.5 Get Question with Visibility

Get question with tier-based visibility rules applied.

**Endpoint**: `GET /progress/questions/:id`
**Authentication**: Required (JWT Bearer Token)

#### Path Parameters

- `id` (required): Question ID

#### Query Parameters

- `userId` (required): User ID for visibility check

#### Request Example

```
GET /progress/questions/question-uuid-456?userId=user-uuid-123
```

#### Response (200 OK)

```json
{
  "success": true,
  "data": {
    "questionId": "question-uuid-456",
    "questionText": "Who was the founder of Maurya Empire?",
    "hasAccess": true,
    "tierRequired": "Free",
    "category": "PRELIMS"
  }
}
```

---

### 6.6 Record Question Attempt

Record a user's attempt at answering a question.

**Endpoint**: `POST /progress/questions/:id/attempt`
**Authentication**: Required (JWT Bearer Token)

#### Path Parameters

- `id` (required): Question ID

#### Request Body

```json
{
  "userId": "user-uuid-123",
  "selectedOption": 0,
  "isCorrect": true,
  "timeSpentSeconds": 45
}
```

#### Response (201 Created)

```json
{
  "success": true,
  "message": "Attempt recorded successfully"
}
```

---

### 6.7 Get User Attempts for Question

Get all attempts by a user for a specific question.

**Endpoint**: `GET /progress/users/:id/attempts/:questionId`
**Authentication**: Required (JWT Bearer Token)

#### Path Parameters

- `id` (required): User ID
- `questionId` (required): Question ID

#### Response (200 OK)

```json
{
  "success": true,
  "data": [
    {
      "attemptId": "attempt-uuid-111",
      "selectedOption": 0,
      "isCorrect": true,
      "timeSpentSeconds": 45,
      "attemptedAt": "2025-10-10T10:30:00.000Z"
    },
    {
      "attemptId": "attempt-uuid-222",
      "selectedOption": 1,
      "isCorrect": false,
      "timeSpentSeconds": 30,
      "attemptedAt": "2025-10-09T15:20:00.000Z"
    }
  ]
}
```

---

### 6.8 Get User Question Statistics

Get comprehensive statistics about user's question attempts.

**Endpoint**: `GET /progress/users/:id/stats`
**Authentication**: Required (JWT Bearer Token)

#### Path Parameters

- `id` (required): User ID

#### Response (200 OK)

```json
{
  "success": true,
  "data": {
    "totalAttempts": 250,
    "correctAnswers": 200,
    "incorrectAnswers": 50,
    "accuracy": 80,
    "averageTimePerQuestion": 42.5,
    "subjectWiseStats": {
      "History": {
        "attempts": 100,
        "correct": 85,
        "accuracy": 85
      },
      "Geography": {
        "attempts": 150,
        "correct": 115,
        "accuracy": 76.67
      }
    }
  }
}
```

---

### 6.9 Record Daily Session

Record a user's daily study session.

**Endpoint**: `POST /progress/sessions`
**Authentication**: Required (JWT Bearer Token)

#### Request Body

```json
{
  "userId": "user-uuid-123",
  "date": "2025-10-10",
  "hoursSpent": 3.5,
  "questionsAttempted": 50,
  "correctAnswers": 42,
  "topics": ["History", "Geography"]
}
```

#### Response (201 Created)

```json
{
  "success": true,
  "message": "Session recorded successfully"
}
```

---

### 6.10 Get User Streak

Get user's current study streak information.

**Endpoint**: `GET /progress/users/:id/streak`
**Authentication**: Required (JWT Bearer Token)

#### Path Parameters

- `id` (required): User ID

#### Response (200 OK)

```json
{
  "success": true,
  "data": {
    "currentStreak": 15,
    "longestStreak": 45,
    "lastSessionDate": "2025-10-10",
    "totalDaysStudied": 120
  }
}
```

---

### 6.11 Get Session History

Get user's session history for a specified number of days.

**Endpoint**: `GET /progress/users/:id/sessions`
**Authentication**: Required (JWT Bearer Token)

#### Path Parameters

- `id` (required): User ID

#### Query Parameters

- `days` (optional, default: 30): Number of days to look back

#### Request Example

```
GET /progress/users/user-uuid-123/sessions?days=7
```

#### Response (200 OK)

```json
{
  "success": true,
  "data": [
    {
      "date": "2025-10-10",
      "hoursSpent": 3.5,
      "questionsAttempted": 50,
      "correctAnswers": 42,
      "topics": ["History", "Geography"]
    },
    {
      "date": "2025-10-09",
      "hoursSpent": 2.0,
      "questionsAttempted": 30,
      "correctAnswers": 25,
      "topics": ["Polity"]
    }
  ]
}
```

---

### 6.12 Create Assessment

Create a new assessment in the graph database.

**Endpoint**: `POST /progress/assessments`
**Authentication**: Required (JWT Bearer Token + Admin Role)

#### Request Body

```json
{
  "assessmentId": "assessment-uuid-789",
  "title": "Full Length Mock Test 1",
  "category": "PRELIMS",
  "totalQuestions": 100,
  "duration": 7200,
  "maxScore": 200
}
```

#### Response (201 Created)

```json
{
  "success": true,
  "data": {
    "assessmentId": "assessment-uuid-789",
    "title": "Full Length Mock Test 1",
    "category": "PRELIMS",
    "createdAt": "2025-10-10T10:30:00.000Z"
  }
}
```

---

### 6.13 Record Assessment Completion

Record a user's completion of an assessment.

**Endpoint**: `POST /progress/assessments/:id/complete`
**Authentication**: Required (JWT Bearer Token)

#### Path Parameters

- `id` (required): Assessment ID

#### Request Body

```json
{
  "userId": "user-uuid-123",
  "score": 175,
  "totalQuestions": 100,
  "correctAnswers": 87,
  "timeSpentSeconds": 6800
}
```

#### Response (201 Created)

```json
{
  "success": true,
  "message": "Assessment completion recorded successfully"
}
```

---

### 6.14 Get User Progress Dashboard

Get comprehensive progress dashboard for a user.

**Endpoint**: `GET /progress/users/:id/progress`
**Authentication**: Required (JWT Bearer Token)

#### Path Parameters

- `id` (required): User ID

#### Response (200 OK)

```json
{
  "success": true,
  "data": {
    "assessments": {
      "totalAttempted": 5,
      "averageScore": 165,
      "bestScore": 185,
      "recentAssessments": [
        {
          "assessmentId": "assessment-uuid-789",
          "title": "Full Length Mock Test 1",
          "score": 175,
          "completedAt": "2025-10-10T10:30:00.000Z"
        }
      ]
    },
    "weekly": {
      "hoursSpent": 20,
      "questionsAttempted": 300,
      "accuracy": 82
    },
    "monthly": {
      "hoursSpent": 85,
      "questionsAttempted": 1200,
      "accuracy": 80
    },
    "streak": {
      "currentStreak": 15,
      "longestStreak": 45
    },
    "questions": {
      "totalAttempts": 1200,
      "correctAnswers": 960,
      "accuracy": 80
    }
  }
}
```

---

### 6.15 Get Questions by Category

Get questions filtered by category (PRELIMS or MAINS).

**Endpoint**: `GET /progress/questions`
**Authentication**: Required (JWT Bearer Token)

#### Query Parameters

- `category` (required): Question category - `PRELIMS` or `MAINS`
- `limit` (optional, default: 20): Maximum number of questions to return

#### Request Example

```
GET /progress/questions?category=PRELIMS&limit=10
```

#### Response (200 OK)

```json
{
  "success": true,
  "data": [
    {
      "questionId": "question-uuid-456",
      "questionText": "Who was the founder of Maurya Empire?",
      "category": "PRELIMS",
      "difficulty": "MEDIUM",
      "subject": "History",
      "topic": "Ancient India"
    }
  ]
}
```

---

## 7. Health Check Module

Base path: `/health`

All health check endpoints are public (no authentication required).

### 7.1 Basic Health Check

Check basic application health.

**Endpoint**: `GET /health`
**Authentication**: Public (No token required)

#### Response (200 OK)

```json
{
  "status": "ok",
  "info": {
    "database": {
      "status": "up"
    }
  },
  "error": {},
  "details": {
    "database": {
      "status": "up"
    }
  }
}
```

---

### 7.2 Database Health Check

Check PostgreSQL database connectivity.

**Endpoint**: `GET /health/database`
**Authentication**: Public (No token required)

#### Response (200 OK)

```json
{
  "status": "ok",
  "database": "connected",
  "timestamp": "2025-10-10T10:30:00.000Z"
}
```

#### Error Response (503 Service Unavailable)

```json
{
  "status": "error",
  "database": "disconnected",
  "timestamp": "2025-10-10T10:30:00.000Z"
}
```

---

### 7.3 Neo4j Health Check

Check Neo4j graph database connectivity.

**Endpoint**: `GET /health/neo4j`
**Authentication**: Public (No token required)

#### Response (200 OK)

```json
{
  "status": "healthy",
  "neo4j": "connected",
  "version": "5.12.0",
  "timestamp": "2025-10-10T10:30:00.000Z"
}
```

#### Error Response (503 Service Unavailable)

```json
{
  "status": "unhealthy",
  "neo4j": "connection_error",
  "error": "Connection timeout",
  "timestamp": "2025-10-10T10:30:00.000Z"
}
```

---

### 7.4 Valkey/Redis Health Check

Check Valkey (Redis) cache connectivity.

**Endpoint**: `GET /health/valkey`
**Authentication**: Public (No token required)

#### Response (200 OK)

```json
{
  "status": "healthy",
  "valkey": "connected",
  "operations": ["set", "get", "del"],
  "timestamp": "2025-10-10T10:30:00.000Z"
}
```

#### Error Response (503 Service Unavailable)

```json
{
  "status": "unhealthy",
  "valkey": "connection_error",
  "error": "Connection refused",
  "timestamp": "2025-10-10T10:30:00.000Z"
}
```

---

### 7.5 Full System Health Check

Check all services health (PostgreSQL, Neo4j, Valkey).

**Endpoint**: `GET /health/full`
**Authentication**: Public (No token required)

#### Response (200 OK)

```json
{
  "status": "ok",
  "services": {
    "postgres": {
      "status": "healthy",
      "connection": "connected"
    },
    "neo4j": {
      "status": "healthy",
      "neo4j": "connected",
      "version": "5.12.0"
    },
    "valkey": {
      "status": "healthy",
      "valkey": "connected",
      "operations": ["set", "get", "del"]
    }
  },
  "timestamp": "2025-10-10T10:30:00.000Z"
}
```

#### Error Response (503 Service Unavailable)

```json
{
  "status": "error",
  "services": {
    "postgres": {
      "status": "healthy",
      "connection": "connected"
    },
    "neo4j": {
      "status": "unhealthy",
      "neo4j": "connection_error"
    },
    "valkey": {
      "status": "healthy",
      "valkey": "connected"
    }
  },
  "timestamp": "2025-10-10T10:30:00.000Z"
}
```

---

## Common HTTP Status Codes

### Success Codes

- `200 OK` - Request successful
- `201 Created` - Resource created successfully

### Client Error Codes

- `400 Bad Request` - Invalid request data or validation failed
- `401 Unauthorized` - Missing or invalid authentication token
- `403 Forbidden` - Insufficient permissions
- `404 Not Found` - Resource not found
- `409 Conflict` - Conflict with existing resource (e.g., duplicate entry)
- `413 Payload Too Large` - Request body exceeds size limit
- `422 Unprocessable Entity` - Request data is semantically invalid
- `429 Too Many Requests` - Rate limit exceeded

### Server Error Codes

- `500 Internal Server Error` - Unexpected server error
- `503 Service Unavailable` - Service temporarily unavailable (e.g., database down)

---

## Rate Limiting

The following endpoints have rate limits:

- `POST /auth/cognito/exchange` - 5 requests per minute
- `POST /auth/refresh` - Default throttle applies

Rate limit exceeded response:

```json
{
  "statusCode": 429,
  "message": "Too many requests",
  "error": "Too Many Requests"
}
```

---

## Pagination Best Practices

1. Always specify reasonable `limit` values (recommended: 10-50)
2. Page numbers start from 1, not 0
3. Use `hasNext` and `hasPrev` flags for navigation UI
4. Cache results when appropriate

Example pagination usage:

```javascript
// First page
GET /pyq?page=1&limit=10

// Next page (if hasNext is true)
GET /pyq?page=2&limit=10

// Previous page (if hasPrev is true)
GET /pyq?page=1&limit=10
```

---

## Filtering Best Practices

1. Combine multiple filters for precise results
2. Use appropriate data types (numbers for `year`, strings for `subject`)
3. Filter values are case-sensitive unless specified otherwise
4. Title filters typically support partial matching

Example combined filtering:

```
GET /pyq?subject=History&year=2023&difficulty=MEDIUM&page=1&limit=10
```

---

## Security Best Practices

1. Always use HTTPS in production
2. Store JWT tokens securely (not in localStorage for web apps)
3. Refresh tokens before they expire
4. Implement proper CORS policies
5. Never expose sensitive data in query parameters
6. Validate and sanitize all user inputs

---

## Version History

- **v1.0** - Initial API release (2025-10-10)

---

## Support and Contact

For API support, issues, or feature requests, please contact the development team or file an issue in the project repository.

---

**Last Updated**: October 10, 2025
**API Version**: v1
**Document Version**: 1.0
