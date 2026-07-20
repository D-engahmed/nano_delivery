# SmartFleet API Specification
## Version 1.0.0 | OpenAPI 3.0.0

---

## Base URLs

| Environment | URL |
|-------------|-----|
| Local | `http://localhost:8000` |
| Development | `https://api-dev.smartfleet.io` |
| Production | `https://api.smartfleet.io` |

---

## Authentication

All endpoints (except `/health` and `/docs`) require JWT authentication.

**Header:** `Authorization: Bearer <token>`

**Token Types:**
- `access_token` - Valid for 15 minutes
- `refresh_token` - Valid for 7 days

### Token Refresh
```http
POST /auth/refresh
Content-Type: application/json

{
  "refresh_token": "eyJhbGciOiJIUzI1NiIs..."
}
```

**Response:**
```json
{
  "access_token": "eyJhbGciOiJIUzI1NiIs...",
  "token_type": "bearer",
  "expires_in": 900
}
```

---

## Rate Limiting

| Tier | Requests/Minute | Burst |
|------|-----------------|-------|
| Anonymous | 10 | 10 |
| Customer | 100 | 150 |
| Operator | 500 | 750 |
| Admin | 1000 | 1500 |

**Headers:**
- `X-RateLimit-Limit`: 100
- `X-RateLimit-Remaining`: 87
- `X-RateLimit-Reset`: 1625097600

---

## Common Response Format

### Success (2xx)
```json
{
  "success": true,
  "data": { ... },
  "meta": {
    "timestamp": "2026-07-20T10:30:00Z",
    "request_id": "req_abc123"
  }
}
```

### Error (4xx/5xx)
```json
{
  "success": false,
  "error": {
    "code": "ORDER_NOT_FOUND",
    "message": "Order with ID '550e8400-e29b' not found",
    "details": { ... },
    "request_id": "req_abc123"
  }
}
```

---

## Endpoints

### 1. ORDER SERVICE

#### Create Order
```http
POST /api/v1/orders
Authorization: Bearer <token>
Content-Type: application/json
```

**Request Body:**
```json
{
  "pickup": {
    "address": "123 Main St, Beijing",
    "lat": 39.9042,
    "lng": 116.4074,
    "contact_name": "Alice Chen",
    "contact_phone": "+86-138-0000-0000",
    "instructions": "Call upon arrival"
  },
  "delivery": {
    "address": "456 Park Ave, Beijing",
    "lat": 39.9156,
    "lng": 116.4114,
    "contact_name": "Bob Wang",
    "contact_phone": "+86-139-0000-0000",
    "instructions": "Leave at reception"
  },
  "package": {
    "weight": 2.5,
    "dimensions": "30x20x15",
    "description": "Medical supplies - fragile",
    "category": "medical",
    "is_fragile": true,
    "requires_signature": true
  },
  "priority": "normal",
  "scheduled_pickup_time": "2026-07-20T14:00:00Z"
}
```

**Response (201 Created):**
```json
{
  "success": true,
  "data": {
    "id": "550e8400-e29b-41d4-a716-446655440000",
    "tracking_number": "SF7A3B9C2D1E4F5",
    "status": "pending",
    "priority": "normal",
    "pickup": {
      "address": "123 Main St, Beijing",
      "lat": 39.9042,
      "lng": 116.4074,
      "estimated_time": "2026-07-20T14:00:00Z"
    },
    "delivery": {
      "address": "456 Park Ave, Beijing",
      "lat": 39.9156,
      "lng": 116.4114,
      "estimated_time": "2026-07-20T14:25:00Z"
    },
    "pricing": {
      "base_fare": 5.00,
      "distance_fare": 3.50,
      "priority_fee": 0.00,
      "total": 8.50,
      "currency": "CNY"
    },
    "qr_code": "https://api.smartfleet.io/qr/SF7A3B9C2D1E4F5",
    "created_at": "2026-07-20T10:30:00Z"
  }
}
```

---

#### List Orders
```http
GET /api/v1/orders?page=1&limit=20&status=in_transit&priority=high
Authorization: Bearer <token>
```

**Query Parameters:**
| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `page` | integer | No | Page number (default: 1) |
| `limit` | integer | No | Items per page (default: 20, max: 100) |
| `status` | string | No | Filter by status |
| `priority` | string | No | Filter by priority |
| `start_date` | ISO 8601 | No | Filter from date |
| `end_date` | ISO 8601 | No | Filter to date |
| `vehicle_id` | UUID | No | Filter by assigned vehicle |

**Response:**
```json
{
  "success": true,
  "data": {
    "items": [
      {
        "id": "550e8400-e29b-41d4-a716-446655440000",
        "tracking_number": "SF7A3B9C2D1E4F5",
        "status": "in_transit",
        "priority": "high",
        "pickup_address": "123 Main St",
        "delivery_address": "456 Park Ave",
        "estimated_delivery": "2026-07-20T14:25:00Z",
        "vehicle": {
          "id": "660e8400-e29b-41d4-a716-446655440001",
          "type": "drone",
          "model": "DJI-M300-RTK"
        }
      }
    ],
    "pagination": {
      "page": 1,
      "limit": 20,
      "total": 156,
      "pages": 8,
      "has_next": true,
      "has_prev": false
    }
  }
}
```

---

#### Get Order Detail
```http
GET /api/v1/orders/{order_id}
Authorization: Bearer <token>
```

**Response:**
```json
{
  "success": true,
  "data": {
    "id": "550e8400-e29b-41d4-a716-446655440000",
    "tracking_number": "SF7A3B9C2D1E4F5",
    "status": "in_transit",
    "status_history": [
      {
        "status": "pending",
        "timestamp": "2026-07-20T10:30:00Z",
        "location": null,
        "notes": "Order created"
      },
      {
        "status": "assigned",
        "timestamp": "2026-07-20T10:32:00Z",
        "location": null,
        "notes": "Assigned to drone DJI-001"
      },
      {
        "status": "picked_up",
        "timestamp": "2026-07-20T14:05:00Z",
        "location": {"lat": 39.9042, "lng": 116.4074},
        "notes": "Package collected from sender"
      },
      {
        "status": "in_transit",
        "timestamp": "2026-07-20T14:08:00Z",
        "location": {"lat": 39.9080, "lng": 116.4090},
        "notes": "En route to destination"
      }
    ],
    "route": {
      "waypoints": [
        {"lat": 39.9042, "lng": 116.4074, "altitude": 50, "timestamp": "2026-07-20T14:00:00Z"},
        {"lat": 39.9080, "lng": 116.4090, "altitude": 80, "timestamp": "2026-07-20T14:08:00Z"},
        {"lat": 39.9156, "lng": 116.4114, "altitude": 50, "timestamp": "2026-07-20T14:20:00Z"}
      ],
      "distance_meters": 2500,
      "estimated_duration_seconds": 1200
    },
    "vehicle": {
      "id": "660e8400-e29b-41d4-a716-446655440001",
      "type": "drone",
      "model": "DJI-M300-RTK",
      "current_position": {"lat": 39.9100, "lng": 116.4100, "altitude": 75},
      "battery_level": 78,
      "signal_strength": -65
    }
  }
}
```

---

#### Update Order Status
```http
PUT /api/v1/orders/{order_id}/status
Authorization: Bearer <token>
Content-Type: application/json
```

**Request Body:**
```json
{
  "status": "delivered",
  "location": {
    "lat": 39.9156,
    "lng": 116.4114
  },
  "notes": "Delivered to reception desk",
  "proof_of_delivery": {
    "photo_url": "https://cdn.smartfleet.io/pod/abc123.jpg",
    "signature_data": "base64_encoded_signature",
    "recipient_name": "Bob Wang"
  }
}
```

**Valid Status Transitions:**
```
pending → assigned → picked_up → in_transit → delivered
   ↓         ↓          ↓           ↓            ↓
cancelled  failed    returned    diverted    cancelled
```

---

#### Dispatch Order (AI Assignment)
```http
POST /api/v1/orders/{order_id}/dispatch
Authorization: Bearer <token>
```

**Response:**
```json
{
  "success": true,
  "data": {
    "order_id": "550e8400-e29b-41d4-a716-446655440000",
    "vehicle": {
      "id": "660e8400-e29b-41d4-a716-446655440001",
      "type": "drone",
      "model": "DJI-M300-RTK",
      "current_position": {"lat": 39.9000, "lng": 116.4000}
    },
    "route": {
      "waypoints": [
        {"lat": 39.9000, "lng": 116.4000, "altitude": 80},
        {"lat": 39.9042, "lng": 116.4074, "altitude": 80},
        {"lat": 39.9156, "lng": 116.4114, "altitude": 50}
      ],
      "estimated_distance": 2500,
      "estimated_duration": 1200,
      "risk_score": 0.12
    },
    "eta": {
      "pickup": "2026-07-20T14:00:00Z",
      "delivery": "2026-07-20T14:20:00Z"
    },
    "dispatch_reason": "Optimal: closest drone (1.2km), 85% battery, clear weather"
  }
}
```

---

#### Track Order (Real-time)
```http
GET /api/v1/orders/{order_id}/tracking
```

**Response:**
```json
{
  "success": true,
  "data": {
    "order_id": "550e8400-e29b-41d4-a716-446655440000",
    "tracking_number": "SF7A3B9C2D1E4F5",
    "status": "in_transit",
    "progress_percentage": 65,
    "current_position": {
      "lat": 39.9100,
      "lng": 116.4100,
      "altitude": 75,
      "heading": 45,
      "speed": 12.5
    },
    "remaining_distance": 875,
    "estimated_arrival": "2026-07-20T14:18:00Z",
    "next_waypoint": {
      "lat": 39.9156,
      "lng": 116.4114,
      "distance_to": 875,
      "eta": "2026-07-20T14:18:00Z"
    }
  }
}
```

**WebSocket Alternative:**
```javascript
// Connect to real-time tracking stream
const ws = new WebSocket('wss://api.smartfleet.io/ws/tracking/SF7A3B9C2D1E4F5');

ws.onmessage = (event) => {
  const update = JSON.parse(event.data);
  // { position, speed, battery, status, timestamp }
};
```

---

### 2. FLEET SERVICE

#### List Vehicles
```http
GET /api/v1/fleet/vehicles?status=available&type=drone
Authorization: Bearer <token>
```

**Response:**
```json
{
  "success": true,
  "data": {
    "items": [
      {
        "id": "660e8400-e29b-41d4-a716-446655440001",
        "type": "drone",
        "model": "DJI-M300-RTK",
        "serial_number": "DJI-2024-001",
        "status": "available",
        "battery_level": 85,
        "max_payload": 2.7,
        "current_position": {"lat": 39.9000, "lng": 116.4000, "altitude": 50},
        "home_base": {"lat": 39.9000, "lng": 116.4000},
        "total_deliveries": 1247,
        "total_distance_km": 8560,
        "last_maintenance": "2026-07-15T08:00:00Z",
        "sensors": {
          "gps": true,
          "lidar": true,
          "camera": true,
          "imu": true
        }
      }
    ],
    "summary": {
      "total": 45,
      "available": 12,
      "busy": 28,
      "charging": 3,
      "maintenance": 2
    }
  }
}
```

---

#### Get Vehicle Telemetry
```http
GET /api/v1/fleet/vehicles/{vehicle_id}/telemetry?start=2026-07-20T10:00:00Z&end=2026-07-20T11:00:00Z
```

**Response:**
```json
{
  "success": true,
  "data": {
    "vehicle_id": "660e8400-e29b-41d4-a716-446655440001",
    "points": [
      {
        "timestamp": "2026-07-20T10:00:00Z",
        "position": {"lat": 39.9000, "lng": 116.4000, "altitude": 50},
        "speed": 0,
        "battery": 100,
        "temperature": 28.5,
        "obstacle_detected": false
      },
      {
        "timestamp": "2026-07-20T10:05:00Z",
        "position": {"lat": 39.9042, "lng": 116.4074, "altitude": 80},
        "speed": 15.2,
        "battery": 95,
        "temperature": 29.1,
        "obstacle_detected": false
      }
    ]
  }
}
```

---

#### Send Command to Vehicle
```http
POST /api/v1/fleet/vehicles/{vehicle_id}/commands
Authorization: Bearer <token>
Content-Type: application/json
```

**Request Body:**
```json
{
  "command": "RETURN_TO_BASE",
  "priority": "high",
  "reason": "Low battery - 15% remaining"
}
```

**Valid Commands:**
| Command | Description | Required Role |
|---------|-------------|---------------|
| `RETURN_TO_BASE` | Abort mission, return home | Operator+ |
| `LAND_IMMEDIATELY` | Emergency landing at nearest safe spot | Operator+ |
| `HOLD_POSITION` | Hover in place | Operator+ |
| `RESUME_MISSION` | Continue current delivery | Operator+ |
| `DIVERT` | Change route to new waypoint | Operator+ |
| `UPDATE_ROUTE` | Modify waypoints | System |

---

### 3. UTM SERVICE (Unmanned Traffic Management)

#### Register Flight Plan
```http
POST /api/v1/utm/flight-plans
Authorization: Bearer <token>
Content-Type: application/json
```

**Request Body:**
```json
{
  "vehicle_id": "660e8400-e29b-41d4-a716-446655440001",
  "order_id": "550e8400-e29b-41d4-a716-446655440000",
  "waypoints": [
    {"lat": 39.9000, "lng": 116.4000, "altitude": 80, "time": "2026-07-20T14:00:00Z"},
    {"lat": 39.9042, "lng": 116.4074, "altitude": 80, "time": "2026-07-20T14:05:00Z"},
    {"lat": 39.9156, "lng": 116.4114, "altitude": 50, "time": "2026-07-20T14:20:00Z"}
  ],
  "max_altitude": 120,
  "estimated_duration": 1200,
  "purpose": "delivery"
}
```

**Response:**
```json
{
  "success": true,
  "data": {
    "flight_plan_id": "fp-770e8400-e29b-41d4",
    "status": "approved",
    "approval_time": "2026-07-20T10:30:05Z",
    "restrictions": [
      "Maintain altitude below 120m",
      "Avoid zone Z-001 (hospital) within 100m",
      "Report position every 5 seconds"
    ],
    "conflicts": [],
    "geofences_applied": ["Z-001", "Z-002", "Z-003"]
  }
}
```

---

#### Get Active Airspace
```http
GET /api/v1/utm/airspace?bounds=39.90,116.40,39.92,116.42&altitude_max=150
```

**Response:**
```json
{
  "success": true,
  "data": {
    "bounds": {
      "north": 39.92,
      "south": 39.90,
      "east": 116.42,
      "west": 116.40
    },
    "active_vehicles": [
      {
        "vehicle_id": "660e8400-e29b-41d4-a716-446655440001",
        "position": {"lat": 39.9100, "lng": 116.4100, "altitude": 75},
        "velocity": {"x": 5.2, "y": 3.1, "z": 0},
        "flight_plan_id": "fp-770e8400-e29b-41d4",
        "priority": "normal"
      }
    ],
    "geofences": [
      {
        "id": "Z-001",
        "name": "Hospital Zone",
        "type": "restricted",
        "geometry": {
          "type": "Polygon",
          "coordinates": [[[116.405, 39.908], [116.407, 39.908], ...]]
        },
        "altitude_limit": 0,
        "buffer_meters": 100
      }
    ],
    "weather": {
      "wind_speed": 8.5,
      "wind_direction": 270,
      "visibility": 10000,
      "precipitation": 0
    }
  }
}
```

---

#### Report Position (Called by vehicle every 5 seconds)
```http
POST /api/v1/utm/position-reports
Authorization: Bearer <vehicle-token>
Content-Type: application/json
```

**Request Body:**
```json
{
  "vehicle_id": "660e8400-e29b-41d4-a716-446655440001",
  "timestamp": "2026-07-20T10:05:00Z",
  "position": {
    "lat": 39.9100,
    "lng": 116.4100,
    "altitude": 75,
    "accuracy": 1.2
  },
  "velocity": {
    "speed": 12.5,
    "heading": 45,
    "vertical_speed": 0.5
  },
  "battery_level": 78,
  "status": "in_transit"
}
```

**Response (if conflict detected):**
```json
{
  "success": true,
  "data": {
    "status": "warning",
    "alerts": [
      {
        "type": "proximity",
        "severity": "medium",
        "message": "Vehicle DJI-002 approaching within 200m",
        "recommended_action": "DESCEND_TO_60M",
        "time_to_conflict": 45
      }
    ]
  }
}
```

---

### 4. ANALYTICS SERVICE

#### Get Dashboard Metrics
```http
GET /api/v1/analytics/dashboard?period=today
Authorization: Bearer <token>
```

**Response:**
```json
{
  "success": true,
  "data": {
    "period": "2026-07-20",
    "orders": {
      "total": 1247,
      "completed": 1180,
      "in_progress": 45,
      "failed": 12,
      "cancelled": 10,
      "avg_delivery_time": 18.5,
      "on_time_rate": 0.94
    },
    "fleet": {
      "active_drones": 28,
      "active_robots": 17,
      "avg_battery": 72,
      "utilization_rate": 0.87
    },
    "revenue": {
      "total": 15840.00,
      "avg_order_value": 12.70,
      "distance_fare": 6230.00,
      "priority_fare": 1240.00
    },
    "performance": {
      "p50_latency_ms": 45,
      "p95_latency_ms": 120,
      "p99_latency_ms": 280,
      "error_rate": 0.002
    }
  }
}
```

---

#### Get Demand Heatmap
```http
GET /api/v1/analytics/heatmap?date=2026-07-20&hour=14
```

**Response:**
```json
{
  "success": true,
  "data": {
    "grid_size": 100,
    "bounds": {"north": 39.95, "south": 39.85, "east": 116.45, "west": 116.35},
    "cells": [
      {"lat": 39.900, "lng": 116.400, "demand": 45, "intensity": 0.9},
      {"lat": 39.905, "lng": 116.405, "demand": 32, "intensity": 0.7},
      {"lat": 39.910, "lng": 116.410, "demand": 12, "intensity": 0.3}
    ]
  }
}
```

---

## Error Codes

| Code | HTTP Status | Description |
|------|-------------|-------------|
| `INVALID_REQUEST` | 400 | Malformed request body |
| `UNAUTHORIZED` | 401 | Missing or invalid token |
| `FORBIDDEN` | 403 | Insufficient permissions |
| `ORDER_NOT_FOUND` | 404 | Order ID does not exist |
| `VEHICLE_UNAVAILABLE` | 409 | No vehicles available for dispatch |
| `GEOFENCE_VIOLATION` | 409 | Route violates restricted airspace |
| `RATE_LIMIT_EXCEEDED` | 429 | Too many requests |
| `INTERNAL_ERROR` | 500 | Server error |
| `SERVICE_UNAVAILABLE` | 503 | Service temporarily down |

---

## WebSocket Events

### Real-time Tracking Stream
```
wss://api.smartfleet.io/ws/tracking/{tracking_number}
```

**Events:**
```json
// position_update
{
  "event": "position_update",
  "timestamp": "2026-07-20T10:05:00Z",
  "data": {
    "position": {"lat": 39.9100, "lng": 116.4100, "altitude": 75},
    "speed": 12.5,
    "heading": 45,
    "battery": 78,
    "next_waypoint_distance": 875
  }
}

// status_change
{
  "event": "status_change",
  "timestamp": "2026-07-20T10:05:00Z",
  "data": {
    "old_status": "picked_up",
    "new_status": "in_transit",
    "location": {"lat": 39.9100, "lng": 116.4100}
  }
}

// delivery_complete
{
  "event": "delivery_complete",
  "timestamp": "2026-07-20T10:20:00Z",
  "data": {
    "delivery_time": "2026-07-20T10:20:00Z",
    "total_distance": 2500,
    "battery_remaining": 45,
    "proof_of_delivery": {
      "photo_url": "https://cdn.smartfleet.io/pod/abc123.jpg",
      "recipient": "Bob Wang"
    }
  }
}

// alert
{
  "event": "alert",
  "timestamp": "2026-07-20T10:15:00Z",
  "data": {
    "type": "weather_warning",
    "severity": "medium",
    "message": "Wind speed increasing to 12 m/s. Route adjusted.",
    "action_taken": "route_recalculated"
  }
}
```
