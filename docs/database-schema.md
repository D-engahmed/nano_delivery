# SmartFleet Database Schema
## PostgreSQL 15 + PostGIS 3.3

---

## Schema Overview

```
┌─────────────────────────────────────────────────────────────┐
│                    smartfleet_db                             │
│                                                              │
│  ┌────────────┐  ┌────────────┐  ┌────────────┐            │
│  │   users    │  │  vehicles  │  │   orders   │            │
│  └────────────┘  └────────────┘  └────────────┘            │
│         │              │              │                    │
│         └──────────────┼──────────────┘                    │
│                        │                                    │
│  ┌────────────┐  ┌────┴───────┐  ┌────────────┐            │
│  │  geofences │  │order_status│  │ flight_plans           │
│  └────────────┘  │  _history  │  └────────────┘            │
│                  └────────────┘                             │
│  ┌────────────────────────────────────────────┐            │
│  │         vehicle_telemetry                   │            │
│  │         (Time-series, Partitioned)        │            │
│  └────────────────────────────────────────────┘            │
└─────────────────────────────────────────────────────────────┘
```

---

## 1. USERS TABLE

```sql
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email VARCHAR(255) UNIQUE NOT NULL,
    hashed_password VARCHAR(255) NOT NULL,
    full_name VARCHAR(255),
    phone VARCHAR(50),

    -- Role-based access control
    role VARCHAR(50) DEFAULT 'customer',
    -- Valid roles: 'admin', 'operator', 'analyst', 'customer', 'system'

    is_active BOOLEAN DEFAULT TRUE,
    email_verified BOOLEAN DEFAULT FALSE,
    phone_verified BOOLEAN DEFAULT FALSE,

    -- Profile
    avatar_url VARCHAR(500),
    preferred_language VARCHAR(10) DEFAULT 'zh-CN',

    -- Timestamps
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    last_login_at TIMESTAMP WITH TIME ZONE,

    -- Constraints
    CONSTRAINT chk_email_format CHECK (email ~* '^[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}$'),
    CONSTRAINT chk_role CHECK (role IN ('admin', 'operator', 'analyst', 'customer', 'system'))
);

-- Indexes
CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_role ON users(role) WHERE is_active = TRUE;
CREATE INDEX idx_users_created_at ON users(created_at DESC);

-- Trigger for updated_at
CREATE OR REPLACE FUNCTION update_updated_at_column()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = CURRENT_TIMESTAMP;
    RETURN NEW;
END;
$$ language 'plpgsql';

CREATE TRIGGER update_users_updated_at
    BEFORE UPDATE ON users
    FOR EACH ROW
    EXECUTE FUNCTION update_updated_at_column();
```

---

## 2. VEHICLES TABLE

```sql
CREATE TABLE vehicles (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),

    -- Vehicle identification
    vehicle_type VARCHAR(50) NOT NULL,  -- 'drone', 'robot'
    model VARCHAR(100) NOT NULL,         -- 'DJI-M300-RTK', 'JD-Auto-1'
    serial_number VARCHAR(100) UNIQUE NOT NULL,

    -- Operational status
    status VARCHAR(50) DEFAULT 'available',
    -- Valid: 'available', 'busy', 'charging', 'maintenance', 'offline', 'emergency'

    -- Battery & Power
    battery_level INTEGER CHECK (battery_level >= 0 AND battery_level <= 100),
    battery_voltage DECIMAL(5, 2),       -- e.g., 22.50V
    battery_cycles INTEGER DEFAULT 0,     -- Charge/discharge cycles

    -- Payload capacity
    max_payload_weight DECIMAL(10, 3),   -- kg
    max_payload_dimensions VARCHAR(50),  -- "LxWxH in cm"
    remaining_capacity DECIMAL(10, 3),   -- Current available capacity

    -- Position (GPS + PostGIS)
    current_position GEOGRAPHY(POINT, 4326),
    current_lat DECIMAL(10, 8),
    current_lng DECIMAL(11, 8),
    current_altitude DECIMAL(10, 2),     -- meters
    current_heading DECIMAL(10, 2),      -- degrees (0-360)
    position_accuracy DECIMAL(5, 2),     -- GPS accuracy in meters
    last_position_update TIMESTAMP WITH TIME ZONE,

    -- Home base (charging station / depot)
    home_base_position GEOGRAPHY(POINT, 4326),
    home_base_lat DECIMAL(10, 8),
    home_base_lng DECIMAL(11, 8),

    -- Performance metrics
    total_deliveries INTEGER DEFAULT 0,
    total_distance_km DECIMAL(10, 2) DEFAULT 0,
    total_flight_hours DECIMAL(10, 2) DEFAULT 0,

    -- Maintenance
    last_maintenance_at TIMESTAMP WITH TIME ZONE,
    next_maintenance_at TIMESTAMP WITH TIME ZONE,
    maintenance_notes TEXT,

    -- Sensors & capabilities
    has_gps BOOLEAN DEFAULT TRUE,
    has_lidar BOOLEAN DEFAULT FALSE,
    has_camera BOOLEAN DEFAULT TRUE,
    has_infrared BOOLEAN DEFAULT FALSE,

    -- Communication
    signal_strength INTEGER,             -- dBm
    network_type VARCHAR(20),            -- '4G', '5G', 'WiFi'
    last_heartbeat_at TIMESTAMP WITH TIME ZONE,

    -- Firmware
    firmware_version VARCHAR(50),

    -- Timestamps
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,

    -- Constraints
    CONSTRAINT chk_vehicle_type CHECK (vehicle_type IN ('drone', 'robot')),
    CONSTRAINT chk_status CHECK (status IN ('available', 'busy', 'charging', 'maintenance', 'offline', 'emergency')),
    CONSTRAINT chk_battery CHECK (battery_level >= 0 AND battery_level <= 100)
);

-- Spatial indexes for location-based queries
CREATE INDEX idx_vehicles_position ON vehicles USING GIST (current_position);
CREATE INDEX idx_vehicles_home_base ON vehicles USING GIST (home_base_position);
CREATE INDEX idx_vehicles_status ON vehicles(status) WHERE status IN ('available', 'busy');
CREATE INDEX idx_vehicles_type_status ON vehicles(vehicle_type, status);
CREATE INDEX idx_vehicles_battery ON vehicles(battery_level) WHERE status = 'available';
CREATE INDEX idx_vehicles_last_heartbeat ON vehicles(last_heartbeat_at DESC);

CREATE TRIGGER update_vehicles_updated_at
    BEFORE UPDATE ON vehicles
    FOR EACH ROW
    EXECUTE FUNCTION update_updated_at_column();
```

### Vehicle Status State Machine
```
                    ┌─────────────┐
                    │  available  │◄────────────────────┐
                    └──────┬──────┘                     │
                           │ assign()                   │ return_to_base()
                           ▼                            │
                    ┌─────────────┐                     │
                    │    busy     │                     │
                    └──────┬──────┘                     │
                           │                            │
           ┌───────────────┼───────────────┐            │
           │               │               │            │
           ▼               ▼               ▼            │
    ┌─────────────┐ ┌─────────────┐ ┌─────────────┐   │
    │   charging  │ │ maintenance │ │   offline   │───┘
    └──────┬──────┘ └──────┬──────┘ └──────┬──────┘
           │               │               │
           └───────────────┴───────────────┘
                           │
                           ▼
                    ┌─────────────┐
                    │  emergency  │ (manual intervention required)
                    └─────────────┘
```

---

## 3. ORDERS TABLE

```sql
CREATE TABLE orders (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),

    -- Relationships
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE RESTRICT,
    vehicle_id UUID REFERENCES vehicles(id) ON DELETE SET NULL,

    -- Order status
    status VARCHAR(50) DEFAULT 'pending',
    -- Valid: 'pending', 'assigned', 'picked_up', 'in_transit', 'delivered', 
    --        'cancelled', 'failed', 'returned', 'diverted'

    priority VARCHAR(20) DEFAULT 'normal',
    -- Valid: 'low', 'normal', 'high', 'urgent'

    -- Pickup details
    pickup_address TEXT NOT NULL,
    pickup_lat DECIMAL(10, 8) NOT NULL,
    pickup_lng DECIMAL(11, 8) NOT NULL,
    pickup_position GEOGRAPHY(POINT, 4326),
    pickup_contact_name VARCHAR(255),
    pickup_contact_phone VARCHAR(50),
    pickup_instructions TEXT,
    pickup_building_name VARCHAR(255),
    pickup_floor VARCHAR(50),
    actual_pickup_at TIMESTAMP WITH TIME ZONE,

    -- Delivery details
    delivery_address TEXT NOT NULL,
    delivery_lat DECIMAL(10, 8) NOT NULL,
    delivery_lng DECIMAL(11, 8) NOT NULL,
    delivery_position GEOGRAPHY(POINT, 4326),
    delivery_contact_name VARCHAR(255),
    delivery_contact_phone VARCHAR(50),
    delivery_instructions TEXT,
    delivery_building_name VARCHAR(255),
    delivery_floor VARCHAR(50),
    actual_delivery_at TIMESTAMP WITH TIME ZONE,

    -- Package details
    package_weight DECIMAL(10, 3),        -- kg
    package_dimensions VARCHAR(50),         -- "LxWxH cm"
    package_description TEXT,
    package_category VARCHAR(50),         -- 'food', 'medical', 'document', 'electronics'
    is_fragile BOOLEAN DEFAULT FALSE,
    requires_signature BOOLEAN DEFAULT FALSE,
    is_temperature_sensitive BOOLEAN DEFAULT FALSE,
    temperature_required DECIMAL(5, 2),   -- Celsius, if temperature sensitive

    -- Pricing
    base_fare DECIMAL(10, 2) NOT NULL DEFAULT 5.00,
    distance_fare DECIMAL(10, 2) NOT NULL DEFAULT 0.00,
    weight_fare DECIMAL(10, 2) DEFAULT 0.00,
    priority_fee DECIMAL(10, 2) DEFAULT 0.00,
    weather_fee DECIMAL(10, 2) DEFAULT 0.00,
    discount_amount DECIMAL(10, 2) DEFAULT 0.00,
    total_amount DECIMAL(10, 2) NOT NULL,
    currency VARCHAR(3) DEFAULT 'CNY',

    -- Payment
    payment_status VARCHAR(50) DEFAULT 'pending',
    -- Valid: 'pending', 'paid', 'failed', 'refunded', 'disputed'
    payment_method VARCHAR(50),           -- 'alipay', 'wechat', 'card', 'cash'
    payment_at TIMESTAMP WITH TIME ZONE,

    -- Route & tracking
    route_waypoints JSONB,                -- Array of {lat, lng, altitude, time}
    estimated_distance_meters INTEGER,
    estimated_duration_seconds INTEGER,
    actual_distance_meters INTEGER,
    actual_duration_seconds INTEGER,

    -- QR code for pickup verification
    qr_code VARCHAR(255) UNIQUE,
    qr_code_expires_at TIMESTAMP WITH TIME ZONE,

    -- Tracking
    tracking_number VARCHAR(100) UNIQUE NOT NULL,

    -- Delivery proof
    proof_of_delivery JSONB,              -- {photo_url, signature_data, recipient_name}
    delivery_photo_url VARCHAR(500),
    delivery_signature_data TEXT,
    delivery_recipient_name VARCHAR(255),

    -- Cancellation
    cancelled_at TIMESTAMP WITH TIME ZONE,
    cancellation_reason TEXT,
    cancelled_by UUID REFERENCES users(id),

    -- Failure
    failed_at TIMESTAMP WITH TIME ZONE,
    failure_reason TEXT,
    failure_code VARCHAR(50),

    -- Ratings
    customer_rating INTEGER CHECK (customer_rating >= 1 AND customer_rating <= 5),
    customer_comment TEXT,
    rated_at TIMESTAMP WITH TIME ZONE,

    -- Scheduling
    scheduled_pickup_at TIMESTAMP WITH TIME ZONE,
    scheduled_delivery_at TIMESTAMP WITH TIME ZONE,

    -- Timestamps
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    completed_at TIMESTAMP WITH TIME ZONE,

    -- Constraints
    CONSTRAINT chk_order_status CHECK (
        status IN ('pending', 'assigned', 'picked_up', 'in_transit', 
                   'delivered', 'cancelled', 'failed', 'returned', 'diverted')
    ),
    CONSTRAINT chk_priority CHECK (priority IN ('low', 'normal', 'high', 'urgent')),
    CONSTRAINT chk_payment_status CHECK (
        payment_status IN ('pending', 'paid', 'failed', 'refunded', 'disputed')
    ),
    CONSTRAINT chk_customer_rating CHECK (customer_rating >= 1 AND customer_rating <= 5),
    CONSTRAINT chk_amount_positive CHECK (total_amount >= 0)
);

-- Indexes
CREATE INDEX idx_orders_user_id ON orders(user_id);
CREATE INDEX idx_orders_vehicle_id ON orders(vehicle_id) WHERE vehicle_id IS NOT NULL;
CREATE INDEX idx_orders_status ON orders(status);
CREATE INDEX idx_orders_tracking ON orders(tracking_number);
CREATE INDEX idx_orders_created_at ON orders(created_at DESC);
CREATE INDEX idx_orders_status_created ON orders(status, created_at DESC);
CREATE INDEX idx_orders_pickup_position ON orders USING GIST (pickup_position);
CREATE INDEX idx_orders_delivery_position ON orders USING GIST (delivery_position);
CREATE INDEX idx_orders_scheduled ON orders(scheduled_pickup_at) WHERE status = 'pending';
CREATE INDEX idx_orders_completed ON orders(completed_at DESC) WHERE status = 'delivered';

-- Composite index for common query: find orders by user + status
CREATE INDEX idx_orders_user_status ON orders(user_id, status, created_at DESC);

CREATE TRIGGER update_orders_updated_at
    BEFORE UPDATE ON orders
    FOR EACH ROW
    EXECUTE FUNCTION update_updated_at_column();
```

### Order Status State Machine
```
┌─────────┐    assign()    ┌──────────┐   pickup()   ┌────────────┐
│ pending │──────────────►│ assigned │─────────────►│ picked_up  │
└────┬────┘               └────┬─────┘              └─────┬──────┘
     │                         │                          │
     │ cancel()                │ cancel()                 │ deliver()
     ▼                         ▼                          ▼
┌──────────┐               ┌──────────┐              ┌────────────┐
│cancelled │               │cancelled │              │ delivered  │
└──────────┘               └──────────┘              └─────┬──────┘
                                                           │
                                                           │ fail()
                                                           ▼
                                                    ┌────────────┐
                                                    │   failed   │
                                                    └────────────┘
```

---

## 4. ORDER STATUS HISTORY (Audit Trail)

```sql
CREATE TABLE order_status_history (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    order_id UUID NOT NULL REFERENCES orders(id) ON DELETE CASCADE,

    status VARCHAR(50) NOT NULL,
    previous_status VARCHAR(50),

    -- Location at status change
    location_lat DECIMAL(10, 8),
    location_lng DECIMAL(11, 8),
    location_position GEOGRAPHY(POINT, 4326),
    altitude DECIMAL(10, 2),

    -- Context
    notes TEXT,
    changed_by UUID REFERENCES users(id),  -- NULL if system-initiated
    changed_by_system BOOLEAN DEFAULT FALSE, -- TRUE if automated (AI dispatch, etc.)

    -- Vehicle state at change
    vehicle_battery_level INTEGER,
    vehicle_position_lat DECIMAL(10, 8),
    vehicle_position_lng DECIMAL(11, 8),

    -- Timestamps
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,

    -- Partitioning by month for performance
    CONSTRAINT status_history_partition_check 
        CHECK (created_at >= '2024-01-01'::TIMESTAMP WITH TIME ZONE)
) PARTITION BY RANGE (created_at);

-- Create monthly partitions
CREATE TABLE order_status_history_2024_01 PARTITION OF order_status_history
    FOR VALUES FROM ('2024-01-01') TO ('2024-02-01');
CREATE TABLE order_status_history_2024_02 PARTITION OF order_status_history
    FOR VALUES FROM ('2024-02-01') TO ('2024-03-01');
-- ... continue for each month

-- Indexes
CREATE INDEX idx_status_history_order ON order_status_history(order_id, created_at DESC);
CREATE INDEX idx_status_history_created ON order_status_history(created_at DESC);
CREATE INDEX idx_status_history_status ON order_status_history(status, created_at DESC);
```

---

## 5. GEOFENCES (No-Fly Zones)

```sql
CREATE TABLE geofences (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),

    name VARCHAR(255) NOT NULL,
    description TEXT,

    -- Zone classification
    zone_type VARCHAR(50) NOT NULL,
    -- Valid: 'restricted' (no entry), 'warning' (altitude limit), 
    --        'temporary' (time-limited), 'dynamic' (weather-based)

    -- Geometry (PostGIS)
    geometry GEOGRAPHY(POLYGON, 4326) NOT NULL,

    -- Altitude restrictions
    altitude_min INTEGER,  -- meters above ground
    altitude_max INTEGER,  -- meters above ground

    -- Time restrictions (for temporary zones)
    effective_from TIMESTAMP WITH TIME ZONE,
    effective_until TIMESTAMP WITH TIME ZONE,
    recurring_schedule JSONB,  -- {"days": [1,2,3,4,5], "start_time": "08:00", "end_time": "18:00"}

    -- Metadata
    created_by UUID REFERENCES users(id),
    approved_by UUID REFERENCES users(id),
    approved_at TIMESTAMP WITH TIME ZONE,

    -- Source
    source VARCHAR(100),  -- 'civil_aviation', 'military', 'police', 'weather', 'operator'
    external_id VARCHAR(100),  -- ID from external system (e.g., CAAC)

    is_active BOOLEAN DEFAULT TRUE,

    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,

    CONSTRAINT chk_zone_type CHECK (
        zone_type IN ('restricted', 'warning', 'temporary', 'dynamic')
    )
);

-- Spatial index for fast intersection queries
CREATE INDEX idx_geofences_geometry ON geofences USING GIST (geometry);
CREATE INDEX idx_geofences_active ON geofences(is_active) WHERE is_active = TRUE;
CREATE INDEX idx_geofences_type ON geofences(zone_type, is_active);

-- Function to check if a point is inside any active geofence
CREATE OR REPLACE FUNCTION check_geofence_violation(
    p_lat DECIMAL(10, 8),
    p_lng DECIMAL(11, 8),
    p_altitude INTEGER
) RETURNS TABLE (
    geofence_id UUID,
    geofence_name VARCHAR(255),
    violation_type VARCHAR(50)
) AS $$
BEGIN
    RETURN QUERY
    SELECT 
        g.id,
        g.name,
        g.zone_type AS violation_type
    FROM geofences g
    WHERE g.is_active = TRUE
      AND ST_Contains(
          g.geometry::geometry,
          ST_SetSRID(ST_MakePoint(p_lng, p_lat), 4326)
      )
      AND (g.altitude_min IS NULL OR p_altitude >= g.altitude_min)
      AND (g.altitude_max IS NULL OR p_altitude <= g.altitude_max)
      AND (g.effective_from IS NULL OR NOW() >= g.effective_from)
      AND (g.effective_until IS NULL OR NOW() <= g.effective_until);
END;
$$ LANGUAGE plpgsql;
```

---

## 6. FLIGHT PLANS

```sql
CREATE TABLE flight_plans (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),

    -- Relationships
    vehicle_id UUID NOT NULL REFERENCES vehicles(id),
    order_id UUID REFERENCES orders(id) ON DELETE SET NULL,

    -- Status
    status VARCHAR(50) DEFAULT 'pending',
    -- Valid: 'pending', 'submitted', 'under_review', 'approved', 'rejected',
    --        'active', 'completed', 'cancelled', 'emergency_terminated'

    -- Route
    waypoints JSONB NOT NULL,
    -- [
    --   {"lat": 39.9042, "lng": 116.4074, "altitude": 80, 
    --    "estimated_time": "2026-07-20T14:00:00Z", "type": "pickup"},
    --   {"lat": 39.9100, "lng": 116.4100, "altitude": 100, 
    --    "estimated_time": "2026-07-20T14:10:00Z", "type": "transit"},
    --   {"lat": 39.9156, "lng": 116.4114, "altitude": 50, 
    --    "estimated_time": "2026-07-20T14:20:00Z", "type": "delivery"}
    -- ]

    route_geometry GEOGRAPHY(LINESTRING, 4326),

    -- Metrics
    estimated_distance_meters INTEGER,
    estimated_duration_seconds INTEGER,
    actual_distance_meters INTEGER,
    actual_duration_seconds INTEGER,

    -- Altitude
    max_altitude INTEGER,
    min_altitude INTEGER,

    -- Safety
    geofences_checked UUID[],  -- Array of geofence IDs checked
    conflicts_detected JSONB,   -- [{"vehicle_id": "...", "distance": 150, "time": "..."}]
    risk_score DECIMAL(3, 2),   -- 0.00 to 1.00

    -- Weather at submission
    weather_at_submission JSONB,
    -- {"wind_speed": 8.5, "wind_direction": 270, "visibility": 10000, "precipitation": 0}

    -- Timing
    planned_start_at TIMESTAMP WITH TIME ZONE,
    planned_end_at TIMESTAMP WITH TIME ZONE,
    actual_start_at TIMESTAMP WITH TIME ZONE,
    actual_end_at TIMESTAMP WITH TIME ZONE,

    -- Approval
    submitted_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    reviewed_at TIMESTAMP WITH TIME ZONE,
    reviewed_by UUID REFERENCES users(id),
    rejection_reason TEXT,

    -- Emergency
    emergency_declared_at TIMESTAMP WITH TIME ZONE,
    emergency_reason TEXT,
    emergency_action_taken TEXT,

    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,

    CONSTRAINT chk_flight_status CHECK (
        status IN ('pending', 'submitted', 'under_review', 'approved', 'rejected',
                   'active', 'completed', 'cancelled', 'emergency_terminated')
    )
);

CREATE INDEX idx_flight_plans_vehicle ON flight_plans(vehicle_id, status);
CREATE INDEX idx_flight_plans_order ON flight_plans(order_id);
CREATE INDEX idx_flight_plans_status ON flight_plans(status, planned_start_at);
CREATE INDEX idx_flight_plans_time ON flight_plans(planned_start_at, planned_end_at);
```

---

## 7. VEHICLE TELEMETRY (Time-Series)

```sql
CREATE TABLE vehicle_telemetry (
    id BIGSERIAL,

    -- Vehicle reference
    vehicle_id UUID NOT NULL REFERENCES vehicles(id),

    -- Timestamp (partition key)
    timestamp TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP,

    -- Position
    position GEOGRAPHY(POINT, 4326),
    lat DECIMAL(10, 8) NOT NULL,
    lng DECIMAL(11, 8) NOT NULL,
    altitude DECIMAL(10, 2),           -- meters
    accuracy DECIMAL(5, 2),            -- GPS accuracy in meters

    -- Movement
    speed DECIMAL(10, 2),              -- m/s
    heading DECIMAL(10, 2),            -- degrees (0-360)
    vertical_speed DECIMAL(10, 2),     -- m/s (positive = ascending)

    -- Battery
    battery_level INTEGER,
    battery_voltage DECIMAL(5, 2),
    battery_current DECIMAL(6, 3),     -- Amps (positive = charging)
    battery_temperature DECIMAL(5, 2), -- Celsius

    -- Environment
    ambient_temperature DECIMAL(5, 2),
    wind_speed DECIMAL(5, 2),
    wind_direction DECIMAL(10, 2),

    -- Communication
    signal_strength INTEGER,           -- dBm
    network_latency_ms INTEGER,
    packet_loss_rate DECIMAL(5, 4),   -- 0.0000 to 1.0000

    -- Sensors
    obstacle_detected BOOLEAN DEFAULT FALSE,
    obstacle_distance DECIMAL(10, 2),  -- meters
    obstacle_direction DECIMAL(10, 2), -- degrees

    gps_satellites INTEGER,
    gps_hdop DECIMAL(5, 2),           -- Horizontal Dilution of Precision

    -- IMU (Inertial Measurement Unit)
    acceleration_x DECIMAL(10, 6),    -- m/s^2
    acceleration_y DECIMAL(10, 6),
    acceleration_z DECIMAL(10, 6),
    gyro_x DECIMAL(10, 6),            -- rad/s
    gyro_y DECIMAL(10, 6),
    gyro_z DECIMAL(10, 6),

    -- Status
    flight_mode VARCHAR(50),          -- 'manual', 'auto', 'rtl', 'land', 'hover'
    motor_status VARCHAR(50),         -- 'armed', 'disarmed', 'error'

    -- Order reference (if on delivery)
    active_order_id UUID REFERENCES orders(id),
    waypoint_index INTEGER,           -- Current waypoint in route
    distance_to_next_waypoint DECIMAL(10, 2),

    -- Partitioning
    PRIMARY KEY (id, timestamp)
) PARTITION BY RANGE (timestamp);

-- Create monthly partitions (automated via cron or pg_partman)
CREATE TABLE vehicle_telemetry_2024_01 PARTITION OF vehicle_telemetry
    FOR VALUES FROM ('2024-01-01') TO ('2024-02-01');
CREATE TABLE vehicle_telemetry_2024_02 PARTITION OF vehicle_telemetry
    FOR VALUES FROM ('2024-02-01') TO ('2024-03-01');
-- ... etc

-- Indexes
CREATE INDEX idx_telemetry_vehicle_time ON vehicle_telemetry(vehicle_id, timestamp DESC);
CREATE INDEX idx_telemetry_time ON vehicle_telemetry(timestamp DESC);
CREATE INDEX idx_telemetry_position ON vehicle_telemetry USING GIST (position);
CREATE INDEX idx_telemetry_order ON vehicle_telemetry(active_order_id, timestamp DESC);

-- Hyperloglog for approximate distinct counts (optional extension)
-- For "how many unique vehicles were active today?"
```

---

## 8. NOTIFICATIONS

```sql
CREATE TABLE notifications (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),

    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    order_id UUID REFERENCES orders(id) ON DELETE CASCADE,

    -- Content
    type VARCHAR(50) NOT NULL,
    -- Valid: 'order_status', 'delivery_eta', 'delivery_complete', 
    --        'payment_success', 'payment_failed', 'promotion',
    --        'system_alert', 'vehicle_status'

    title VARCHAR(255) NOT NULL,
    message TEXT NOT NULL,

    -- Deep link / action
    action_url VARCHAR(500),
    action_type VARCHAR(50),  -- 'open_order', 'track_delivery', 'rate_order'
    action_data JSONB,        -- {"order_id": "...", "tracking_number": "..."}

    -- Delivery
    channel VARCHAR(50),      -- 'push', 'sms', 'email', 'in_app'
    sent_at TIMESTAMP WITH TIME ZONE,
    delivered_at TIMESTAMP WITH TIME ZONE,
    read_at TIMESTAMP WITH TIME ZONE,

    -- Status
    status VARCHAR(50) DEFAULT 'pending',
    -- Valid: 'pending', 'sent', 'delivered', 'read', 'failed'

    -- Retry
    retry_count INTEGER DEFAULT 0,
    last_error TEXT,

    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,

    CONSTRAINT chk_notification_type CHECK (
        type IN ('order_status', 'delivery_eta', 'delivery_complete',
                 'payment_success', 'payment_failed', 'promotion',
                 'system_alert', 'vehicle_status')
    ),
    CONSTRAINT chk_notification_status CHECK (
        status IN ('pending', 'sent', 'delivered', 'read', 'failed')
    )
);

CREATE INDEX idx_notifications_user ON notifications(user_id, created_at DESC);
CREATE INDEX idx_notifications_unread ON notifications(user_id, status) WHERE status != 'read';
CREATE INDEX idx_notifications_order ON notifications(order_id, created_at DESC);
```

---

## 9. AUDIT LOG (Security & Compliance)

```sql
CREATE TABLE audit_logs (
    id BIGSERIAL PRIMARY KEY,

    -- Who
    user_id UUID REFERENCES users(id),
    user_email VARCHAR(255),
    user_role VARCHAR(50),

    -- What
    action VARCHAR(100) NOT NULL,     -- 'order_created', 'vehicle_dispatched', 'login_failed'
    resource_type VARCHAR(50),        -- 'order', 'vehicle', 'user', 'flight_plan'
    resource_id UUID,

    -- Details
    old_values JSONB,
    new_values JSONB,
    changes JSONB GENERATED ALWAYS AS (
        jsonb_diff(old_values, new_values)
    ) STORED,

    -- Context
    ip_address INET,
    user_agent TEXT,
    session_id VARCHAR(255),
    request_id VARCHAR(255),

    -- Result
    success BOOLEAN DEFAULT TRUE,
    error_message TEXT,

    -- Timestamp
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,

    -- Partitioning by day for high volume
    CONSTRAINT audit_logs_partition_check 
        CHECK (created_at >= '2024-01-01'::TIMESTAMP WITH TIME ZONE)
) PARTITION BY RANGE (created_at);

CREATE INDEX idx_audit_user ON audit_logs(user_id, created_at DESC);
CREATE INDEX idx_audit_action ON audit_logs(action, created_at DESC);
CREATE INDEX idx_audit_resource ON audit_logs(resource_type, resource_id);
CREATE INDEX idx_audit_request ON audit_logs(request_id);
```

---

## 10. MATERIALIZED VIEWS (Performance)

```sql
-- Daily order statistics (refreshed every hour)
CREATE MATERIALIZED VIEW mv_daily_order_stats AS
SELECT 
    DATE(created_at) AS date,
    COUNT(*) AS total_orders,
    COUNT(*) FILTER (WHERE status = 'delivered') AS delivered_orders,
    COUNT(*) FILTER (WHERE status = 'cancelled') AS cancelled_orders,
    COUNT(*) FILTER (WHERE status = 'failed') AS failed_orders,
    AVG(EXTRACT(EPOCH FROM (completed_at - created_at))) FILTER (WHERE status = 'delivered') AS avg_delivery_time_seconds,
    SUM(total_amount) AS total_revenue,
    AVG(total_amount) AS avg_order_value
FROM orders
WHERE created_at >= CURRENT_DATE - INTERVAL '90 days'
GROUP BY DATE(created_at);

CREATE UNIQUE INDEX idx_mv_daily_stats_date ON mv_daily_order_stats(date);

-- Refresh strategy: cron job or pg_cron extension
-- SELECT cron.schedule('refresh-stats', '0 * * * *', 'REFRESH MATERIALIZED VIEW CONCURRENTLY mv_daily_order_stats');

-- Vehicle utilization (refreshed every 5 minutes)
CREATE MATERIALIZED VIEW mv_vehicle_utilization AS
SELECT 
    v.id AS vehicle_id,
    v.vehicle_type,
    v.model,
    v.status,
    COUNT(o.id) FILTER (WHERE o.status IN ('assigned', 'picked_up', 'in_transit')) AS active_orders,
    COUNT(o.id) FILTER (WHERE o.status = 'delivered' AND o.completed_at >= CURRENT_DATE) AS today_deliveries,
    SUM(o.actual_distance_meters) FILTER (WHERE o.status = 'delivered' AND o.completed_at >= CURRENT_DATE) AS today_distance_meters,
    AVG(EXTRACT(EPOCH FROM (o.completed_at - o.created_at))) FILTER (WHERE o.status = 'delivered' AND o.created_at >= CURRENT_DATE - INTERVAL '7 days') AS avg_delivery_time_7d
FROM vehicles v
LEFT JOIN orders o ON o.vehicle_id = v.id AND o.created_at >= CURRENT_DATE - INTERVAL '7 days'
GROUP BY v.id, v.vehicle_type, v.model, v.status;

CREATE UNIQUE INDEX idx_mv_vehicle_util_id ON mv_vehicle_utilization(vehicle_id);
```

---

## Database Configuration

```sql
-- Enable required extensions
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
CREATE EXTENSION IF NOT EXISTS "postgis";
CREATE EXTENSION IF NOT EXISTS "pg_stat_statements";  -- Query performance monitoring

-- Connection pooling settings (for PgBouncer)
-- max_connections = 200
-- shared_buffers = 4GB
-- effective_cache_size = 12GB
-- work_mem = 16MB
-- maintenance_work_mem = 512MB

-- Auto-vacuum for high-update tables
ALTER TABLE vehicle_telemetry SET (
    autovacuum_vacuum_scale_factor = 0.05,
    autovacuum_analyze_scale_factor = 0.02
);

-- Table partitioning automation
CREATE OR REPLACE FUNCTION create_telemetry_partition()
RETURNS void AS $$
DECLARE
    partition_date DATE;
    partition_name TEXT;
    start_date DATE;
    end_date DATE;
BEGIN
    partition_date := DATE_TRUNC('month', CURRENT_DATE + INTERVAL '1 month');
    partition_name := 'vehicle_telemetry_' || TO_CHAR(partition_date, 'YYYY_MM');
    start_date := partition_date;
    end_date := partition_date + INTERVAL '1 month';

    EXECUTE format(
        'CREATE TABLE IF NOT EXISTS %I PARTITION OF vehicle_telemetry FOR VALUES FROM (%L) TO (%L)',
        partition_name, start_date, end_date
    );
END;
$$ LANGUAGE plpgsql;
```
