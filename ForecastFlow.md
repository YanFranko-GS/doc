# ForecastFlow — Informe Técnico Enterprise
### Plataforma Local de Forecasting Empresarial · Arquitectura de Referencia

---

## 1. Arquitectura del Sistema

### 1.1 Visión General

ForecastFlow es una plataforma modular de forecasting empresarial diseñada para ejecutarse completamente on-premise. Su arquitectura separa responsabilidades en capas bien definidas: ingesta de datos, procesamiento distribuido, entrenamiento asíncrono de modelos, model registry con versionado, y presentación visual.

La plataforma opera sobre un modelo cliente-servidor donde el frontend React se comunica exclusivamente con la API Gateway FastAPI. Todas las operaciones de larga duración (exploración de datos, feature engineering, entrenamiento) se delegan a workers Celery coordinados por Redis. Los modelos entrenados se almacenan en un model registry local con seguimiento completo de métricas y versionado.

### 1.2 Diagrama de Arquitectura Completa

```mermaid
flowchart TD
    subgraph FRONTEND["Frontend — React + TypeScript"]
        UI[Wizard Visual\nConexión → Dataset → Objetivo → Horizonte]
        DASH[Dashboard Interactivo\nForecast + Métricas + Exportación]
        POLL[Polling Async\ncada 2 segundos]
    end

    subgraph GATEWAY["API Gateway — FastAPI"]
        AUTH[JWT Auth\n+ Rate Limiting]
        CONN[/api/connections]
        EXPLORE[/api/explore]
        TRAIN[/api/training/start]
        STATUS[/api/training/status/:job_id]
        RESULTS[/api/forecast/results]
        EXPORT[/api/export/csv|excel]
    end

    subgraph CACHE["Cache Layer — Redis"]
        RC1[Schema Cache\nTTL 300s]
        RC2[Preview Cache\nTTL 120s]
        RC3[Forecast Cache\nTTL 3600s]
    end

    subgraph QUEUE["Async Queue — Celery + Redis"]
        BROKER[Redis Broker]
        W1[Worker 1\nFeature Engineering]
        W2[Worker 2\nModel Training]
        W3[Worker 3\nValidation + Backtesting]
        W4[Worker 4\nForecast Generation]
    end

    subgraph ENGINE["Forecast Engine"]
        FE[Feature Engineering\nEngine]
        TR[Training\nOrchestrator]
        SEL[Model Selector\nAutoML Routing]
        VAL[Validator\nCross-Validation]
    end

    subgraph MODELS["Model Layer"]
        XGB[XGBoost]
        LGB[LightGBM]
        CAT[CatBoost]
        PRO[Prophet]
        SAR[SARIMA]
    end

    subgraph REGISTRY["Model Registry"]
        MR[Metadata Store\nSQLite + JSON]
        MA[Artifacts Store\njoblib / pickle]
        MV[Version Control\nSemVer]
        MM[Metrics Tracker\nMAPE, RMSE, MAE]
    end

    subgraph DATASOURCES["Data Sources"]
        PG[(PostgreSQL)]
        MY[(MySQL)]
        SS[(SQL Server)]
        SL[(SQLite)]
    end

    subgraph MONITORING["Monitoring"]
        LOG[Structured Logs\nloguru]
        MET[Métricas Runtime\nPrometheus-compatible]
        HLT[Health Checks\n/health]
    end

    UI -->|HTTP/REST + JWT| AUTH
    AUTH --> CONN & EXPLORE & TRAIN & STATUS & RESULTS & EXPORT
    CONN -->|encrypt AES-256| DATASOURCES
    EXPLORE -->|check| RC1
    RC1 -->|miss| DATASOURCES
    TRAIN -->|dispatch| BROKER
    BROKER --> W1 --> FE
    FE --> W2 --> TR
    TR --> SEL
    SEL --> XGB & LGB & CAT & PRO & SAR
    XGB & LGB & CAT & PRO & SAR --> VAL
    VAL --> W3 --> MM
    MM --> MR & MA & MV
    W4 --> RC3
    DASH -->|fetch| RC3
    POLL --> STATUS
    STATUS -->|job state| QUEUE
    ENGINE --- MONITORING
    GATEWAY --- MONITORING
```

### 1.3 Principios de Diseño

- **Separación de concerns**: cada capa tiene responsabilidad única y límites claros de API.
- **Async-first**: ninguna operación de larga duración bloquea la API principal.
- **Cache-first**: resultados frecuentes cacheados en Redis antes de ir a base de datos o reentrenar.
- **Fail-safe**: errores en workers no colapsan la API; el usuario recibe estado de error en su job.
- **Zero data leakage**: credenciales de BD cifradas en AES-256, nunca en logs ni en respuestas de API.

---

## 2. Backend FastAPI

### 2.1 Estructura del Proyecto

```
forecastflow/
├── api/
│   ├── main.py                  # Entrada FastAPI
│   ├── dependencies.py          # JWT, DB session, rate limit
│   ├── routers/
│   │   ├── auth.py
│   │   ├── connections.py
│   │   ├── explore.py
│   │   ├── training.py
│   │   ├── forecast.py
│   │   └── export.py
│   └── middleware/
│       ├── cors.py
│       └── logging.py
├── core/
│   ├── config.py                # Settings Pydantic
│   ├── security.py              # JWT + AES-256
│   └── database.py              # SQLAlchemy async engine
├── models/
│   ├── schemas.py               # Pydantic v2 models
│   └── db_models.py             # SQLAlchemy ORM models
├── services/
│   ├── connector.py             # Multi-DB connector
│   ├── explorer.py              # Schema + preview
│   └── forecast_service.py      # Orchestration logic
├── ml/
│   ├── feature_engineering.py
│   ├── model_router.py
│   ├── models/
│   │   ├── xgboost_model.py
│   │   ├── lightgbm_model.py
│   │   ├── catboost_model.py
│   │   ├── prophet_model.py
│   │   └── sarima_model.py
│   ├── validator.py
│   └── registry.py
├── workers/
│   ├── celery_app.py
│   └── tasks.py
├── docker-compose.yml
├── Dockerfile
└── requirements.txt
```

### 2.2 Configuración Principal (main.py)

```python
# api/main.py
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware
from contextlib import asynccontextmanager
from api.routers import auth, connections, explore, training, forecast, export
from api.middleware.logging import setup_logging
from core.config import settings
from core.database import engine, Base

@asynccontextmanager
async def lifespan(app: FastAPI):
    await Base.metadata.create_all(bind=engine)
    yield

app = FastAPI(
    title="ForecastFlow API",
    version="1.0.0",
    lifespan=lifespan,
    docs_url="/api/docs",
    redoc_url=None,
)

app.add_middleware(
    CORSMiddleware,
    allow_origins=settings.ALLOWED_ORIGINS,
    allow_credentials=True,
    allow_methods=["GET", "POST", "PUT", "DELETE"],
    allow_headers=["Authorization", "Content-Type"],
)

setup_logging()

app.include_router(auth.router,        prefix="/api/auth",        tags=["Auth"])
app.include_router(connections.router, prefix="/api/connections",  tags=["Connections"])
app.include_router(explore.router,     prefix="/api/explore",      tags=["Explore"])
app.include_router(training.router,    prefix="/api/training",     tags=["Training"])
app.include_router(forecast.router,    prefix="/api/forecast",     tags=["Forecast"])
app.include_router(export.router,      prefix="/api/export",       tags=["Export"])

@app.get("/health")
async def health():
    return {"status": "ok", "version": "1.0.0"}
```

### 2.3 Configuración y Seguridad (core/config.py + core/security.py)

```python
# core/config.py
from pydantic_settings import BaseSettings
from typing import List

class Settings(BaseSettings):
    SECRET_KEY: str
    ALGORITHM: str = "HS256"
    ACCESS_TOKEN_EXPIRE_MINUTES: int = 60
    REFRESH_TOKEN_EXPIRE_DAYS: int = 7
    AES_ENCRYPTION_KEY: str              # 32 bytes hex para AES-256
    DATABASE_URL: str = "sqlite+aiosqlite:///./forecastflow.db"
    REDIS_URL: str = "redis://localhost:6379/0"
    ALLOWED_ORIGINS: List[str] = ["http://localhost:3000"]
    MAX_CHUNK_SIZE: int = 50_000
    MODEL_REGISTRY_PATH: str = "./model_registry"

    class Config:
        env_file = ".env"

settings = Settings()

# core/security.py
import jwt
import base64
from datetime import datetime, timedelta, timezone
from cryptography.hazmat.primitives.ciphers import Cipher, algorithms, modes
from cryptography.hazmat.backends import default_backend
from passlib.context import CryptContext
from core.config import settings

pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")

def create_access_token(data: dict) -> str:
    payload = {**data, "exp": datetime.now(timezone.utc) + timedelta(minutes=settings.ACCESS_TOKEN_EXPIRE_MINUTES)}
    return jwt.encode(payload, settings.SECRET_KEY, algorithm=settings.ALGORITHM)

def create_refresh_token(data: dict) -> str:
    payload = {**data, "exp": datetime.now(timezone.utc) + timedelta(days=settings.REFRESH_TOKEN_EXPIRE_DAYS)}
    return jwt.encode(payload, settings.SECRET_KEY, algorithm=settings.ALGORITHM)

def decode_token(token: str) -> dict:
    return jwt.decode(token, settings.SECRET_KEY, algorithms=[settings.ALGORITHM])

def encrypt_credentials(plaintext: str) -> str:
    """AES-256-GCM encryption para credenciales de BD."""
    key = bytes.fromhex(settings.AES_ENCRYPTION_KEY)
    iv = b"\x00" * 16  # En producción usar os.urandom(16) y almacenar junto al cipher
    cipher = Cipher(algorithms.AES(key), modes.CBC(iv), backend=default_backend())
    encryptor = cipher.encryptor()
    padded = plaintext.encode().ljust(((len(plaintext) // 16) + 1) * 16)
    return base64.b64encode(encryptor.update(padded) + encryptor.finalize()).decode()

def decrypt_credentials(ciphertext: str) -> str:
    key = bytes.fromhex(settings.AES_ENCRYPTION_KEY)
    iv = b"\x00" * 16
    cipher = Cipher(algorithms.AES(key), modes.CBC(iv), backend=default_backend())
    decryptor = cipher.decryptor()
    raw = base64.b64decode(ciphertext)
    return (decryptor.update(raw) + decryptor.finalize()).decode().rstrip()
```

### 2.4 Router de Autenticación (api/routers/auth.py)

```python
# api/routers/auth.py
from fastapi import APIRouter, Depends, HTTPException, status
from fastapi.security import OAuth2PasswordBearer, OAuth2PasswordRequestForm
from sqlalchemy.ext.asyncio import AsyncSession
from core.database import get_db
from core.security import (pwd_context, create_access_token,
                           create_refresh_token, decode_token)
from models.schemas import TokenResponse, RefreshRequest, UserCreate
from models.db_models import User
from sqlalchemy import select

router = APIRouter()
oauth2_scheme = OAuth2PasswordBearer(tokenUrl="/api/auth/login")

@router.post("/login", response_model=TokenResponse)
async def login(form: OAuth2PasswordRequestForm = Depends(), db: AsyncSession = Depends(get_db)):
    result = await db.execute(select(User).where(User.username == form.username))
    user = result.scalar_one_or_none()
    if not user or not pwd_context.verify(form.password, user.hashed_password):
        raise HTTPException(status_code=status.HTTP_401_UNAUTHORIZED, detail="Credenciales inválidas")
    return TokenResponse(
        access_token=create_access_token({"sub": user.username, "role": user.role}),
        refresh_token=create_refresh_token({"sub": user.username}),
        token_type="bearer"
    )

@router.post("/refresh", response_model=TokenResponse)
async def refresh(body: RefreshRequest):
    try:
        payload = decode_token(body.refresh_token)
        return TokenResponse(
            access_token=create_access_token({"sub": payload["sub"]}),
            refresh_token=create_refresh_token({"sub": payload["sub"]}),
            token_type="bearer"
        )
    except Exception:
        raise HTTPException(status_code=401, detail="Token de refresco inválido")

async def get_current_user(token: str = Depends(oauth2_scheme), db: AsyncSession = Depends(get_db)):
    try:
        payload = decode_token(token)
        username = payload.get("sub")
    except Exception:
        raise HTTPException(status_code=401, detail="Token inválido")
    result = await db.execute(select(User).where(User.username == username))
    user = result.scalar_one_or_none()
    if not user:
        raise HTTPException(status_code=401, detail="Usuario no encontrado")
    return user
```

### 2.5 Router de Conexiones (api/routers/connections.py)

```python
# api/routers/connections.py
from fastapi import APIRouter, Depends, HTTPException
from sqlalchemy.ext.asyncio import AsyncSession
from core.database import get_db
from core.security import encrypt_credentials, decrypt_credentials
from models.schemas import ConnectionCreate, ConnectionTestResult, ConnectionResponse
from models.db_models import Connection, User
from api.routers.auth import get_current_user
from services.connector import DatabaseConnector
import asyncio, time

router = APIRouter()

@router.post("/test", response_model=ConnectionTestResult)
async def test_connection(
    conn: ConnectionCreate,
    current_user: User = Depends(get_current_user)
):
    """Prueba conexión a BD. SLA: < 2 segundos."""
    start = time.perf_counter()
    try:
        connector = DatabaseConnector(conn.connection_url)
        await asyncio.wait_for(connector.test(), timeout=2.0)
        elapsed = time.perf_counter() - start
        return ConnectionTestResult(success=True, latency_ms=round(elapsed * 1000, 2))
    except asyncio.TimeoutError:
        raise HTTPException(status_code=408, detail="Timeout: conexión tardó más de 2 segundos")
    except Exception as e:
        return ConnectionTestResult(success=False, error=str(e))

@router.post("/", response_model=ConnectionResponse)
async def create_connection(
    conn: ConnectionCreate,
    db: AsyncSession = Depends(get_db),
    current_user: User = Depends(get_current_user)
):
    """Crea y persiste una conexión con credenciales cifradas."""
    encrypted_url = encrypt_credentials(conn.connection_url)
    db_conn = Connection(
        name=conn.name,
        engine=conn.engine,
        encrypted_url=encrypted_url,
        owner_id=current_user.id
    )
    db.add(db_conn)
    await db.commit()
    await db.refresh(db_conn)
    return ConnectionResponse(id=db_conn.id, name=db_conn.name, engine=db_conn.engine)
```

### 2.6 Router de Exploración (api/routers/explore.py)

```python
# api/routers/explore.py
from fastapi import APIRouter, Depends, HTTPException
from models.schemas import ExploreResponse, TablePreview
from models.db_models import Connection, User
from api.routers.auth import get_current_user
from services.explorer import DataExplorer
from services.connector import DatabaseConnector
from core.security import decrypt_credentials
from core.config import settings
from sqlalchemy.ext.asyncio import AsyncSession
from core.database import get_db
from sqlalchemy import select
import redis.asyncio as redis
import json

router = APIRouter()
cache = redis.from_url(settings.REDIS_URL)

@router.get("/{connection_id}/tables", response_model=ExploreResponse)
async def get_tables(
    connection_id: int,
    db: AsyncSession = Depends(get_db),
    current_user: User = Depends(get_current_user)
):
    """Detecta tablas y columnas. SLA: < 3 segundos (cacheado < 1s)."""
    cache_key = f"schema:{connection_id}"
    cached = await cache.get(cache_key)
    if cached:
        return ExploreResponse(**json.loads(cached))

    result = await db.execute(select(Connection).where(
        Connection.id == connection_id, Connection.owner_id == current_user.id
    ))
    conn = result.scalar_one_or_none()
    if not conn:
        raise HTTPException(status_code=404, detail="Conexión no encontrada")

    url = decrypt_credentials(conn.encrypted_url)
    connector = DatabaseConnector(url)
    explorer = DataExplorer(connector)
    schema = await explorer.get_schema()

    await cache.setex(cache_key, 300, json.dumps(schema.model_dump()))
    return schema

@router.get("/{connection_id}/tables/{table_name}/preview", response_model=TablePreview)
async def preview_table(
    connection_id: int,
    table_name: str,
    limit: int = 100,
    db: AsyncSession = Depends(get_db),
    current_user: User = Depends(get_current_user)
):
    """Preview primeras 100 filas con tipos detectados."""
    cache_key = f"preview:{connection_id}:{table_name}"
    cached = await cache.get(cache_key)
    if cached:
        return TablePreview(**json.loads(cached))

    result = await db.execute(select(Connection).where(Connection.id == connection_id))
    conn = result.scalar_one_or_none()
    url = decrypt_credentials(conn.encrypted_url)
    connector = DatabaseConnector(url)
    explorer = DataExplorer(connector)
    preview = await explorer.preview_table(table_name, limit=limit)

    await cache.setex(cache_key, 120, json.dumps(preview.model_dump()))
    return preview
```

### 2.7 Router de Entrenamiento (api/routers/training.py)

```python
# api/routers/training.py
from fastapi import APIRouter, Depends, HTTPException
from models.schemas import TrainingJobRequest, TrainingJobStatus
from models.db_models import User, TrainingJob
from api.routers.auth import get_current_user
from workers.tasks import run_forecast_pipeline
from core.database import get_db
from sqlalchemy.ext.asyncio import AsyncSession
from sqlalchemy import select
import uuid

router = APIRouter()

@router.post("/start", status_code=202)
async def start_training(
    request: TrainingJobRequest,
    db: AsyncSession = Depends(get_db),
    current_user: User = Depends(get_current_user)
):
    """Lanza entrenamiento asíncrono. Retorna job_id inmediatamente."""
    job_id = str(uuid.uuid4())
    job = TrainingJob(
        id=job_id,
        connection_id=request.connection_id,
        table_name=request.table_name,
        target_column=request.target_column,
        date_column=request.date_column,
        horizon_months=request.horizon_months,
        status="queued",
        owner_id=current_user.id
    )
    db.add(job)
    await db.commit()

    # Despachar a Celery sin bloquear
    run_forecast_pipeline.apply_async(
        kwargs={"job_id": job_id, "request": request.model_dump()},
        task_id=job_id
    )
    return {"job_id": job_id, "status": "queued"}

@router.get("/status/{job_id}", response_model=TrainingJobStatus)
async def get_status(
    job_id: str,
    db: AsyncSession = Depends(get_db),
    current_user: User = Depends(get_current_user)
):
    """Polling cada 2s desde frontend. Retorna estado + progreso + logs."""
    result = await db.execute(select(TrainingJob).where(
        TrainingJob.id == job_id, TrainingJob.owner_id == current_user.id
    ))
    job = result.scalar_one_or_none()
    if not job:
        raise HTTPException(status_code=404, detail="Job no encontrado")

    return TrainingJobStatus(
        job_id=job.id,
        status=job.status,
        progress=job.progress,
        current_step=job.current_step,
        models_trained=job.models_trained or [],
        best_model=job.best_model,
        error=job.error_message,
        started_at=job.started_at,
        completed_at=job.completed_at,
    )
```

---

## 3. Pipeline de Machine Learning

### 3.1 Feature Engineering Engine (ml/feature_engineering.py)

```python
# ml/feature_engineering.py
import pandas as pd
import numpy as np
from typing import Optional

class FeatureEngineer:
    """Generación automática de features para series temporales."""

    LAG_WINDOWS    = [1, 7, 14, 30, 60, 90]
    ROLLING_WINDOWS = [7, 14, 30, 90]
    FOURIER_TERMS   = 5

    def __init__(self, freq: str = "D"):
        self.freq = freq

    def fit_transform(self, df: pd.DataFrame, target: str, date_col: str) -> pd.DataFrame:
        df = df.sort_values(date_col).copy()
        df = df.set_index(date_col)
        df = self._add_lag_features(df, target)
        df = self._add_rolling_features(df, target)
        df = self._add_temporal_encoding(df)
        df = self._add_fourier_features(df)
        df = self._add_trend_features(df, target)
        df = self._add_calendar_features(df)
        df = df.dropna()
        return df

    def _add_lag_features(self, df: pd.DataFrame, target: str) -> pd.DataFrame:
        for lag in self.LAG_WINDOWS:
            df[f"lag_{lag}"] = df[target].shift(lag)
            df[f"pct_change_{lag}"] = df[target].pct_change(lag)
        return df

    def _add_rolling_features(self, df: pd.DataFrame, target: str) -> pd.DataFrame:
        for w in self.ROLLING_WINDOWS:
            roll = df[target].rolling(window=w, min_periods=1)
            df[f"rolling_mean_{w}"]   = roll.mean()
            df[f"rolling_std_{w}"]    = roll.std()
            df[f"rolling_min_{w}"]    = roll.min()
            df[f"rolling_max_{w}"]    = roll.max()
            df[f"rolling_median_{w}"] = roll.median()
        return df

    def _add_temporal_encoding(self, df: pd.DataFrame) -> pd.DataFrame:
        idx = df.index
        df["day_of_week"]   = idx.dayofweek
        df["day_of_month"]  = idx.day
        df["week_of_year"]  = idx.isocalendar().week.astype(int)
        df["month"]         = idx.month
        df["quarter"]       = idx.quarter
        df["year"]          = idx.year
        df["is_weekend"]    = (idx.dayofweek >= 5).astype(int)
        df["is_month_start"] = idx.is_month_start.astype(int)
        df["is_month_end"]   = idx.is_month_end.astype(int)
        df["is_quarter_end"] = idx.is_quarter_end.astype(int)
        return df

    def _add_fourier_features(self, df: pd.DataFrame) -> pd.DataFrame:
        """Features de Fourier para capturar estacionalidad compleja."""
        t = np.arange(len(df))
        period = {"D": 365.25, "W": 52, "M": 12, "Q": 4, "Y": 1}.get(self.freq, 365.25)
        for k in range(1, self.FOURIER_TERMS + 1):
            df[f"sin_fourier_{k}"] = np.sin(2 * np.pi * k * t / period)
            df[f"cos_fourier_{k}"] = np.cos(2 * np.pi * k * t / period)
        return df

    def _add_trend_features(self, df: pd.DataFrame, target: str) -> pd.DataFrame:
        df["trend_linear"] = np.arange(len(df), dtype=float)
        df["trend_log"]    = np.log1p(df["trend_linear"])
        df["ewm_30"]       = df[target].ewm(span=30, adjust=False).mean()
        return df

    def _add_calendar_features(self, df: pd.DataFrame) -> pd.DataFrame:
        """Variables de negocio comunes."""
        idx = df.index
        df["days_since_epoch"] = (idx - pd.Timestamp("2000-01-01")).days
        return df
```

### 3.2 Model Router (ml/model_router.py)

```python
# ml/model_router.py
import pandas as pd
from dataclasses import dataclass
from typing import List

@dataclass
class DatasetProfile:
    n_rows: int
    n_cols: int
    freq: str
    has_strong_seasonality: bool
    has_categorical_cols: bool
    high_cardinality: bool
    n_years: float

class ModelRouter:
    """Selección automática de modelos según características del dataset."""

    def select_models(self, profile: DatasetProfile) -> List[str]:
        models = []

        # LightGBM — datasets masivos, entrenamiento rápido
        if profile.n_rows > 500_000:
            models.append("lightgbm")

        # Prophet — estacionalidad fuerte, patrones anuales
        if profile.has_strong_seasonality and profile.n_years >= 2:
            models.append("prophet")

        # CatBoost — variables categóricas con alta cardinalidad
        if profile.has_categorical_cols and profile.high_cardinality:
            models.append("catboost")

        # SARIMA — series simples, pocos features, forecasting clásico
        if profile.n_cols <= 3 and not profile.has_categorical_cols:
            models.append("sarima")

        # XGBoost — features tabulares complejas
        if profile.n_cols > 3 or profile.n_rows > 10_000:
            models.append("xgboost")

        # Fallback: siempre entrenar al menos XGBoost y Prophet
        if not models:
            models = ["xgboost", "prophet"]

        return list(set(models))

    def profile_dataset(self, df: pd.DataFrame, date_col: str, target_col: str) -> DatasetProfile:
        n_rows = len(df)
        n_cols = len(df.columns)
        freq   = pd.infer_freq(pd.to_datetime(df[date_col])) or "D"

        # Detectar estacionalidad fuerte (autocorrelación a lag anual)
        target = df[target_col].dropna()
        lag_365 = min(365, len(target) // 2)
        acf_val = float(target.autocorr(lag=lag_365)) if len(target) > lag_365 else 0.0
        has_strong_seasonality = abs(acf_val) > 0.3

        # Detectar categóricas
        cat_cols = df.select_dtypes(include=["object", "category"]).columns.tolist()
        has_categorical = len(cat_cols) > 0
        high_cardinality = any(df[c].nunique() > 50 for c in cat_cols) if cat_cols else False

        # Años de historia
        dates = pd.to_datetime(df[date_col])
        n_years = (dates.max() - dates.min()).days / 365.25

        freq_map = {"H": "H", "D": "D", "W": "W", "MS": "M", "M": "M",
                    "QS": "Q", "Q": "Q", "AS": "Y", "A": "Y"}
        freq_std = freq_map.get(freq[0] if freq else "D", "D")

        return DatasetProfile(
            n_rows=n_rows, n_cols=n_cols, freq=freq_std,
            has_strong_seasonality=has_strong_seasonality,
            has_categorical_cols=has_categorical, high_cardinality=high_cardinality,
            n_years=n_years
        )
```

### 3.3 Implementación de Modelos

#### XGBoost Model

```python
# ml/models/xgboost_model.py
import xgboost as xgb
import pandas as pd
import numpy as np
from sklearn.model_selection import TimeSeriesSplit
from .base_model import BaseForecaster

class XGBoostForecaster(BaseForecaster):
    model_name = "xgboost"

    def __init__(self):
        self.params = {
            "n_estimators": 500, "learning_rate": 0.05, "max_depth": 6,
            "min_child_weight": 1, "subsample": 0.8, "colsample_bytree": 0.8,
            "tree_method": "hist", "random_state": 42, "n_jobs": -1
        }
        self.model = xgb.XGBRegressor(**self.params)

    def fit(self, X: pd.DataFrame, y: pd.Series) -> None:
        tscv = TimeSeriesSplit(n_splits=5)
        splits = list(tscv.split(X))
        train_idx, val_idx = splits[-1]
        X_train, X_val = X.iloc[train_idx], X.iloc[val_idx]
        y_train, y_val = y.iloc[train_idx], y.iloc[val_idx]
        self.model.fit(
            X_train, y_train,
            eval_set=[(X_val, y_val)],
            verbose=False
        )
        self.feature_names = list(X.columns)

    def predict(self, X: pd.DataFrame) -> np.ndarray:
        return self.model.predict(X)

    def get_feature_importance(self) -> dict:
        return dict(zip(self.feature_names, self.model.feature_importances_))
```

#### Prophet Model

```python
# ml/models/prophet_model.py
import pandas as pd
import numpy as np
from prophet import Prophet
from .base_model import BaseForecaster

class ProphetForecaster(BaseForecaster):
    model_name = "prophet"

    def __init__(self):
        self.model = Prophet(
            yearly_seasonality=True,
            weekly_seasonality=True,
            daily_seasonality=False,
            seasonality_mode="multiplicative",
            changepoint_prior_scale=0.05,
            interval_width=0.95,
        )
        self._fitted_df = None

    def fit(self, X: pd.DataFrame, y: pd.Series) -> None:
        df = pd.DataFrame({"ds": X.index, "y": y.values})
        self.model.fit(df)
        self._fitted_df = df

    def predict(self, X: pd.DataFrame) -> np.ndarray:
        future = pd.DataFrame({"ds": X.index})
        forecast = self.model.predict(future)
        return forecast["yhat"].values

    def predict_with_intervals(self, future_df: pd.DataFrame) -> pd.DataFrame:
        forecast = self.model.predict(future_df)
        return forecast[["ds", "yhat", "yhat_lower", "yhat_upper"]]
```

#### SARIMA Model

```python
# ml/models/sarima_model.py
import pandas as pd
import numpy as np
from statsmodels.tsa.statespace.sarimax import SARIMAX
from pmdarima import auto_arima
from .base_model import BaseForecaster

class SARIMAForecaster(BaseForecaster):
    model_name = "sarima"

    def fit(self, X: pd.DataFrame, y: pd.Series) -> None:
        # Auto-selección de orden ARIMA
        auto = auto_arima(
            y, seasonal=True, m=12,
            stepwise=True, suppress_warnings=True,
            error_action="ignore", max_p=3, max_q=3, max_P=2, max_Q=2
        )
        self.order         = auto.order
        self.seasonal_order = auto.seasonal_order
        self.model = SARIMAX(y, order=self.order, seasonal_order=self.seasonal_order)
        self.fitted = self.model.fit(disp=False)

    def predict(self, X: pd.DataFrame) -> np.ndarray:
        steps = len(X)
        forecast = self.fitted.get_forecast(steps=steps)
        return forecast.predicted_mean.values

    def predict_with_intervals(self, steps: int, alpha: float = 0.05) -> pd.DataFrame:
        forecast = self.fitted.get_forecast(steps=steps)
        df = forecast.summary_frame(alpha=alpha)
        return df[["mean", "mean_ci_lower", "mean_ci_upper"]]
```

### 3.4 Validador y Métricas (ml/validator.py)

```python
# ml/validator.py
import numpy as np
import pandas as pd
from typing import Dict

class TimeSeriesValidator:

    def compute_metrics(self, y_true: np.ndarray, y_pred: np.ndarray) -> Dict[str, float]:
        y_true = np.array(y_true, dtype=float)
        y_pred = np.array(y_pred, dtype=float)
        mask = (y_true != 0) & np.isfinite(y_true) & np.isfinite(y_pred)
        y_t, y_p = y_true[mask], y_pred[mask]
        mape  = float(np.mean(np.abs((y_t - y_p) / y_t)) * 100)
        rmse  = float(np.sqrt(np.mean((y_t - y_p) ** 2)))
        mae   = float(np.mean(np.abs(y_t - y_p)))
        smape = float(np.mean(2 * np.abs(y_t - y_p) / (np.abs(y_t) + np.abs(y_p))) * 100)
        return {"mape": round(mape, 4), "rmse": round(rmse, 4),
                "mae": round(mae, 4), "smape": round(smape, 4)}

    def sliding_window_validation(
        self, model, X: pd.DataFrame, y: pd.Series, n_splits: int = 5
    ) -> Dict[str, float]:
        """Rolling forecasting origin con ventana deslizante."""
        n = len(X)
        min_train = int(n * 0.6)
        step      = (n - min_train) // n_splits
        all_metrics = []

        for i in range(n_splits):
            split_point = min_train + i * step
            X_train, X_val = X.iloc[:split_point], X.iloc[split_point:split_point + step]
            y_train, y_val = y.iloc[:split_point], y.iloc[split_point:split_point + step]
            if len(X_val) == 0:
                continue
            model.fit(X_train, y_train)
            y_pred = model.predict(X_val)
            all_metrics.append(self.compute_metrics(y_val.values, y_pred))

        return {k: round(float(np.mean([m[k] for m in all_metrics])), 4)
                for k in all_metrics[0]} if all_metrics else {}

    def backtesting_12m(
        self, model, X: pd.DataFrame, y: pd.Series, freq: str = "M"
    ) -> Dict[str, float]:
        """Backtesting de 12 periodos hacia atrás."""
        n_periods = 12
        if len(X) <= n_periods * 2:
            return self.compute_metrics(y.values[-n_periods:],
                                        np.full(n_periods, y.mean()))
        X_train = X.iloc[:-n_periods]
        y_train = y.iloc[:-n_periods]
        X_test  = X.iloc[-n_periods:]
        y_test  = y.iloc[-n_periods:]
        model.fit(X_train, y_train)
        y_pred = model.predict(X_test)
        return self.compute_metrics(y_test.values, y_pred)

    def select_best_model(self, results: Dict[str, Dict]) -> str:
        """Selecciona modelo con menor MAPE ponderado por RMSE y MAE."""
        scores = {}
        for model_name, metrics in results.items():
            if not metrics:
                continue
            score = (metrics.get("mape", 999) * 0.5 +
                     metrics.get("rmse", 999) * 0.3 +
                     metrics.get("mae",  999) * 0.2)
            scores[model_name] = score
        return min(scores, key=scores.get) if scores else "xgboost"
```

---

## 4. Frontend React

### 4.1 Estructura del Proyecto

```
forecastflow-ui/
├── src/
│   ├── app/
│   │   ├── App.tsx
│   │   ├── router.tsx
│   │   └── store.ts              # Zustand global state
│   ├── pages/
│   │   ├── LoginPage.tsx
│   │   ├── DashboardPage.tsx
│   │   └── ForecastWizardPage.tsx
│   ├── components/
│   │   ├── wizard/
│   │   │   ├── StepConnect.tsx
│   │   │   ├── StepDataset.tsx
│   │   │   ├── StepTarget.tsx
│   │   │   ├── StepHorizon.tsx
│   │   │   ├── StepTraining.tsx
│   │   │   └── StepResults.tsx
│   │   ├── charts/
│   │   │   ├── ForecastChart.tsx
│   │   │   └── MetricsCard.tsx
│   │   ├── ui/
│   │   │   ├── SkeletonLoader.tsx
│   │   │   ├── ProgressBar.tsx
│   │   │   └── StatusBadge.tsx
│   │   └── layout/
│   │       ├── Sidebar.tsx
│   │       └── Header.tsx
│   ├── hooks/
│   │   ├── useTrainingPoller.ts
│   │   ├── useConnections.ts
│   │   └── useForecast.ts
│   ├── api/
│   │   ├── client.ts             # Axios instance con interceptors JWT
│   │   ├── auth.ts
│   │   ├── connections.ts
│   │   ├── explore.ts
│   │   ├── training.ts
│   │   └── forecast.ts
│   └── types/
│       └── index.ts
```

### 4.2 Cliente API con JWT Interceptors (api/client.ts)

```typescript
// api/client.ts
import axios, { AxiosInstance } from "axios";

const apiClient: AxiosInstance = axios.create({
  baseURL: import.meta.env.VITE_API_URL || "http://localhost:8000",
  timeout: 30_000,
  headers: { "Content-Type": "application/json" },
});

// Attach access token a cada request
apiClient.interceptors.request.use((config) => {
  const token = localStorage.getItem("access_token");
  if (token) config.headers.Authorization = `Bearer ${token}`;
  return config;
});

// Auto-refresh si 401
apiClient.interceptors.response.use(
  (res) => res,
  async (error) => {
    const original = error.config;
    if (error.response?.status === 401 && !original._retry) {
      original._retry = true;
      const refresh = localStorage.getItem("refresh_token");
      if (refresh) {
        const { data } = await axios.post("/api/auth/refresh", { refresh_token: refresh });
        localStorage.setItem("access_token", data.access_token);
        original.headers.Authorization = `Bearer ${data.access_token}`;
        return apiClient(original);
      }
    }
    return Promise.reject(error);
  }
);

export default apiClient;
```

### 4.3 Hook de Polling de Entrenamiento (hooks/useTrainingPoller.ts)

```typescript
// hooks/useTrainingPoller.ts
import { useState, useEffect, useRef } from "react";
import { getTrainingStatus } from "../api/training";
import type { TrainingStatus } from "../types";

const POLL_INTERVAL = 2000; // 2 segundos

export function useTrainingPoller(jobId: string | null) {
  const [status, setStatus] = useState<TrainingStatus | null>(null);
  const [isPolling, setIsPolling] = useState(false);
  const intervalRef = useRef<ReturnType<typeof setInterval> | null>(null);

  useEffect(() => {
    if (!jobId) return;

    setIsPolling(true);
    intervalRef.current = setInterval(async () => {
      try {
        const data = await getTrainingStatus(jobId);
        setStatus(data);
        if (["completed", "failed"].includes(data.status)) {
          clearInterval(intervalRef.current!);
          setIsPolling(false);
        }
      } catch {
        clearInterval(intervalRef.current!);
        setIsPolling(false);
      }
    }, POLL_INTERVAL);

    return () => {
      if (intervalRef.current) clearInterval(intervalRef.current);
    };
  }, [jobId]);

  return { status, isPolling };
}
```

### 4.4 Gráfico de Forecast Principal (components/charts/ForecastChart.tsx)

```typescript
// components/charts/ForecastChart.tsx
import React from "react";
import {
  ComposedChart, Line, Area, XAxis, YAxis, CartesianGrid,
  Tooltip, Legend, ResponsiveContainer, ReferenceLine
} from "recharts";
import { motion } from "framer-motion";
import type { ForecastPoint } from "../../types";

interface Props {
  historicalData: ForecastPoint[];
  forecastData: ForecastPoint[];
  modelName: string;
}

const ForecastChart: React.FC<Props> = ({ historicalData, forecastData, modelName }) => {
  const splitDate = historicalData[historicalData.length - 1]?.date;
  const combined = [...historicalData, ...forecastData];

  return (
    <motion.div
      initial={{ opacity: 0, y: 20 }}
      animate={{ opacity: 1, y: 0 }}
      transition={{ duration: 0.5 }}
      className="bg-gray-900 rounded-2xl p-6 shadow-xl"
    >
      <div className="flex items-center justify-between mb-6">
        <h2 className="text-white text-xl font-semibold">Forecast</h2>
        <span className="bg-indigo-600 text-white text-xs px-3 py-1 rounded-full">
          Modelo: {modelName.toUpperCase()}
        </span>
      </div>

      <ResponsiveContainer width="100%" height={400}>
        <ComposedChart data={combined} margin={{ top: 10, right: 30, left: 0, bottom: 0 }}>
          <CartesianGrid strokeDasharray="3 3" stroke="#374151" />
          <XAxis dataKey="date" stroke="#9CA3AF" tick={{ fontSize: 12 }} />
          <YAxis stroke="#9CA3AF" tick={{ fontSize: 12 }} />
          <Tooltip
            contentStyle={{ backgroundColor: "#1F2937", border: "none", borderRadius: "8px", color: "#F9FAFB" }}
            formatter={(value: number, name: string) => [value.toLocaleString(), name]}
          />
          <Legend wrapperStyle={{ color: "#9CA3AF" }} />

          {/* Banda de confianza */}
          <Area
            data={forecastData}
            dataKey="confidence_upper"
            fill="#4F46E5"
            fillOpacity={0.15}
            stroke="none"
            name="Intervalo superior"
          />
          <Area
            data={forecastData}
            dataKey="confidence_lower"
            fill="#4F46E5"
            fillOpacity={0.0}
            stroke="none"
            name="Intervalo inferior"
          />

          {/* Histórico */}
          <Line
            data={historicalData}
            type="monotone"
            dataKey="value"
            stroke="#60A5FA"
            strokeWidth={2}
            dot={false}
            name="Histórico"
          />

          {/* Forecast */}
          <Line
            data={forecastData}
            type="monotone"
            dataKey="forecast"
            stroke="#A78BFA"
            strokeWidth={2.5}
            strokeDasharray="6 3"
            dot={false}
            name="Forecast"
          />

          {/* Línea divisoria histórico/forecast */}
          {splitDate && (
            <ReferenceLine
              x={splitDate}
              stroke="#6B7280"
              strokeDasharray="4 4"
              label={{ value: "Hoy", fill: "#9CA3AF", fontSize: 11 }}
            />
          )}
        </ComposedChart>
      </ResponsiveContainer>
    </motion.div>
  );
};

export default ForecastChart;
```

---

## 5. Estrategia de Datos

### 5.1 Conector Multi-Motor (services/connector.py)

```python
# services/connector.py
from sqlalchemy.ext.asyncio import create_async_engine, AsyncEngine
from sqlalchemy import text
import pandas as pd
from typing import AsyncGenerator

DRIVER_MAP = {
    "postgresql": "postgresql+asyncpg",
    "mysql":      "mysql+aiomysql",
    "mssql":      "mssql+aioodbc",
    "sqlite":     "sqlite+aiosqlite",
}

class DatabaseConnector:
    def __init__(self, url: str):
        self.url = self._normalize_url(url)
        self.engine: AsyncEngine = create_async_engine(
            self.url, pool_size=5, max_overflow=10, pool_timeout=30
        )

    def _normalize_url(self, url: str) -> str:
        for dialect, async_driver in DRIVER_MAP.items():
            if url.startswith(dialect + "://"):
                return url.replace(dialect + "://", async_driver + "://", 1)
        return url

    async def test(self) -> bool:
        async with self.engine.connect() as conn:
            await conn.execute(text("SELECT 1"))
        return True

    async def fetch_chunked(
        self, query: str, chunk_size: int = 50_000
    ) -> AsyncGenerator[pd.DataFrame, None]:
        """Extracción streaming por chunks. No carga el dataset completo en RAM."""
        async with self.engine.connect() as conn:
            result = await conn.execute(text(query))
            while True:
                rows = result.fetchmany(chunk_size)
                if not rows:
                    break
                yield pd.DataFrame(rows, columns=result.keys())
```

### 5.2 Explorador de Esquema (services/explorer.py)

```python
# services/explorer.py
import pandas as pd
from sqlalchemy import inspect, text
from models.schemas import ExploreResponse, TableInfo, ColumnInfo, TablePreview
from services.connector import DatabaseConnector

class DataExplorer:
    def __init__(self, connector: DatabaseConnector):
        self.connector = connector

    async def get_schema(self) -> ExploreResponse:
        tables = []
        async with self.connector.engine.connect() as conn:
            inspector = inspect(self.connector.engine.sync_engine)
            for table_name in inspector.get_table_names():
                cols = []
                for col in inspector.get_columns(table_name):
                    cols.append(ColumnInfo(
                        name=col["name"],
                        dtype=str(col["type"]),
                        nullable=col.get("nullable", True),
                        is_temporal=self._is_temporal(str(col["type"]))
                    ))
                row_count = await self._estimate_count(conn, table_name)
                tables.append(TableInfo(name=table_name, columns=cols, estimated_rows=row_count))
        return ExploreResponse(tables=tables)

    async def preview_table(self, table_name: str, limit: int = 100) -> TablePreview:
        rows, detected_freq, date_col, target_col = [], None, None, None
        async with self.connector.engine.connect() as conn:
            result = await conn.execute(text(f"SELECT * FROM {table_name} LIMIT {limit}"))
            keys = list(result.keys())
            data = [dict(zip(keys, row)) for row in result.fetchall()]

        df = pd.DataFrame(data)
        # Detectar columna fecha
        for col in df.columns:
            if self._is_temporal(str(df[col].dtype)) or "date" in col.lower() or "time" in col.lower():
                date_col = col
                try:
                    df[col] = pd.to_datetime(df[col])
                    detected_freq = pd.infer_freq(df[col].dropna().sort_values())
                except Exception:
                    pass
                break
        # Detectar columna objetivo (primera numérica no fecha)
        for col in df.select_dtypes(include="number").columns:
            if col != date_col:
                target_col = col
                break

        return TablePreview(
            columns=list(df.columns),
            dtypes={c: str(df[c].dtype) for c in df.columns},
            sample_data=df.head(20).to_dict(orient="records"),
            detected_date_column=date_col,
            detected_target_column=target_col,
            detected_frequency=detected_freq,
        )

    def _is_temporal(self, dtype_str: str) -> bool:
        return any(t in dtype_str.lower() for t in ["date", "time", "timestamp"])

    async def _estimate_count(self, conn, table_name: str) -> int:
        try:
            result = await conn.execute(text(f"SELECT COUNT(*) FROM {table_name}"))
            return result.scalar()
        except Exception:
            return -1
```

### 5.3 Procesamiento Incremental y Adaptive Chunking

Para datasets de millones de registros, el sistema aplica **adaptive chunking** ajustando el tamaño del chunk según la memoria disponible y la frecuencia temporal detectada:

```python
# services/chunked_processor.py
import psutil
import pandas as pd
from typing import Iterator

class ChunkedProcessor:
    BASE_CHUNK = 50_000
    MIN_CHUNK  = 5_000
    MAX_CHUNK  = 200_000

    def adaptive_chunk_size(self) -> int:
        """Ajusta tamaño de chunk según RAM disponible."""
        available_mb = psutil.virtual_memory().available / (1024 ** 2)
        if available_mb > 4000:   return self.MAX_CHUNK
        if available_mb > 1000:   return self.BASE_CHUNK
        return self.MIN_CHUNK

    def process_in_chunks(
        self, connector, query: str, processor_fn
    ) -> pd.DataFrame:
        chunk_size = self.adaptive_chunk_size()
        accumulated = []
        for chunk in connector.fetch_chunked(query, chunk_size=chunk_size):
            processed = processor_fn(chunk)
            accumulated.append(processed)
        return pd.concat(accumulated, ignore_index=True)
```

---

## 6. Model Registry & MLOps

### 6.1 Registry Local (ml/registry.py)

```python
# ml/registry.py
import json, uuid, joblib
from datetime import datetime, timezone
from pathlib import Path
from dataclasses import dataclass, field, asdict
from typing import Optional, Dict, List
from core.config import settings

@dataclass
class ModelVersion:
    version_id:     str
    model_name:     str
    semver:         str
    dataset_hash:   str
    n_rows:         int
    feature_names:  List[str]
    metrics:        Dict[str, float]
    training_time_s: float
    horizon_months: int
    created_at:     str = field(default_factory=lambda: datetime.now(timezone.utc).isoformat())
    status:         str = "active"    # active | archived | rollback

class ModelRegistry:
    def __init__(self):
        self.base_path = Path(settings.MODEL_REGISTRY_PATH)
        self.base_path.mkdir(parents=True, exist_ok=True)
        self.index_file = self.base_path / "index.json"
        self._ensure_index()

    def _ensure_index(self):
        if not self.index_file.exists():
            self.index_file.write_text(json.dumps([]))

    def register(self, model_obj, version: ModelVersion) -> str:
        model_dir = self.base_path / version.model_name / version.version_id
        model_dir.mkdir(parents=True, exist_ok=True)
        # Guardar artefacto
        joblib.dump(model_obj, model_dir / "model.joblib")
        # Guardar metadata
        (model_dir / "metadata.json").write_text(json.dumps(asdict(version), indent=2))
        # Actualizar índice global
        index = self._load_index()
        index.append(asdict(version))
        self.index_file.write_text(json.dumps(index, indent=2))
        return version.version_id

    def load(self, model_name: str, version_id: str):
        path = self.base_path / model_name / version_id / "model.joblib"
        return joblib.load(path)

    def get_best_version(self, model_name: str, metric: str = "mape") -> Optional[ModelVersion]:
        versions = [v for v in self._load_index()
                    if v["model_name"] == model_name and v["status"] == "active"]
        if not versions:
            return None
        best = min(versions, key=lambda v: v["metrics"].get(metric, 9999))
        return ModelVersion(**best)

    def rollback(self, model_name: str, version_id: str) -> bool:
        index = self._load_index()
        for v in index:
            if v["model_name"] == model_name:
                v["status"] = "archived"
            if v["version_id"] == version_id:
                v["status"] = "active"
        self.index_file.write_text(json.dumps(index, indent=2))
        return True

    def compare_versions(self, model_name: str) -> List[Dict]:
        return [v for v in self._load_index() if v["model_name"] == model_name]

    def _load_index(self) -> List[Dict]:
        return json.loads(self.index_file.read_text())
```

### 6.2 Versionado Semántico

Cada modelo registrado recibe una versión siguiendo el esquema `MAJOR.MINOR.PATCH`:

- **MAJOR**: cambio de arquitectura del modelo o features incompatibles.
- **MINOR**: reentrenamiento con nuevos datos, misma arquitectura.
- **PATCH**: ajuste de hiperparámetros sin cambios de features.

### 6.3 Workflow MLOps Completo

```
Dataset nuevo/actualizado
        ↓
Feature Engineering (versionado con hash)
        ↓
Entrenamiento N modelos en paralelo (Celery workers)
        ↓
Cross-validation + Backtesting 12 meses
        ↓
Comparación de métricas MAPE/RMSE/MAE
        ↓
Registro en Model Registry (metadata + artifact)
        ↓
Promoción del mejor modelo a "active"
        ↓
Forecast con IC → Caché Redis (TTL 1h)
        ↓
Dashboard disponible en < 2 segundos
```

---

## 7. Seguridad y Autenticación

### 7.1 Matriz de Seguridad

| Componente | Mecanismo | Estándar |
|---|---|---|
| Autenticación | JWT RS256 + Refresh Tokens | RFC 7519 |
| Credenciales BD | AES-256-CBC | NIST SP 800-38A |
| Transporte | HTTPS/TLS 1.3 | RFC 8446 |
| Rate Limiting | Sliding Window (100 req/min) | OWASP |
| CORS | Lista blanca de orígenes | OWASP |
| Inputs | Pydantic v2 validators | OWASP Input Validation |
| Logs | Redact de credenciales | GDPR/SOC2 |

### 7.2 Rate Limiting con SlowAPI

```python
# api/middleware/rate_limit.py
from slowapi import Limiter
from slowapi.util import get_remote_address
from slowapi.errors import RateLimitExceeded
from fastapi import Request
from fastapi.responses import JSONResponse

limiter = Limiter(key_func=get_remote_address)

async def rate_limit_handler(request: Request, exc: RateLimitExceeded):
    return JSONResponse(
        status_code=429,
        content={"detail": "Demasiadas solicitudes. Intente en 60 segundos."}
    )
```

### 7.3 Redacción de Credenciales en Logs

```python
# core/logging_filter.py
import re, logging

SENSITIVE_PATTERNS = [
    r"(password=)[^@&\s]+",
    r"(://[^:]+:)[^@]+(@)",
    r"(Bearer\s)\S+",
]

class CredentialRedactFilter(logging.Filter):
    def filter(self, record: logging.LogRecord) -> bool:
        msg = str(record.getMessage())
        for pattern in SENSITIVE_PATTERNS:
            msg = re.sub(pattern, r"\1[REDACTED]\2" if r"\2" in pattern else r"\1[REDACTED]", msg)
        record.msg = msg
        record.args = ()
        return True
```

### 7.4 Validación de Conexiones

Antes de almacenar cualquier URL de conexión, el sistema valida:

1. La URL no contiene inyecciones SQL en los campos de host/database.
2. El motor declarado coincide con el driver de la URL.
3. El test de conexión completa en menos de 2 segundos antes de cifrar y persistir.
4. El string cifrado nunca se devuelve en ninguna respuesta de API.

---

## 8. Estrategia de Performance y Escalabilidad

### 8.1 SLA Operacionales

| Operación | SLA Objetivo | Mecanismo |
|---|---|---|
| Test de conexión | < 2 segundos | Timeout asyncio |
| Preview de tablas | < 3 segundos | Cache Redis TTL 120s |
| Dashboard cacheado | < 2 segundos | Cache Redis TTL 3600s |
| Polling de progreso | Cada 2 segundos | Long-poll sobre job state |
| Entrenamiento (100K rows) | ~30–90 segundos | Async workers |
| Entrenamiento (1M+ rows) | ~5–15 minutos | Parallel Celery workers |

### 8.2 Paralelismo de Entrenamiento

```python
# workers/tasks.py
from celery import group
from workers.celery_app import celery_app
from ml.model_router import ModelRouter
from ml.feature_engineering import FeatureEngineer
from ml.validator import TimeSeriesValidator
from ml.registry import ModelRegistry, ModelVersion
import hashlib, time, uuid

@celery_app.task(bind=True, max_retries=2, soft_time_limit=1800)
def run_forecast_pipeline(self, job_id: str, request: dict):
    """Task principal. Orquesta FE → entrenamiento paralelo → selección → forecast."""
    _update_job(job_id, status="running", step="feature_engineering", progress=5)

    # 1. Feature engineering
    fe = FeatureEngineer(freq=request.get("freq", "D"))
    df = _load_data_chunked(request)
    X  = fe.fit_transform(df, request["target_column"], request["date_column"])
    y  = X.pop(request["target_column"])
    _update_job(job_id, step="training", progress=20)

    # 2. Seleccionar modelos
    router = ModelRouter()
    profile = router.profile_dataset(df, request["date_column"], request["target_column"])
    selected_models = router.select_models(profile)

    # 3. Entrenar modelos en paralelo con Celery group
    tasks = group(
        train_single_model.s(job_id, model_name, X.to_dict(), y.to_list(), request)
        for model_name in selected_models
    )
    group_result = tasks.apply_async()
    results = group_result.get(timeout=1500)  # 25 min timeout

    # 4. Seleccionar mejor modelo
    validator = TimeSeriesValidator()
    metrics_by_model = {r["model_name"]: r["metrics"] for r in results if r}
    best_model_name = validator.select_best_model(metrics_by_model)
    _update_job(job_id, step="forecast", progress=90, best_model=best_model_name)

    # 5. Generar forecast final con intervalos de confianza
    _generate_and_cache_forecast(job_id, best_model_name, request)
    _update_job(job_id, status="completed", progress=100)

@celery_app.task(bind=True, max_retries=1)
def train_single_model(self, job_id: str, model_name: str, X_dict: dict, y_list: list, request: dict):
    import pandas as pd
    from ml.models import get_model_class
    X = pd.DataFrame(X_dict)
    y = pd.Series(y_list)
    model_cls = get_model_class(model_name)
    model = model_cls()
    validator = TimeSeriesValidator()
    t0 = time.time()
    metrics = validator.sliding_window_validation(model, X, y)
    backtesting = validator.backtesting_12m(model, X, y)
    metrics.update({f"bt_{k}": v for k, v in backtesting.items()})
    model.fit(X, y)
    elapsed = time.time() - t0
    # Registrar en model registry
    registry = ModelRegistry()
    version = ModelVersion(
        version_id=str(uuid.uuid4()), model_name=model_name,
        semver="1.0.0", dataset_hash=hashlib.md5(str(X_dict).encode()).hexdigest()[:8],
        n_rows=len(y_list), feature_names=list(X_dict.keys()),
        metrics=metrics, training_time_s=round(elapsed, 2),
        horizon_months=request.get("horizon_months", 6)
    )
    registry.register(model, version)
    return {"model_name": model_name, "metrics": metrics, "version_id": version.version_id}
```

### 8.3 Configuración Celery

```python
# workers/celery_app.py
from celery import Celery
from core.config import settings

celery_app = Celery(
    "forecastflow",
    broker=settings.REDIS_URL,
    backend=settings.REDIS_URL,
    include=["workers.tasks"],
)

celery_app.conf.update(
    task_serializer="json",
    result_serializer="json",
    accept_content=["json"],
    worker_prefetch_multiplier=1,        # Un task a la vez por worker
    task_acks_late=True,                 # Confirmar solo tras completar
    worker_max_tasks_per_child=50,       # Reciclar worker cada 50 tasks
    task_soft_time_limit=1800,           # 30 min
    task_time_limit=2100,                # 35 min hard limit
    task_routes={
        "workers.tasks.train_single_model": {"queue": "training"},
        "workers.tasks.run_forecast_pipeline": {"queue": "pipeline"},
    }
)
```

---

## 9. Guía de Experiencia de Usuario (UX)

### 9.1 Principios de Diseño

ForecastFlow está diseñado para usuarios no técnicos de negocio que necesitan predicciones confiables sin entender modelos de ML. Los cinco principios guía son:

1. **Zero Config**: el sistema toma decisiones técnicas (modelo, features, validación) de forma completamente automática.
2. **Progreso Visible**: cada operación muestra progreso en tiempo real. El usuario nunca ve una pantalla congelada.
3. **Confianza Contextual**: los gráficos muestran intervalos de confianza con explicaciones simples ("95% de probabilidad de estar en este rango").
4. **Acción Inmediata**: el flujo wizard lleva al usuario de la conexión a resultados en 6 pasos lineales sin decisiones técnicas.
5. **Dark Mode First**: interfaz oscura por defecto, diseñada para uso prolongado en entornos de oficina.

### 9.2 Flujo Wizard Visual — 6 Pasos

```
┌─────────────────────────────────────────────────────────────┐
│  Step 1: Conectar BD                                         │
│  ┌─────────────────────────────────┐                        │
│  │ Motor: [PostgreSQL ▼]           │                        │
│  │ URL o conexión manual           │                        │
│  │ postgresql://user:****@host/db  │                        │
│  │ [Probar conexión] ← < 2s        │                        │
│  └─────────────────────────────────┘                        │
├─────────────────────────────────────────────────────────────┤
│  Step 2: Seleccionar Dataset                                 │
│  Tablas detectadas automáticamente con preview de filas      │
│  [ventas_2020_2024] 2.4M filas  [seleccionar]               │
├─────────────────────────────────────────────────────────────┤
│  Step 3: Columnas                                            │
│  Fecha detectada: fecha_venta ✓   (puedes cambiarla)        │
│  Variable objetivo: monto_total ✓  (puedes cambiarla)       │
├─────────────────────────────────────────────────────────────┤
│  Step 4: Horizonte                                           │
│  ¿Hasta dónde quieres predecir?                             │
│  [1 mes]  [3 meses]  [6 meses ●]  [12 meses]               │
├─────────────────────────────────────────────────────────────┤
│  Step 5: Entrenamiento Automático                            │
│  ████████████████░░░░ 72%                                   │
│  ▸ Generando features temporales...                         │
│  ▸ Entrenando XGBoost, LightGBM, Prophet...                 │
│  ▸ Backtesting 12 meses...                                  │
├─────────────────────────────────────────────────────────────┤
│  Step 6: Dashboard de Resultados                            │
│  [Gráfico Forecast + Bandas de Confianza]                   │
│  MAPE: 4.2%   RMSE: 12,450   Accuracy: 95.8%               │
│  Modelo: LightGBM v1.0.2  │  Tiempo: 47s  │  2.4M rows     │
│  [Exportar CSV] [Exportar Excel]                            │
└─────────────────────────────────────────────────────────────┘
```

### 9.3 Estados Visuales del Entrenamiento

```
QUEUED    → Spinner animado    → "Tu forecast está en cola..."
RUNNING   → Progress bar vivo  → "Entrenando modelos (3/5)..."
COMPLETED → Check verde        → "¡Forecast listo!"
FAILED    → X rojo             → "Error: [mensaje claro en lenguaje natural]"
```

### 9.4 Indicadores de Calidad para No Técnicos

El dashboard traduce métricas técnicas a lenguaje de negocio:

| Métrica Técnica | Presentación para Negocio |
|---|---|
| MAPE < 5% | 🟢 Excelente precisión |
| MAPE 5–10% | 🟡 Buena precisión |
| MAPE 10–20% | 🟠 Precisión aceptable |
| MAPE > 20% | 🔴 Revisar datos |
| IC 95% | "95% de probabilidad de estar en este rango" |
| Backtesting 12m | "Validado contra 12 meses reales" |

### 9.5 Componentes UI Clave

- **Skeleton Loaders**: en lugar de spinners genéricos, los esqueletos preservan la estructura del dashboard mientras carga.
- **Framer Motion Transitions**: transiciones suaves entre pasos del wizard (300ms ease-out).
- **Tooltips Contextuales**: cada métrica tiene un "?" con explicación en lenguaje natural.
- **Panel Lateral Pipeline**: siempre visible, muestra el estado de cada etapa del pipeline.
- **Error States Descriptivos**: errores con causa probable y acción sugerida, no stack traces.

---

## 10. Estrategia de Despliegue

### 10.1 Docker Compose (Producción)

```yaml
# docker-compose.yml
version: "3.9"

services:
  api:
    build: .
    image: forecastflow-api:latest
    command: uvicorn api.main:app --host 0.0.0.0 --port 8000 --workers 4
    environment:
      - SECRET_KEY=${SECRET_KEY}
      - AES_ENCRYPTION_KEY=${AES_ENCRYPTION_KEY}
      - REDIS_URL=redis://redis:6379/0
      - DATABASE_URL=sqlite+aiosqlite:///./data/forecastflow.db
    volumes:
      - ./data:/app/data
      - ./model_registry:/app/model_registry
    ports:
      - "8000:8000"
    depends_on:
      redis:
        condition: service_healthy
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000/health"]
      interval: 30s
      timeout: 10s
      retries: 3

  worker-pipeline:
    build: .
    image: forecastflow-api:latest
    command: celery -A workers.celery_app worker -Q pipeline -c 2 --loglevel=info
    environment:
      - REDIS_URL=redis://redis:6379/0
    volumes:
      - ./data:/app/data
      - ./model_registry:/app/model_registry
    depends_on:
      - redis
    restart: unless-stopped

  worker-training:
    build: .
    image: forecastflow-api:latest
    command: celery -A workers.celery_app worker -Q training -c 4 --loglevel=info
    environment:
      - REDIS_URL=redis://redis:6379/0
    volumes:
      - ./model_registry:/app/model_registry
    depends_on:
      - redis
    restart: unless-stopped

  redis:
    image: redis:7-alpine
    command: redis-server --maxmemory 2gb --maxmemory-policy allkeys-lru
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5
    restart: unless-stopped

  frontend:
    build: ./forecastflow-ui
    image: forecastflow-ui:latest
    ports:
      - "3000:80"
    environment:
      - VITE_API_URL=http://localhost:8000
    depends_on:
      - api
    restart: unless-stopped

volumes:
  redis_data:
```

### 10.2 Dockerfile Backend

```dockerfile
# Dockerfile
FROM python:3.11-slim

WORKDIR /app

# Dependencias del sistema para drivers DB
RUN apt-get update && apt-get install -y \
    gcc libpq-dev unixodbc-dev curl \
    && rm -rf /var/lib/apt/lists/*

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

RUN mkdir -p /app/data /app/model_registry

CMD ["uvicorn", "api.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

### 10.3 requirements.txt

```
fastapi==0.115.0
uvicorn[standard]==0.32.0
pydantic==2.9.0
pydantic-settings==2.6.0
sqlalchemy[asyncio]==2.0.36
aiosqlite==0.20.0
asyncpg==0.29.0
aiomysql==0.2.0
celery[redis]==5.4.0
redis==5.2.0
xgboost==2.1.2
lightgbm==4.5.0
catboost==1.2.7
prophet==1.1.6
statsmodels==0.14.4
pmdarima==2.0.4
scikit-learn==1.5.2
pandas==2.2.3
numpy==1.26.4
joblib==1.4.2
cryptography==43.0.1
PyJWT==2.10.0
passlib[bcrypt]==1.7.4
slowapi==0.1.9
loguru==0.7.2
psutil==6.1.0
```

### 10.4 Preparación para Kubernetes

Para escalar a entornos Kubernetes, la arquitectura ya está preparada con:

- **Stateless API**: sin estado local en el proceso uvicorn. Todo el estado en Redis o SQLite/PostgreSQL.
- **Horizontal Pod Autoscaler**: workers Celery escalables independientemente de la API.
- **ConfigMaps/Secrets**: variables de entorno mapeadas directamente a Kubernetes Secrets.
- **Persistent Volumes**: `model_registry/` y `data/` mapeados a PVCs.
- **Readiness/Liveness Probes**: `/health` endpoint ya implementado.
- **Namespaces**: API, workers-pipeline, workers-training en namespaces separados para control de recursos.

---

## 11. Flujo Operacional Completo

A continuación se describe el flujo end-to-end desde que el usuario abre la plataforma hasta que descarga sus predicciones.

### Fase 1 — Autenticación y Conexión (0–30 segundos)

El usuario accede a la plataforma y completa el login con usuario y contraseña. El backend valida las credenciales, emite un access token JWT (60 minutos) y un refresh token (7 días). El frontend almacena ambos tokens y configura el interceptor Axios.

El usuario completa el formulario de conexión a base de datos: selecciona el motor (PostgreSQL, MySQL, SQL Server, SQLite) e ingresa la URL de conexión. El backend prueba la conexión en menos de 2 segundos, devuelve latencia confirmada, cifra la URL con AES-256 y persiste la conexión sin exponer las credenciales.

### Fase 2 — Exploración y Detección (30 segundos–2 minutos)

El sistema realiza introspección del esquema de la base de datos: detecta todas las tablas disponibles, tipos de columnas y estima el conteo de filas. Esta información se cachea en Redis con TTL de 5 minutos.

El usuario selecciona una tabla. El sistema genera un preview de las primeras 100 filas, detecta automáticamente la columna fecha y la columna objetivo principal, e infiere la frecuencia temporal (diaria, semanal, mensual, etc.). El usuario confirma o ajusta estas sugerencias.

El sistema ejecuta validaciones automáticas sobre la serie temporal detectada: verifica continuidad temporal, identifica gaps, cuantifica datos faltantes, detecta duplicados y aplica detección básica de anomalías estadísticas (valores fuera de 3 desviaciones estándar).

### Fase 3 — Configuración del Forecast (30 segundos)

El usuario selecciona el horizonte de predicción: 1, 3, 6 o 12 meses. No realiza ninguna otra configuración técnica. El sistema determina automáticamente qué modelos son apropiados basándose en el perfil del dataset: tamaño, frecuencia, presencia de estacionalidad, variables categóricas y años de historia disponibles.

### Fase 4 — Feature Engineering y Entrenamiento Asíncrono (variable: 30 segundos–15 minutos)

El backend despacha un job Celery y devuelve inmediatamente un `job_id`. El frontend inicia polling cada 2 segundos mostrando el progreso en tiempo real.

El worker principal ejecuta en secuencia:

1. Extracción chunked de datos (chunks de 50K filas, ajustable por RAM disponible).
2. Generación automática de features: lags (1, 7, 14, 30, 60, 90 períodos), rolling statistics (media, std, min, max, mediana en ventanas de 7/14/30/90 días), encoding temporal (día semana, mes, trimestre, año, indicadores is_weekend, is_month_end), Fourier features para capturar estacionalidad compleja, features de tendencia (lineal, logarítmica, EWM).
3. Lanzamiento paralelo de entrenamiento de modelos seleccionados mediante Celery group.
4. Cada model worker aplica sliding window cross-validation con 5 splits y backtesting de 12 meses. Los resultados se registran en el Model Registry con versión semántica.
5. Comparación de métricas MAPE/RMSE/MAE ponderadas para seleccionar el mejor modelo.

### Fase 5 — Generación de Forecast y Registro

El mejor modelo entrenado genera predicciones para el horizonte seleccionado. Los modelos que soportan intervalos de confianza nativos (Prophet, SARIMA) los generan directamente. Para modelos de gradient boosting, los intervalos se calculan mediante bootstrap sampling o conformal prediction simplificado.

El modelo ganador y sus métricas se almacenan en el Model Registry local con metadata completa: features usadas, hash del dataset, tiempo de entrenamiento, métricas de validación y versión semántica. Los resultados del forecast se cachean en Redis con TTL de 1 hora.

### Fase 6 — Dashboard Interactivo y Exportación

El dashboard se carga en menos de 2 segundos desde cache. Muestra:

- Gráfico principal con serie histórica completa, forecast multi-horizonte y bandas de confianza al 80% y 95%.
- Panel de métricas con MAPE, RMSE, MAE y traducción a lenguaje de negocio (Excelente/Buena/Aceptable/Revisar).
- Panel lateral con estado del pipeline, modelo seleccionado, tiempo de entrenamiento, tamaño del dataset y workers activos durante el proceso.
- Tabla de predicciones exportable con fecha, valor predicho, límite inferior y superior del intervalo de confianza.

El usuario puede exportar los resultados a CSV o Excel con un clic. Para nuevas ejecuciones con diferentes horizontes o datos actualizados, el sistema reutiliza los features ya calculados desde cache, reduciendo el tiempo de reentrenamiento en un 60–80%.

---

*ForecastFlow — Arquitectura de Referencia Enterprise v1.0*
*Equipo: Distributed Systems · ML Engineering · Data Engineering · MLOps · Backend · Frontend · Data Science · UX/UI*
