# Stock Analysis & Trading Intelligence Platform - Production Architecture

> **CTO-Level System Design Document**  
> A battle-tested, institutional-grade fintech platform designed to scale to millions of users while maintaining sub-millisecond latency for market data and ML inference.

## Table of Contents

1. [Executive Summary](#executive-summary)
2. [High-Level Architecture](#high-level-architecture)
3. [Technology Stack](#technology-stack)
4. [Microservices Breakdown](#microservices-breakdown)
5. [Infrastructure Design](#infrastructure-design)
6. [Data Storage Design](#data-storage-design)
7. [Real-Time Data Flow](#real-time-data-flow)
8. [Machine Learning Architecture](#machine-learning-architecture)
9. [Security & Compliance](#security--compliance)
10. [Scalability & Performance](#scalability--performance)
11. [Monitoring & Observability](#monitoring--observability)
12. [Deployment Strategy](#deployment-strategy)
13. [Cost Optimization](#cost-optimization)
14. [Risk Analysis & Mitigation](#risk-analysis--mitigation)

---

## Executive Summary

This document outlines the production architecture for a comprehensive stock analysis and trading intelligence platform. The system is designed to:

- **Handle high-throughput market data**: Process 100k+ market ticks per second during trading hours
- **Provide real-time analysis**: Sub-100ms ML inference and signal generation
- **Scale horizontally**: Support millions of concurrent users with WebSocket connections
- **Ensure reliability**: 99.95% uptime SLA with multi-region failover
- **Enable rapid iteration**: GitOps-based CI/CD with blue-green deployments

### Key Capabilities

| Capability | Target Metric | Implementation |
|-----------|---------------|----------------|
| Market Data Latency | < 50ms | WebSocket → Dragonfly cache → In-memory buffers |
| ML Inference | < 100ms | ONNX Runtime with optimized models |
| Concurrent WebSocket | 100k+ | Node.js Fastify with sticky sessions |
| Backtest Performance | < 30s for 2yr data | ClickHouse columnar analytics |
| System Uptime | 99.95% | Multi-AZ deployment, auto-scaling |

---

## High-Level Architecture

### System Diagram

```
┌─────────────────────────────────────────────────────────────┐
│                     FRONTEND LAYER                          │
│  React 18 + TypeScript + TradingView Charts                │
│  (Web Dashboard, Real-time Charts, Backtest Reports)        │
└──────────────────────┬──────────────────────────────────────┘
                       │ HTTPS/WSS (CloudFront + WAF)
┌──────────────────────▼──────────────────────────────────────┐
│                    API GATEWAY LAYER                        │
│  Kong/Nginx + AWS ALB                                       │
│  - Rate Limiting (per-user, per-IP)                        │
│  - JWT Authentication                                       │
│  - Request/Response transformation                          │
│  - Circuit Breaker                                          │
└─────────┬──────────┬──────────┬──────────────┬──────────────┘
          │          │          │              │
    ┌─────▼──┐  ┌───▼─────┐  ┌─▼───────┐  ┌──▼─────────┐
    │ Auth   │  │ Market  │  │ Strategy │  │  Backtest  │
    │Service │  │Data Svc │  │ Engine   │  │   Service  │
    │        │  │         │  │          │  │            │
    │FastAPI │  │FastAPI  │  │FastAPI   │  │  FastAPI   │
    └────┬───┘  └────┬────┘  └────┬─────┘  └─────┬──────┘
         │           │            │              │
         │      ┌────▼────────────▼──────────────▼───┐
         │      │                                     │
    ┌────▼──────▼────┐   WebSocket Gateway           │
    │                │   (Node.js Fastify)           │
    │  Data Bus      │   - 100k concurrent conns     │
    │  Apache Kafka  │   - Real-time broadcasting    │
    │                │                                │
    │  Topics:       │◄──────────────────────────────┘
    │  - market-ticks│
    │  - signals     │
    │  - ml-preds    │
    │  - alerts      │
    └────┬───────────┘
         │
    ┌────▼──────────────────────────────────────────────┐
    │                                                    │
    ▼                                                    ▼
┌───────────────────┐                        ┌──────────────────┐
│  Real-Time Layer  │                        │  Batch Layer     │
│                   │                        │                  │
│  Dragonfly Cache  │                        │  Airflow (ETL)   │
│  - Market data    │                        │  - Daily ingestion
│  - Session store  │                        │  - Feature eng.  │
│  - Feature cache  │                        │  - Model training│
│  (sub-ms access)  │                        │                  │
└─────────┬─────────┘                        └────────┬─────────┘
          │                                           │
          ▼                                           ▼
┌─────────────────────┐                    ┌────────────────────┐
│  TimescaleDB        │                    │  ClickHouse (OLAP) │
│  (Time-Series)      │                    │  - Backtest results│
│  - OHLCV historical │                    │  - Analytics       │
│  - Optimized for    │                    │  - Reporting       │
│    fintech queries  │                    │                    │
└─────────────────────┘                    └────────────────────┘
          │                                           │
          └──────────────┬────────────────────────────┘
                         │
                         ▼
              ┌──────────────────────┐
              │  PostgreSQL (OLTP)   │
              │  - User accounts     │
              │  - Watchlists        │
              │  - Alerts & settings │
              │  (RDS Multi-AZ)      │
              └──────────────────────┘
                         │
                         │
              ┌──────────▼───────────┐
              │  S3 Object Storage   │
              │  - Model artifacts   │
              │  - Backtest reports  │
              │  - Training datasets │
              │  - Archive (5yr+)    │
              └──────────────────────┘
                         │
              ┌──────────▼───────────────────────────┐
              │  ML Training & Inference Platform    │
              │  - MLflow (experiment tracking)      │
              │  - Ray Tune (hyperparameter tuning)  │
              │  - ONNX Runtime (inference)          │
              │  - Feast/Tecton (feature store)      │
              └──────────┬───────────────────────────┘
                         │
              ┌──────────▼───────────────────────────┐
              │  Upstox API Integration Layer        │
              │  - WebSocket (live market hours)     │
              │  - REST API (EOD/historical)         │
              │  - Auto-switching (9:15-15:30)       │
              │  - Circuit breaker & retry logic     │
              └──────────────────────────────────────┘
```

### Architecture Principles

1. **Event-Driven Architecture**: Kafka as the central nervous system for all events
2. **Microservices**: Independent services with clear boundaries and responsibilities
3. **Polyglot Persistence**: Right database for each use case (OLTP, OLAP, Time-Series, Cache)
4. **Horizontal Scalability**: All services stateless, scale with K8s auto-scaling
5. **Observability-First**: Comprehensive monitoring, logging, and tracing
6. **Security in Depth**: Multiple layers (WAF, API Gateway, service-level auth, encryption)

---

## Technology Stack

### Frontend Technologies

| Component | Technology | Justification |
|-----------|-----------|---------------|
| Framework | **React 18 + TypeScript** | Industry standard, type safety, excellent ecosystem |
| Charts | **TradingView Lightweight Charts** | Purpose-built for financial data, 60fps performance |
| State Management | **Redux Toolkit + RTK Query** | Predictable state, built-in caching, DevTools |
| Real-time | **Socket.io Client** | Automatic reconnection, fallback transports |
| Styling | **Tailwind CSS + shadcn/ui** | Utility-first, consistent design system |
| Build Tool | **Vite** | 10x faster than webpack, HMR, ESM-native |
| Testing | **Vitest + React Testing Library** | Fast unit tests, user-centric integration tests |
| Monitoring | **Sentry + LogRocket** | Error tracking, session replay |

### Backend Technologies

#### Core Services

| Service | Technology | Rationale |
|---------|-----------|-----------|
| API Framework | **FastAPI (Python 3.11+)** | Async support, automatic OpenAPI docs, Pydantic validation |
| WebSocket Gateway | **Node.js + Fastify** | Event loop handles 100k+ concurrent connections efficiently |
| Inter-Service Comm | **gRPC** | Low-latency binary protocol, type-safe contracts |
| Validation | **Pydantic v2** | 17x faster than v1, JSON Schema generation |
| Async Tasks | **Celery + Redis** | Distributed task queue for background jobs |

#### Data Layer

| Type | Technology | Use Case |
|------|-----------|----------|
| **OLTP** | PostgreSQL 15 (RDS Multi-AZ) | User data, watchlists, alerts, settings |
| **Time-Series** | TimescaleDB | OHLCV data with hypertables, 20x compression |
| **OLAP** | ClickHouse | Backtest analytics, reporting, columnar storage |
| **Cache** | Dragonfly | Redis-compatible, 6x faster, multi-threaded |
| **Feature Store** | Tecton / Feast | Versioned ML features, point-in-time correctness |
| **Object Storage** | AWS S3 | Model artifacts, backtest reports, archives |

#### Streaming & Events

```
Apache Kafka (MSK)
├── market-ticks (100k msg/sec during market hours)
│   ├── Partitions: 20 (by symbol hash)
│   ├── Retention: 24 hours (in-memory)
│   └── Consumers: Strategy Engine, ML Service
│
├── strategy-signals (1k msg/sec)
│   ├── Partitions: 10
│   ├── Retention: 7 days
│   └── Consumers: WebSocket Gateway, Alert Service
│
├── ml-predictions (5k msg/sec)
│   ├── Partitions: 10
│   ├── Retention: 3 days
│   └── Consumers: Strategy Engine, Dashboard
│
└── user-alerts (fan-out)
    ├── Partitions: 5
    ├── Retention: 1 day
    └── Consumers: WebSocket Gateway, Email Service
```

#### Machine Learning Stack

| Component | Technology | Purpose |
|-----------|-----------|---------|
| Training | **PyTorch 2.0** | Deep learning models (LSTMs, Transformers) |
| Classical ML | **scikit-learn + XGBoost** | Gradient boosting for tabular data |
| Feature Eng | **Pandas + Polars** | Polars 5-10x faster for large datasets |
| Orchestration | **Apache Airflow** | DAG scheduling for ETL, training pipelines |
| Experiment Tracking | **MLflow** | Model versioning, metrics, artifact storage |
| Hyperparameter Tuning | **Optuna + Ray Tune** | Distributed hyperparameter search |
| Model Serving | **ONNX Runtime** | Cross-platform, GPU/CPU inference |
| Model Monitoring | **Evidently AI** | Data drift detection, model performance |

#### Infrastructure & DevOps

```yaml
Cloud Provider: AWS
  Compute:
    - EKS (Kubernetes 1.28): Container orchestration
    - EC2 (Auto Scaling Groups): Node groups
    - Lambda: Scheduled jobs, event-driven tasks
  
  Networking:
    - VPC: Isolated network per environment
    - ALB: Application Load Balancer (WebSocket support)
    - Route53: DNS management
    - CloudFront: CDN for static assets
  
  Storage:
    - RDS Multi-AZ: PostgreSQL (OLTP)
    - MemoryDB: Dragonfly cluster
    - EBS: Persistent volumes for Kafka, ClickHouse
    - S3: Object storage (Glacier for archives)
  
  Security:
    - IAM: Fine-grained access control
    - Secrets Manager: Encrypted secrets
    - AWS Shield + WAF: DDoS protection
    - KMS: Encryption keys
  
  Monitoring:
    - CloudWatch: Metrics, logs, alarms
    - X-Ray: Distributed tracing
    - DataDog: APM, infrastructure monitoring
  
  CI/CD:
    - ECR: Container registry
    - CodePipeline: CI/CD orchestration
    - ArgoCD: GitOps deployment to EKS
```

---

## Microservices Breakdown

### Service A: Market Data Ingestion Service

**Technology**: Python FastAPI  
**Responsibility**: Ingest live and historical data from Upstox API

#### Architecture

```python
# market_data_service/
├── main.py                 # FastAPI application
├── ws_manager.py           # WebSocket connection manager
├── rest_fallback.py        # REST API fallback (after hours)
├── candle_aggregator.py    # Tick-to-candle aggregation
├── cache_writer.py         # Dragonfly cache writer
├── kafka_producer.py       # Kafka event publisher
├── circuit_breaker.py      # Circuit breaker pattern
├── health_check.py         # Health & readiness probes
└── config.py               # Configuration management
```

#### Key Features

1. **Dual Mode Operation**:
   - **Market Hours (9:15 AM - 3:30 PM IST)**: WebSocket streaming
   - **After Hours**: REST API polling every 5 minutes

2. **Connection Management**:
```python
class UpstoxWebSocketManager:
    def __init__(self, symbols: List[str], api_key: str):
        self.symbols = symbols
        self.connection_pool = []
        self.circuit_breaker = CircuitBreaker(
            failure_threshold=5,
            recovery_timeout=60
        )
    
    async def connect(self):
        """Establish WebSocket connections (max 50 symbols per connection)"""
        for chunk in chunked(self.symbols, 50):
            ws = await self._create_connection(chunk)
            self.connection_pool.append(ws)
    
    async def on_tick(self, tick: MarketTick):
        """Handle incoming tick data"""
        # 1. Write to Dragonfly cache (hot data)
        await cache.setex(f"tick:{tick.symbol}:latest", 60, tick.json())
        
        # 2. Publish to Kafka
        await kafka_producer.send("market-ticks", tick.dict())
        
        # 3. Aggregate into candles (in-memory buffer)
        self.candle_aggregator.add_tick(tick)
```

3. **Candle Aggregation** (1m, 5m, 15m, 1h, 1d):
```python
class CandleAggregator:
    def __init__(self):
        self.buffers = {
            "1m": defaultdict(list),
            "5m": defaultdict(list),
            "15m": defaultdict(list),
            "1h": defaultdict(list),
            "1d": defaultdict(list)
        }
    
    def add_tick(self, tick: MarketTick):
        """Add tick to all interval buffers"""
        for interval in self.buffers:
            self.buffers[interval][tick.symbol].append(tick)
    
    async def flush_candles(self):
        """Flush completed candles to TimescaleDB + Dragonfly"""
        for interval, symbol_ticks in self.buffers.items():
            for symbol, ticks in symbol_ticks.items():
                if self._is_candle_complete(ticks, interval):
                    candle = self._create_candle(ticks)
                    
                    # Write to TimescaleDB (persistent)
                    await db.insert_candle(candle)
                    
                    # Write to Dragonfly (hot cache)
                    await cache.lpush(f"candles:{symbol}:{interval}", candle.json())
                    await cache.ltrim(f"candles:{symbol}:{interval}", 0, 500)
                    
                    # Clear buffer
                    self.buffers[interval][symbol] = []
```

4. **Circuit Breaker for Upstox API**:
```python
@circuit_breaker(
    failure_threshold=5,
    recovery_timeout=60,
    expected_exception=UpstoxAPIException
)
async def fetch_candles_rest(symbol: str, interval: str, from_date: datetime):
    """Fallback REST API call with circuit breaker"""
    response = await upstox_client.get_candles(symbol, interval, from_date)
    return response.data
```

#### Scaling Strategy

- **Horizontal**: 3-5 pods in production
- **Partitioning**: Each pod handles subset of symbols (sharded by hash)
- **Resource Limits**:
  ```yaml
  resources:
    requests:
      cpu: 2
      memory: 4Gi
    limits:
      cpu: 4
      memory: 8Gi
  ```

---

### Service B: Real-Time WebSocket Gateway

**Technology**: Node.js + Fastify + Socket.io  
**Responsibility**: Manage WebSocket connections to frontend clients

#### Architecture

```javascript
// websocket_gateway/
├── server.js               // Fastify server
├── connection_manager.js   // Connection pooling & heartbeat
├── message_router.js       // Route messages to subscribed clients
├── subscription_handler.js // Manage user subscriptions
├── rate_limiter.js         // Per-user rate limiting
├── auth_middleware.js      // JWT validation
└── redis_adapter.js        // Redis adapter for horizontal scaling
```

#### Key Implementation

```javascript
// server.js
const fastify = require('fastify')({ logger: true });
const socketio = require('fastify-socket.io');
const { Kafka } = require('kafkajs');
const Redis = require('ioredis');

// Register Socket.io with Redis adapter (for horizontal scaling)
fastify.register(socketio, {
  cors: { origin: process.env.FRONTEND_URL },
  adapter: require('socket.io-redis')({
    pubClient: new Redis(process.env.REDIS_URL),
    subClient: new Redis(process.env.REDIS_URL)
  })
});

// Kafka consumer for market data
const kafka = new Kafka({ brokers: [process.env.KAFKA_BROKER] });
const consumer = kafka.consumer({ groupId: 'websocket-gateway' });

fastify.ready(async () => {
  const io = fastify.io;
  
  // Handle client connections
  io.on('connection', async (socket) => {
    console.log(`Client connected: ${socket.id}`);
    
    // Verify JWT token
    const token = socket.handshake.auth.token;
    const user = await verifyJWT(token);
    if (!user) {
      socket.disconnect();
      return;
    }
    
    socket.user = user;
    
    // Handle subscriptions
    socket.on('subscribe', async (symbols) => {
      // Rate limiting: max 50 symbols per user
      if (symbols.length > 50) {
        socket.emit('error', { message: 'Max 50 symbols allowed' });
        return;
      }
      
      // Join rooms for each symbol
      symbols.forEach(symbol => {
        socket.join(`symbol:${symbol}`);
      });
      
      // Send latest cached data immediately
      for (const symbol of symbols) {
        const latestTick = await redis.get(`tick:${symbol}:latest`);
        if (latestTick) {
          socket.emit('tick', JSON.parse(latestTick));
        }
      }
    });
    
    socket.on('unsubscribe', (symbols) => {
      symbols.forEach(symbol => {
        socket.leave(`symbol:${symbol}`);
      });
    });
    
    socket.on('disconnect', () => {
      console.log(`Client disconnected: ${socket.id}`);
    });
  });
  
  // Consume Kafka events and broadcast to clients
  await consumer.connect();
  await consumer.subscribe({ topics: ['market-ticks', 'strategy-signals', 'ml-predictions'] });
  
  await consumer.run({
    eachMessage: async ({ topic, partition, message }) => {
      const data = JSON.parse(message.value.toString());
      
      switch(topic) {
        case 'market-ticks':
          // Broadcast tick to subscribers of this symbol
          io.to(`symbol:${data.symbol}`).emit('tick', data);
          break;
        
        case 'strategy-signals':
          // Broadcast signal to all connected users
          io.emit('signal', data);
          break;
        
        case 'ml-predictions':
          // Broadcast ML prediction
          io.to(`symbol:${data.symbol}`).emit('prediction', data);
          break;
      }
    }
  });
});

fastify.listen({ port: 3000, host: '0.0.0.0' });
```

#### Horizontal Scaling with Redis Adapter

```
         ┌─────────────┐
         │   Client 1  │
         └──────┬──────┘
                │
         ┌──────▼──────────┐
         │  ALB (Sticky)   │
         └──────┬──────────┘
                │
    ┌───────────┴───────────┐
    │                       │
┌───▼────┐            ┌─────▼───┐
│ WS Pod │            │ WS Pod  │
│   1    │            │    2    │
└───┬────┘            └────┬────┘
    │                      │
    └──────┬───────────────┘
           │
      ┌────▼─────┐
      │  Redis   │  (Pub/Sub for message routing)
      └──────────┘
```

#### Resource Allocation

- **CPU**: 2 cores per pod (single-threaded event loop)
- **Memory**: 2 GB per pod
- **Connections**: ~10k per pod (horizontal scaling for more)
- **Auto-scaling**: Scale up when avg connections > 8k

---

### Service C: Strategy Engine

**Technology**: Python FastAPI  
**Responsibility**: Real-time technical analysis & signal generation

#### Architecture

```python
# strategy_engine/
├── main.py                 # FastAPI application
├── strategies/
│   ├── base.py             # Abstract base class
│   ├── bollinger_bands.py  # Bollinger Bands strategy
│   ├── donchian_channel.py # Donchian Channel strategy
│   ├── head_shoulders.py   # Head & Shoulders pattern
│   ├── rsi_divergence.py   # RSI divergence strategy
│   └── custom/             # User-defined strategies (Enterprise)
├── indicators/
│   ├── technical.py        # TA-Lib wrapper
│   ├── custom.py           # Custom indicators
│   └── pattern_detection.py # Chart pattern detection
├── signal_generator.py     # Signal generation logic
├── candle_buffer.py        # In-memory candle buffers
├── kafka_consumer.py       # Consume market-ticks topic
└── kafka_producer.py       # Produce strategy-signals topic
```

#### Strategy Plugin System

```python
# strategies/base.py
from abc import ABC, abstractmethod
from dataclasses import dataclass
import pandas as pd

@dataclass
class Signal:
    symbol: str
    timestamp: datetime
    action: str  # BUY, SELL, HOLD
    confidence: float  # 0.0 - 1.0
    price: float
    metadata: dict  # Additional info (indicators, patterns)

class StrategyBase(ABC):
    """Abstract base class for all strategies"""
    
    def __init__(self, params: dict):
        self.params = params
    
    @abstractmethod
    def calculate(self, ohlcv: pd.DataFrame) -> Optional[Signal]:
        """
        Implement strategy logic
        
        Args:
            ohlcv: DataFrame with columns [time, open, high, low, close, volume]
        
        Returns:
            Signal object or None
        """
        pass
    
    @abstractmethod
    def get_required_candles(self) -> int:
        """Return minimum number of candles needed"""
        pass
```

#### Example: Bollinger Bands Strategy

```python
# strategies/bollinger_bands.py
import talib
from typing import Optional

class BollingerBandsBreakout(StrategyBase):
    """
    Strategy: Buy when price breaks above upper band, sell when breaks below lower band
    """
    
    def __init__(self, params: dict = None):
        default_params = {
            'period': 20,
            'std_dev': 2.0,
            'volume_threshold': 1.5  # Volume must be 1.5x average
        }
        super().__init__({**default_params, **(params or {})})
    
    def get_required_candles(self) -> int:
        return self.params['period'] + 10  # Extra buffer
    
    def calculate(self, ohlcv: pd.DataFrame) -> Optional[Signal]:
        if len(ohlcv) < self.get_required_candles():
            return None
        
        # Calculate Bollinger Bands
        upper, middle, lower = talib.BBANDS(
            ohlcv['close'],
            timeperiod=self.params['period'],
            nbdevup=self.params['std_dev'],
            nbdevdn=self.params['std_dev']
        )
        
        # Calculate average volume
        avg_volume = ohlcv['volume'].rolling(window=20).mean()
        
        # Get latest values
        latest = ohlcv.iloc[-1]
        prev = ohlcv.iloc[-2]
        
        # Check volume spike
        volume_ratio = latest['volume'] / avg_volume.iloc[-1]
        if volume_ratio < self.params['volume_threshold']:
            return None  # No signal without volume confirmation
        
        # Breakout detection
        if prev['close'] <= upper.iloc[-2] and latest['close'] > upper.iloc[-1]:
            # Bullish breakout
            confidence = min(
                (latest['close'] - upper.iloc[-1]) / upper.iloc[-1],
                1.0
            )
            return Signal(
                symbol=latest['symbol'],
                timestamp=latest['time'],
                action='BUY',
                confidence=confidence,
                price=latest['close'],
                metadata={
                    'upper_band': upper.iloc[-1],
                    'middle_band': middle.iloc[-1],
                    'lower_band': lower.iloc[-1],
                    'volume_ratio': volume_ratio
                }
            )
        
        elif prev['close'] >= lower.iloc[-2] and latest['close'] < lower.iloc[-1]:
            # Bearish breakdown
            confidence = min(
                (lower.iloc[-1] - latest['close']) / lower.iloc[-1],
                1.0
            )
            return Signal(
                symbol=latest['symbol'],
                timestamp=latest['time'],
                action='SELL',
                confidence=confidence,
                price=latest['close'],
                metadata={
                    'upper_band': upper.iloc[-1],
                    'middle_band': middle.iloc[-1],
                    'lower_band': lower.iloc[-1],
                    'volume_ratio': volume_ratio
                }
            )
        
        return None  # No signal
```

#### Real-Time Signal Generation Flow

```python
# signal_generator.py
from typing import Dict, List
import asyncio

class SignalGenerator:
    def __init__(self, strategies: List[StrategyBase], intervals: List[str]):
        self.strategies = strategies
        self.intervals = intervals  # ['1m', '5m', '15m', '1h', '1d']
        self.candle_buffers = CandleBufferManager(intervals)
        self.kafka_consumer = KafkaConsumer(topic='market-ticks')
        self.kafka_producer = KafkaProducer(topic='strategy-signals')
    
    async def run(self):
        """Main event loop"""
        async for message in self.kafka_consumer:
            tick = MarketTick.parse_obj(message.value)
            
            # Update candle buffers
            completed_candles = self.candle_buffers.add_tick(tick)
            
            # Process completed candles
            for symbol, interval, candles_df in completed_candles:
                await self._process_candle(symbol, interval, candles_df)
    
    async def _process_candle(self, symbol: str, interval: str, ohlcv: pd.DataFrame):
        """Run all strategies on completed candle"""
        for strategy in self.strategies:
            try:
                signal = strategy.calculate(ohlcv)
                if signal:
                    # Publish signal to Kafka
                    await self.kafka_producer.send(signal.dict())
                    
                    # Cache signal in Dragonfly
                    await cache.lpush(
                        f"signals:{symbol}:{interval}",
                        signal.json()
                    )
                    
                    logger.info(f"Signal generated: {signal}")
            
            except Exception as e:
                logger.error(f"Strategy {strategy.__class__.__name__} failed: {e}")
```

#### Candle Buffer Management (In-Memory)

```python
# candle_buffer.py
from collections import defaultdict, deque

class CandleBufferManager:
    """Maintains in-memory circular buffers for each symbol/interval"""
    
    def __init__(self, intervals: List[str], buffer_size: int = 500):
        self.intervals = intervals
        self.buffer_size = buffer_size
        
        # Structure: buffers[symbol][interval] = deque of OHLCV candles
        self.buffers = defaultdict(lambda: {
            interval: deque(maxlen=buffer_size) for interval in intervals
        })
        
        # Track current candle being aggregated
        self.current_candle = defaultdict(dict)
    
    def add_tick(self, tick: MarketTick) -> List[Tuple[str, str, pd.DataFrame]]:
        """
        Add tick to buffers, return completed candles
        
        Returns:
            List of (symbol, interval, candles_df) for completed candles
        """
        completed = []
        
        for interval in self.intervals:
            candle_start = self._get_candle_start_time(tick.timestamp, interval)
            
            # Check if new candle has started
            if (tick.symbol, interval) not in self.current_candle or \
               self.current_candle[(tick.symbol, interval)]['start_time'] != candle_start:
                
                # Previous candle is complete
                if (tick.symbol, interval) in self.current_candle:
                    prev_candle = self.current_candle[(tick.symbol, interval)]
                    self.buffers[tick.symbol][interval].append(prev_candle)
                    
                    # Convert buffer to DataFrame
                    candles_df = pd.DataFrame(list(self.buffers[tick.symbol][interval]))
                    completed.append((tick.symbol, interval, candles_df))
                
                # Start new candle
                self.current_candle[(tick.symbol, interval)] = {
                    'symbol': tick.symbol,
                    'interval': interval,
                    'start_time': candle_start,
                    'open': tick.price,
                    'high': tick.price,
                    'low': tick.price,
                    'close': tick.price,
                    'volume': tick.volume
                }
            else:
                # Update current candle
                candle = self.current_candle[(tick.symbol, interval)]
                candle['high'] = max(candle['high'], tick.price)
                candle['low'] = min(candle['low'], tick.price)
                candle['close'] = tick.price
                candle['volume'] += tick.volume
        
        return completed
    
    def _get_candle_start_time(self, timestamp: datetime, interval: str) -> datetime:
        """Calculate candle start time based on interval"""
        interval_minutes = {
            '1m': 1, '5m': 5, '15m': 15, '1h': 60, '1d': 1440
        }[interval]
        
        minutes = (timestamp.hour * 60 + timestamp.minute) // interval_minutes * interval_minutes
        return timestamp.replace(hour=minutes // 60, minute=minutes % 60, second=0, microsecond=0)
```

---

### Service D: Backtesting Service

**Technology**: Python FastAPI + ClickHouse  
**Responsibility**: Historical testing & walk-forward optimization

#### Architecture

```python
# backtest_service/
├── main.py                 # FastAPI application
├── engine/
│   ├── backtest_engine.py  # Core backtesting logic
│   ├── optimizer.py        # Parameter optimization (Ray Tune + Optuna)
│   ├── walk_forward.py     # Walk-forward analysis
│   └── metrics.py          # Performance metrics calculation
├── data_loader.py          # Load historical data from TimescaleDB
├── report_generator.py     # Generate PDF/HTML reports
└── storage.py              # Store results in ClickHouse + S3
```

#### Backtest Engine

```python
# engine/backtest_engine.py
from dataclasses import dataclass, field
from typing import List, Dict
import pandas as pd
import numpy as np

@dataclass
class Trade:
    entry_time: datetime
    exit_time: datetime
    entry_price: float
    exit_price: float
    quantity: int
    direction: str  # LONG or SHORT
    pnl: float
    pnl_pct: float
    metadata: dict = field(default_factory=dict)

@dataclass
class BacktestResult:
    strategy_name: str
    symbol: str
    interval: str
    start_date: datetime
    end_date: datetime
    initial_capital: float
    final_capital: float
    
    # Trade statistics
    trades: List[Trade]
    total_trades: int
    winning_trades: int
    losing_trades: int
    
    # Performance metrics
    total_return: float
    cagr: float
    sharpe_ratio: float
    sortino_ratio: float
    max_drawdown: float
    max_drawdown_duration: int  # days
    win_rate: float
    profit_factor: float
    
    # Equity curve
    equity_curve: pd.Series
    drawdown_curve: pd.Series
    
    # Monthly returns
    monthly_returns: pd.Series

class BacktestEngine:
    def __init__(
        self,
        strategy: StrategyBase,
        initial_capital: float = 100000,
        commission: float = 0.001,  # 0.1% per trade
        slippage: float = 0.0005     # 0.05% slippage
    ):
        self.strategy = strategy
        self.initial_capital = initial_capital
        self.commission = commission
        self.slippage = slippage
        
        self.equity_curve = []
        self.trades = []
        self.current_position = None
    
    def run(
        self,
        symbol: str,
        interval: str,
        start_date: datetime,
        end_date: datetime
    ) -> BacktestResult:
        """
        Run backtest on historical data
        """
        # Load historical data from TimescaleDB
        ohlcv = self._load_data(symbol, interval, start_date, end_date)
        
        capital = self.initial_capital
        self.equity_curve = [capital]
        self.trades = []
        self.current_position = None
        
        # Iterate through each candle
        required_candles = self.strategy.get_required_candles()
        
        for i in range(required_candles, len(ohlcv)):
            # Get historical window for strategy
            window = ohlcv.iloc[:i+1]
            
            # Generate signal
            signal = self.strategy.calculate(window)
            
            # Execute trade logic
            current_candle = ohlcv.iloc[i]
            capital = self._process_signal(signal, current_candle, capital)
            
            self.equity_curve.append(capital)
        
        # Close any open position at end
        if self.current_position:
            final_candle = ohlcv.iloc[-1]
            capital = self._close_position(final_candle, capital)
            self.equity_curve.append(capital)
        
        # Calculate metrics
        metrics = self._calculate_metrics(ohlcv)
        
        return BacktestResult(
            strategy_name=self.strategy.__class__.__name__,
            symbol=symbol,
            interval=interval,
            start_date=start_date,
            end_date=end_date,
            initial_capital=self.initial_capital,
            final_capital=capital,
            trades=self.trades,
            **metrics
        )
    
    def _process_signal(self, signal: Optional[Signal], candle: pd.Series, capital: float) -> float:
        """Process trading signal"""
        if signal is None:
            return capital
        
        # Apply slippage
        execution_price = candle['close'] * (1 + self.slippage if signal.action == 'BUY' else 1 - self.slippage)
        
        if signal.action == 'BUY' and self.current_position is None:
            # Open long position
            quantity = int((capital * 0.95) / execution_price)  # Use 95% of capital
            cost = quantity * execution_price * (1 + self.commission)
            
            if cost <= capital:
                self.current_position = {
                    'direction': 'LONG',
                    'entry_time': candle['time'],
                    'entry_price': execution_price,
                    'quantity': quantity,
                    'metadata': signal.metadata
                }
                capital -= cost
        
        elif signal.action == 'SELL' and self.current_position and self.current_position['direction'] == 'LONG':
            # Close long position
            capital = self._close_position(candle, capital)
        
        return capital
    
    def _close_position(self, candle: pd.Series, capital: float) -> float:
        """Close current position"""
        if not self.current_position:
            return capital
        
        pos = self.current_position
        execution_price = candle['close'] * (1 - self.slippage)  # Slippage on exit
        proceeds = pos['quantity'] * execution_price * (1 - self.commission)
        
        # Record trade
        pnl = proceeds - (pos['quantity'] * pos['entry_price'])
        pnl_pct = pnl / (pos['quantity'] * pos['entry_price'])
        
        trade = Trade(
            entry_time=pos['entry_time'],
            exit_time=candle['time'],
            entry_price=pos['entry_price'],
            exit_price=execution_price,
            quantity=pos['quantity'],
            direction=pos['direction'],
            pnl=pnl,
            pnl_pct=pnl_pct,
            metadata=pos['metadata']
        )
        
        self.trades.append(trade)
        capital += proceeds
        self.current_position = None
        
        return capital
    
    def _calculate_metrics(self, ohlcv: pd.DataFrame) -> Dict:
        """Calculate performance metrics"""
        equity_series = pd.Series(self.equity_curve)
        returns = equity_series.pct_change().dropna()
        
        # Basic metrics
        total_return = (equity_series.iloc[-1] - equity_series.iloc[0]) / equity_series.iloc[0]
        
        # CAGR
        years = len(ohlcv) / 252  # Assuming daily candles
        cagr = (equity_series.iloc[-1] / equity_series.iloc[0]) ** (1 / years) - 1
        
        # Sharpe Ratio (annualized)
        sharpe = (returns.mean() / returns.std()) * np.sqrt(252) if returns.std() > 0 else 0
        
        # Sortino Ratio (downside deviation)
        downside_returns = returns[returns < 0]
        sortino = (returns.mean() / downside_returns.std()) * np.sqrt(252) if len(downside_returns) > 0 else 0
        
        # Drawdown
        cumulative_returns = (1 + returns).cumprod()
        running_max = cumulative_returns.cummax()
        drawdown = (cumulative_returns - running_max) / running_max
        max_drawdown = drawdown.min()
        
        # Max drawdown duration
        in_drawdown = drawdown < 0
        drawdown_periods = in_drawdown.astype(int).groupby((in_drawdown != in_drawdown.shift()).cumsum()).sum()
        max_dd_duration = drawdown_periods.max() if len(drawdown_periods) > 0 else 0
        
        # Trade statistics
        winning_trades = [t for t in self.trades if t.pnl > 0]
        losing_trades = [t for t in self.trades if t.pnl <= 0]
        
        win_rate = len(winning_trades) / len(self.trades) if self.trades else 0
        
        # Profit factor
        gross_profit = sum(t.pnl for t in winning_trades) if winning_trades else 0
        gross_loss = abs(sum(t.pnl for t in losing_trades)) if losing_trades else 0
        profit_factor = gross_profit / gross_loss if gross_loss > 0 else float('inf')
        
        return {
            'total_trades': len(self.trades),
            'winning_trades': len(winning_trades),
            'losing_trades': len(losing_trades),
            'total_return': total_return,
            'cagr': cagr,
            'sharpe_ratio': sharpe,
            'sortino_ratio': sortino,
            'max_drawdown': max_drawdown,
            'max_drawdown_duration': max_dd_duration,
            'win_rate': win_rate,
            'profit_factor': profit_factor,
            'equity_curve': equity_series,
            'drawdown_curve': drawdown
        }
```

#### Walk-Forward Optimization

```python
# engine/walk_forward.py
class WalkForwardOptimizer:
    """
    Walk-forward optimization:
    1. Split data into training and testing windows
    2. Optimize parameters on training window
    3. Test on out-of-sample testing window
    4. Roll forward and repeat
    """
    
    def __init__(
        self,
        strategy_class: Type[StrategyBase],
        param_space: Dict[str, List],
        train_period: int = 252,  # 1 year
        test_period: int = 63,    # 3 months
        step_size: int = 63       # Roll forward 3 months
    ):
        self.strategy_class = strategy_class
        self.param_space = param_space
        self.train_period = train_period
        self.test_period = test_period
        self.step_size = step_size
    
    def run(
        self,
        symbol: str,
        interval: str,
        start_date: datetime,
        end_date: datetime
    ) -> Dict:
        """Run walk-forward optimization"""
        
        # Load full dataset
        ohlcv = load_data(symbol, interval, start_date, end_date)
        
        results = []
        current_start = 0
        
        while current_start + self.train_period + self.test_period < len(ohlcv):
            # Define train and test windows
            train_end = current_start + self.train_period
            test_end = train_end + self.test_period
            
            train_data = ohlcv.iloc[current_start:train_end]
            test_data = ohlcv.iloc[train_end:test_end]
            
            # Optimize on training window
            best_params = self._optimize_on_window(train_data, symbol, interval)
            
            # Test on out-of-sample window
            strategy = self.strategy_class(best_params)
            engine = BacktestEngine(strategy)
            test_result = engine.run(symbol, interval, test_data.iloc[0]['time'], test_data.iloc[-1]['time'])
            
            results.append({
                'train_start': train_data.iloc[0]['time'],
                'train_end': train_data.iloc[-1]['time'],
                'test_start': test_data.iloc[0]['time'],
                'test_end': test_data.iloc[-1]['time'],
                'best_params': best_params,
                'test_sharpe': test_result.sharpe_ratio,
                'test_return': test_result.total_return,
                'test_max_dd': test_result.max_drawdown
            })
            
            # Roll forward
            current_start += self.step_size
        
        return {
            'windows': results,
            'avg_test_sharpe': np.mean([r['test_sharpe'] for r in results]),
            'avg_test_return': np.mean([r['test_return'] for r in results]),
            'avg_max_dd': np.mean([r['test_max_dd'] for r in results])
        }
    
    def _optimize_on_window(self, data: pd.DataFrame, symbol: str, interval: str) -> Dict:
        """Optimize parameters using Optuna"""
        import optuna
        
        def objective(trial):
            # Sample parameters from search space
            params = {
                key: trial.suggest_categorical(key, values)
                for key, values in self.param_space.items()
            }
            
            # Run backtest with these parameters
            strategy = self.strategy_class(params)
            engine = BacktestEngine(strategy)
            result = engine.run(symbol, interval, data.iloc[0]['time'], data.iloc[-1]['time'])
            
            # Optimize for Sharpe ratio
            return result.sharpe_ratio
        
        # Run optimization
        study = optuna.create_study(direction='maximize')
        study.optimize(objective, n_trials=100, timeout=300)  # 5 minute timeout
        
        return study.best_params
```

#### Storage in ClickHouse

```sql
-- ClickHouse table for backtest results
CREATE TABLE backtest_results (
    backtest_id UUID DEFAULT generateUUIDv4(),
    user_id String,
    strategy_name String,
    symbol String,
    interval String,
    parameters String,  -- JSON serialized
    
    start_date Date,
    end_date Date,
    
    -- Performance metrics
    initial_capital Float64,
    final_capital Float64,
    total_return Float64,
    cagr Float64,
    sharpe_ratio Float64,
    sortino_ratio Float64,
    max_drawdown Float64,
    max_drawdown_duration Int32,
    
    -- Trade statistics
    total_trades Int32,
    winning_trades Int32,
    losing_trades Int32,
    win_rate Float64,
    profit_factor Float64,
    
    -- Metadata
    execution_time_ms Int32,
    created_at DateTime DEFAULT now(),
    
    INDEX idx_user_id (user_id) TYPE bloom_filter GRANULARITY 1,
    INDEX idx_strategy (strategy_name) TYPE bloom_filter GRANULARITY 1
    
) ENGINE = MergeTree()
ORDER BY (strategy_name, symbol, created_at)
PARTITION BY toYYYYMM(created_at)
TTL created_at + INTERVAL 2 YEAR;

-- Store detailed trade history
CREATE TABLE backtest_trades (
    backtest_id UUID,
    trade_number Int32,
    entry_time DateTime,
    exit_time DateTime,
    entry_price Float64,
    exit_price Float64,
    quantity Int32,
    direction Enum8('LONG' = 1, 'SHORT' = 2),
    pnl Float64,
    pnl_pct Float64,
    metadata String  -- JSON
    
) ENGINE = MergeTree()
ORDER BY (backtest_id, trade_number)
PARTITION BY toYYYYMM(entry_time)
TTL entry_time + INTERVAL 2 YEAR;
```

---

### Service E: ML Service

**Technology**: Python FastAPI + PyTorch + MLflow  
**Responsibility**: Model training, feature engineering, real-time inference

[Continued in next file due to length...]

