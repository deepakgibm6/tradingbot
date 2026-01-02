# Database Schemas

## PostgreSQL (OLTP) - User Data

### Users Table

```sql
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email VARCHAR(255) UNIQUE NOT NULL,
    name VARCHAR(255) NOT NULL,
    picture_url TEXT,
    role VARCHAR(50) NOT NULL DEFAULT 'free',
    email_verified BOOLEAN DEFAULT FALSE,
    
    -- Timestamps
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    last_login_at TIMESTAMP WITH TIME ZONE,
    
    -- Metadata
    metadata JSONB DEFAULT '{}',
    
    -- Indexes
    CONSTRAINT chk_role CHECK (role IN ('free', 'enterprise', 'admin'))
);

CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_role ON users(role);
CREATE INDEX idx_users_created_at ON users(created_at DESC);
```

### Watchlists Table

```sql
CREATE TABLE watchlists (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    name VARCHAR(255) NOT NULL,
    symbols TEXT[] NOT NULL,  -- Array of stock symbols
    
    -- Settings
    is_default BOOLEAN DEFAULT FALSE,
    sort_order INTEGER DEFAULT 0,
    
    -- Timestamps
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    
    -- Constraints
    CONSTRAINT fk_user FOREIGN KEY (user_id) REFERENCES users(id)
);

CREATE INDEX idx_watchlists_user_id ON watchlists(user_id);
CREATE INDEX idx_watchlists_symbols ON watchlists USING GIN(symbols);
```

### Alerts Table

```sql
CREATE TABLE alerts (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    symbol VARCHAR(50) NOT NULL,
    
    -- Alert conditions
    condition_type VARCHAR(50) NOT NULL,  -- 'price_above', 'price_below', 'signal_generated', 'ml_prediction'
    condition_value DECIMAL(20, 4),
    
    -- Status
    is_active BOOLEAN DEFAULT TRUE,
    triggered_at TIMESTAMP WITH TIME ZONE,
    notification_sent BOOLEAN DEFAULT FALSE,
    
    -- Timestamps
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    
    -- Constraints
    CONSTRAINT chk_condition_type CHECK (
        condition_type IN ('price_above', 'price_below', 'signal_generated', 'ml_prediction', 'volume_spike')
    )
);

CREATE INDEX idx_alerts_user_id ON alerts(user_id);
CREATE INDEX idx_alerts_symbol ON alerts(symbol);
CREATE INDEX idx_alerts_active ON alerts(is_active) WHERE is_active = TRUE;
```

### API Keys Table (Enterprise Users)

```sql
CREATE TABLE api_keys (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    
    -- Key details
    key_hash VARCHAR(255) UNIQUE NOT NULL,  -- SHA-256 hash of API key
    key_prefix VARCHAR(20) NOT NULL,  -- First 8 chars for identification
    name VARCHAR(255),
    
    -- Permissions
    scopes TEXT[] DEFAULT ARRAY['read:market_data', 'read:signals'],  -- Permission scopes
    
    -- Rate limiting
    rate_limit_per_minute INTEGER DEFAULT 60,
    rate_limit_per_day INTEGER DEFAULT 10000,
    
    -- Status
    is_active BOOLEAN DEFAULT TRUE,
    last_used_at TIMESTAMP WITH TIME ZONE,
    expires_at TIMESTAMP WITH TIME ZONE,
    
    -- Timestamps
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    
    CONSTRAINT fk_user FOREIGN KEY (user_id) REFERENCES users(id)
);

CREATE INDEX idx_api_keys_user_id ON api_keys(user_id);
CREATE INDEX idx_api_keys_key_hash ON api_keys(key_hash);
CREATE INDEX idx_api_keys_active ON api_keys(is_active) WHERE is_active = TRUE;
```

### User Sessions Table

```sql
CREATE TABLE user_sessions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    
    -- Session details
    token_hash VARCHAR(255) UNIQUE NOT NULL,
    ip_address INET,
    user_agent TEXT,
    
    -- Timestamps
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    expires_at TIMESTAMP WITH TIME ZONE NOT NULL,
    last_activity_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    
    CONSTRAINT fk_user FOREIGN KEY (user_id) REFERENCES users(id)
);

CREATE INDEX idx_sessions_user_id ON user_sessions(user_id);
CREATE INDEX idx_sessions_expires_at ON user_sessions(expires_at);

-- Auto-delete expired sessions
CREATE OR REPLACE FUNCTION delete_expired_sessions()
RETURNS void AS $$
BEGIN
    DELETE FROM user_sessions WHERE expires_at < NOW();
END;
$$ LANGUAGE plpgsql;

-- Schedule cleanup (requires pg_cron extension)
-- SELECT cron.schedule('cleanup-sessions', '0 */6 * * *', 'SELECT delete_expired_sessions()');
```

---

## TimescaleDB - Market Data (Time-Series)

### Market Data Hypertable

```sql
-- Enable TimescaleDB extension
CREATE EXTENSION IF NOT EXISTS timescaledb;

-- Create market data table
CREATE TABLE market_data (
    time TIMESTAMPTZ NOT NULL,
    symbol TEXT NOT NULL,
    interval TEXT NOT NULL,  -- '1m', '5m', '15m', '1h', '1d'
    
    -- OHLCV data
    open DOUBLE PRECISION NOT NULL,
    high DOUBLE PRECISION NOT NULL,
    low DOUBLE PRECISION NOT NULL,
    close DOUBLE PRECISION NOT NULL,
    volume BIGINT NOT NULL,
    
    -- Additional fields
    trades_count INTEGER,
    vwap DOUBLE PRECISION,  -- Volume-weighted average price
    
    -- Constraints
    CONSTRAINT chk_interval CHECK (interval IN ('1m', '5m', '15m', '1h', '1d'))
);

-- Convert to hypertable (partitioned by time)
SELECT create_hypertable('market_data', 'time', chunk_time_interval => INTERVAL '1 day');

-- Create indexes
CREATE INDEX idx_market_data_symbol_interval_time 
    ON market_data (symbol, interval, time DESC);

CREATE INDEX idx_market_data_time 
    ON market_data (time DESC);

-- Compression policy (compress data older than 7 days)
ALTER TABLE market_data SET (
    timescaledb.compress,
    timescaledb.compress_segmentby = 'symbol, interval',
    timescaledb.compress_orderby = 'time DESC'
);

SELECT add_compression_policy('market_data', INTERVAL '7 days');

-- Retention policy (delete data older than 2 years)
SELECT add_retention_policy('market_data', INTERVAL '2 years');

-- Continuous aggregates for faster queries
CREATE MATERIALIZED VIEW market_data_1h
WITH (timescaledb.continuous) AS
SELECT
    time_bucket('1 hour', time) AS bucket,
    symbol,
    interval,
    first(open, time) AS open,
    max(high) AS high,
    min(low) AS low,
    last(close, time) AS close,
    sum(volume) AS volume
FROM market_data
WHERE interval = '1m'
GROUP BY bucket, symbol, interval;

-- Refresh policy (refresh every 10 minutes)
SELECT add_continuous_aggregate_policy('market_data_1h',
    start_offset => INTERVAL '2 hours',
    end_offset => INTERVAL '10 minutes',
    schedule_interval => INTERVAL '10 minutes'
);
```

### Market Ticks (Real-time, short retention)

```sql
CREATE TABLE market_ticks (
    time TIMESTAMPTZ NOT NULL,
    symbol TEXT NOT NULL,
    price DOUBLE PRECISION NOT NULL,
    volume INTEGER NOT NULL,
    bid DOUBLE PRECISION,
    ask DOUBLE PRECISION,
    bid_quantity INTEGER,
    ask_quantity INTEGER
);

SELECT create_hypertable('market_ticks', 'time', chunk_time_interval => INTERVAL '1 hour');

CREATE INDEX idx_market_ticks_symbol_time ON market_ticks (symbol, time DESC);

-- Retention: Keep only 24 hours of tick data
SELECT add_retention_policy('market_ticks', INTERVAL '24 hours');
```

---

## ClickHouse - Analytics (OLAP)

### Backtest Results Table

```sql
CREATE TABLE backtest_results (
    backtest_id UUID DEFAULT generateUUIDv4(),
    user_id String,
    strategy_name String,
    symbol String,
    interval String,
    
    -- Parameters
    parameters String,  -- JSON serialized
    
    -- Date range
    start_date Date,
    end_date Date,
    
    -- Capital
    initial_capital Float64,
    final_capital Float64,
    
    -- Performance metrics
    total_return Float64,
    cagr Float64,
    sharpe_ratio Float64,
    sortino_ratio Float64,
    max_drawdown Float64,
    max_drawdown_duration Int32,
    calmar_ratio Float64,
    
    -- Trade statistics
    total_trades Int32,
    winning_trades Int32,
    losing_trades Int32,
    win_rate Float64,
    profit_factor Float64,
    avg_win Float64,
    avg_loss Float64,
    largest_win Float64,
    largest_loss Float64,
    
    -- Execution metadata
    execution_time_ms Int32,
    created_at DateTime DEFAULT now(),
    
    -- Indexes
    INDEX idx_user_id (user_id) TYPE bloom_filter GRANULARITY 1,
    INDEX idx_strategy (strategy_name) TYPE bloom_filter GRANULARITY 1,
    INDEX idx_symbol (symbol) TYPE bloom_filter GRANULARITY 1
    
) ENGINE = MergeTree()
ORDER BY (strategy_name, symbol, created_at)
PARTITION BY toYYYYMM(created_at)
TTL created_at + INTERVAL 2 YEAR  -- Auto-delete after 2 years
SETTINGS index_granularity = 8192;
```

### Backtest Trades Table

```sql
CREATE TABLE backtest_trades (
    backtest_id UUID,
    trade_number Int32,
    
    -- Trade details
    entry_time DateTime,
    exit_time DateTime,
    entry_price Float64,
    exit_price Float64,
    quantity Int32,
    direction Enum8('LONG' = 1, 'SHORT' = 2),
    
    -- P&L
    pnl Float64,
    pnl_pct Float64,
    cumulative_pnl Float64,
    
    -- Trade metadata
    entry_signal_confidence Float64,
    exit_reason String,  -- 'signal', 'stop_loss', 'take_profit', 'timeout'
    metadata String  -- JSON with indicator values at entry/exit
    
) ENGINE = MergeTree()
ORDER BY (backtest_id, trade_number)
PARTITION BY toYYYYMM(entry_time)
TTL entry_time + INTERVAL 2 YEAR
SETTINGS index_granularity = 8192;
```

### Strategy Signals (Historical)

```sql
CREATE TABLE strategy_signals (
    signal_id UUID DEFAULT generateUUIDv4(),
    symbol String,
    interval String,
    strategy_name String,
    
    -- Signal details
    timestamp DateTime,
    action Enum8('BUY' = 1, 'SELL' = 2, 'HOLD' = 3),
    confidence Float64,
    price Float64,
    
    -- Metadata
    indicators String,  -- JSON with indicator values
    
    -- Outcome (for backtesting signal accuracy)
    future_return_1h Float64,
    future_return_4h Float64,
    future_return_1d Float64,
    was_profitable Bool,
    
    created_at DateTime DEFAULT now(),
    
    INDEX idx_symbol (symbol) TYPE bloom_filter GRANULARITY 1,
    INDEX idx_strategy (strategy_name) TYPE bloom_filter GRANULARITY 1
    
) ENGINE = MergeTree()
ORDER BY (symbol, strategy_name, timestamp)
PARTITION BY toYYYYMM(timestamp)
TTL timestamp + INTERVAL 1 YEAR
SETTINGS index_granularity = 8192;
```

### ML Predictions Table

```sql
CREATE TABLE ml_predictions (
    prediction_id UUID DEFAULT generateUUIDv4(),
    symbol String,
    interval String,
    model_name String,
    model_version String,
    
    -- Prediction
    timestamp DateTime,
    prediction_direction Enum8('UP' = 1, 'DOWN' = 2),
    confidence Float64,
    predicted_return Float64,
    
    -- Features used (for debugging)
    features String,  -- JSON
    
    -- Actual outcome (populated later)
    actual_direction Enum8('UP' = 1, 'DOWN' = 2, 'UNKNOWN' = 0) DEFAULT 'UNKNOWN',
    actual_return Float64 DEFAULT 0,
    prediction_accuracy Bool,
    
    created_at DateTime DEFAULT now(),
    
    INDEX idx_symbol (symbol) TYPE bloom_filter GRANULARITY 1,
    INDEX idx_model (model_name) TYPE bloom_filter GRANULARITY 1
    
) ENGINE = MergeTree()
ORDER BY (symbol, model_name, timestamp)
PARTITION BY toYYYYMM(timestamp)
TTL timestamp + INTERVAL 6 MONTH
SETTINGS index_granularity = 8192;
```

### User Activity Events

```sql
CREATE TABLE user_activity_events (
    event_id UUID DEFAULT generateUUIDv4(),
    user_id String,
    event_type String,  -- 'login', 'backtest_run', 'watchlist_created', 'alert_triggered'
    
    -- Event details
    timestamp DateTime,
    ip_address IPv4,
    user_agent String,
    
    -- Event metadata
    metadata String,  -- JSON
    
    -- Session info
    session_id String,
    
    created_at DateTime DEFAULT now(),
    
    INDEX idx_user_id (user_id) TYPE bloom_filter GRANULARITY 1,
    INDEX idx_event_type (event_type) TYPE bloom_filter GRANULARITY 1
    
) ENGINE = MergeTree()
ORDER BY (user_id, timestamp)
PARTITION BY toYYYYMM(timestamp)
TTL timestamp + INTERVAL 1 YEAR
SETTINGS index_granularity = 8192;
```

---

## Dragonfly (Redis) - Cache Structure

### Key Patterns

```
# Market Data Cache
tick:{symbol}:latest                    # Latest tick (TTL: 60s)
candles:{symbol}:{interval}             # List of recent candles (LPUSH, keep 500)
ohlcv:{symbol}:{interval}:{date}        # Daily candle data (Hash, TTL: 7 days)

# Strategy Signals
signals:{symbol}:{interval}             # Recent signals (List, keep 100)
signal:{signal_id}                      # Signal details (Hash, TTL: 24 hours)

# ML Predictions
prediction:{symbol}:latest              # Latest ML prediction (Hash, TTL: 1 hour)
predictions:{symbol}                    # Recent predictions (List, keep 50)

# User Sessions
session:{session_id}                    # Session data (Hash, TTL: 24 hours)
user:{user_id}:active_session           # Active session mapping (String, TTL: 24 hours)

# Rate Limiting
ratelimit:user:{user_id}:{window}       # User rate limit counter (String, TTL: dynamic)
ratelimit:ip:{ip_address}:{window}      # IP rate limit counter (String, TTL: dynamic)

# Feature Store (Hot Features)
features:{symbol}:latest                # Latest features for ML (Hash, TTL: 5 minutes)

# WebSocket Subscriptions
ws:subscriptions:{symbol}               # Set of socket IDs subscribed to symbol
ws:user:{user_id}:sockets               # Set of socket IDs for user

# Leaderboard
leaderboard:strategies:{interval}       # Sorted set of strategies by performance
leaderboard:users:{interval}            # Sorted set of users by total returns
```

### Example Operations

```python
import redis
from datetime import datetime, timedelta

# Initialize Dragonfly client
cache = redis.Redis(host='dragonfly.svc.cluster.local', port=6379, decode_responses=True)

# === Market Data Operations ===

# Store latest tick
def cache_latest_tick(symbol: str, tick: dict):
    """Cache the latest tick with 60s TTL"""
    cache.setex(
        f"tick:{symbol}:latest",
        60,
        json.dumps(tick)
    )

# Store candle
def cache_candle(symbol: str, interval: str, candle: dict):
    """Add candle to list, keep only 500 recent"""
    key = f"candles:{symbol}:{interval}"
    cache.lpush(key, json.dumps(candle))
    cache.ltrim(key, 0, 499)

# Get recent candles
def get_recent_candles(symbol: str, interval: str, count: int = 100):
    """Retrieve recent candles"""
    key = f"candles:{symbol}:{interval}"
    candles_json = cache.lrange(key, 0, count - 1)
    return [json.loads(c) for c in candles_json]

# === Rate Limiting ===

def check_rate_limit(user_id: str, limit: int = 60, window: int = 60) -> bool:
    """
    Check if user is within rate limit
    
    Args:
        user_id: User identifier
        limit: Max requests allowed
        window: Time window in seconds
    
    Returns:
        True if allowed, False if rate limit exceeded
    """
    key = f"ratelimit:user:{user_id}:{window}"
    
    # Increment counter
    current = cache.incr(key)
    
    # Set expiry on first request
    if current == 1:
        cache.expire(key, window)
    
    return current <= limit

# === WebSocket Subscriptions ===

def subscribe_to_symbol(socket_id: str, symbol: str):
    """Register socket subscription to symbol"""
    cache.sadd(f"ws:subscriptions:{symbol}", socket_id)

def unsubscribe_from_symbol(socket_id: str, symbol: str):
    """Remove socket subscription"""
    cache.srem(f"ws:subscriptions:{symbol}", socket_id)

def get_symbol_subscribers(symbol: str) -> set:
    """Get all socket IDs subscribed to symbol"""
    return cache.smembers(f"ws:subscriptions:{symbol}")

# === Leaderboard ===

def update_strategy_leaderboard(strategy_name: str, sharpe_ratio: float):
    """Update strategy leaderboard with new performance"""
    cache.zadd(
        "leaderboard:strategies:all_time",
        {strategy_name: sharpe_ratio}
    )

def get_top_strategies(limit: int = 10) -> list:
    """Get top performing strategies"""
    return cache.zrevrange(
        "leaderboard:strategies:all_time",
        0,
        limit - 1,
        withscores=True
    )
```

---

## Feature Store (Feast/Tecton)

### Feature Definitions

```python
# features/feature_definitions.py
from feast import Entity, Feature, FeatureView, Field, ValueType
from feast.types import Float32, Int32, String
from datetime import timedelta

# Define entities
symbol_entity = Entity(
    name="symbol",
    value_type=ValueType.STRING,
    description="Stock symbol identifier"
)

# Technical Indicators Feature View
technical_indicators = FeatureView(
    name="technical_indicators",
    entities=["symbol"],
    ttl=timedelta(days=1),
    schema=[
        Field(name="rsi_14", dtype=Float32),
        Field(name="rsi_7", dtype=Float32),
        Field(name="macd", dtype=Float32),
        Field(name="macd_signal", dtype=Float32),
        Field(name="macd_hist", dtype=Float32),
        Field(name="atr_14", dtype=Float32),
        Field(name="atr_pct", dtype=Float32),
        Field(name="adx_14", dtype=Float32),
        Field(name="bb_width", dtype=Float32),
        Field(name="bb_position", dtype=Float32),
    ],
    online=True,
    source="timescaledb_source",  # Connect to TimescaleDB
    tags={"category": "technical_analysis"}
)

# Volume Features
volume_features = FeatureView(
    name="volume_features",
    entities=["symbol"],
    ttl=timedelta(days=1),
    schema=[
        Field(name="volume_ratio", dtype=Float32),
        Field(name="obv", dtype=Float32),
        Field(name="obv_ema", dtype=Float32),
        Field(name="adosc", dtype=Float32),
    ],
    online=True,
    source="timescaledb_source",
    tags={"category": "volume_analysis"}
)

# Regime Features
regime_features = FeatureView(
    name="regime_features",
    entities=["symbol"],
    ttl=timedelta(hours=1),
    schema=[
        Field(name="volatility_20", dtype=Float32),
        Field(name="volatility_regime", dtype=String),  # 'low', 'medium', 'high'
        Field(name="trend_strength", dtype=String),  # 'weak', 'moderate', 'strong'
        Field(name="market_regime", dtype=Int32),  # 0=ranging, 1=trending
    ],
    online=True,
    source="timescaledb_source",
    tags={"category": "regime_detection"}
)
```

---

## Data Migration Strategy

### Phase 1: Cold Start (Initial Data Load)

```python
# scripts/cold_start_data_load.py
"""
Load historical data from Upstox API into TimescaleDB
Run once during initial setup
"""

import asyncio
from datetime import datetime, timedelta

async def cold_start_data_load():
    symbols = get_nifty_50_symbols()
    intervals = ['1m', '5m', '15m', '1h', '1d']
    
    end_date = datetime.now()
    start_date = end_date - timedelta(days=730)  # 2 years
    
    for symbol in symbols:
        for interval in intervals:
            print(f"Loading {symbol} {interval} data...")
            
            # Fetch from Upstox
            candles = await upstox_client.get_historical_candles(
                symbol=symbol,
                interval=interval,
                from_date=start_date,
                to_date=end_date
            )
            
            # Batch insert to TimescaleDB
            await db.batch_insert_candles(candles)
            
            print(f"Loaded {len(candles)} candles for {symbol} {interval}")
    
    print("Cold start complete!")

if __name__ == "__main__":
    asyncio.run(cold_start_data_load())
```

### Phase 2: Continuous Sync

```python
# Airflow DAG for daily sync
from airflow import DAG
from airflow.operators.python import PythonOperator
from datetime import datetime, timedelta

def sync_missing_candles():
    """Sync any missing candles from previous day"""
    symbols = get_all_symbols()
    intervals = ['1m', '5m', '15m', '1h', '1d']
    
    yesterday = datetime.now().date() - timedelta(days=1)
    
    for symbol in symbols:
        for interval in intervals:
            # Check for gaps
            gaps = db.find_data_gaps(symbol, interval, yesterday)
            
            for gap_start, gap_end in gaps:
                # Fetch missing data
                candles = upstox_client.get_historical_candles(
                    symbol, interval, gap_start, gap_end
                )
                db.insert_candles(candles)

with DAG(
    dag_id='daily_data_sync',
    schedule_interval='0 23 * * *',  # 11 PM daily
    start_date=datetime(2024, 1, 1),
    catchup=False
) as dag:
    
    sync_task = PythonOperator(
        task_id='sync_missing_candles',
        python_callable=sync_missing_candles
    )
```

---

This comprehensive database schema documentation provides the foundation for all data storage needs in the platform, from real-time market data to historical analytics and ML feature storage.
