# API Reference

This document provides comprehensive API documentation for PadnetXpress backend services, including authentication, health data endpoints, and integration patterns.

## 📋 Table of Contents

- [Overview](#overview)
- [Authentication](#authentication)
- [Base URLs](#base-urls)
- [Health Data API](#health-data-api)
- [User Management API](#user-management-api)
- [Analytics API](#analytics-api)
- [Device Integration API](#device-integration-api)
- [Error Handling](#error-handling)
- [Rate Limiting](#rate-limiting)

## 🌐 Overview

PadnetXpress API is a RESTful service that provides secure access to health monitoring data, user management, and analytics. The API supports:

- JWT-based authentication
- Real-time health data synchronization
- Secure data encryption
- Comprehensive analytics
- Medical device integration

### API Characteristics

- **Protocol**: HTTPS only (TLS 1.3)
- **Format**: JSON
- **Authentication**: JWT Bearer tokens
- **Versioning**: URL-based (`/api/v1/`)
- **Rate Limiting**: Token bucket algorithm

## 🔐 Authentication

### Authentication Flow

```mermaid
sequenceDiagram
    participant C as Client
    participant A as Auth Service
    participant R as Resource Server
    
    C->>A: POST /auth/login
    A-->>C: JWT Access Token + Refresh Token
    C->>R: GET /api/v1/health-data (with JWT)
    R->>A: Validate Token
    A-->>R: Token Valid
    R-->>C: Health Data Response
```

### Login Endpoint

**POST** `/auth/login`

```json
{
  "email": "user@example.com",
  "password": "securePassword123",
  "deviceId": "iPhone-12345-ABCD"
}
```

**Response:**
```json
{
  "success": true,
  "data": {
    "accessToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
    "refreshToken": "def50200a1b2c3d4e5f6...",
    "expiresIn": 3600,
    "tokenType": "Bearer",
    "user": {
      "id": "user123",
      "email": "user@example.com",
      "firstName": "John",
      "lastName": "Doe",
      "healthKitEnabled": true
    }
  }
}
```

### Token Refresh

**POST** `/auth/refresh`

```json
{
  "refreshToken": "def50200a1b2c3d4e5f6..."
}
```

### Authorization Header

All authenticated requests must include:

```
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```

## 🌍 Base URLs

| Environment | Base URL |
|-------------|----------|
| Development | `https://dev-api.padnetxpress.com/api/v1` |
| Staging | `https://staging-api.padnetxpress.com/api/v1` |
| Production | `https://api.padnetxpress.com/api/v1` |

## ❤️ Health Data API

### Heart Rate Endpoints

#### Get Heart Rate Data

**GET** `/health-data/heart-rate`

**Query Parameters:**
- `startDate` (ISO 8601): Start date filter
- `endDate` (ISO 8601): End date filter
- `granularity` (string): `minute`, `hour`, `day`
- `limit` (integer): Maximum records (default: 100)

```bash
GET /health-data/heart-rate?startDate=2024-01-01T00:00:00Z&endDate=2024-01-02T00:00:00Z&granularity=hour
```

**Response:**
```json
{
  "success": true,
  "data": {
    "readings": [
      {
        "id": "hr_123456",
        "timestamp": "2024-01-01T10:30:00Z",
        "value": 72,
        "unit": "bpm",
        "source": "apple_watch",
        "confidence": 0.95,
        "metadata": {
          "activity": "resting",
          "context": "morning_measurement"
        }
      }
    ],
    "statistics": {
      "average": 74.5,
      "minimum": 62,
      "maximum": 89,
      "count": 24
    },
    "pagination": {
      "page": 1,
      "pageSize": 100,
      "totalPages": 1,
      "totalRecords": 24
    }
  }
}
```

#### Add Heart Rate Reading

**POST** `/health-data/heart-rate`

```json
{
  "readings": [
    {
      "timestamp": "2024-01-01T10:30:00Z",
      "value": 72,
      "source": "manual_entry",
      "metadata": {
        "activity": "post_exercise",
        "notes": "After 30-minute jog"
      }
    }
  ]
}
```

### Blood Pressure Endpoints

#### Get Blood Pressure Data

**GET** `/health-data/blood-pressure`

**Response:**
```json
{
  "success": true,
  "data": {
    "readings": [
      {
        "id": "bp_789012",
        "timestamp": "2024-01-01T08:00:00Z",
        "systolic": 120,
        "diastolic": 80,
        "unit": "mmHg",
        "source": "omron_device",
        "metadata": {
          "position": "sitting",
          "arm": "left",
          "cuffSize": "standard"
        }
      }
    ],
    "trends": {
      "systolicTrend": "stable",
      "diastolicTrend": "decreasing",
      "averageSystolic": 118.5,
      "averageDiastolic": 78.2
    }
  }
}
```

#### Add Blood Pressure Reading

**POST** `/health-data/blood-pressure`

```json
{
  "readings": [
    {
      "timestamp": "2024-01-01T08:00:00Z",
      "systolic": 120,
      "diastolic": 80,
      "source": "manual_entry",
      "metadata": {
        "position": "sitting",
        "arm": "left",
        "notes": "Morning reading"
      }
    }
  ]
}
```

### Sleep Data Endpoints

#### Get Sleep Analysis

**GET** `/health-data/sleep`

**Response:**
```json
{
  "success": true,
  "data": {
    "sleepSessions": [
      {
        "id": "sleep_345678",
        "bedTime": "2024-01-01T22:30:00Z",
        "sleepTime": "2024-01-01T23:15:00Z",
        "wakeTime": "2024-01-02T07:00:00Z",
        "totalSleepDuration": 465,
        "sleepEfficiency": 0.87,
        "stages": {
          "awake": 15,
          "light": 245,
          "deep": 125,
          "rem": 80
        },
        "quality": {
          "score": 82,
          "factors": {
            "duration": "good",
            "efficiency": "excellent",
            "consistency": "fair"
          }
        }
      }
    ],
    "weeklyAverage": {
      "totalSleep": 442,
      "efficiency": 0.84,
      "qualityScore": 79
    }
  }
}
```

## 👤 User Management API

### User Profile

#### Get User Profile

**GET** `/users/profile`

**Response:**
```json
{
  "success": true,
  "data": {
    "id": "user123",
    "email": "user@example.com",
    "firstName": "John",
    "lastName": "Doe",
    "dateOfBirth": "1990-05-15",
    "gender": "male",
    "height": 175,
    "weight": 70,
    "healthProfile": {
      "healthKitEnabled": true,
      "medicalConditions": ["hypertension"],
      "medications": [
        {
          "name": "Lisinopril",
          "dosage": "10mg",
          "frequency": "daily"
        }
      ],
      "allergies": ["penicillin"]
    },
    "preferences": {
      "units": "metric",
      "notifications": {
        "heartRateAlerts": true,
        "medicationReminders": true,
        "weeklyReports": true
      }
    }
  }
}
```

#### Update User Profile

**PUT** `/users/profile`

```json
{
  "firstName": "John",
  "lastName": "Smith",
  "weight": 72,
  "preferences": {
    "notifications": {
      "heartRateAlerts": false
    }
  }
}
```

### Health Goals

#### Get Health Goals

**GET** `/users/goals`

**Response:**
```json
{
  "success": true,
  "data": {
    "goals": [
      {
        "id": "goal_001",
        "type": "heart_rate",
        "target": {
          "restingHeartRate": 65,
          "targetRange": {
            "min": 60,
            "max": 70
          }
        },
        "progress": {
          "currentValue": 68,
          "achievementPercentage": 85,
          "trend": "improving"
        },
        "timeline": {
          "startDate": "2024-01-01",
          "endDate": "2024-03-31",
          "checkpoints": ["2024-01-31", "2024-02-29"]
        }
      }
    ]
  }
}
```

## 📊 Analytics API

### Health Insights

#### Get Health Analytics

**GET** `/analytics/insights`

**Query Parameters:**
- `period` (string): `week`, `month`, `quarter`, `year`
- `metrics` (array): List of metrics to include

**Response:**
```json
{
  "success": true,
  "data": {
    "period": "month",
    "insights": [
      {
        "metric": "heart_rate_variability",
        "trend": "improving",
        "change": "+12%",
        "significance": "moderate",
        "recommendation": "Continue current exercise routine"
      },
      {
        "metric": "sleep_quality",
        "trend": "stable",
        "change": "+2%",
        "significance": "low",
        "recommendation": "Consider earlier bedtime"
      }
    ],
    "riskFactors": [
      {
        "category": "cardiovascular",
        "level": "low",
        "factors": ["age", "blood_pressure"],
        "score": 23
      }
    ],
    "achievements": [
      {
        "title": "7-Day Heart Rate Goal",
        "description": "Maintained target heart rate for 7 consecutive days",
        "earnedDate": "2024-01-15"
      }
    ]
  }
}
```

### Export Data

#### Export Health Data

**POST** `/analytics/export`

```json
{
  "format": "csv",
  "dateRange": {
    "startDate": "2024-01-01",
    "endDate": "2024-01-31"
  },
  "metrics": ["heart_rate", "blood_pressure", "sleep"],
  "email": "user@example.com"
}
```

**Response:**
```json
{
  "success": true,
  "data": {
    "exportId": "export_123456",
    "status": "processing",
    "estimatedCompletion": "2024-01-15T14:30:00Z",
    "downloadUrl": null
  }
}
```

## 🔗 Device Integration API

### Connected Devices

#### Get Connected Devices

**GET** `/devices/connected`

**Response:**
```json
{
  "success": true,
  "data": {
    "devices": [
      {
        "id": "device_001",
        "name": "Apple Watch Series 8",
        "type": "smartwatch",
        "manufacturer": "Apple",
        "model": "A2376",
        "connectionStatus": "connected",
        "lastSync": "2024-01-15T10:30:00Z",
        "capabilities": ["heart_rate", "sleep_tracking", "activity"],
        "batteryLevel": 85
      },
      {
        "id": "device_002",
        "name": "Omron BP Monitor",
        "type": "blood_pressure_monitor",
        "manufacturer": "Omron",
        "model": "HEM-7156T",
        "connectionStatus": "paired",
        "lastSync": "2024-01-15T08:00:00Z",
        "capabilities": ["blood_pressure"]
      }
    ]
  }
}
```

#### Sync Device Data

**POST** `/devices/{deviceId}/sync`

```json
{
  "syncType": "incremental",
  "lastSyncTimestamp": "2024-01-15T10:00:00Z"
}
```

## ⚠️ Error Handling

### Error Response Format

```json
{
  "success": false,
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Invalid request parameters",
    "details": [
      {
        "field": "email",
        "message": "Invalid email format"
      }
    ],
    "timestamp": "2024-01-15T10:30:00Z",
    "requestId": "req_123456789"
  }
}
```

### Error Codes

| Code | HTTP Status | Description |
|------|-------------|-------------|
| `AUTHENTICATION_REQUIRED` | 401 | Authentication token required |
| `INVALID_TOKEN` | 401 | JWT token is invalid or expired |
| `FORBIDDEN` | 403 | Insufficient permissions |
| `VALIDATION_ERROR` | 400 | Request validation failed |
| `RESOURCE_NOT_FOUND` | 404 | Requested resource not found |
| `RATE_LIMIT_EXCEEDED` | 429 | Too many requests |
| `INTERNAL_ERROR` | 500 | Internal server error |
| `SERVICE_UNAVAILABLE` | 503 | Service temporarily unavailable |

### Health-Specific Errors

| Code | Description |
|------|-------------|
| `HEALTHKIT_PERMISSION_DENIED` | HealthKit access not granted |
| `INVALID_HEALTH_DATA` | Health data validation failed |
| `DEVICE_NOT_CONNECTED` | Medical device not connected |
| `DATA_SYNC_FAILED` | Failed to sync health data |

## ⏱️ Rate Limiting

### Rate Limits

| Endpoint Category | Requests per Minute | Requests per Hour |
|------------------|-------------------|------------------|
| Authentication | 5 | 20 |
| Health Data Read | 100 | 1000 |
| Health Data Write | 30 | 300 |
| Analytics | 20 | 200 |
| Device Integration | 10 | 100 |

### Rate Limit Headers

```http
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 95
X-RateLimit-Reset: 1642234800
X-RateLimit-Window: 60
```

### Rate Limit Exceeded Response

```json
{
  "success": false,
  "error": {
    "code": "RATE_LIMIT_EXCEEDED",
    "message": "Too many requests. Please try again later.",
    "retryAfter": 30
  }
}
```

## 🔒 Security Considerations

### API Security Features

- **HTTPS Only**: All API communication encrypted with TLS 1.3
- **JWT Tokens**: Stateless authentication with configurable expiration
- **Request Signing**: Optional request signing for sensitive operations
- **IP Whitelisting**: Optional IP-based access control
- **CORS**: Configurable cross-origin resource sharing
- **Input Validation**: Comprehensive input sanitization

### Health Data Compliance

- **HIPAA Compliance**: All health data handling follows HIPAA guidelines
- **Data Minimization**: Only necessary data is collected and stored
- **Audit Logging**: Comprehensive audit trail for all data access
- **Encryption**: AES-256 encryption for data at rest
- **Data Retention**: Configurable data retention policies

---

For additional API support, contact: api-support@padnetxpress.com 