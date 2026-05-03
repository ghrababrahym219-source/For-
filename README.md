# For-
AI-Powered Autonomous Dropshipping System with Dynamic Pricing,  Generative Marketing, and Real-time Analytics
#!/usr/bin/env bash
set -euo pipefail

# تعديل: اختر الفرع الذي تريد الدفع إليه
TARGET_BRANCH="${1:-main}"
NEW_DIR="weather-dashboard"

echo "Preparing files under ./${NEW_DIR} and committing to branch ${TARGET_BRANCH}..."

# Ensure repo is clean
if [ -n "$(git status --porcelain)" ]; then
  echo "Repo has uncommitted changes. Commit or stash them first."
  exit 1
fi

# Create branch if needed
if git ls-remote --exit-code --heads origin "${TARGET_BRANCH}" >/dev/null 2>&1; then
  git checkout "${TARGET_BRANCH}"
else
  git checkout -b "${TARGET_BRANCH}"
fi

mkdir -p "${NEW_DIR}/app/api" "${NEW_DIR}/app/core" "${NEW_DIR}/app/infrastructure" "${NEW_DIR}/app/utils" "${NEW_DIR}/tests" ".github/workflows"

# Write files
cat > "${NEW_DIR}/README.md" <<'MD'
# Weather Dashboard Microservice

Lightweight FastAPI microservice that fetches weather data from OpenWeatherMap,
with Redis caching, async I/O, Pydantic validation, structured logging and basic CI.

Run locally:
1. cp .env.example .env and set WEATHER_API_OPENWEATHER_API_KEY and REDIS_URL
2. pip install -r requirements.txt
3. uvicorn app.main:app --reload --port 8000
MD

cat > "${NEW_DIR}/.env.example" <<'ENV'
# Weather Dashboard .env.example
WEATHER_API_OPENWEATHER_API_KEY=your_openweather_api_key_here
WEATHER_API_BASE_URL=https://api.openweathermap.org/data/2.5
WEATHER_API_TIMEOUT=10
REDIS_URL=redis://localhost:6379/0
CACHE_TTL_SECONDS=600
APP_HOST=0.0.0.0
APP_PORT=8000
ENVIRONMENT=development
LOG_LEVEL=INFO
ENV

cat > "${NEW_DIR}/requirements.txt" <<'REQ'
fastapi==0.104.1
uvicorn[standard]==0.24.0
pydantic==2.5.0
aiohttp==3.9.1
aioredis==2.0.1
python-dotenv==1.0.0
prometheus-client==0.19.0
pytest==7.4.3
pytest-asyncio==0.21.1
REQ

cat > "${NEW_DIR}/Dockerfile" <<'DOCK'
FROM python:3.11-slim
WORKDIR /app
COPY requirements.txt .
RUN apt-get update && apt-get install -y gcc && pip install --no-cache-dir -r requirements.txt && apt-get remove -y gcc && rm -rf /var/lib/apt/lists/*
COPY . .
EXPOSE 8000
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
DOCK

# app/main.py
cat > "${NEW_DIR}/app/main.py" <<'PY'
"""FastAPI Weather Dashboard microservice entrypoint."""
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware
from contextlib import asynccontextmanager
import logging

from app.config import settings
from app.api.routes import router as weather_router
from app.utils.logger import setup_logging

setup_logging(level=settings.log_level)
logger = logging.getLogger("weather-dashboard")

@asynccontextmanager
async def lifespan(app: FastAPI):
    logger.info("Starting Weather Dashboard service")
    yield
    logger.info("Stopping Weather Dashboard service")

app = FastAPI(title="Weather Dashboard", version="0.1.0", lifespan=lifespan)

app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

app.include_router(weather_router, prefix="/api/v1")

@app.get("/", tags=["root"])
async def root():
    return {"message": "Weather Dashboard", "version": "0.1.0"}
PY

# app/config.py
cat > "${NEW_DIR}/app/config.py" <<'PY'
"""Configuration via environment and Pydantic."""
from pydantic_settings import BaseSettings, SettingsConfigDict
from pydantic import Field

class Settings(BaseSettings):
    model_config = SettingsConfigDict(env_file=".env", case_sensitive=False)

    weather_api_key: str = Field(..., env="WEATHER_API_OPENWEATHER_API_KEY")
    weather_base_url: str = Field("https://api.openweathermap.org/data/2.5", env="WEATHER_API_BASE_URL")
    weather_timeout: int = Field(10, env="WEATHER_API_TIMEOUT")

    redis_url: str = Field("redis://localhost:6379/0", env="REDIS_URL")
    cache_ttl_seconds: int = Field(600, env="CACHE_TTL_SECONDS")

    app_host: str = Field("0.0.0.0", env="APP_HOST")
    app_port: int = Field(8000, env="APP_PORT")
    environment: str = Field("development", env="ENVIRONMENT")
    log_level: str = Field("INFO", env="LOG_LEVEL")

settings = Settings()
PY

# app/utils/logger.py
cat > "${NEW_DIR}/app/utils/logger.py" <<'PY'
"""Structured JSON logging util."""
import logging
import json
from datetime import datetime

class JSONFormatter(logging.Formatter):
    def format(self, record: logging.LogRecord) -> str:
        data = {
            "timestamp": datetime.utcfromtimestamp(record.created).isoformat() + "Z",
            "level": record.levelname,
            "logger": record.name,
            "message": record.getMessage(),
            "module": record.module,
            "func": record.funcName,
            "lineno": record.lineno
        }
        if record.exc_info:
            data["exc_info"] = self.formatException(record.exc_info)
        return json.dumps(data)

def setup_logging(level: str = "INFO") -> None:
    root = logging.getLogger()
    root.setLevel(level.upper())
    handler = logging.StreamHandler()
    handler.setFormatter(JSONFormatter())
    if not root.handlers:
        root.addHandler(handler)
PY

# app/api/routes.py
cat > "${NEW_DIR}/app/api/routes.py" <<'PY'
"""API routes for weather service."""
from fastapi import APIRouter, Query, HTTPException
from typing import Optional

from app.core.weather_service import WeatherService
from app.core.cache_service import CacheService
from app.config import settings

router = APIRouter()
_cache = CacheService(settings.redis_url)
_service = WeatherService(cache=_cache)

@router.on_event("startup")
async def startup():
    await _cache.connect()

@router.on_event("shutdown")
async def shutdown():
    await _cache.disconnect()

@router.get("/weather/current")
async def current_weather(
    latitude: float = Query(..., ge=-90, le=90),
    longitude: float = Query(..., ge=-180, le=180),
    units: str = Query("metric")
):
    try:
        data = await _service.get_combined_current(latitude, longitude, units)
        return data
    except Exception as exc:
        raise HTTPException(status_code=503, detail=str(exc))
PY

# app/core/cache_service.py
cat > "${NEW_DIR}/app/core/cache_service.py" <<'PY'
"""Simple Redis cache wrapper using aioredis."""
from typing import Optional, Any
import json
import aioredis
import logging

logger = logging.getLogger("cache")

class CacheService:
    def __init__(self, redis_url: str):
        self._url = redis_url
        self._redis: Optional[aioredis.Redis] = None

    async def connect(self) -> None:
        try:
            self._redis = aioredis.from_url(self._url, encoding="utf8", decode_responses=True)
            await self._redis.ping()
            logger.info("Connected to Redis")
        except Exception as e:
            logger.error(f"Redis connect failed: {e}")
            self._redis = None

    async def disconnect(self) -> None:
        if self._redis:
            await self._redis.close()
            logger.info("Redis disconnected")

    async def get(self, key: str) -> Optional[Any]:
        if not self._redis:
            return None
        v = await self._redis.get(key)
        if v is None:
            return None
        try:
            return json.loads(v)
        except Exception:
            return v

    async def set(self, key: str, value: Any, ttl: int = 600) -> bool:
        if not self._redis:
            return False
        try:
            await self._redis.set(key, json.dumps(value), ex=ttl)
            return True
        except Exception as e:
            logger.error(f"Redis set error: {e}")
            return False
PY

# app/infrastructure/external_api.py
cat > "${NEW_DIR}/app/infrastructure/external_api.py" <<'PY'
"""External API integration (OpenWeatherMap)."""
from typing import Dict, Any
import aiohttp
import asyncio
from app.config import settings
import logging

logger = logging.getLogger("external_api")

async def fetch_current_openweather(lat: float, lon: float, units: str = "metric") -> Dict[str, Any]:
    url = f"{settings.weather_base_url}/weather"
    params = {
        "lat": lat,
        "lon": lon,
        "appid": settings.weather_api_key,
        "units": units
    }
    timeout = aiohttp.ClientTimeout(total=settings.weather_timeout)
    async with aiohttp.ClientSession(timeout=timeout) as session:
        async with session.get(url, params=params) as resp:
            if resp.status != 200:
                text = await resp.text()
                logger.error(f"OpenWeatherMap error {resp.status}: {text}")
                raise RuntimeError(f"Weather provider error: {resp.status}")
            return await resp.json()

async def fetch_forecast_openweather(lat: float, lon: float, units: str = "metric") -> Dict[str, Any]:
    url = f"{settings.weather_base_url}/forecast"
    params = {
        "lat": lat,
        "lon": lon,
        "appid": settings.weather_api_key,
        "units": units,
        "cnt": 40
    }
    timeout = aiohttp.ClientTimeout(total=settings.weather_timeout)
    async with aiohttp.ClientSession(timeout=timeout) as session:
        async with session.get(url, params=params) as resp:
            if resp.status != 200:
                text = await resp.text()
                logger.error(f"OpenWeatherMap forecast error {resp.status}: {text}")
                raise RuntimeError(f"Weather provider error: {resp.status}")
            return await resp.json()
PY

# app/core/weather_service.py
cat > "${NEW_DIR}/app/core/weather_service.py" <<'PY'
"""Business logic combining external API and cache."""
from typing import Any, Dict
import asyncio
from datetime import datetime
from app.infrastructure.external_api import fetch_current_openweather, fetch_forecast_openweather
from app.core.cache_service import CacheService
import logging

logger = logging.getLogger("weather_service")

class WeatherService:
    def __init__(self, cache: CacheService):
        self.cache = cache

    async def get_combined_current(self, lat: float, lon: float, units: str = "metric") -> Dict[str, Any]:
        cache_key = f"weather:current:{lat}:{lon}:{units}"
        cached = await self.cache.get(cache_key)
        if cached:
            cached["cached"] = True
            return cached

        # fetch in parallel
        current_task = fetch_current_openweather(lat, lon, units)
        forecast_task = fetch_forecast_openweather(lat, lon, units)
        current, forecast = await asyncio.gather(current_task, forecast_task)
        result = {
            "current": current,
            "forecast_summary": {
                "cnt": len(forecast.get("list", [])),
            },
            "fetched_at": datetime.utcnow().isoformat() + "Z",
            "cached": False
        }
        await self.cache.set(cache_key, result, ttl=600)
        return result
PY

# tests/test_basic.py
cat > "${NEW_DIR}/tests/test_basic.py" <<'PY'
import pytest
from fastapi.testclient import TestClient
from app.main import app

client = TestClient(app)

def test_root():
    r = client.get("/")
    assert r.status_code == 200
    body = r.json()
    assert "message" in body
PY

# CI workflow
cat > ".github/workflows/ci.yml" <<'YML'
name: CI
on: [push, pull_request]
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Set up Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.11'
      - name: Install deps
        run: |
          python -m pip install --upgrade pip
          pip install -r weather-dashboard/requirements.txt
      - name: Run tests
        run: |
          pytest -q
YML

# End of file writes
echo "Files created."
git add "${NEW_DIR}" .github/workflows/ci.yml
git commit -m "feat(weather-dashboard): add weather-dashboard microservice (FastAPI, cache, external provider, tests, CI)"
git push origin "${TARGET_BRANCH}"

echo "Done. Pushed changes to branch ${TARGET_BRANCH}."
