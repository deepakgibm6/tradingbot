# Technology Stack - Detailed Breakdown

## Service E: ML Service (Continued)

### Architecture

```python
# ml_service/
├── main.py                     # FastAPI application
├── training/
│   ├── feature_engineering.py  # Feature calculation
│   ├── direction_predictor.py  # Binary classifier (UP/DOWN)
│   ├── breakout_predictor.py   # Breakout probability
│   ├── regime_detector.py      # Unsupervised clustering
│   └── trainer.py              # Training orchestration
├── inference/
│   ├── model_loader.py         # Load models from MLflow
│   ├── feature_extractor.py    # Real-time feature extraction
│   └── predictor.py            # Inference service (ONNX)
├── feature_store/
│   ├── feast_config.py         # Feast feature store config
│   └── materialization.py      # Feature materialization
└── monitoring/
    ├── drift_detector.py       # Data drift detection
    └── model_monitor.py        # Model performance monitoring
```

### Feature Engineering Pipeline

```python
# training/feature_engineering.py
import pandas as pd
import talib
import numpy as np
from typing import Dict

class FeatureEngineer:
    """
    Calculate technical indicators as ML features
    """
    
    @staticmethod
    def calculate_features(ohlcv: pd.DataFrame) -> pd.DataFrame:
        """
        Calculate comprehensive feature set from OHLCV data
        
        Returns DataFrame with features for each timestamp
        """
        df = ohlcv.copy()
        
        # === MOMENTUM INDICATORS ===
        df['rsi_14'] = talib.RSI(df['close'], timeperiod=14)
        df['rsi_7'] = talib.RSI(df['close'], timeperiod=7)
        df['rsi_21'] = talib.RSI(df['close'], timeperiod=21)
        
        df['macd'], df['macd_signal'], df['macd_hist'] = talib.MACD(
            df['close'],
            fastperiod=12,
            slowperiod=26,
            signalperiod=9
        )
        
        df['roc_10'] = talib.ROC(df['close'], timeperiod=10)
        df['roc_20'] = talib.ROC(df['close'], timeperiod=20)
        
        df['momentum_10'] = talib.MOM(df['close'], timeperiod=10)
        
        # === VOLATILITY INDICATORS ===
        df['atr_14'] = talib.ATR(df['high'], df['low'], df['close'], timeperiod=14)
        df['atr_pct'] = df['atr_14'] / df['close']  # Normalized ATR
        
        upper, middle, lower = talib.BBANDS(
            df['close'],
            timeperiod=20,
            nbdevup=2,
            nbdevdn=2
        )
        df['bb_upper'] = upper
        df['bb_middle'] = middle
        df['bb_lower'] = lower
        df['bb_width'] = (upper - lower) / middle
        df['bb_position'] = (df['close'] - lower) / (upper - lower)  # 0-1 scale
        
        # === TREND INDICATORS ===
        df['adx_14'] = talib.ADX(df['high'], df['low'], df['close'], timeperiod=14)
        df['adx_20'] = talib.ADX(df['high'], df['low'], df['close'], timeperiod=20)
        
        df['ema_9'] = talib.EMA(df['close'], timeperiod=9)
        df['ema_21'] = talib.EMA(df['close'], timeperiod=21)
        df['ema_50'] = talib.EMA(df['close'], timeperiod=50)
        df['ema_200'] = talib.EMA(df['close'], timeperiod=200)
        
        # EMA crossover signals
        df['ema_9_21_cross'] = np.where(df['ema_9'] > df['ema_21'], 1, -1)
        df['ema_50_200_cross'] = np.where(df['ema_50'] > df['ema_200'], 1, -1)
        
        # === VOLUME INDICATORS ===
        df['volume_sma_20'] = talib.SMA(df['volume'], timeperiod=20)
        df['volume_ratio'] = df['volume'] / df['volume_sma_20']
        
        df['obv'] = talib.OBV(df['close'], df['volume'])
        df['obv_ema'] = talib.EMA(df['obv'], timeperiod=20)
        
        df['ad'] = talib.AD(df['high'], df['low'], df['close'], df['volume'])
        df['adosc'] = talib.ADOSC(df['high'], df['low'], df['close'], df['volume'])
        
        # === PRICE PATTERNS ===
        # Donchian Channel (support/resistance)
        df['donchian_high_20'] = df['high'].rolling(window=20).max()
        df['donchian_low_20'] = df['low'].rolling(window=20).min()
        df['donchian_mid'] = (df['donchian_high_20'] + df['donchian_low_20']) / 2
        df['donchian_position'] = (df['close'] - df['donchian_low_20']) / \
                                   (df['donchian_high_20'] - df['donchian_low_20'])
        
        # Pivot points
        df['pivot'] = (df['high'] + df['low'] + df['close']) / 3
        df['r1'] = 2 * df['pivot'] - df['low']
        df['s1'] = 2 * df['pivot'] - df['high']
        
        # === MARKET MICROSTRUCTURE ===
        # Price action features
        df['body_size'] = abs(df['close'] - df['open']) / df['open']
        df['upper_wick'] = (df['high'] - df[['open', 'close']].max(axis=1)) / df['open']
        df['lower_wick'] = (df[['open', 'close']].min(axis=1) - df['low']) / df['open']
        df['candle_range'] = (df['high'] - df['low']) / df['open']
        
        # Gap detection
        df['gap_up'] = (df['open'] - df['close'].shift(1)) / df['close'].shift(1)
        df['gap_up'] = df['gap_up'].clip(lower=0)
        
        # === REGIME FEATURES ===
        # Volatility regime
        df['volatility_20'] = df['close'].pct_change().rolling(window=20).std()
        df['volatility_regime'] = pd.qcut(
            df['volatility_20'],
            q=3,
            labels=['low', 'medium', 'high'],
            duplicates='drop'
        )
        
        # Trend strength
        df['trend_strength'] = np.where(
            df['adx_14'] > 25,
            'strong',
            np.where(df['adx_14'] > 15, 'moderate', 'weak')
        )
        
        # === TIME-BASED FEATURES ===
        df['hour'] = df['time'].dt.hour
        df['day_of_week'] = df['time'].dt.dayofweek
        df['is_market_open'] = ((df['hour'] >= 9) & (df['hour'] < 15)).astype(int)
        
        # First/last hour of trading (higher volatility)
        df['is_first_hour'] = ((df['hour'] == 9)).astype(int)
        df['is_last_hour'] = ((df['hour'] == 14) | (df['hour'] == 15)).astype(int)
        
        return df.dropna()  # Remove rows with NaN from indicator calculations
    
    @staticmethod
    def create_labels(ohlcv: pd.DataFrame, horizon: int = 60) -> pd.Series:
        """
        Create labels for supervised learning
        
        Args:
            ohlcv: OHLCV dataframe
            horizon: Number of minutes ahead to predict
        
        Returns:
            Series with labels (1 = UP, 0 = DOWN)
        """
        # Calculate future return
        future_return = ohlcv['close'].shift(-horizon) / ohlcv['close'] - 1
        
        # Binary classification: >0.1% move = 1, else 0
        labels = (future_return > 0.001).astype(int)
        
        return labels
```

### Model Training Pipeline

```python
# training/direction_predictor.py
import mlflow
import mlflow.sklearn
from sklearn.model_selection import train_test_split, TimeSeriesSplit
from sklearn.metrics import accuracy_score, precision_score, recall_score, f1_score, roc_auc_score
from xgboost import XGBClassifier
import optuna

class DirectionPredictor:
    """
    Predict if price will move UP or DOWN in next 1 hour
    """
    
    def __init__(self, symbol: str, interval: str = '15m'):
        self.symbol = symbol
        self.interval = interval
        self.model = None
        self.feature_columns = None
    
    def prepare_data(self, start_date: datetime, end_date: datetime):
        """Load and prepare training data"""
        # Load OHLCV from TimescaleDB
        ohlcv = load_ohlcv(self.symbol, self.interval, start_date, end_date)
        
        # Feature engineering
        features_df = FeatureEngineer.calculate_features(ohlcv)
        labels = FeatureEngineer.create_labels(ohlcv, horizon=60)  # 1 hour ahead
        
        # Align features and labels
        df = features_df.join(labels.rename('target')).dropna()
        
        # Select feature columns (exclude metadata)
        self.feature_columns = [
            col for col in df.columns
            if col not in ['time', 'symbol', 'open', 'high', 'low', 'close', 'volume', 'target']
        ]
        
        X = df[self.feature_columns]
        y = df['target']
        
        return X, y
    
    def train(self, X, y, optimize: bool = True):
        """
        Train XGBoost classifier with optional hyperparameter tuning
        """
        # Time-series split (respects temporal order)
        tscv = TimeSeriesSplit(n_splits=5)
        
        if optimize:
            # Hyperparameter optimization with Optuna
            best_params = self._optimize_hyperparameters(X, y, tscv)
        else:
            best_params = {
                'max_depth': 8,
                'learning_rate': 0.05,
                'n_estimators': 200,
                'subsample': 0.8,
                'colsample_bytree': 0.8
            }
        
        # Train final model with best parameters
        self.model = XGBClassifier(
            **best_params,
            random_state=42,
            use_label_encoder=False,
            eval_metric='logloss'
        )
        
        # Split data (80/20)
        X_train, X_test, y_train, y_test = train_test_split(
            X, y,
            test_size=0.2,
            shuffle=False  # Maintain temporal order
        )
        
        # Train
        self.model.fit(
            X_train, y_train,
            eval_set=[(X_test, y_test)],
            early_stopping_rounds=20,
            verbose=False
        )
        
        # Evaluate
        metrics = self._evaluate(X_test, y_test)
        
        return metrics
    
    def _optimize_hyperparameters(self, X, y, tscv):
        """Hyperparameter tuning with Optuna"""
        
        def objective(trial):
            params = {
                'max_depth': trial.suggest_int('max_depth', 3, 12),
                'learning_rate': trial.suggest_float('learning_rate', 0.01, 0.3, log=True),
                'n_estimators': trial.suggest_int('n_estimators', 50, 500),
                'subsample': trial.suggest_float('subsample', 0.6, 1.0),
                'colsample_bytree': trial.suggest_float('colsample_bytree', 0.6, 1.0),
                'min_child_weight': trial.suggest_int('min_child_weight', 1, 10),
                'gamma': trial.suggest_float('gamma', 0, 5)
            }
            
            # Cross-validation
            scores = []
            for train_idx, val_idx in tscv.split(X):
                X_train, X_val = X.iloc[train_idx], X.iloc[val_idx]
                y_train, y_val = y.iloc[train_idx], y.iloc[val_idx]
                
                model = XGBClassifier(**params, random_state=42, use_label_encoder=False, eval_metric='logloss')
                model.fit(X_train, y_train, verbose=False)
                
                y_pred_proba = model.predict_proba(X_val)[:, 1]
                score = roc_auc_score(y_val, y_pred_proba)
                scores.append(score)
            
            return np.mean(scores)
        
        study = optuna.create_study(direction='maximize')
        study.optimize(objective, n_trials=50, timeout=600)  # 10 min timeout
        
        return study.best_params
    
    def _evaluate(self, X_test, y_test):
        """Evaluate model performance"""
        y_pred = self.model.predict(X_test)
        y_pred_proba = self.model.predict_proba(X_test)[:, 1]
        
        metrics = {
            'accuracy': accuracy_score(y_test, y_pred),
            'precision': precision_score(y_test, y_pred),
            'recall': recall_score(y_test, y_pred),
            'f1_score': f1_score(y_test, y_pred),
            'roc_auc': roc_auc_score(y_test, y_pred_proba)
        }
        
        return metrics
    
    def save_model(self, version: str = None):
        """Save model to MLflow registry"""
        with mlflow.start_run():
            # Log model
            mlflow.sklearn.log_model(self.model, "direction_predictor")
            
            # Log feature names
            mlflow.log_param("feature_columns", self.feature_columns)
            mlflow.log_param("symbol", self.symbol)
            mlflow.log_param("interval", self.interval)
            
            # Register model
            model_uri = f"runs:/{mlflow.active_run().info.run_id}/direction_predictor"
            mlflow.register_model(model_uri, f"direction_predictor_{self.symbol}")
```

### Real-Time Inference Service

```python
# inference/predictor.py
import onnxruntime as ort
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
import numpy as np

app = FastAPI(title="ML Inference Service")

class PredictionRequest(BaseModel):
    symbol: str
    interval: str

class PredictionResponse(BaseModel):
    symbol: str
    direction: str  # UP or DOWN
    confidence: float
    timestamp: datetime
    features: dict

class MLPredictor:
    def __init__(self):
        self.models = {}
        self.feature_columns = {}
        self._load_production_models()
    
    def _load_production_models(self):
        """Load all production models from MLflow"""
        client = mlflow.tracking.MlflowClient()
        
        # Get all registered models
        for rm in client.list_registered_models():
            # Get production version
            for mv in rm.latest_versions:
                if mv.current_stage == 'Production':
                    # Download and convert to ONNX
                    model_uri = f"models:/{rm.name}/{mv.version}"
                    model = mlflow.sklearn.load_model(model_uri)
                    
                    # Convert to ONNX for faster inference
                    onnx_model = self._convert_to_onnx(model)
                    
                    # Create ONNX Runtime session
                    self.models[rm.name] = ort.InferenceSession(onnx_model)
                    
                    # Load feature columns
                    run = client.get_run(mv.run_id)
                    self.feature_columns[rm.name] = eval(run.data.params['feature_columns'])
    
    def _convert_to_onnx(self, sklearn_model):
        """Convert sklearn model to ONNX format"""
        from skl2onnx import convert_sklearn
        from skl2onnx.common.data_types import FloatTensorType
        
        initial_type = [('float_input', FloatTensorType([None, len(self.feature_columns)]))]
        onnx_model = convert_sklearn(sklearn_model, initial_types=initial_type)
        return onnx_model.SerializeToString()
    
    async def predict(self, symbol: str, interval: str) -> PredictionResponse:
        """
        Real-time prediction with sub-100ms latency
        """
        # 1. Load recent candles from Dragonfly cache
        candles_json = await cache.lrange(f"candles:{symbol}:{interval}", 0, 500)
        if not candles_json:
            raise HTTPException(status_code=404, detail="No data available")
        
        candles = pd.DataFrame([json.loads(c) for c in candles_json])
        
        # 2. Extract features (in-memory, fast)
        features_df = FeatureEngineer.calculate_features(candles)
        latest_features = features_df.iloc[-1]
        
        # 3. Prepare model input
        model_name = f"direction_predictor_{symbol}"
        if model_name not in self.models:
            model_name = "direction_predictor_default"  # Fallback to default model
        
        feature_values = latest_features[self.feature_columns[model_name]].values
        input_array = feature_values.reshape(1, -1).astype(np.float32)
        
        # 4. Run inference (ONNX Runtime - optimized)
        ort_session = self.models[model_name]
        ort_inputs = {ort_session.get_inputs()[0].name: input_array}
        ort_outs = ort_session.run(None, ort_inputs)
        
        # 5. Parse results
        probabilities = ort_outs[0][0]  # [prob_down, prob_up]
        confidence = float(max(probabilities))
        direction = "UP" if probabilities[1] > probabilities[0] else "DOWN"
        
        # 6. Publish prediction to Kafka
        prediction_event = {
            'symbol': symbol,
            'interval': interval,
            'direction': direction,
            'confidence': confidence,
            'timestamp': datetime.utcnow().isoformat()
        }
        await kafka_producer.send('ml-predictions', prediction_event)
        
        return PredictionResponse(
            symbol=symbol,
            direction=direction,
            confidence=confidence,
            timestamp=datetime.utcnow(),
            features=latest_features[self.feature_columns[model_name]].to_dict()
        )

# Initialize predictor
predictor = MLPredictor()

@app.post("/predict", response_model=PredictionResponse)
async def predict_direction(request: PredictionRequest):
    """Predict price direction for given symbol"""
    return await predictor.predict(request.symbol, request.interval)

@app.get("/health")
async def health_check():
    return {"status": "healthy", "models_loaded": len(predictor.models)}
```

### Airflow DAG for Training Pipeline

```python
# dags/ml_training_pipeline.py
from airflow import DAG
from airflow.operators.python import PythonOperator
from datetime import datetime, timedelta

default_args = {
    'owner': 'ml-team',
    'depends_on_past': False,
    'email_on_failure': True,
    'email_on_retry': False,
    'retries': 2,
    'retry_delay': timedelta(minutes=5)
}

def feature_engineering_task():
    """Extract features from historical data"""
    symbols = get_all_symbols()
    
    for symbol in symbols:
        # Load 2 years of data
        end_date = datetime.now()
        start_date = end_date - timedelta(days=730)
        
        ohlcv = load_ohlcv(symbol, '15m', start_date, end_date)
        
        # Calculate features
        features = FeatureEngineer.calculate_features(ohlcv)
        labels = FeatureEngineer.create_labels(ohlcv, horizon=60)
        
        # Store in feature store
        feature_store.write_features(
            symbol=symbol,
            features=features,
            labels=labels
        )

def model_training_task():
    """Train models for all symbols"""
    symbols = get_all_symbols()
    
    for symbol in symbols:
        # Load features from feature store
        X, y = feature_store.read_features(symbol)
        
        # Train model
        predictor = DirectionPredictor(symbol, interval='15m')
        metrics = predictor.train(X, y, optimize=True)
        
        # Check if model performs better than current production
        current_metrics = get_production_model_metrics(symbol)
        
        if metrics['roc_auc'] > current_metrics.get('roc_auc', 0.5):
            # Save and promote to production
            predictor.save_model()
            promote_to_production(f"direction_predictor_{symbol}")

def drift_detection_task():
    """Detect data drift and trigger retraining if needed"""
    from evidently.pipeline.column_mapping import ColumnMapping
    from evidently.model_profile import Profile
    from evidently.model_profile.sections import DataDriftProfileSection
    
    symbols = get_all_symbols()
    
    for symbol in symbols:
        # Load reference data (training data)
        reference_data = feature_store.read_features(symbol, reference=True)
        
        # Load current data (last 7 days)
        current_data = feature_store.read_features(symbol, days=7)
        
        # Calculate drift
        profile = Profile(sections=[DataDriftProfileSection()])
        profile.calculate(reference_data, current_data)
        
        drift_report = profile.json()
        
        # Check if drift detected
        if drift_report['data_drift']['data_drift_detected']:
            # Trigger retraining
            trigger_retraining(symbol)

with DAG(
    dag_id='ml_training_pipeline',
    default_args=default_args,
    description='ML model training and monitoring pipeline',
    schedule_interval='@weekly',  # Run every week
    start_date=datetime(2024, 1, 1),
    catchup=False,
    tags=['ml', 'training']
) as dag:
    
    feature_eng = PythonOperator(
        task_id='feature_engineering',
        python_callable=feature_engineering_task
    )
    
    model_training = PythonOperator(
        task_id='model_training',
        python_callable=model_training_task
    )
    
    drift_detection = PythonOperator(
        task_id='drift_detection',
        python_callable=drift_detection_task
    )
    
    # Define task dependencies
    feature_eng >> model_training >> drift_detection
```

---

## Service F: Auth & User Service

**Technology**: Python FastAPI + PostgreSQL  
**Responsibility**: Authentication, authorization, user management

### Architecture

```python
# auth_service/
├── main.py                 # FastAPI application
├── auth/
│   ├── google_oauth.py     # Google Sign-In integration
│   ├── jwt_handler.py      # JWT token generation & validation
│   └── rate_limiter.py     # Rate limiting middleware
├── models/
│   ├── user.py             # User data model
│   ├── watchlist.py        # Watchlist model
│   └── alert.py            # Alert model
├── crud/
│   ├── user_crud.py        # User CRUD operations
│   ├── watchlist_crud.py   # Watchlist operations
│   └── alert_crud.py       # Alert operations
└── rbac/
    ├── roles.py            # Role definitions
    └── permissions.py      # Permission checks
```

### Google OAuth Integration

```python
# auth/google_oauth.py
from google.oauth2 import id_token
from google.auth.transport import requests
from fastapi import HTTPException

class GoogleAuthService:
    def __init__(self, client_id: str):
        self.client_id = client_id
    
    async def verify_token(self, token: str) -> dict:
        """
        Verify Google ID token
        
        Returns user info: {email, name, picture, email_verified}
        """
        try:
            idinfo = id_token.verify_oauth2_token(
                token,
                requests.Request(),
                self.client_id
            )
            
            if idinfo['iss'] not in ['accounts.google.com', 'https://accounts.google.com']:
                raise ValueError('Wrong issuer')
            
            return {
                'email': idinfo['email'],
                'name': idinfo['name'],
                'picture': idinfo['picture'],
                'email_verified': idinfo['email_verified']
            }
        
        except ValueError as e:
            raise HTTPException(status_code=401, detail=f"Invalid token: {str(e)}")

# main.py endpoints
@app.post("/auth/google/signin")
async def google_signin(id_token: str):
    """
    Google Sign-In authentication
    
    1. Verify Google token
    2. Create/update user in database
    3. Generate JWT token
    4. Return access token
    """
    # Verify token
    google_auth = GoogleAuthService(settings.GOOGLE_CLIENT_ID)
    user_info = await google_auth.verify_token(id_token)
    
    # Create or update user
    user = await upsert_user(user_info)
    
    # Generate JWT
    jwt_token = create_jwt_token(user)
    
    return {
        'access_token': jwt_token,
        'token_type': 'bearer',
        'user': user.dict()
    }
```

### JWT Token Handling

```python
# auth/jwt_handler.py
from jose import JWTError, jwt
from datetime import datetime, timedelta
from fastapi import Depends, HTTPException, status
from fastapi.security import OAuth2PasswordBearer

oauth2_scheme = OAuth2PasswordBearer(tokenUrl="token")

SECRET_KEY = settings.JWT_SECRET_KEY
ALGORITHM = "HS256"
ACCESS_TOKEN_EXPIRE_MINUTES = 60 * 24  # 24 hours

def create_jwt_token(user: User) -> str:
    """Create JWT access token"""
    to_encode = {
        'sub': str(user.id),
        'email': user.email,
        'role': user.role,
        'exp': datetime.utcnow() + timedelta(minutes=ACCESS_TOKEN_EXPIRE_MINUTES)
    }
    return jwt.encode(to_encode, SECRET_KEY, algorithm=ALGORITHM)

async def get_current_user(token: str = Depends(oauth2_scheme)) -> User:
    """Dependency to get current authenticated user"""
    credentials_exception = HTTPException(
        status_code=status.HTTP_401_UNAUTHORIZED,
        detail="Could not validate credentials",
        headers={"WWW-Authenticate": "Bearer"},
    )
    
    try:
        payload = jwt.decode(token, SECRET_KEY, algorithms=[ALGORITHM])
        user_id: str = payload.get("sub")
        if user_id is None:
            raise credentials_exception
        
        # Load user from database
        user = await db.get_user_by_id(user_id)
        if user is None:
            raise credentials_exception
        
        return user
    
    except JWTError:
        raise credentials_exception
```

### RBAC System

```python
# rbac/roles.py
from enum import Enum
from pydantic import BaseModel

class UserRole(str, Enum):
    FREE = "free"
    ENTERPRISE = "enterprise"
    ADMIN = "admin"

class RolePermissions(BaseModel):
    """Define permissions for each role"""
    max_watchlists: int
    max_alerts: int
    max_backtests_per_day: int
    data_delay_seconds: int  # 0 = real-time, 900 = 15 min delay
    strategies_access: list[str]
    api_access: bool
    custom_strategies: bool
    ml_predictions: bool
    
ROLE_CONFIG = {
    UserRole.FREE: RolePermissions(
        max_watchlists=1,
        max_alerts=5,
        max_backtests_per_day=3,
        data_delay_seconds=900,  # 15 min delay
        strategies_access=['bollinger_bands', 'rsi_divergence'],
        api_access=False,
        custom_strategies=False,
        ml_predictions=False
    ),
    UserRole.ENTERPRISE: RolePermissions(
        max_watchlists=999,
        max_alerts=999,
        max_backtests_per_day=999,
        data_delay_seconds=0,  # Real-time
        strategies_access=['all'],
        api_access=True,
        custom_strategies=True,
        ml_predictions=True
    ),
    UserRole.ADMIN: RolePermissions(
        max_watchlists=999,
        max_alerts=999,
        max_backtests_per_day=999,
        data_delay_seconds=0,
        strategies_access=['all'],
        api_access=True,
        custom_strategies=True,
        ml_predictions=True
    )
}

def check_permission(user: User, permission: str) -> bool:
    """Check if user has specific permission"""
    role_perms = ROLE_CONFIG[user.role]
    return getattr(role_perms, permission, False)

def require_permission(permission: str):
    """Decorator to enforce permission checks"""
    def decorator(func):
        async def wrapper(*args, current_user: User = Depends(get_current_user), **kwargs):
            if not check_permission(current_user, permission):
                raise HTTPException(
                    status_code=403,
                    detail=f"Permission denied: {permission}"
                )
            return await func(*args, current_user=current_user, **kwargs)
        return wrapper
    return decorator
```

### User Endpoints

```python
# main.py
@app.get("/users/me")
async def get_current_user_info(current_user: User = Depends(get_current_user)):
    """Get current user profile"""
    return current_user

@app.get("/users/me/watchlists")
async def get_watchlists(current_user: User = Depends(get_current_user)):
    """Get user's watchlists"""
    watchlists = await db.get_watchlists(current_user.id)
    return watchlists

@app.post("/users/me/watchlists")
async def create_watchlist(
    name: str,
    symbols: list[str],
    current_user: User = Depends(get_current_user)
):
    """Create new watchlist"""
    # Check permission
    role_perms = ROLE_CONFIG[current_user.role]
    existing_count = await db.count_watchlists(current_user.id)
    
    if existing_count >= role_perms.max_watchlists:
        raise HTTPException(
            status_code=403,
            detail=f"Maximum watchlists reached ({role_perms.max_watchlists})"
        )
    
    watchlist = await db.create_watchlist(current_user.id, name, symbols)
    return watchlist

@app.get("/users/me/alerts")
async def get_alerts(current_user: User = Depends(get_current_user)):
    """Get user's alerts"""
    alerts = await db.get_alerts(current_user.id)
    return alerts

@app.post("/users/me/alerts")
async def create_alert(
    symbol: str,
    condition: str,  # 'price_above', 'price_below', 'signal_generated'
    value: float,
    current_user: User = Depends(get_current_user)
):
    """Create price/signal alert"""
    role_perms = ROLE_CONFIG[current_user.role]
    existing_count = await db.count_alerts(current_user.id)
    
    if existing_count >= role_perms.max_alerts:
        raise HTTPException(
            status_code=403,
            detail=f"Maximum alerts reached ({role_perms.max_alerts})"
        )
    
    alert = await db.create_alert(current_user.id, symbol, condition, value)
    
    # Publish alert creation event to Kafka
    await kafka_producer.send('user-alerts', {
        'type': 'alert_created',
        'user_id': str(current_user.id),
        'alert_id': str(alert.id),
        'symbol': symbol,
        'condition': condition,
        'value': value
    })
    
    return alert
```

---

## Infrastructure Design (AWS)

### EKS Cluster Configuration

```yaml
# eks-cluster.yaml
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: trading-platform-prod
  region: us-east-1
  version: "1.28"

vpc:
  cidr: 10.0.0.0/16
  nat:
    gateway: HighlyAvailable  # NAT gateway in each AZ

availabilityZones: ["us-east-1a", "us-east-1b", "us-east-1c"]

managedNodeGroups:
  # API Services (FastAPI microservices)
  - name: api-services
    instanceType: r6i.2xlarge  # Memory-optimized
    desiredCapacity: 3
    minSize: 3
    maxSize: 20
    volumeSize: 100
    volumeType: gp3
    labels:
      workload: api
    tags:
      Environment: production
      Team: backend
    iam:
      withAddonPolicies:
        ebs: true
        efs: true
        albIngress: true
    
  # WebSocket Gateway (Node.js)
  - name: websocket-gateway
    instanceType: c6i.4xlarge  # Compute-optimized
    desiredCapacity: 5
    minSize: 3
    maxSize: 50
    volumeSize: 50
    labels:
      workload: websocket
    taints:
      - key: workload
        value: websocket
        effect: NoSchedule
    
  # Kafka Cluster
  - name: kafka-cluster
    instanceType: i3.2xlarge  # Storage-optimized (NVMe SSD)
    desiredCapacity: 3
    minSize: 3
    maxSize: 10
    volumeSize: 500
    volumeType: gp3
    labels:
      workload: kafka
    taints:
      - key: workload
        value: kafka
        effect: NoSchedule
    
  # ML Training (GPU, spot instances)
  - name: ml-training
    instanceType: p3.2xlarge  # GPU instances
    desiredCapacity: 0
    minSize: 0
    maxSize: 5
    spot: true
    volumeSize: 200
    labels:
      workload: ml-training
    taints:
      - key: nvidia.com/gpu
        value: "true"
        effect: NoSchedule

cloudWatch:
  clusterLogging:
    enableTypes: ["api", "audit", "authenticator", "controllerManager", "scheduler"]

addons:
  - name: vpc-cni
  - name: coredns
  - name: kube-proxy
  - name: aws-ebs-csi-driver
```

### Kubernetes Deployments

#### Market Data Service Deployment

```yaml
# k8s/market-data-service.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: market-data-service
  namespace: trading-platform
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  selector:
    matchLabels:
      app: market-data-service
  template:
    metadata:
      labels:
        app: market-data-service
        version: v1
    spec:
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchExpressions:
                - key: app
                  operator: In
                  values:
                  - market-data-service
              topologyKey: topology.kubernetes.io/zone
      
      containers:
      - name: market-data
        image: 123456789.dkr.ecr.us-east-1.amazonaws.com/market-data-service:latest
        ports:
        - containerPort: 8000
          name: http
        - containerPort: 9090
          name: metrics
        
        resources:
          requests:
            cpu: 2000m
            memory: 4Gi
          limits:
            cpu: 4000m
            memory: 8Gi
        
        env:
        - name: ENVIRONMENT
          value: "production"
        - name: LOG_LEVEL
          value: "INFO"
        - name: UPSTOX_API_KEY
          valueFrom:
            secretKeyRef:
              name: upstox-secrets
              key: api_key
        - name: DRAGONFLY_HOST
          value: "dragonfly.default.svc.cluster.local:6379"
        - name: KAFKA_BROKERS
          value: "kafka-0.kafka-headless:9092,kafka-1.kafka-headless:9092,kafka-2.kafka-headless:9092"
        - name: POSTGRES_HOST
          valueFrom:
            secretKeyRef:
              name: db-secrets
              key: host
        
        livenessProbe:
          httpGet:
            path: /health
            port: 8000
          initialDelaySeconds: 30
          periodSeconds: 10
          timeoutSeconds: 5
          failureThreshold: 3
        
        readinessProbe:
          httpGet:
            path: /ready
            port: 8000
          initialDelaySeconds: 10
          periodSeconds: 5
          timeoutSeconds: 3
          failureThreshold: 3
        
        volumeMounts:
        - name: config
          mountPath: /etc/config
          readOnly: true
      
      volumes:
      - name: config
        configMap:
          name: market-data-config

---
apiVersion: v1
kind: Service
metadata:
  name: market-data-service
  namespace: trading-platform
spec:
  type: ClusterIP
  ports:
  - port: 8000
    targetPort: 8000
    name: http
  - port: 9090
    targetPort: 9090
    name: metrics
  selector:
    app: market-data-service

---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: market-data-service-hpa
  namespace: trading-platform
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: market-data-service
  minReplicas: 3
  maxReplicas: 20
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 60
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 70
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 60
      policies:
      - type: Percent
        value: 50
        periodSeconds: 60
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
      - type: Pods
        value: 1
        periodSeconds: 180
```

---

## Deployment Strategy

### CI/CD Pipeline (GitHub Actions + ArgoCD)

```yaml
# .github/workflows/deploy.yml
name: CI/CD Pipeline

on:
  push:
    branches:
      - main
      - develop
    paths:
      - 'services/**'

env:
  AWS_REGION: us-east-1
  ECR_REGISTRY: 123456789.dkr.ecr.us-east-1.amazonaws.com

jobs:
  build-and-push:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        service:
          - market-data-service
          - strategy-engine
          - backtest-service
          - ml-service
          - auth-service
    
    steps:
    - name: Checkout code
      uses: actions/checkout@v3
    
    - name: Configure AWS credentials
      uses: aws-actions/configure-aws-credentials@v2
      with:
        aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
        aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
        aws-region: ${{ env.AWS_REGION }}
    
    - name: Login to Amazon ECR
      id: login-ecr
      uses: aws-actions/amazon-ecr-login@v1
    
    - name: Build, tag, and push image
      env:
        ECR_REPOSITORY: ${{ matrix.service }}
        IMAGE_TAG: ${{ github.sha }}
      run: |
        cd services/${{ matrix.service }}
        docker build -t $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG .
        docker push $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG
        docker tag $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG $ECR_REGISTRY/$ECR_REPOSITORY:latest
        docker push $ECR_REGISTRY/$ECR_REPOSITORY:latest
    
    - name: Update Kubernetes manifests
      run: |
        cd k8s/overlays/${{ github.ref_name }}
        kustomize edit set image ${{ matrix.service }}=$ECR_REGISTRY/${{ matrix.service }}:${{ github.sha }}
        git config user.name "GitHub Actions"
        git config user.email "actions@github.com"
        git add .
        git commit -m "Update ${{ matrix.service }} image to ${{ github.sha }}"
        git push

  deploy:
    needs: build-and-push
    runs-on: ubuntu-latest
    
    steps:
    - name: Trigger ArgoCD Sync
      run: |
        argocd app sync trading-platform-${{ github.ref_name }} \
          --server ${{ secrets.ARGOCD_SERVER }} \
          --auth-token ${{ secrets.ARGOCD_TOKEN }} \
          --prune \
          --async
    
    - name: Wait for rollout
      run: |
        argocd app wait trading-platform-${{ github.ref_name }} \
          --server ${{ secrets.ARGOCD_SERVER }} \
          --auth-token ${{ secrets.ARGOCD_TOKEN }} \
          --health \
          --timeout 600
```

### Blue-Green Deployment Strategy

```yaml
# k8s/blue-green-deployment.yaml
apiVersion: v1
kind: Service
metadata:
  name: market-data-service
spec:
  selector:
    app: market-data-service
    version: blue  # Switch to 'green' during deployment
  ports:
  - port: 8000

---
# Blue deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: market-data-service-blue
spec:
  replicas: 3
  selector:
    matchLabels:
      app: market-data-service
      version: blue
  template:
    metadata:
      labels:
        app: market-data-service
        version: blue
    spec:
      containers:
      - name: market-data
        image: market-data-service:v1.0.0

---
# Green deployment (new version)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: market-data-service-green
spec:
  replicas: 3
  selector:
    matchLabels:
      app: market-data-service
      version: green
  template:
    metadata:
      labels:
        app: market-data-service
        version: green
    spec:
      containers:
      - name: market-data
        image: market-data-service:v1.1.0

---
# Deployment script
# 1. Deploy green with 0 replicas
# 2. Scale green to desired replicas
# 3. Run smoke tests
# 4. If success: Switch service selector to green
# 5. Scale blue to 0
# 6. Monitor for 1 hour
# 7. Delete blue deployment
```

---

## Cost Optimization

### Reserved Instances & Savings Plans

```yaml
Cost Optimization Strategy:

1. EKS Node Groups:
   - Base capacity (always-on): Reserved Instances (1-year commitment)
     - api-services: 3x r6i.2xlarge = ~$900/month (save 40%)
     - kafka-cluster: 3x i3.2xlarge = ~$1,200/month (save 40%)
   
   - Burst capacity: On-Demand Auto-Scaling
     - Scale up during market hours (9:15-15:30)
     - Scale down after hours

2. RDS:
   - Multi-AZ PostgreSQL: Reserved Instance (3-year commitment)
   - Savings: ~60% ($500/month vs $1,250/month)

3. ElastiCache/MemoryDB:
   - Dragonfly cluster: Reserved Instances
   - Savings: ~45%

4. ML Training:
   - Use Spot Instances (p3.2xlarge)
   - Savings: ~70% ($0.918/hour vs $3.06/hour)
   - Implement spot instance interruption handling

5. Data Storage:
   - S3 Intelligent-Tiering: Automatic cost optimization
   - Archive old backtest reports to Glacier ($0.004/GB vs $0.023/GB)
   - TimescaleDB compression: 20x reduction in storage

6. Data Transfer:
   - Use CloudFront CDN: Reduce origin requests
   - VPC Endpoints: Avoid data transfer charges for AWS services

Estimated Monthly Cost:
- Compute (EKS): $3,500
- Databases (RDS + ClickHouse + Dragonfly): $2,000
- Storage (S3 + EBS): $500
- Networking (ALB + CloudFront + Data Transfer): $800
- Kafka (MSK): $1,200
- Monitoring (DataDog): $600
---------------------------------
Total: ~$8,600/month

With 10,000 users: $0.86/user/month
With 100,000 users: $0.086/user/month (economies of scale)
```

---

This comprehensive technical documentation provides a production-ready architecture for a stock analysis and trading platform. The design prioritizes:

1. **Low Latency**: Sub-100ms for real-time data and ML inference
2. **High Scalability**: Horizontal scaling across all services
3. **Reliability**: Multi-AZ deployment, auto-healing, circuit breakers
4. **Cost Efficiency**: Reserved instances, spot instances, intelligent storage tiering
5. **Observability**: Comprehensive monitoring, logging, and tracing
6. **Security**: Multiple layers of defense (WAF, API Gateway, RBAC, encryption)

The architecture is designed to handle millions of users while maintaining institutional-grade performance and reliability standards.
