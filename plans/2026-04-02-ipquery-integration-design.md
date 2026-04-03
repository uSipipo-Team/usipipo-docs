# 🧠 IPQuery.io Integration Design - uSipipo VPN

> Diseño técnico para integración de IPQuery.io API en el ecosistema uSipipo

---

## 📊 Información del Diseño

| Campo | Valor |
|-------|-------|
| **Documento** | IPQuery.io Integration Design |
| **Estado** | ✅ Aprobado |
| **Autor** | uSipipo Team |
| **Fecha** | 2026-04-02 |
| **Versión** | 1.0.0 |
| **Repositorios** | usipipo-backend, usipipo-commons |

---

## 🎯 Visión General

Integración de la API de **IPQuery.io** para mejorar la experiencia de usuario mediante recomendación inteligente de servidores VPN y monitoreo proactivo de la calidad de las IPs de salida.

---

## 📋 Casos de Uso

### **Feature 1: Smart Server Recommendation**

**Propósito:** Recomendar automáticamente los servidores VPN con menor latencia basándose en la ubicación geográfica del usuario.

**Problema que resuelve:**
- Usuarios seleccionan servidores manualmente sin conocer cuál es óptimo
- Alta latencia por selección incorrecta de servidor
- Soporte saturado con consultas de "¿qué servidor uso?"

**Solución:**
- Detectar ubicación del usuario (país, ciudad) vía IPQuery.io
- Calcular servidores recomendados basados en proximidad geográfica
- Mostrar recomendación en UI con indicador de latencia estimada

---

### **Feature 2: VPN Detection Quality Monitoring**

**Propósito:** Monitorear proactivamente que las IPs de salida de uSipipo NO sean detectadas como VPN/proxy/tor por servicios externos.

**Problema que resuelve:**
- IPs de salida bloqueadas por servicios (Netflix, Hulu, etc.)
- Usuarios reportan que VPN "no funciona" para ciertos sitios
- Degradación gradual de calidad sin detección temprana

**Solución:**
- Background job consulta IPQuery.io diariamente para cada IP de salida
- Alertas automáticas si `risk_score > 50` o `is_vpn/is_proxy/is_tor = true`
- Dashboard de admin con estado de salud de IPs
- Rotación proactiva de IPs comprometidas

---

## 🏗️ Arquitectura

### **Diagrama de Alto Nivel**

```
┌─────────────────────────────────────────────────────────────────────┐
│                         uSipipo Ecosystem                           │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌──────────────┐         ┌──────────────┐                        │
│  │   Telegram   │         │   Android    │                        │
│  │     Bot      │         │     App      │                        │
│  └──────┬───────┘         └──────┬───────┘                        │
│         │                        │                                  │
│         └───────────┬────────────┘                                  │
│                     │                                               │
│                     ▼                                               │
│         ┌─────────────────────┐                                    │
│         │   Backend API       │                                    │
│         │   (FastAPI)         │                                    │
│         │   Port: 8000        │                                    │
│         └──────────┬──────────┘                                    │
│                    │                                               │
│         ┌──────────┼──────────┐                                   │
│         │          │          │                                    │
│         ▼          ▼          ▼                                    │
│  ┌────────────┐ ┌────────────┐ ┌────────────┐                     │
│  │ IPQuery    │ │ PostgreSQL │ │   Redis    │                     │
│  │ Service    │ │  (Datos)   │ │  (Cache)   │                     │
│  │ (External) │ │            │ │            │                     │
│  └────────────┘ └────────────┘ └────────────┘                     │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 📦 Componentes

### **1. IPQuery Client (usipipo-commons)**

**Responsabilidad:** Cliente HTTP reutilizable para consultar IPQuery.io API.

**Ubicación:** `usipipo-commons/usipipo_commons/clients/ipquery_client.py`

**Interfaz:**

```python
class IPQueryClient:
    """Cliente para IPQuery.io API"""
    
    def __init__(
        self,
        base_url: str = "https://api.ipquery.io",
        timeout: int = 5,
        cache_ttl: int = 3600  # 1 hora
    ):
        ...
    
    async def get_ip_info(self, ip: str) -> IPQueryResponse:
        """Obtener información de una IP"""
        ...
    
    async def get_bulk_ip_info(self, ips: list[str]) -> list[IPQueryResponse]:
        """Obtener información de múltiples IPs (bulk)"""
        ...
```

**Entidades:**

```python
class IPQueryResponse(BaseModel):
    """Respuesta de IPQuery.io"""
    
    # Location
    country: str
    country_code: str  # ISO 3166-1 alpha-2
    city: str
    state: str
    zipcode: str
    timezone: str
    
    # Risk
    is_mobile: bool
    is_vpn: bool
    is_tor: bool
    is_proxy: bool
    risk_score: int  # 0-100
    
    # Metadata
    ip: str
    queried_at: datetime
```

---

### **2. Smart Server Recommendation Service (usipipo-backend)**

**Responsabilidad:** Calcular recomendaciones de servidores basadas en ubicación.

**Ubicación:** `usipipo-backend/src/core/application/services/smart_server_service.py`

**Interfaz:**

```python
class SmartServerService:
    """Servicio para recomendación inteligente de servidores"""
    
    def __init__(
        self,
        ipquery_client: IPQueryClient,
        server_repository: ServerRepository,
        cache: RedisCache
    ):
        ...
    
    async def get_recommended_servers(
        self,
        user_ip: str,
        protocol: str
    ) -> RecommendedServersResponse:
        """
        Obtener servidores recomendados para un usuario
        
        Args:
            user_ip: IP del usuario (detectada del request)
            protocol: 'wireguard' o 'outline'
        
        Returns:
            Lista de servidores con campo 'recommended' priorizado
        """
        ...
```

**Algoritmo de Recomendación:**

```python
RECOMMENDATION_RULES = {
    # Región → Servidores recomendados (en orden de prioridad)
    "MX": ["US-West-1", "US-East-1"],  # México → US West primero
    "CO": ["US-East-1", "US-West-1"],  # Colombia → US East primero
    "AR": ["US-East-1", "EU-West-1"],  # Argentina → US East
    "CL": ["US-East-1", "EU-West-1"],  # Chile → US East
    "PE": ["US-East-1", "US-West-1"],  # Perú → US East
    "BR": ["US-East-1", "EU-West-1"],  # Brasil → US East
    "EC": ["US-West-1", "US-East-1"],  # Ecuador → US West
    # Default: US East como fallback
    "DEFAULT": ["US-East-1", "US-West-1"]
}
```

**Response:**

```python
class RecommendedServersResponse(BaseModel):
    servers: list[ServerInfo]
    recommended: list[ServerInfo]  # Top 5 recomendados
    user_location: UserLocation
    
class ServerInfo(BaseModel):
    id: str
    name: str
    country_code: str
    country_name: str
    city: str
    load_percentage: int
    load_level: str  # "low", "medium", "high"
    status: str  # "online", "offline", "maintenance"
    is_recommended: bool
    recommended_reason: str | None  # "Más cercano a tu ubicación"
    
class UserLocation(BaseModel):
    country: str
    country_code: str
    city: str
    detected_from: str  # "ipquery", "cache", "user_profile"
```

---

### **3. VPN Quality Monitor (usipipo-backend)**

**Responsabilidad:** Background job para monitorear calidad de IPs de salida.

**Ubicación:** `usipipo-backend/src/infrastructure/jobs/vpn_quality_monitor.py`

**Interfaz:**

```python
class VPNQualityMonitor:
    """Monitor de calidad de VPN"""
    
    def __init__(
        self,
        ipquery_client: IPQueryClient,
        vpn_server_repository: VPNServerRepository,
        alert_service: AlertService,
        db_session: AsyncSession
    ):
        ...
    
    async def run_monitoring(self) -> VPNQualityReport:
        """
        Ejecutar monitoreo de todas las IPs de salida
        
        Returns:
            Reporte con estado de cada IP
        """
        ...
    
    async def check_ip_quality(
        self,
        server_ip: str,
        server_id: str
    ) -> IPQualityResult:
        """
        Verificar calidad de una IP específica
        
        Returns:
            Resultado con flags de riesgo
        """
        ...
```

**Entidades:**

```python
class IPQualityResult(BaseModel):
    """Resultado de verificación de IP"""
    
    server_id: str
    server_ip: str
    server_name: str
    
    # IPQuery data
    is_vpn: bool
    is_proxy: bool
    is_tor: bool
    risk_score: int
    
    # Quality assessment
    quality_status: str  # "good", "warning", "critical"
    issues: list[str]  # ["high_risk_score", "detected_as_proxy", ...]
    
    # Metadata
    checked_at: datetime
    previous_check: datetime | None
    
class VPNQualityReport(BaseModel):
    """Reporte completo de monitoreo"""
    
    total_servers: int
    good_quality: int
    warning_quality: int
    critical_quality: int
    
    servers: list[IPQualityResult]
    critical_alerts: list[CriticalAlert]
    
    generated_at: datetime
    
class CriticalAlert(BaseModel):
    """Alerta crítica para admin"""
    
    server_id: str
    server_ip: str
    severity: str  # "high", "critical"
    reason: str
    recommended_action: str
```

**Reglas de Calidad:**

```python
QUALITY_THRESHOLDS = {
    "good": {
        "max_risk_score": 30,
        "allowed_vpn": True,  # Es esperado que sea VPN
        "allowed_proxy": False,
        "allowed_tor": False
    },
    "warning": {
        "max_risk_score": 50,
        "allowed_vpn": True,
        "allowed_proxy": False,
        "allowed_tor": False
    },
    "critical": {
        "max_risk_score": 100,  # Cualquier score > 50
        "allowed_vpn": False,
        "allowed_proxy": False,
        "allowed_tor": False
    }
}
```

---

### **4. API Endpoints (usipipo-backend)**

#### **Endpoint 1: Smart Server Recommendation**

**Mejora al endpoint existente:**

```http
GET /api/v1/vpn/servers?protocol=outline
Authorization: Bearer <access_token>
```

**Cambios:**
- El backend detecta automáticamente la IP del usuario desde `X-Forwarded-For` o `X-Real-IP`
- La respuesta ahora incluye campo `recommended_reason` en cada servidor
- El array `recommended` está priorizado por cercanía geográfica

**Response (mejorada):**

```json
{
  "servers": [
    {
      "id": "550e8400-e29b-41d4-a716-446655440000",
      "name": "US-East-1",
      "country_code": "US",
      "country_name": "United States",
      "city": "New York",
      "load_percentage": 23,
      "load_level": "low",
      "status": "online",
      "is_recommended": true,
      "recommended_reason": "Más cercano a tu ubicación (México)"
    },
    {
      "id": "660e8400-e29b-41d4-a716-446655440001",
      "name": "EU-West-1",
      "country_code": "DE",
      "country_name": "Germany",
      "city": "Frankfurt",
      "load_percentage": 67,
      "load_level": "medium",
      "status": "online",
      "is_recommended": false,
      "recommended_reason": null
    }
  ],
  "recommended": [
    {
      "id": "550e8400-e29b-41d4-a716-446655440000",
      "name": "US-East-1",
      "country_code": "US",
      "country_name": "United States",
      "city": "New York",
      "load_percentage": 23,
      "load_level": "low",
      "status": "online",
      "is_recommended": true,
      "recommended_reason": "Más cercano a tu ubicación (México)"
    }
  ],
  "user_location": {
    "country": "Mexico",
    "country_code": "MX",
    "city": "Mexico City",
    "detected_from": "ipquery"
  }
}
```

---

#### **Endpoint 2: VPN Quality Dashboard (Admin)**

**Nuevo endpoint para admin:**

```http
GET /api/v1/admin/vpn/quality
Authorization: Bearer <admin_access_token>
```

**Response:**

```json
{
  "status": "success",
  "data": {
    "total_servers": 10,
    "good_quality": 8,
    "warning_quality": 1,
    "critical_quality": 1,
    "last_check": "2026-04-02T08:00:00Z",
    "next_check": "2026-04-03T08:00:00Z",
    "servers": [
      {
        "server_id": "srv-001",
        "server_name": "US-East-1",
        "server_ip": "198.51.100.1",
        "quality_status": "good",
        "risk_score": 15,
        "is_vpn": true,
        "is_proxy": false,
        "is_tor": false,
        "issues": [],
        "checked_at": "2026-04-02T08:00:00Z"
      },
      {
        "server_id": "srv-002",
        "server_name": "EU-West-1",
        "server_ip": "203.0.113.50",
        "quality_status": "critical",
        "risk_score": 75,
        "is_vpn": true,
        "is_proxy": true,
        "is_tor": false,
        "issues": ["high_risk_score", "detected_as_proxy"],
        "checked_at": "2026-04-02T08:00:00Z"
      }
    ],
    "critical_alerts": [
      {
        "server_id": "srv-002",
        "server_name": "EU-West-1",
        "severity": "critical",
        "reason": "IP detectada como proxy con risk_score=75",
        "recommended_action": "Rotar IP o reemplazar servidor"
      }
    ]
  }
}
```

---

#### **Endpoint 3: Forzar Re-check de Calidad (Admin)**

**Nuevo endpoint para admin:**

```http
POST /api/v1/admin/vpn/quality/check
Authorization: Bearer <admin_access_token>
Content-Type: application/json

{
  "server_ids": ["srv-002", "srv-003"]  // Opcional, si no se envía chequea todos
}
```

**Response:**

```json
{
  "status": "success",
  "message": "Verificación de calidad iniciada",
  "data": {
    "check_id": "check-123",
    "servers_queued": 2,
    "estimated_completion": "2026-04-02T08:05:00Z"
  }
}
```

---

## 🔄 Data Flow

### **Flow 1: Smart Server Recommendation**

```
┌──────────────┐
│   Usuario    │
│  (Telegram/  │
│   Android)   │
└──────┬───────┘
       │ 1. GET /api/v1/vpn/servers?protocol=outline
       │    Headers: X-Forwarded-For: 187.141.240.1
       ▼
┌──────────────┐
│   Backend    │
│   (FastAPI)  │
└──────┬───────┘
       │ 2. Extraer IP del usuario (187.141.240.1)
       │ 3. Check Redis cache: GET ipquery:187.141.240.1
       │
       ├─→ [CACHE HIT] ──→ Usar datos cacheados
       │
       └─→ [CACHE MISS]
              │
              ▼
       ┌──────────────┐
       │  IPQuery.io  │
       │  API         │
       └──────┬───────┘
              │ 4. GET /187.141.240.1
              │ Response: {country: "MX", city: "Mexico City", ...}
              ▼
       ┌──────────────┐
       │   Backend    │
       │   (FastAPI)  │
       └──────┬───────┘
              │ 5. Guardar en Redis: SET ipquery:187.141.240.1 (TTL 1h)
              │ 6. Calcular servidores recomendados:
              │    - MX → US-West-1, US-East-1
              │ 7. Ordenar servidores por recomendación
              ▼
       ┌──────────────┐
       │   Usuario    │
       │  Recibe:     │
       │  servers +   │
       │  recommended │
       └──────────────┘
```

---

### **Flow 2: VPN Quality Monitoring (Background Job)**

```
┌──────────────┐
│  Scheduler   │
│  (Cron:      │
│   08:00      │
│   diario)    │
└──────┬───────┘
       │ 1. Trigger vpn_quality_monitor job
       ▼
┌──────────────┐
│   Backend    │
│   (Job)      │
└──────┬───────┘
       │ 2. SELECT * FROM vpn_servers WHERE status='active'
       │    Result: [srv-001 (198.51.100.1), srv-002 (203.0.113.50), ...]
       │
       │ 3. FOR EACH server:
       │    a. Check Redis cache: GET ipquery:{server_ip}
       │    b. [CACHE MISS] → IPQuery API
       │    c. Parse response
       │    d. Evaluate quality rules
       │    e. Update DB: vpn_server_quality_logs
       │    f. If critical → Create alert
       │
       ▼
┌──────────────┐
│  IPQuery.io  │
│  API         │
│  (Bulk)      │
└──────┬───────┘
       │ 4. GET /198.51.100.1,203.0.113.50,...
       │    Response: [{is_vpn: true, risk_score: 15, ...}, ...]
       │
       ▼
┌──────────────┐
│   Backend    │
│   (Job)      │
└──────┬───────┘
       │ 5. INSERT INTO vpn_server_quality_logs
       │ 6. IF risk_score > 50 OR is_proxy=true:
       │    - INSERT INTO admin_alerts
       │    - Send notification (FCM/Email)
       │
       ▼
┌──────────────┐
│   Admin      │
│  Dashboard   │
│  + Alerts    │
└──────────────┘
```

---

## 🗄️ Database Schema

### **Nuevas Tablas**

#### **1. vpn_server_quality_logs**

```sql
CREATE TABLE vpn_server_quality_logs (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    server_id UUID NOT NULL REFERENCES vpn_servers(id),
    server_ip INET NOT NULL,
    
    -- IPQuery data
    country VARCHAR(100),
    country_code VARCHAR(2),
    city VARCHAR(100),
    is_vpn BOOLEAN DEFAULT FALSE,
    is_proxy BOOLEAN DEFAULT FALSE,
    is_tor BOOLEAN DEFAULT FALSE,
    risk_score INTEGER CHECK (risk_score >= 0 AND risk_score <= 100),
    
    -- Quality assessment
    quality_status VARCHAR(20) CHECK (quality_status IN ('good', 'warning', 'critical')),
    issues TEXT[],  -- Array de strings: ['high_risk_score', 'detected_as_proxy']
    
    -- Metadata
    checked_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- Indexes
CREATE INDEX idx_vpn_quality_logs_server_id ON vpn_server_quality_logs(server_id);
CREATE INDEX idx_vpn_quality_logs_checked_at ON vpn_server_quality_logs(checked_at DESC);
CREATE INDEX idx_vpn_quality_logs_status ON vpn_server_quality_logs(quality_status);
```

---

#### **2. admin_alerts**

```sql
CREATE TABLE admin_alerts (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    alert_type VARCHAR(50) NOT NULL,
    severity VARCHAR(20) NOT NULL CHECK (severity IN ('low', 'medium', 'high', 'critical')),
    
    -- Context
    server_id UUID REFERENCES vpn_servers(id),
    server_ip INET,
    title VARCHAR(255) NOT NULL,
    message TEXT NOT NULL,
    recommended_action TEXT,
    
    -- Status
    status VARCHAR(20) DEFAULT 'pending' CHECK (status IN ('pending', 'acknowledged', 'resolved', 'dismissed')),
    acknowledged_by UUID REFERENCES users(id),
    acknowledged_at TIMESTAMP WITH TIME ZONE,
    resolved_by UUID REFERENCES users(id),
    resolved_at TIMESTAMP WITH TIME ZONE,
    
    -- Metadata
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- Indexes
CREATE INDEX idx_admin_alerts_status ON admin_alerts(status);
CREATE INDEX idx_admin_alerts_severity ON admin_alerts(severity);
CREATE INDEX idx_admin_alerts_created_at ON admin_alerts(created_at DESC);
```

---

#### **3. vpn_server_recommendations (Cache de recomendaciones)**

```sql
CREATE TABLE vpn_server_recommendations (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    country_code VARCHAR(2) NOT NULL,
    protocol VARCHAR(20) NOT NULL,
    
    -- Recommended servers (ordenados por prioridad)
    recommended_server_ids UUID[],  -- Array de server IDs
    
    -- Metadata
    calculated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    expires_at TIMESTAMP WITH TIME ZONE NOT NULL,
    is_active BOOLEAN DEFAULT TRUE
);

-- Indexes
CREATE INDEX idx_vpn_recommendations_country ON vpn_server_recommendations(country_code);
CREATE INDEX idx_vpn_recommendations_protocol ON vpn_server_recommendations(protocol);
CREATE INDEX idx_vpn_recommendations_active ON vpn_server_recommendations(is_active) WHERE is_active = TRUE;
```

---

## 🧪 Error Handling

### **Escenarios de Error**

| Escenario | Comportamiento | Fallback |
|-----------|----------------|----------|
| **IPQuery API timeout** | Timeout después de 5s | Usar datos cacheados en Redis (si existen) |
| **IPQuery API rate limit (429)** | Reintentar con exponential backoff (3 intentos) | Si falla → usar fallback geográfico por IP ranges |
| **IP inválida** | Retornar error 400 | N/A (error de cliente) |
| **Redis cache unavailable** | Consultar IPQuery directamente | Sin cache, mayor latencia |
| **VPN server sin IP** | Saltar servidor en monitoreo | Log warning, continuar con siguientes |

---

### **Retry Logic (Exponential Backoff)**

```python
import asyncio
from tenacity import retry, stop_after_attempt, wait_exponential

@retry(
    stop=stop_after_attempt(3),
    wait=wait_exponential(multiplier=1, min=1, max=10),
    reraise=True
)
async def query_ipquery_with_retry(ip: str):
    """Consultar IPQuery con reintentos"""
    async with httpx.AsyncClient(timeout=5.0) as client:
        response = await client.get(f"https://api.ipquery.io/{ip}")
        response.raise_for_status()
        return response.json()
```

---

### **Circuit Breaker Pattern**

```python
from circuitbreaker import circuit

@circuit(
    failure_threshold=5,  # 5 fallos consecutivos
    recovery_timeout=60,  # Esperar 60s antes de reintentar
    expected_exception=httpx.HTTPError
)
async def ipquery_api_call(ip: str):
    """Circuit breaker para IPQuery API"""
    async with httpx.AsyncClient(timeout=5.0) as client:
        response = await client.get(f"https://api.ipquery.io/{ip}")
        response.raise_for_status()
        return response.json()
```

---

## 🧪 Testing Strategy

### **Unit Tests**

```python
# tests/unit/test_ipquery_client.py

class TestIPQueryClient:
    
    async def test_get_ip_info_success(self, mock_httpx):
        """Test consulta exitosa"""
        client = IPQueryClient()
        result = await client.get_ip_info("187.141.240.1")
        
        assert result.country == "Mexico"
        assert result.country_code == "MX"
        assert result.city == "Mexico City"
    
    async def test_get_ip_info_cache_hit(self, redis_cache):
        """Test cache hit"""
        client = IPQueryClient(cache=redis_cache)
        
        # Primera consulta (cache miss)
        await client.get_ip_info("187.141.240.1")
        
        # Segunda consulta (cache hit)
        with patch.object(httpx.AsyncClient, 'get') as mock_get:
            await client.get_ip_info("187.141.240.1")
            mock_get.assert_not_called()  # No llamó a API
    
    async def test_get_ip_info_timeout(self, mock_httpx_timeout):
        """Test timeout de API"""
        client = IPQueryClient()
        
        with pytest.raises(IPQueryTimeoutError):
            await client.get_ip_info("187.141.240.1")
```

---

### **Integration Tests**

```python
# tests/integration/test_smart_server_service.py

class TestSmartServerService:
    
    async def test_get_recommended_servers_mexico(self):
        """Test recomendación para México"""
        service = SmartServerService()
        result = await service.get_recommended_servers(
            user_ip="187.141.240.1",
            protocol="outline"
        )
        
        assert result.user_location.country_code == "MX"
        assert result.recommended[0].country_code == "US"
        assert result.recommended[0].is_recommended == True
        assert result.recommended[0].recommended_reason is not None
    
    async def test_get_recommended_servers_unknown_country(self):
        """Test país no soportado → fallback a US-East"""
        service = SmartServerService()
        result = await service.get_recommended_servers(
            user_ip="41.234.56.78",  # IP de África (no soportada)
            protocol="outline"
        )
        
        assert result.recommended[0].country_code == "US"
        assert result.recommended[0].name == "US-East-1"
```

---

### **End-to-End Tests**

```python
# tests/e2e/test_vpn_quality_monitor.py

class TestVPNQualityMonitor:
    
    async def test_full_monitoring_run(self):
        """Test ejecución completa del monitor"""
        monitor = VPNQualityMonitor()
        report = await monitor.run_monitoring()
        
        assert report.total_servers > 0
        assert report.good_quality + report.warning_quality + report.critical_quality == report.total_servers
        
        # Verificar que se guardó en DB
        async with db_session() as session:
            logs = await session.execute(
                select(VPNServerQualityLog).where(
                    VPNServerQualityLog.checked_at >= datetime.now() - timedelta(minutes=5)
                )
            )
            assert len(logs.scalars().all()) == report.total_servers
```

---

## 📊 Monitoring & Observability

### **Métricas a Monitorear**

```python
# Prometheus metrics

# IPQuery API
ipquery_api_requests_total = Counter(
    'ipquery_api_requests_total',
    'Total IPQuery API requests',
    ['status']  # success, error, timeout
)

ipquery_api_latency_seconds = Histogram(
    'ipquery_api_latency_seconds',
    'IPQuery API latency',
    buckets=[0.1, 0.5, 1.0, 2.0, 5.0]
)

ipquery_cache_hit_ratio = Gauge(
    'ipquery_cache_hit_ratio',
    'IPQuery cache hit ratio'
)

# VPN Quality
vpn_quality_servers_total = Gauge(
    'vpn_quality_servers_total',
    'Total VPN servers by quality status',
    ['status']  # good, warning, critical
)

vpn_quality_check_duration_seconds = Histogram(
    'vpn_quality_check_duration_seconds',
    'Duration of VPN quality monitoring job'
)
```

---

### **Logging**

```python
# Structured logging

import structlog

logger = structlog.get_logger()

# Smart Server Recommendation
logger.info(
    "smart_server_recommendation",
    user_ip=user_ip,
    detected_country=country_code,
    recommended_servers=len(recommended),
    cache_hit=cache_hit,
    latency_ms=latency_ms
)

# VPN Quality Monitor
logger.info(
    "vpn_quality_check",
    server_id=server_id,
    server_ip=server_ip,
    quality_status=quality_status,
    risk_score=risk_score,
    issues=issues
)

# Critical Alert
logger.error(
    "vpn_quality_critical_alert",
    server_id=server_id,
    server_ip=server_ip,
    risk_score=risk_score,
    recommended_action=recommended_action
)
```

---

### **Alerting**

**Alertas críticas (envían notificación inmediata):**

```yaml
# alertmanager.yml

groups:
  - name: vpn_quality
    rules:
      - alert: VPNQualityCritical
        expr: vpn_quality_servers_total{status="critical"} > 0
        for: 0m
        labels:
          severity: critical
        annotations:
          summary: "Servidor VPN con calidad crítica detectada"
          description: "El servidor {{ $labels.server }} tiene risk_score > 50"
      
      - alert: IPQueryAPIDown
        expr: rate(ipquery_api_requests_total{status="error"}[5m]) > 0.5
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "IPQuery API con alta tasa de errores"
          description: "{{ $value }} errores por segundo en los últimos 5 minutos"
```

---

## 🔐 Security Considerations

### **1. Privacy del Usuario**

**Datos almacenados:**
- ✅ IP del usuario (temporal, en cache Redis por 1h)
- ✅ País/ciudad detectado (asociado al usuario para analytics)
- ❌ NO almacenar histórico de IPs del usuario

**Política de retención:**
- Cache Redis: 1 hora TTL automático
- Analytics: Agregar por país (no por IP individual)
- Logs: 30 días, luego anonymizar

---

### **2. Rate Limiting**

**Límites de IPQuery.io:**
- Free tier: Unlimited
- Bulk queries: 10,000 IPs por request

**Nuestros límites internos:**
```python
# Para Smart Server Recommendation (por usuario)
@rate_limit(
    requests=30,  # 30 requests
    period=60     # por minuto
)
async def get_servers(user_id: str):
    ...

# Para VPN Quality Monitor (bulk job)
BATCH_SIZE = 1000  # Consultar de 1000 en 1000 IPs
DELAY_BETWEEN_BATCHES = 1  # 1 segundo entre batches
```

---

### **3. Fallback Geográfico**

Si IPQuery.io está down, usar fallback por IP ranges:

```python
IP_RANGE_FALLBACK = {
    # Ranges de IPs por país (simplificado)
    "187.0.0.0/8": "MX",
    "190.0.0.0/8": "CO",
    "181.0.0.0/8": "AR",
    "200.0.0.0/8": "BR",
    # Default
    "DEFAULT": "US"
}

def get_country_from_ip_range(ip: str) -> str:
    """Fallback geográfico por IP range"""
    ip_prefix = ip.split('.')[0]
    return IP_RANGE_FALLBACK.get(f"{ip_prefix}.0.0.0/8", "US")
```

---

## 🚀 Deployment

### **Variables de Entorno**

```bash
# .env (backend)

# IPQuery.io
IPQUERY_BASE_URL=https://api.ipquery.io
IPQUERY_TIMEOUT=5
IPQUERY_CACHE_TTL=3600

# VPN Quality Monitor
VPN_QUALITY_MONITOR_ENABLED=true
VPN_QUALITY_MONITOR_CRON="0 8 * * *"  # Diario a las 08:00 UTC
VPN_QUALITY_RISK_THRESHOLD=50
VPN_QUALITY_ALERT_EMAIL=admin@usipipo.com

# Feature flags
SMART_SERVER_RECOMMENDATION_ENABLED=true
```

---

### **Migraciones de Base de Datos**

```bash
# Crear migración
cd usipipo-backend
uv run alembic revision --autogenerate -m "Add IPQuery integration tables"

# Aplicar migración
uv run alembic upgrade head
```

---

### **Background Job Setup**

```python
# src/infrastructure/jobs/scheduler.py

from apscheduler.schedulers.asyncio import AsyncIOScheduler

scheduler = AsyncIOScheduler()

# VPN Quality Monitor (diario, 08:00 UTC)
scheduler.add_job(
    vpn_quality_monitor.run_monitoring,
    trigger='cron',
    hour=8,
    minute=0,
    id='vpn_quality_monitor',
    name='VPN Quality Monitoring'
)

# Iniciar scheduler
scheduler.start()
```

---

## 📈 Success Metrics

### **KPIs de Éxito**

| KPI | Línea Base | Objetivo | Medición |
|-----|------------|----------|----------|
| **Latencia promedio de conexión** | 120ms | < 80ms | Prometheus metrics |
| **% usuarios usando servidor recomendado** | N/A | > 70% | Analytics |
| **% servidores con calidad "good"** | N/A | > 90% | VPN Quality Dashboard |
| **Tiempo de detección de IP comprometida** | Manual (días) | < 24h | Alert timestamps |
| **Tickets de soporte por "VPN lenta"** | 15/mes | < 5/mes | Support tickets |

---

## 📚 Recursos Relacionados

- [IPQuery.io API Docs](http://ipquery.io/)
- [Backend API Reference](../apis/backend-api-reference.md)
- [VPN Connection Flow](../flows/vpn-connection-flow.md)
- [Infrastructure Stack](../technology/infrastructure-stack.md)

---

## ✅ Checklist de Implementación

### **Fase 1: Core (Sprint 1)**

- [ ] Crear `IPQueryClient` en usipipo-commons
- [ ] Agregar entidades `IPQueryResponse` en usipipo-commons
- [ ] Implementar `SmartServerService` en usipipo-backend
- [ ] Mejorar endpoint `GET /api/v1/vpn/servers` con recomendaciones
- [ ] Agregar tests unitarios para IPQueryClient
- [ ] Agregar tests de integración para SmartServerService

### **Fase 2: VPN Quality (Sprint 2)**

- [ ] Crear tablas `vpn_server_quality_logs` y `admin_alerts`
- [ ] Implementar `VPNQualityMonitor` job
- [ ] Implementar endpoint `GET /api/v1/admin/vpn/quality`
- [ ] Implementar endpoint `POST /api/v1/admin/vpn/quality/check`
- [ ] Configurar scheduler para job diario
- [ ] Agregar tests e2e para VPN Quality Monitor

### **Fase 3: Polish (Sprint 3)**

- [ ] Agregar métricas Prometheus
- [ ] Configurar alertas en AlertManager
- [ ] Mejorar logging estructurado
- [ ] Documentar en API reference
- [ ] Actualizar changelog
- [ ] Deploy a producción

---

**Última actualización:** 2026-04-02  
**Estado:** ✅ Aprobado para implementación
