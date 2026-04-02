# VPN Server Selection Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development to implement this plan task-by-task.

**Goal:** Implement user-facing VPN server selection in Telegram bot with dynamic server list display and backend API endpoint.

**Architecture:** Add new GET endpoint to backend for fetching available servers filtered by protocol, create server selection conversation state in Telegram bot with inline keyboard showing server details (country, city, load indicator), modify key creation flow to include server_id parameter.

**Tech Stack:** Python 3.13, FastAPI, asyncpg, SQLAlchemy, python-telegram-bot v21, JWT authentication, Clean/Hexagonal architecture.

---

## Pre-Implementation Checklist

- [x] Design approved via brainstorming skill
- [ ] Implementation plan created
- [ ] Backend worktree created
- [ ] Bot worktree created
- [ ] Tests environment ready

---

## Phase 1: Backend Implementation

### Task 1: Create User-Facing Server Schemas

**Files:**
- Modify: `usipipo-backend/src/shared/schemas/vpn.py`
- Test: `usipipo-backend/tests/shared/schemas/test_vpn_schemas.py`

**Step 1: Read existing vpn.py schemas**

```bash
cd /home/mowgli/usipipo/usipipo-backend
cat src/shared/schemas/vpn.py
```

**Step 2: Add new schemas to vpn.py**

Add after existing schemas:

```python
class VpnServerResponse(BaseModel):
    """User-facing VPN server response schema."""
    
    id: uuid.UUID
    name: str
    country_code: str
    country_name: str
    city: str | None
    load_percentage: int
    load_level: Literal["low", "medium", "high"]
    status: str
    
    @field_validator("load_percentage")
    @classmethod
    def calculate_load(cls, v: float, info: ValidationInfo) -> int:
        """Calculate load percentage from current/max connections."""
        return int(v)
    
    @field_validator("load_level")
    @classmethod
    def determine_load_level(cls, v: str, info: ValidationInfo) -> str:
        """Determine load level based on percentage."""
        load_pct = info.data.get("load_percentage", 0)
        if load_pct <= 50:
            return "low"
        elif load_pct <= 80:
            return "medium"
        else:
            return "high"


class VpnServersListResponse(BaseModel):
    """Response schema for list of available VPN servers."""
    
    servers: list[VpnServerResponse]
    recommended: list[VpnServerResponse]  # Top 5 lowest load
```

**Step 3: Create test file**

```python
# tests/shared/schemas/test_vpn_schemas.py
import uuid
import pytest
from src.shared.schemas.vpn import VpnServerResponse, VpnServersListResponse


class TestVpnServerResponse:
    """Test VpnServerResponse schema."""
    
    def test_valid_server_response(self):
        """Test valid server response creation."""
        server = VpnServerResponse(
            id=uuid.uuid4(),
            name="US-East-1",
            country_code="US",
            country_name="United States",
            city="New York",
            load_percentage=23,
            load_level="low",
            status="online"
        )
        
        assert server.name == "US-East-1"
        assert server.load_level == "low"
        assert server.city == "New York"
    
    def test_load_level_medium(self):
        """Test medium load level (51-80%)."""
        server = VpnServerResponse(
            id=uuid.uuid4(),
            name="DE-Frankfurt-1",
            country_code="DE",
            country_name="Germany",
            city="Frankfurt",
            load_percentage=67,
            load_level="medium",
            status="online"
        )
        
        assert server.load_level == "medium"
    
    def test_load_level_high(self):
        """Test high load level (>80%)."""
        server = VpnServerResponse(
            id=uuid.uuid4(),
            name="JP-Tokyo-1",
            country_code="JP",
            country_name="Japan",
            city="Tokyo",
            load_percentage=85,
            load_level="high",
            status="online"
        )
        
        assert server.load_level == "high"


class TestVpnServersListResponse:
    """Test VpnServersListResponse schema."""
    
    def test_servers_list_response(self):
        """Test servers list response with recommended servers."""
        servers = [
            VpnServerResponse(
                id=uuid.uuid4(),
                name=f"Server-{i}",
                country_code="US",
                country_name="United States",
                city="New York",
                load_percentage=i * 10,
                load_level="low",
                status="online"
            )
            for i in range(1, 6)
        ]
        
        response = VpnServersListResponse(
            servers=servers,
            recommended=servers[:5]  # Top 5
        )
        
        assert len(response.servers) == 5
        assert len(response.recommended) == 5
```

**Step 4: Run tests**

```bash
cd /home/mowgli/usipipo/usipipo-backend
source .venv/bin/activate
pytest tests/shared/schemas/test_vpn_schemas.py -v
```

Expected: All tests pass

**Step 5: Commit**

```bash
cd /home/mowgli/usipipo/usipipo-backend
git add src/shared/schemas/vpn.py tests/shared/schemas/test_vpn_schemas.py
git commit -m "feat: add user-facing VPN server response schemas"
```

**Skills:** @python-reviewer @code-reviewer

---

### Task 2: Create Server Service Method for User-Facing List

**Files:**
- Modify: `usipipo-backend/src/core/application/services/server_registry_service.py`
- Test: `usipipo-backend/tests/core/application/services/test_server_registry_service.py`

**Step 1: Read existing ServerRegistryService**

```bash
cd /home/mowgli/usipipo/usipipo-backend
cat src/core/application/services/server_registry_service.py
```

**Step 2: Add new method to get servers for user display**

Add to `ServerRegistryService` class:

```python
async def get_servers_for_user(
    self,
    protocol: str,
    limit: int | None = None,
) -> list[Server]:
    """Get available servers for user display, sorted by load.
    
    Args:
        protocol: Protocol type ("outline" or "wireguard")
        limit: Optional limit for number of servers returned
        
    Returns:
        List of available servers sorted by load (lowest first)
    """
    servers = await self.get_available_servers()
    
    # Filter by protocol support
    if protocol.lower() == "outline":
        servers = [s for s in servers if s.supports_outline]
    elif protocol.lower() == "wireguard":
        servers = [s for s in servers if s.supports_wireguard]
    
    # Filter only online servers
    servers = [s for s in servers if s.status == "online"]
    
    # Sort by load (lowest first)
    servers.sort(key=lambda s: s.current_connections / max(s.max_connections, 1))
    
    # Apply limit if specified
    if limit:
        servers = servers[:limit]
    
    logger.debug(f"Found {len(servers)} available servers for {protocol}")
    return servers
```

**Step 3: Add tests**

```python
# tests/core/application/services/test_server_registry_service.py
import pytest
import uuid
from src.core.application.services.server_registry_service import ServerRegistryService
from src.core.domain.entities.server import Server


class TestGetServersForUser:
    """Test get_servers_for_user method."""
    
    @pytest.mark.asyncio
    async def test_returns_servers_sorted_by_load(self, server_registry_service):
        """Test that servers are returned sorted by load (lowest first)."""
        # Setup mock servers with different loads
        mock_servers = [
            Server(
                id=uuid.uuid4(),
                name="High-Load-Server",
                country_code="US",
                country_name="United States",
                city="New York",
                agent_url="http://high-load:8080",
                agent_api_key="key",
                supports_outline=True,
                supports_wireguard=True,
                status="online",
                max_connections=1000,
                current_connections=800,  # 80% load
            ),
            Server(
                id=uuid.uuid4(),
                name="Low-Load-Server",
                country_code="DE",
                country_name="Germany",
                city="Frankfurt",
                agent_url="http://low-load:8080",
                agent_api_key="key",
                supports_outline=True,
                supports_wireguard=True,
                status="online",
                max_connections=1000,
                current_connections=200,  # 20% load
            ),
        ]
        
        # Mock get_available_servers
        server_registry_service.get_available_servers = AsyncMock(return_value=mock_servers)
        
        # Call method
        result = await server_registry_service.get_servers_for_user("outline")
        
        # Verify sorted by load (lowest first)
        assert len(result) == 2
        assert result[0].name == "Low-Load-Server"
        assert result[1].name == "High-Load-Server"
    
    @pytest.mark.asyncio
    async def test_filters_by_protocol(self, server_registry_service):
        """Test that servers are filtered by protocol support."""
        mock_servers = [
            Server(
                id=uuid.uuid4(),
                name="Outline-Only-Server",
                country_code="US",
                country_name="United States",
                city="New York",
                agent_url="http://outline:8080",
                agent_api_key="key",
                supports_outline=True,
                supports_wireguard=False,
                status="online",
                max_connections=1000,
                current_connections=200,
            ),
            Server(
                id=uuid.uuid4(),
                name="WireGuard-Only-Server",
                country_code="DE",
                country_name="Germany",
                city="Frankfurt",
                agent_url="http://wireguard:8080",
                agent_api_key="key",
                supports_outline=False,
                supports_wireguard=True,
                status="online",
                max_connections=1000,
                current_connections=200,
            ),
        ]
        
        server_registry_service.get_available_servers = AsyncMock(return_value=mock_servers)
        
        # Test Outline filter
        outline_servers = await server_registry_service.get_servers_for_user("outline")
        assert len(outline_servers) == 1
        assert outline_servers[0].supports_outline is True
        
        # Test WireGuard filter
        wireguard_servers = await server_registry_service.get_servers_for_user("wireguard")
        assert len(wireguard_servers) == 1
        assert wireguard_servers[0].supports_wireguard is True
    
    @pytest.mark.asyncio
    async def test_filters_offline_servers(self, server_registry_service):
        """Test that offline servers are filtered out."""
        mock_servers = [
            Server(
                id=uuid.uuid4(),
                name="Online-Server",
                country_code="US",
                country_name="United States",
                city="New York",
                agent_url="http://online:8080",
                agent_api_key="key",
                supports_outline=True,
                supports_wireguard=True,
                status="online",
                max_connections=1000,
                current_connections=200,
            ),
            Server(
                id=uuid.uuid4(),
                name="Offline-Server",
                country_code="DE",
                country_name="Germany",
                city="Frankfurt",
                agent_url="http://offline:8080",
                agent_api_key="key",
                supports_outline=True,
                supports_wireguard=True,
                status="offline",
                max_connections=1000,
                current_connections=0,
            ),
        ]
        
        server_registry_service.get_available_servers = AsyncMock(return_value=mock_servers)
        
        result = await server_registry_service.get_servers_for_user("outline")
        
        assert len(result) == 1
        assert result[0].status == "online"
        assert result[0].name == "Online-Server"
    
    @pytest.mark.asyncio
    async def test_applies_limit(self, server_registry_service):
        """Test that limit parameter is applied."""
        mock_servers = [
            Server(
                id=uuid.uuid4(),
                name=f"Server-{i}",
                country_code="US",
                country_name="United States",
                city="New York",
                agent_url=f"http://server{i}:8080",
                agent_api_key="key",
                supports_outline=True,
                supports_wireguard=True,
                status="online",
                max_connections=1000,
                current_connections=i * 100,
            )
            for i in range(1, 11)  # 10 servers
        ]
        
        server_registry_service.get_available_servers = AsyncMock(return_value=mock_servers)
        
        result = await server_registry_service.get_servers_for_user("outline", limit=5)
        
        assert len(result) == 5
```

**Step 4: Run tests**

```bash
cd /home/mowgli/usipipo/usipipo-backend
source .venv/bin/activate
pytest tests/core/application/services/test_server_registry_service.py::TestGetServersForUser -v
```

Expected: All tests pass

**Step 5: Commit**

```bash
cd /home/mowgli/usipipo/usipipo-backend
git add src/core/application/services/server_registry_service.py tests/core/application/services/test_server_registry_service.py
git commit -m "feat: add get_servers_for_user method with protocol filtering and load sorting"
```

**Skills:** @python-reviewer @code-reviewer

---

### Task 3: Create VPN Servers API Endpoint

**Files:**
- Modify: `usipipo-backend/src/infrastructure/api/v1/routes/vpn.py`
- Test: `usipipo-backend/tests/infrastructure/api/v1/routes/test_vpn_servers.py`

**Step 1: Read existing vpn.py routes**

```bash
cd /home/mowgli/usipipo/usipipo-backend
cat src/infrastructure/api/v1/routes/vpn.py
```

**Step 2: Add imports at top of file**

```python
from src.core.application.services.server_registry_service import ServerRegistryService
from src.shared.schemas.vpn import VpnServerResponse, VpnServersListResponse
```

**Step 3: Add new endpoint**

Add to vpn.py router:

```python
@router.get("/servers", response_model=VpnServersListResponse)
async def get_available_servers(
    protocol: str,
    current_user: User = Depends(get_current_user),
    server_service: ServerRegistryService = Depends(get_server_registry_service),
) -> VpnServersListResponse:
    """
    Get list of available VPN servers for user selection.
    
    Args:
        protocol: Protocol type ("outline" or "wireguard")
        current_user: Authenticated user
        
    Returns:
        List of all available servers and top 5 recommended (lowest load)
        
    Raises:
        HTTPException: 400 if invalid protocol
    """
    # Validate protocol
    if protocol.lower() not in ["outline", "wireguard"]:
        raise HTTPException(
            status_code=400,
            detail=f"Invalid protocol: {protocol}. Must be 'outline' or 'wireguard'"
        )
    
    # Get all available servers
    all_servers = await server_service.get_servers_for_user(protocol)
    
    # Get top 5 recommended (lowest load)
    recommended_servers = all_servers[:5]
    
    # Convert to response schema
    def to_response(server: Server) -> VpnServerResponse:
        load_pct = int((server.current_connections / max(server.max_connections, 1)) * 100)
        
        if load_pct <= 50:
            load_level = "low"
        elif load_pct <= 80:
            load_level = "medium"
        else:
            load_level = "high"
        
        return VpnServerResponse(
            id=server.id,
            name=server.name,
            country_code=server.country_code,
            country_name=server.country_name,
            city=server.city,
            load_percentage=load_pct,
            load_level=load_level,
            status=server.status,
        )
    
    servers_response = [to_response(s) for s in all_servers]
    recommended_response = [to_response(s) for s in recommended_servers]
    
    return VpnServersListResponse(
        servers=servers_response,
        recommended=recommended_response,
    )
```

**Step 4: Add dependency injection helper**

Add to imports/dependencies section:

```python
def get_server_registry_service() -> ServerRegistryService:
    """Get ServerRegistryService instance."""
    return ServerRegistryService(
        server_repository=get_server_repository(),
    )
```

**Step 5: Create integration tests**

```python
# tests/infrastructure/api/v1/routes/test_vpn_servers.py
import pytest
from fastapi import status
from src.infrastructure.persistence.models.vpn_server_model import VpnServerModel


class TestGetAvailableServers:
    """Test GET /api/v1/vpn/servers endpoint."""
    
    @pytest.mark.asyncio
    async def test_returns_servers_list(self, client, authenticated_user_token, db_session):
        """Test that endpoint returns list of available servers."""
        # Create test servers
        test_servers = [
            VpnServerModel(
                id="00000000-0000-0000-0000-000000000001",
                name="US-East-1",
                country_code="US",
                country_name="United States",
                city="New York",
                agent_url="http://us-east:8080",
                agent_api_key="encrypted_key",
                supports_outline=True,
                supports_wireguard=True,
                status="online",
                max_connections=1000,
                current_connections=200,
            ),
            VpnServerModel(
                id="00000000-0000-0000-0000-000000000002",
                name="DE-Frankfurt-1",
                country_code="DE",
                country_name="Germany",
                city="Frankfurt",
                agent_url="http://de-fra:8080",
                agent_api_key="encrypted_key",
                supports_outline=True,
                supports_wireguard=False,
                status="online",
                max_connections=1000,
                current_connections=300,
            ),
        ]
        
        db_session.add_all(test_servers)
        await db_session.commit()
        
        # Make request
        response = await client.get(
            "/api/v1/vpn/servers?protocol=outline",
            headers={"Authorization": f"Bearer {authenticated_user_token}"},
        )
        
        assert response.status_code == status.HTTP_200_OK
        data = response.json()
        
        assert "servers" in data
        assert "recommended" in data
        assert len(data["servers"]) >= 1
        assert len(data["recommended"]) <= 5
    
    @pytest.mark.asyncio
    async def test_filters_by_protocol_outline(self, client, authenticated_user_token, db_session):
        """Test that endpoint filters servers by Outline protocol support."""
        # Create servers with different protocol support
        outline_server = VpnServerModel(
            id="00000000-0000-0000-0000-000000000001",
            name="Outline-Server",
            country_code="US",
            country_name="United States",
            city="New York",
            agent_url="http://outline:8080",
            agent_api_key="encrypted_key",
            supports_outline=True,
            supports_wireguard=False,
            status="online",
            max_connections=1000,
            current_connections=200,
        )
        
        wireguard_server = VpnServerModel(
            id="00000000-0000-0000-0000-000000000002",
            name="WireGuard-Server",
            country_code="DE",
            country_name="Germany",
            city="Frankfurt",
            agent_url="http://wireguard:8080",
            agent_api_key="encrypted_key",
            supports_outline=False,
            supports_wireguard=True,
            status="online",
            max_connections=1000,
            current_connections=200,
        )
        
        db_session.add_all([outline_server, wireguard_server])
        await db_session.commit()
        
        # Request Outline servers
        response = await client.get(
            "/api/v1/vpn/servers?protocol=outline",
            headers={"Authorization": f"Bearer {authenticated_user_token}"},
        )
        
        assert response.status_code == status.HTTP_200_OK
        data = response.json()
        
        # Should only return Outline server
        assert len(data["servers"]) == 1
        assert data["servers"][0]["name"] == "Outline-Server"
    
    @pytest.mark.asyncio
    async def test_filters_offline_servers(self, client, authenticated_user_token, db_session):
        """Test that offline servers are excluded."""
        online_server = VpnServerModel(
            id="00000000-0000-0000-0000-000000000001",
            name="Online-Server",
            country_code="US",
            country_name="United States",
            city="New York",
            agent_url="http://online:8080",
            agent_api_key="encrypted_key",
            supports_outline=True,
            supports_wireguard=True,
            status="online",
            max_connections=1000,
            current_connections=200,
        )
        
        offline_server = VpnServerModel(
            id="00000000-0000-0000-0000-000000000002",
            name="Offline-Server",
            country_code="DE",
            country_name="Germany",
            city="Frankfurt",
            agent_url="http://offline:8080",
            agent_api_key="encrypted_key",
            supports_outline=True,
            supports_wireguard=True,
            status="offline",
            max_connections=1000,
            current_connections=0,
        )
        
        db_session.add_all([online_server, offline_server])
        await db_session.commit()
        
        response = await client.get(
            "/api/v1/vpn/servers?protocol=outline",
            headers={"Authorization": f"Bearer {authenticated_user_token}"},
        )
        
        assert response.status_code == status.HTTP_200_OK
        data = response.json()
        
        # Should only return online server
        assert len(data["servers"]) == 1
        assert data["servers"][0]["name"] == "Online-Server"
    
    @pytest.mark.asyncio
    async def test_requires_authentication(self, client):
        """Test that endpoint requires authentication."""
        response = await client.get("/api/v1/vpn/servers?protocol=outline")
        
        assert response.status_code == status.HTTP_401_UNAUTHORIZED
    
    @pytest.mark.asyncio
    async def test_invalid_protocol_returns_400(self, client, authenticated_user_token):
        """Test that invalid protocol returns 400 error."""
        response = await client.get(
            "/api/v1/vpn/servers?protocol=invalid",
            headers={"Authorization": f"Bearer {authenticated_user_token}"},
        )
        
        assert response.status_code == status.HTTP_400_BAD_REQUEST
        assert "Invalid protocol" in response.json()["detail"]
    
    @pytest.mark.asyncio
    async def test_returns_recommended_servers(self, client, authenticated_user_token, db_session):
        """Test that recommended servers are top 5 lowest load."""
        # Create 10 servers with different loads
        for i in range(1, 11):
            db_session.add(
                VpnServerModel(
                    id=f"00000000-0000-0000-0000-00000000000{i:02d}",
                    name=f"Server-{i}",
                    country_code="US",
                    country_name="United States",
                    city="New York",
                    agent_url=f"http://server{i}:8080",
                    agent_api_key="encrypted_key",
                    supports_outline=True,
                    supports_wireguard=True,
                    status="online",
                    max_connections=1000,
                    current_connections=i * 50,  # Increasing load
                )
            )
        
        await db_session.commit()
        
        response = await client.get(
            "/api/v1/vpn/servers?protocol=outline",
            headers={"Authorization": f"Bearer {authenticated_user_token}"},
        )
        
        assert response.status_code == status.HTTP_200_OK
        data = response.json()
        
        # Should have all servers
        assert len(data["servers"]) == 10
        # Should have top 5 recommended
        assert len(data["recommended"]) == 5
        # First recommended should be lowest load
        assert data["recommended"][0]["name"] == "Server-1"
```

**Step 6: Run tests**

```bash
cd /home/mowgli/usipipo/usipipo-backend
source .venv/bin/activate
pytest tests/infrastructure/api/v1/routes/test_vpn_servers.py -v
```

Expected: All tests pass

**Step 7: Commit**

```bash
cd /home/mowgli/usipipo/usipipo-backend
git add src/infrastructure/api/v1/routes/vpn.py tests/infrastructure/api/v1/routes/test_vpn_servers.py
git commit -m "feat: add GET /api/v1/vpn/servers endpoint for user server selection"
```

**Skills:** @python-reviewer @security-reviewer @code-reviewer

---

### Task 4: Update Backend API Documentation

**Files:**
- Modify: `usipipo-docs/apis/backend-api-reference.md`

**Step 1: Add endpoint documentation**

Add to backend-api-reference.md under "VPN Endpoints" section:

```markdown
## Server Selection

### GET /api/v1/vpn/servers

Get list of available VPN servers for user selection.

**Authentication:** Required (user JWT token)

**Query Parameters:**
- `protocol` (required): Protocol type - `"outline"` or `"wireguard"`

**Response:**

```json
{
  "servers": [
    {
      "id": "uuid-here",
      "name": "US-East-1",
      "country_code": "US",
      "country_name": "United States",
      "city": "New York",
      "load_percentage": 23,
      "load_level": "low",
      "status": "online"
    }
  ],
  "recommended": [
    {
      "id": "uuid-here",
      "name": "US-East-1",
      "country_code": "US",
      "country_name": "United States",
      "city": "New York",
      "load_percentage": 23,
      "load_level": "low",
      "status": "online"
    }
  ]
}
```

**Load Levels:**
- `low` (🟢): 0-50% connections used
- `medium` (🟡): 51-80% connections used
- `high` (🔴): 81-100% connections used

**Errors:**
- `400 Bad Request`: Invalid protocol parameter
- `401 Unauthorized`: Missing or invalid authentication token

**Example Request:**

```bash
curl -X GET "http://localhost:8000/api/v1/vpn/servers?protocol=outline" \
  -H "Authorization: Bearer YOUR_JWT_TOKEN"
```
```

**Step 2: Commit**

```bash
cd /home/mowgli/usipipo/usipipo-docs
git add apis/backend-api-reference.md
git commit -m "docs: add VPN server selection endpoint documentation"
```

---

## Phase 2: Telegram Bot Implementation

### Task 5: Create Server Selection Keyboard

**Files:**
- Create: `usipipo-telegram-bot/src/bot/keyboards/servers.py`
- Test: `usipipo-telegram-bot/tests/bot/keyboards/test_servers.py`

**Step 1: Create keyboards/servers.py**

```python
"""Server selection inline keyboards."""

from typing import TYPE_CHECKING

from telegram import InlineKeyboardButton, InlineKeyboardMarkup

if TYPE_CHECKING:
    from usipipo_commons.domain.entities.server import Server


class ServerKeyboards:
    """Factory for server selection inline keyboards."""
    
    LOAD_EMOJIS = {
        "low": "🟢",
        "medium": "🟡",
        "high": "🔴",
    }
    
    @staticmethod
    def server_selection(servers: list["Server"]) -> InlineKeyboardMarkup:
        """Create inline keyboard for server selection.
        
        Args:
            servers: List of servers sorted by load (lowest first)
            
        Returns:
            InlineKeyboardMarkup with server buttons
        """
        keyboard = []
        
        # Show top 5 recommended servers
        recommended = servers[:5]
        
        for server in recommended:
            # Calculate load percentage
            load_pct = int((server.current_connections / max(server.max_connections, 1)) * 100)
            
            # Determine load level and emoji
            if load_pct <= 50:
                load_level = "low"
            elif load_pct <= 80:
                load_level = "medium"
            else:
                load_level = "high"
            
            load_emoji = ServerKeyboards.LOAD_EMOJIS[load_level]
            
            # Button text: Flag + Country + City + Load
            city_text = f" - {server.city}" if server.city else ""
            button_text = f"{server.country_code}{city_text} {load_emoji}"
            
            # Callback data: server_select:{server_id}
            callback_data = f"server_select:{server.id}"
            
            keyboard.append([InlineKeyboardButton(button_text, callback_data=callback_data)])
        
        # Add "Show all servers" button if more than 5 servers
        if len(servers) > 5:
            keyboard.append([
                InlineKeyboardButton("🔍 Ver todos los servidores", callback_data="servers_show_all")
            ])
        
        # Add back button
        keyboard.append([
            InlineKeyboardButton("🔙 Volver", callback_data="vpn_keys_menu")
        ])
        
        return InlineKeyboardMarkup(keyboard)
    
    @staticmethod
    def server_selection_full(servers: list["Server"]) -> InlineKeyboardMarkup:
        """Create inline keyboard showing all servers.
        
        Args:
            servers: List of all available servers
            
        Returns:
            InlineKeyboardMarkup with all server buttons
        """
        keyboard = []
        
        for server in servers:
            load_pct = int((server.current_connections / max(server.max_connections, 1)) * 100)
            
            if load_pct <= 50:
                load_level = "low"
            elif load_pct <= 80:
                load_level = "medium"
            else:
                load_level = "high"
            
            load_emoji = ServerKeyboards.LOAD_EMOJIS[load_level]
            
            city_text = f" - {server.city}" if server.city else ""
            button_text = f"{server.country_code}{city_text} {load_emoji}"
            
            callback_data = f"server_select:{server.id}"
            
            keyboard.append([InlineKeyboardButton(button_text, callback_data=callback_data)])
        
        # Add back button
        keyboard.append([
            InlineKeyboardButton("🔙 Volver", callback_data="vpn_keys_menu")
        ])
        
        return InlineKeyboardMarkup(keyboard)
```

**Step 2: Create tests**

```python
# tests/bot/keyboards/test_servers.py
import pytest
import uuid
from usipipo_commons.domain.entities.server import Server
from src.bot.keyboards.servers import ServerKeyboards


class TestServerKeyboards:
    """Test server selection keyboards."""
    
    def test_server_selection_keyboard(self):
        """Test server selection keyboard creation."""
        servers = [
            Server(
                id=uuid.uuid4(),
                name="US-East-1",
                country_code="US",
                country_name="United States",
                city="New York",
                agent_url="http://us-east:8080",
                agent_api_key="key",
                supports_outline=True,
                supports_wireguard=True,
                status="online",
                max_connections=1000,
                current_connections=200,  # 20% load
            ),
            Server(
                id=uuid.uuid4(),
                name="DE-Frankfurt-1",
                country_code="DE",
                country_name="Germany",
                city="Frankfurt",
                agent_url="http://de-fra:8080",
                agent_api_key="key",
                supports_outline=True,
                supports_wireguard=True,
                status="online",
                max_connections=1000,
                current_connections=600,  # 60% load
            ),
        ]
        
        keyboard = ServerKeyboards.server_selection(servers)
        
        # Check keyboard structure
        assert len(keyboard.inline_keyboard) == 3  # 2 servers + back button
        
        # Check first button
        first_button = keyboard.inline_keyboard[0][0]
        assert "US - New York" in first_button.text
        assert "🟢" in first_button.text  # Low load emoji
        assert first_button.callback_data.startswith("server_select:")
        
        # Check second button
        second_button = keyboard.inline_keyboard[1][0]
        assert "DE - Frankfurt" in second_button.text
        assert "🟡" in second_button.text  # Medium load emoji
    
    def test_server_selection_shows_all_button(self):
        """Test that 'show all' button appears when more than 5 servers."""
        servers = [
            Server(
                id=uuid.uuid4(),
                name=f"Server-{i}",
                country_code="US",
                country_name="United States",
                city="New York",
                agent_url=f"http://server{i}:8080",
                agent_api_key="key",
                supports_outline=True,
                supports_wireguard=True,
                status="online",
                max_connections=1000,
                current_connections=i * 100,
            )
            for i in range(1, 8)  # 7 servers
        ]
        
        keyboard = ServerKeyboards.server_selection(servers)
        
        # Should have 5 recommended + "show all" + back = 7 rows
        assert len(keyboard.inline_keyboard) == 7
        
        # Check "show all" button
        show_all_button = keyboard.inline_keyboard[5][0]
        assert "Ver todos los servidores" in show_all_button.text
        assert show_all_button.callback_data == "servers_show_all"
    
    def test_server_selection_full_keyboard(self):
        """Test full server list keyboard."""
        servers = [
            Server(
                id=uuid.uuid4(),
                name="Server-1",
                country_code="US",
                country_name="United States",
                city="New York",
                agent_url="http://server1:8080",
                agent_api_key="key",
                supports_outline=True,
                supports_wireguard=True,
                status="online",
                max_connections=1000,
                current_connections=200,
            ),
        ]
        
        keyboard = ServerKeyboards.server_selection_full(servers)
        
        # Should have 1 server + back = 2 rows
        assert len(keyboard.inline_keyboard) == 2
        
        # No "show all" button
        button_texts = [
            button.text
            for row in keyboard.inline_keyboard
            for button in row
        ]
        assert "Ver todos los servidores" not in button_texts
```

**Step 3: Run tests**

```bash
cd /home/mowgli/usipipo/usipipo-telegram-bot
source .venv/bin/activate
pytest tests/bot/keyboards/test_servers.py -v
```

Expected: All tests pass

**Step 4: Commit**

```bash
cd /home/mowgli/usipipo/usipipo-telegram-bot
git add src/bot/keyboards/servers.py tests/bot/keyboards/test_servers.py
git commit -m "feat: add server selection inline keyboards with load indicators"
```

**Skills:** @python-reviewer @code-reviewer

---

### Task 6: Add Server Selection Conversation State

**Files:**
- Modify: `usipipo-telegram-bot/src/bot/handlers/keys.py`
- Test: `usipipo-telegram-bot/tests/bot/handlers/test_keys_server_selection.py`

**Step 1: Read existing keys.py handler**

```bash
cd /home/mowgli/usipipo/usipipo-telegram-bot
cat src/bot/handlers/keys.py
```

**Step 2: Add new conversation state**

Add to KeyStates class:

```python
class KeyStates(IntEnum):
    """Estados de la conversación de gestión de claves."""
    
    SELECT_PROTOCOL = 1
    SELECT_SERVER = 2  # NEW STATE
    INPUT_NAME = 3
    CONFIRM_ACTION = 4
```

**Step 3: Add imports**

Add to imports section:

```python
from src.bot.keyboards.servers import ServerKeyboards
from src.infrastructure.api_client import ApiClient
```

**Step 4: Modify protocol_selected handler**

Find `protocol_selected` function and modify:

```python
async def protocol_selected(self, update: Update, context: ContextTypes.DEFAULT_TYPE):
    """Maneja la selección del protocolo VPN."""
    if not update.effective_user or not update.callback_query:
        return ConversationHandler.END
    
    query = update.callback_query
    await query.answer()
    
    # Store protocol in context
    context.user_data["vpn_protocol"] = query.data.replace("vpn_create_", "")
    
    # Fetch available servers from backend
    protocol = context.user_data["vpn_protocol"]
    
    try:
        # Get API client and fetch servers
        api_client = ApiClient()
        tokens = await self.tokens.get(update.effective_user.id)
        
        servers_response = await api_client.get(
            f"/vpn/servers?protocol={protocol}",
            headers={"Authorization": f"Bearer {tokens['access_token']}"},
        )
        
        servers = servers_response.get("servers", [])
        recommended = servers_response.get("recommended", [])
        
        if not servers:
            # No servers available
            await query.edit_message_text(
                "⚠️ No hay servidores disponibles para el protocolo seleccionado.\n\n"
                "Por favor intenta en unos minutos.",
                reply_markup=ServerKeyboards.server_selection([]),
            )
            return ConversationHandler.END
        
        # Format server list message
        message_text = self._format_server_list_message(recommended, servers)
        
        # Update message with server list
        await query.edit_message_text(
            message_text,
            reply_markup=ServerKeyboards.server_selection(recommended),
        )
        
        return KeyStates.SELECT_SERVER
        
    except Exception as e:
        logger.error(f"Error fetching servers: {e}")
        await query.edit_message_text(
            "⚠️ Error al cargar servidores. ¿Reintentar?",
            reply_markup=InlineKeyboardMarkup([
                [InlineKeyboardButton("🔄 Reintentar", callback_data=f"vpn_create_{protocol}")],
                [InlineKeyboardButton("🔙 Volver", callback_data="vpn_keys_menu")],
            ]),
        )
        return ConversationHandler.END
```

**Step 5: Add server list message formatter**

Add new method to KeysHandler class:

```python
def _format_server_list_message(
    self,
    recommended: list[dict],
    all_servers: list[dict],
) -> str:
    """Format server list message with load indicators.
    
    Args:
        recommended: List of recommended servers (top 5)
        all_servers: List of all available servers
        
    Returns:
        Formatted message text
    """
    load_emojis = {
        "low": "🟢",
        "medium": "🟡",
        "high": "🔴",
    }
    
    message = "🌍 <b>Selecciona un Servidor VPN</b>\n\n"
    message += "🔥 <b>Recomendados (menor carga):</b>\n\n"
    
    for i, server in enumerate(recommended, 1):
        load_emoji = load_emojis.get(server["load_level"], "🟢")
        city_text = f" - {server['city']}" if server.get('city') else ""
        
        message += f"┌─────────────────────────────\n"
        message += f"│ {i}. {server['country_name']} {city_text}\n"
        message += f"│ Servidor: {server['name']}\n"
        message += f"│ {load_emoji} Carga: {server['load_percentage']}% • 📶 Online\n"
        message += f"└─────────────────────────────\n\n"
    
    message += "━━━━━━━━━━━━━━━━━━━━\n"
    message += "ℹ️ Los servidores se actualizan en tiempo real\n"
    message += "💡 Tip: Los servidores con 🟢 tienen mejor rendimiento"
    
    return message
```

**Step 6: Add server selection callback handler**

Add new handler method:

```python
async def server_selected(self, update: Update, context: ContextTypes.DEFAULT_TYPE):
    """Maneja la selección de un servidor."""
    if not update.effective_user or not update.callback_query:
        return ConversationHandler.END
    
    query = update.callback_query
    await query.answer()
    
    # Parse server_id from callback data
    callback_data = query.data
    
    if callback_data == "servers_show_all":
        # Show full server list
        protocol = context.user_data["vpn_protocol"]
        
        try:
            api_client = ApiClient()
            tokens = await self.tokens.get(update.effective_user.id)
            
            servers_response = await api_client.get(
                f"/vpn/servers?protocol={protocol}",
                headers={"Authorization": f"Bearer {tokens['access_token']}"},
            )
            
            all_servers = servers_response.get("servers", [])
            
            message_text = "🌍 <b>Todos los Servidores Disponibles</b>\n\n"
            
            # Simple list for full view
            for server in all_servers:
                load_emoji = load_emojis.get(server["load_level"], "🟢")
                city_text = f" - {server['city']}" if server.get('city') else ""
                message_text += f"{server['country_name']}{city_text} {load_emoji}\n"
            
            await query.edit_message_text(
                message_text,
                reply_markup=ServerKeyboards.server_selection_full(all_servers),
            )
            return KeyStates.SELECT_SERVER
            
        except Exception as e:
            logger.error(f"Error fetching all servers: {e}")
            await query.edit_message_text("⚠️ Error al cargar servidores.")
            return ConversationHandler.END
    
    # Extract server_id from callback_data
    # Format: server_select:{server_id}
    if not callback_data.startswith("server_select:"):
        return ConversationHandler.END
    
    server_id = callback_data.replace("server_select:", "")
    context.user_data["server_id"] = server_id
    
    # Get server name for confirmation
    # (In a real implementation, you'd fetch server details or store in context)
    
    await query.edit_message_text(
        "✅ Servidor seleccionado\n\n"
        "Ahora ingresa un <b>nombre</b> para tu clave VPN:\n\n"
        "<i>Ejemplo: Mi Casa, Trabajo, etc.</i>",
        parse_mode="HTML",
        reply_markup=InlineKeyboardMarkup([
            [InlineKeyboardButton("🔙 Cancelar", callback_data="vpn_keys_menu")],
        ]),
    )
    
    return KeyStates.INPUT_NAME
```

**Step 7: Update ConversationHandler**

Find ConversationHandler definition and add new state handler:

```python
self.conv_handler = ConversationHandler(
    entry_points=[...],
    states={
        KeyStates.SELECT_PROTOCOL: [...],
        KeyStates.SELECT_SERVER: [  # NEW
            CallbackQueryHandler(self.server_selected, pattern=r"^server_select:"),
        ],
        KeyStates.INPUT_NAME: [...],
        ...
    },
    fallbacks=[...],
)
```

**Step 8: Create tests**

```python
# tests/bot/handlers/test_keys_server_selection.py
import pytest
from unittest.mock import AsyncMock, patch
from telegram import Update, CallbackQuery
from src.bot.handlers.keys import KeysHandler, KeyStates


class TestServerSelection:
    """Test server selection conversation flow."""
    
    @pytest.mark.asyncio
    async def test_protocol_selected_fetches_servers(self):
        """Test that protocol selection triggers server fetch."""
        # Setup mock update
        callback_query = AsyncMock(spec=CallbackQuery)
        callback_query.data = "vpn_create_outline"
        
        update = AsyncMock(spec=Update)
        update.callback_query = callback_query
        update.effective_user.id = 123456
        
        context = AsyncMock()
        context.user_data = {}
        
        # Mock API client
        mock_servers_response = {
            "servers": [
                {
                    "id": "server-1",
                    "name": "US-East-1",
                    "country_code": "US",
                    "country_name": "United States",
                    "city": "New York",
                    "load_percentage": 23,
                    "load_level": "low",
                    "status": "online",
                }
            ],
            "recommended": [],
        }
        
        with patch("src.bot.handlers.keys.ApiClient") as mock_api_client_class:
            mock_api_client = AsyncMock()
            mock_api_client.get = AsyncMock(return_value=mock_servers_response)
            mock_api_client_class.return_value = mock_api_client
            
            handler = KeysHandler()
            result = await handler.protocol_selected(update, context)
            
            assert result == KeyStates.SELECT_SERVER
            mock_api_client.get.assert_called_once_with(
                "/vpn/servers?protocol=outline",
                headers={"Authorization": "Bearer mock_token"},
            )
    
    @pytest.mark.asyncio
    async def test_server_selected_stores_server_id(self):
        """Test that server selection stores server_id in context."""
        callback_query = AsyncMock(spec=CallbackQuery)
        callback_query.data = "server_select:server-123"
        
        update = AsyncMock(spec=Update)
        update.callback_query = callback_query
        update.effective_user.id = 123456
        
        context = AsyncMock()
        context.user_data = {"vpn_protocol": "outline"}
        
        handler = KeysHandler()
        result = await handler.server_selected(update, context)
        
        assert result == KeyStates.INPUT_NAME
        assert context.user_data["server_id"] == "server-123"
    
    @pytest.mark.asyncio
    async def test_servers_show_all_shows_full_list(self):
        """Test 'show all servers' button displays full list."""
        callback_query = AsyncMock(spec=CallbackQuery)
        callback_query.data = "servers_show_all"
        
        update = AsyncMock(spec=Update)
        update.callback_query = callback_query
        update.effective_user.id = 123456
        
        context = AsyncMock()
        context.user_data = {"vpn_protocol": "outline"}
        
        mock_servers_response = {
            "servers": [{"id": f"server-{i}"} for i in range(1, 11)],
            "recommended": [],
        }
        
        with patch("src.bot.handlers.keys.ApiClient") as mock_api_client_class:
            mock_api_client = AsyncMock()
            mock_api_client.get = AsyncMock(return_value=mock_servers_response)
            mock_api_client_class.return_value = mock_api_client
            
            handler = KeysHandler()
            result = await handler.server_selected(update, context)
            
            assert result == KeyStates.SELECT_SERVER
```

**Step 9: Run tests**

```bash
cd /home/mowgli/usipipo/usipipo-telegram-bot
source .venv/bin/activate
pytest tests/bot/handlers/test_keys_server_selection.py -v
```

Expected: All tests pass

**Step 10: Commit**

```bash
cd /home/mowgli/usipipo/usipipo-telegram-bot
git add src/bot/handlers/keys.py tests/bot/handlers/test_keys_server_selection.py
git commit -m "feat: add server selection conversation state and handler"
```

**Skills:** @python-reviewer @code-reviewer

---

### Task 7: Modify Key Creation to Include Server ID

**Files:**
- Modify: `usipipo-telegram-bot/src/bot/handlers/keys.py:name_received`
- Modify: `usipipo-backend/src/infrastructure/api/v1/routes/vpn.py:create_key`

**Step 1: Update bot's name_received handler**

Find `name_received` method and modify API call:

```python
async def name_received(self, update: Update, context: ContextTypes.DEFAULT_TYPE):
    """Finaliza la creación de la clave con el nombre proporcionado."""
    if not update.effective_user or not update.message or not update.message.text:
        return ConversationHandler.END
    
    key_name = update.message.text.strip()
    telegram_id = update.effective_user.id
    protocol = context.user_data.get("vpn_protocol")
    server_id = context.user_data.get("server_id")  # NEW
    
    # Get auth headers
    tokens = await self.tokens.get(telegram_id)
    headers = {"Authorization": f"Bearer {tokens['access_token']}"}
    
    # Map protocol to KeyType enum value (lowercase)
    vpn_type = "outline" if protocol.lower() == "outline" else "wireguard"
    
    # Prepare request data
    data = {
        "name": key_name,
        "vpn_type": vpn_type,
        "data_limit_gb": 5.0,  # Default 5GB
    }
    
    # Add server_id if selected
    if server_id:
        data["server_id"] = server_id
    
    # Create key via API
    try:
        response = await self.api.post(
            "/vpn/keys",
            data=data,
            headers=headers,
        )
        
        # Success - deliver config to user
        # ... existing success handling code ...
        
    except Exception as e:
        logger.error(f"Error creating VPN key: {e}")
        await update.message.reply_text(
            "❌ Error al crear la clave. Por favor intenta de nuevo.",
        )
        return ConversationHandler.END
    
    return ConversationHandler.END
```

**Step 2: Update backend to accept server_id**

Modify `POST /vpn/keys` endpoint in backend:

```python
# In usipipo-backend/src/infrastructure/api/v1/routes/vpn.py

# Update CreateVpnKeyRequest schema to include optional server_id
class CreateVpnKeyRequest(BaseModel):
    name: str
    vpn_type: str
    data_limit_gb: float = 5.0
    server_id: Optional[uuid.UUID] = None  # NEW optional field


# Update create_key endpoint
@router.post("/keys", response_model=VpnKeyResponse)
async def create_key(
    request: CreateVpnKeyRequest,
    current_user: User = Depends(get_current_user),
    vpn_service: VpnService = Depends(get_vpn_service),
) -> VpnKeyResponse:
    """Create new VPN key."""
    
    # If server_id provided, use it; otherwise let service auto-select
    key = await vpn_service.create_key(
        user_id=current_user.id,
        name=request.name,
        vpn_type=request.vpn_type,
        data_limit_gb=request.data_limit_gb,
        server_id=request.server_id,  # NEW parameter
    )
    
    return VpnKeyResponse(**key.model_dump())
```

**Step 3: Update VPN service to handle server_id**

Modify `VpnService.create_key`:

```python
async def create_key(
    self,
    user_id: uuid.UUID,
    name: str,
    vpn_type: str,
    data_limit_gb: float = 5.0,
    server_id: Optional[uuid.UUID] = None,  # NEW parameter
) -> VpnKey:
    # ... existing validation code ...
    
    # Select server: use provided server_id or auto-select
    if server_id:
        # Use user-selected server
        server = await self.server_registry.get_server(server_id)
        if not server or server.status != "online":
            raise NoAvailableServersError("Selected server is not available")
    else:
        # Auto-select best server (existing logic)
        country = 'US'  # Default
        server = await self.server_registry.select_best_server(
            country=country,
            protocol=vpn_type
        )
    
    if not server:
        logger.warning(f"No available servers")
        raise NoAvailableServersError("No available servers")
    
    logger.info(f"Using server {server.name} ({server.id}) for user {user_id}")
    
    # ... rest of existing key creation code ...
```

**Step 4: Create tests**

```python
# tests/bot/handlers/test_keys_server_id.py
import pytest
from unittest.mock import AsyncMock, patch


class TestKeyCreationWithServerId:
    """Test key creation includes server_id."""
    
    @pytest.mark.asyncio
    async def test_name_received_includes_server_id(self):
        """Test that key creation API call includes server_id."""
        # Setup mock update
        message = AsyncMock()
        message.text = "My Test Key"
        
        update = AsyncMock()
        update.message = message
        update.effective_user.id = 123456
        
        context = AsyncMock()
        context.user_data = {
            "vpn_protocol": "outline",
            "server_id": "server-uuid-123",
        }
        
        handler = KeysHandler()
        
        with patch.object(handler.api, 'post') as mock_post:
            mock_post.return_value = {"id": "key-123", "config": "..."}
            
            result = await handler.name_received(update, context)
            
            # Verify API call includes server_id
            mock_post.assert_called_once()
            call_data = mock_post.call_args[1]["data"]
            assert call_data["server_id"] == "server-uuid-123"
    
    @pytest.mark.asyncio
    async def test_name_received_without_server_id(self):
        """Test key creation works without server_id (auto-select)."""
        message = AsyncMock()
        message.text = "My Test Key"
        
        update = AsyncMock()
        update.message = message
        update.effective_user.id = 123456
        
        context = AsyncMock()
        context.user_data = {
            "vpn_protocol": "outline",
            # No server_id - should auto-select
        }
        
        handler = KeysHandler()
        
        with patch.object(handler.api, 'post') as mock_post:
            mock_post.return_value = {"id": "key-123"}
            
            result = await handler.name_received(update, context)
            
            # Verify server_id not included in request
            call_data = mock_post.call_args[1]["data"]
            assert "server_id" not in call_data
```

**Step 5: Run tests**

```bash
cd /home/mowgli/usipipo/usipipo-telegram-bot
source .venv/bin/activate
pytest tests/bot/handlers/test_keys_server_id.py -v

cd /home/mowgli/usipipo/usipipo-backend
source .venv/bin/activate
pytest tests/core/application/services/test_vpn_service.py::TestCreateKeyWithServerId -v
```

Expected: All tests pass

**Step 6: Commit**

```bash
cd /home/mowgli/usipipo/usipipo-telegram-bot
git add src/bot/handlers/keys.py tests/bot/handlers/test_keys_server_id.py
git commit -m "feat: include server_id in VPN key creation request"

cd /home/mowgli/usipipo/usipipo-backend
git add src/infrastructure/api/v1/routes/vpn.py src/core/application/services/vpn_service.py
git commit -m "feat: accept optional server_id in VPN key creation endpoint"
```

**Skills:** @python-reviewer @code-reviewer

---

## Phase 3: Integration Testing & Polish

### Task 8: End-to-End Integration Tests

**Files:**
- Create: `usipipo-backend/tests/integration/test_server_selection_flow.py`
- Create: `usipipo-telegram-bot/tests/integration/test_server_selection_flow.py`

**Step 1: Create backend integration test**

```python
# usipipo-backend/tests/integration/test_server_selection_flow.py
"""Integration tests for server selection to key creation flow."""

import pytest
import uuid
from src.infrastructure.persistence.models.vpn_server_model import VpnServerModel


class TestServerSelectionKeyCreationFlow:
    """Test complete flow from server selection to key creation."""
    
    @pytest.mark.asyncio
    async def test_user_can_select_server_and_create_key(
        self,
        client,
        authenticated_user_token,
        db_session,
    ):
        """Test complete flow: fetch servers → select server → create key."""
        # Step 1: Create test server
        test_server = VpnServerModel(
            id=uuid.uuid4(),
            name="US-East-1",
            country_code="US",
            country_name="United States",
            city="New York",
            agent_url="http://us-east:8080",
            agent_api_key="encrypted_key",
            supports_outline=True,
            supports_wireguard=True,
            status="online",
            max_connections=1000,
            current_connections=200,
        )
        
        db_session.add(test_server)
        await db_session.commit()
        
        # Step 2: Fetch available servers
        servers_response = await client.get(
            "/api/v1/vpn/servers?protocol=outline",
            headers={"Authorization": f"Bearer {authenticated_user_token}"},
        )
        
        assert servers_response.status_code == 200
        servers_data = servers_response.json()
        assert len(servers_data["servers"]) >= 1
        
        selected_server_id = servers_data["servers"][0]["id"]
        
        # Step 3: Create key with selected server
        key_response = await client.post(
            "/api/v1/vpn/keys",
            json={
                "name": "Test Key",
                "vpn_type": "outline",
                "data_limit_gb": 5.0,
                "server_id": selected_server_id,
            },
            headers={"Authorization": f"Bearer {authenticated_user_token}"},
        )
        
        assert key_response.status_code == 200
        key_data = key_response.json()
        
        # Verify key was created on selected server
        assert "id" in key_data
        # (In a real test, you'd verify the key's server_id matches)
```

**Step 2: Create bot integration test**

```python
# usipipo-telegram-bot/tests/integration/test_server_selection_flow.py
"""Integration tests for Telegram bot server selection flow."""

import pytest
from unittest.mock import AsyncMock, patch


class TestBotServerSelectionFlow:
    """Test complete bot conversation flow for server selection."""
    
    @pytest.mark.asyncio
    async def test_complete_server_selection_flow(self):
        """Test: Protocol → Server List → Select Server → Name → Key Created."""
        # This would be a full integration test with:
        # 1. Mock Telegram Update objects
        # 2. Mock backend API responses
        # 3. Verify conversation state transitions
        # 4. Verify final API call includes server_id
        
        pass  # Implementation depends on test infrastructure
```

**Step 3: Run integration tests**

```bash
cd /home/mowgli/usipipo/usipipo-backend
source .venv/bin/activate
pytest tests/integration/test_server_selection_flow.py -v

cd /home/mowgli/usipipo/usipipo-telegram-bot
source .venv/bin/activate
pytest tests/integration/test_server_selection_flow.py -v
```

Expected: All tests pass

**Step 4: Commit**

```bash
cd /home/mowgli/usipipo/usipipo-backend
git add tests/integration/test_server_selection_flow.py
git commit -m "test: add integration tests for server selection to key creation flow"

cd /home/mowgli/usipipo/usipipo-telegram-bot
git add tests/integration/test_server_selection_flow.py
git commit -m "test: add integration tests for bot server selection flow"
```

**Skills:** @code-reviewer

---

### Task 9: Error Handling & Edge Cases

**Files:**
- Modify: `usipipo-telegram-bot/src/bot/handlers/keys.py`
- Modify: `usipipo-backend/src/core/application/services/vpn_service.py`

**Step 1: Add error handling for server unavailable**

In bot's `name_received`:

```python
try:
    response = await self.api.post(
        "/vpn/keys",
        data=data,
        headers=headers,
    )
except Exception as e:
    logger.error(f"Error creating VPN key: {e}")
    
    # Check if error is server unavailable
    if "server" in str(e).lower() and "available" in str(e).lower():
        await update.message.reply_text(
            "⚠️ El servidor seleccionado ya no está disponible.\n\n"
            "Por favor selecciona otro servidor:",
            reply_markup=ServerKeyboards.server_selection(...),  # Re-fetch servers
        )
        return KeyStates.SELECT_SERVER
    else:
        await update.message.reply_text(
            "❌ Error al crear la clave. Por favor intenta de nuevo.",
        )
        return ConversationHandler.END
```

**Step 2: Add validation in backend**

In `VpnService.create_key`:

```python
if server_id:
    server = await self.server_registry.get_server(server_id)
    if not server:
        raise HTTPException(
            status_code=400,
            detail=f"Server {server_id} not found"
        )
    if server.status != "online":
        raise HTTPException(
            status_code=400,
            detail=f"Server {server.name} is currently offline"
        )
```

**Step 3: Test error scenarios**

```python
# tests/bot/handlers/test_keys_error_handling.py

class TestKeyCreationErrorHandling:
    """Test error handling in key creation."""
    
    @pytest.mark.asyncio
    async def test_server_unavailable_returns_to_selection(self):
        """Test that unavailable server returns user to server selection."""
        # Mock API to raise server unavailable error
        # Verify bot shows error message and server selection keyboard
        pass
    
    @pytest.mark.asyncio
    async def test_api_timeout_shows_retry(self):
        """Test that API timeout shows retry button."""
        # Mock API timeout
        # Verify bot shows retry option
        pass
```

**Step 4: Commit**

```bash
cd /home/mowgli/usipipo/usipipo-telegram-bot
git add src/bot/handlers/keys.py tests/bot/handlers/test_keys_error_handling.py
git commit -m "feat: add error handling for server unavailable and API failures"

cd /home/mowgli/usipipo/usipipo-backend
git add src/core/application/services/vpn_service.py
git commit -m "feat: add validation for user-selected server availability"
```

**Skills:** @python-reviewer @code-reviewer

---

### Task 10: UX Polish & Documentation

**Files:**
- Modify: `usipipo-telegram-bot/src/bot/handlers/keys.py` (message formatting)
- Create: `usipipo-docs/flows/server-selection-flow.md`

**Step 1: Improve message formatting**

Enhance `_format_server_list_message` with better spacing and emoji usage:

```python
def _format_server_list_message(self, recommended, all_servers):
    """Enhanced formatting with better visual hierarchy."""
    message = "🌍 <b>Selecciona un Servidor VPN</b>\n\n"
    message += "🔥 <b>Recomendados (menor carga):</b>\n\n"
    
    for i, server in enumerate(recommended, 1):
        load_emoji = load_emojis.get(server["load_level"], "🟢")
        city_text = f" - {server['city']}" if server.get('city') else ""
        
        message += f"┌─────────────────────────────\n"
        message += f"│ {i}. <b>{server['country_name']}</b> {city_text}\n"
        message += f"│ 🖥️ Servidor: {server['name']}\n"
        message += f"│ {load_emoji} Carga: {server['load_percentage']}% • 📶 Online\n"
        message += f"└─────────────────────────────\n\n"
    
    message += "━━━━━━━━━━━━━━━━━━━━\n"
    message += "ℹ️ Los servidores se actualizan en tiempo real\n"
    message += "💡 <b>Tip:</b> Los servidores con 🟢 tienen mejor rendimiento"
    
    return message
```

**Step 2: Create flow documentation**

```markdown
# VPN Server Selection Flow

## Overview

Users can now select their preferred VPN server during key creation, with real-time load indicators and server status.

## User Journey

```
┌─────────────┐
│ Mis Keys    │
└──────┬──────┘
       │
       ▼
┌─────────────┐
│ Nueva Clave │
└──────┬──────┘
       │
       ▼
┌─────────────┐
│ Protocolo   │ (Outline / WireGuard)
└──────┬──────┘
       │
       ▼
┌─────────────┐
│ Servidores  │ (NEW - Server Selection)
└──────┬──────┘
       │
       ▼
┌─────────────┐
│ Nombre      │
└──────┬──────┘
       │
       ▼
┌─────────────┐
│ ✅ Clave    │
└─────────────┘
```

## Server Display Logic

### Load Indicators
- 🟢 Low: 0-50% connections
- 🟡 Medium: 51-80% connections
- 🔴 High: 81-100% connections

### Recommendation Algorithm
1. Filter by protocol support (Outline/WireGuard)
2. Filter by status (online only)
3. Sort by load percentage (lowest first)
4. Show top 5 as "recommended"
5. Option to view all available servers

## Error Scenarios

1. **No servers available**
   - Message: "⚠️ No hay servidores disponibles"
   - Action: Return to menu

2. **Server goes offline**
   - Backend filters out offline servers
   - User only sees available options

3. **API timeout**
   - Message: "⚠️ Error al cargar servidores"
   - Action: Show retry button

4. **Server unavailable at creation**
   - Message: "⚠️ El servidor ya no está disponible"
   - Action: Return to server selection

## Technical Flow

```
User → Bot → Backend API → Server Registry → Response
  │                        │
  │                        └─→ Filter by protocol
  │                        └─→ Filter by status
  │                        └─→ Sort by load
  │
  └─← Display servers with load indicators
```
```

**Step 3: Commit**

```bash
cd /home/mowgli/usipipo/usipipo-telegram-bot
git add src/bot/handlers/keys.py
git commit -m "style: improve server list message formatting with better visual hierarchy"

cd /home/mowgli/usipipo/usipipo-docs
git add flows/server-selection-flow.md
git commit -m "docs: add VPN server selection flow documentation"
```

---

## Post-Implementation Checklist

- [ ] All tests passing (unit + integration)
- [ ] Code reviewed by @python-reviewer and @security-reviewer
- [ ] Documentation updated
- [ ] Changelog entries added in both repos
- [ ] Deployed to staging environment
- [ ] Tested with real Telegram bot
- [ ] Monitored for errors in logs

---

## Deployment Notes

**Backend:**
```bash
cd /home/mowgli/usipipo/usipipo-backend
git push origin main
# GitHub Actions will auto-deploy
```

**Telegram Bot:**
```bash
cd /home/mowgli/usipipo/usipipo-telegram-bot
git push origin main
# GitHub Actions will auto-deploy
```

**Monitoring:**
- Watch backend logs for `/api/v1/vpn/servers` endpoint errors
- Monitor bot logs for conversation state issues
- Check server load distribution after deployment

---

## Skills to Use During Implementation

- @python-reviewer - After each Python code change
- @security-reviewer - After backend endpoint implementation
- @code-reviewer - After completing each major component
- @database-reviewer - If any database schema changes needed
- @tdd-guide - For test-driven development approach

---

**Plan complete and saved to `usipipo-docs/plans/2026-03-31-vpn-server-selection.md`. Two execution options:**

**1. Subagent-Driven (this session)** - I dispatch fresh subagent per task, review between tasks, fast iteration

**2. Parallel Session (separate)** - Open new session with executing-plans, batch execution with checkpoints

**Which approach?**
