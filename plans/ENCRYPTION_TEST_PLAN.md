# Agent API Key Encryption - Test Plan

## Overview

This document outlines the test plan for verifying that agent API keys are properly encrypted in the database and that encryption is mandatory.

## Prerequisites

1. **Generate an encryption key** (if not already done):
   ```bash
   python -c 'from cryptography.fernet import Fernet; print(Fernet.generate_key().decode())'
   ```

2. **Set the ENCRYPTION_KEY environment variable**:
   ```bash
   export ENCRYPTION_KEY="your-generated-key-here"
   ```

3. **Database access** - Ensure you have access to the test database

---

## Test 1: Encryption Service Initialization

### Test: Service fails without ENCRYPTION_KEY

**Steps:**
1. Unset the ENCRYPTION_KEY: `unset ENCRYPTION_KEY`
2. Try to initialize the encryption service

**Expected Result:**
- `EncryptionNotConfiguredError` is raised
- Error message indicates ENCRYPTION_KEY is required

**Test Code:**
```python
import os
import pytest
from src.shared.encryption import EncryptionService, EncryptionNotConfiguredError

def test_encryption_fails_without_key():
    """Test that encryption service fails without ENCRYPTION_KEY."""
    # Ensure key is not set
    original_key = os.environ.pop("ENCRYPTION_KEY", None)

    # Reset singleton to force re-initialization
    EncryptionService.reset_instance()

    with pytest.raises(EncryptionNotConfiguredError) as exc_info:
        EncryptionService.get_instance()

    assert "ENCRYPTION_KEY" in str(exc_info.value)

    # Restore original key
    if original_key:
        os.environ["ENCRYPTION_KEY"] = original_key
```

---

## Test 2: Encryption/Decryption Round-Trip

### Test: Keys can be encrypted and decrypted correctly

**Steps:**
1. Set ENCRYPTION_KEY
2. Encrypt a test API key
3. Decrypt the result
4. Verify the decrypted value matches the original

**Expected Result:**
- Encrypted value is different from plaintext
- Decrypted value matches original plaintext exactly

**Test Code:**
```python
import os
import pytest
from src.shared.encryption import (
    EncryptionService,
    encrypt_sensitive_data,
    decrypt_sensitive_data,
    EncryptionNotConfiguredError
)

def test_encryption_round_trip():
    """Test that encryption/decryption works correctly."""
    # Ensure key is set
    os.environ["ENCRYPTION_KEY"] = EncryptionService.generate_key()
    EncryptionService.reset_instance()

    original = "agent_test123456789"

    # Encrypt
    encrypted = encrypt_sensitive_data(original)
    assert encrypted != original  # Should be different
    assert len(encrypted) > len(original)  # Should be longer (base64)

    # Decrypt
    decrypted = decrypt_sensitive_data(encrypted)
    assert decrypted == original
```

---

## Test 3: VpnServerModel Encryption

### Test: API keys are encrypted on model save

**Steps:**
1. Create a VpnServerModel instance
2. Set the agent_api_key property
3. Verify the stored value (_agent_api_key) is encrypted
4. Read the agent_api_key property
5. Verify it returns the decrypted value

**Expected Result:**
- Internal `_agent_api_key` contains encrypted data
- Property getter returns decrypted plaintext
- No plaintext is ever written to the database

**Test Code:**
```python
import os
import uuid
import pytest
from src.shared.encryption import EncryptionService, EncryptionNotConfiguredError
from src.infrastructure.persistence.models.vpn_server_model import VpnServerModel

def test_vpn_server_model_encrypts_keys():
    """Test that VpnServerModel encrypts API keys."""
    # Setup encryption
    os.environ["ENCRYPTION_KEY"] = EncryptionService.generate_key()
    EncryptionService.reset_instance()

    # Create model
    server = VpnServerModel(
        id=uuid.uuid4(),
        name="Test Server",
        country_code="US",
        country_name="United States",
        agent_url="https://example.com",
    )

    # Set API key
    original_key = "agent_test123456"
    server.agent_api_key = original_key

    # Verify internal storage is encrypted
    assert server._agent_api_key != original_key
    assert len(server._agent_api_key) > len(original_key)

    # Verify getter returns decrypted value
    assert server.agent_api_key == original_key
```

---

## Test 4: VpnServerModel Fails Without Encryption

### Test: Model raises error when encryption not configured

**Steps:**
1. Unset ENCRYPTION_KEY
2. Try to create a VpnServerModel and set agent_api_key

**Expected Result:**
- `EncryptionNotConfiguredError` is raised
- No plaintext is written to the database

**Test Code:**
```python
import os
import uuid
import pytest
from src.shared.encryption import EncryptionService, EncryptionNotConfiguredError
from src.infrastructure.persistence.models.vpn_server_model import VpnServerModel

def test_vpn_server_model_fails_without_encryption():
    """Test that VpnServerModel fails without encryption."""
    # Ensure key is not set
    os.environ.pop("ENCRYPTION_KEY", None)
    EncryptionService.reset_instance()

    server = VpnServerModel(
        id=uuid.uuid4(),
        name="Test Server",
        country_code="US",
        country_name="United States",
        agent_url="https://example.com",
    )

    # Setting API key should fail
    with pytest.raises(EncryptionNotConfiguredError):
        server.agent_api_key = "agent_test123"
```

---

## Test 5: Migration Encrypts Existing Keys

### Test: Migration successfully encrypts plaintext keys

**Prerequisites:**
- Database with some plaintext API keys
- ENCRYPTION_KEY set

**Steps:**
1. Insert test data with plaintext keys
2. Run the migration: `alembic upgrade head`
3. Query the database
4. Verify keys are now encrypted

**Expected Result:**
- All plaintext keys are encrypted
- Encrypted keys can be decrypted correctly
- Migration logs show count of encrypted/skipped keys

**Test Code:**
```python
import os
from sqlalchemy import create_engine, text
from src.shared.encryption import EncryptionService, decrypt_sensitive_data

def test_migration_encrypts_keys(db_url):
    """Test that migration encrypts existing plaintext keys."""
    # Setup
    os.environ["ENCRYPTION_KEY"] = EncryptionService.generate_key()
    engine = create_engine(db_url)

    with engine.connect() as conn:
        # Insert plaintext key
        conn.execute(text("""
            INSERT INTO vpn_servers (id, name, country_code, country_name, agent_url, agent_api_key)
            VALUES (:id, :name, :country_code, :country_name, :agent_url, :agent_api_key)
        """), {
            "id": "12345678-1234-1234-1234-123456789012",
            "name": "Test Server",
            "country_code": "US",
            "country_name": "United States",
            "agent_url": "https://example.com",
            "agent_api_key": "agent_plaintext123"
        })
        conn.commit()

    # Run migration
    # alembic upgrade head

    # Verify encryption
    with engine.connect() as conn:
        result = conn.execute(text("""
            SELECT agent_api_key FROM vpn_servers WHERE id = :id
        """), {"id": "12345678-1234-1234-1234-123456789012"})

        encrypted_key = result.scalar()

        # Verify it's encrypted (longer than plaintext)
        assert len(encrypted_key) > len("agent_plaintext123")

        # Verify it can be decrypted
        decrypted = decrypt_sensitive_data(encrypted_key)
        assert decrypted == "agent_plaintext123"
```

---

## Test 6: Agent Registration Service

### Test: Registration encrypts keys automatically

**Steps:**
1. Create an API key using the service
2. Register an agent with that key
3. Query the database directly
4. Verify the stored key is encrypted

**Expected Result:**
- Database contains encrypted key
- Service can retrieve and use the key correctly

**Test Code:**
```python
import os
import pytest
from src.shared.encryption import EncryptionService
from src.core.application.services.agent_registration_service import AgentRegistrationService

@pytest.mark.asyncio
async def test_registration_encrypts_keys(session, api_key_repo, vpn_repo):
    """Test that agent registration encrypts API keys."""
    # Setup encryption
    os.environ["ENCRYPTION_KEY"] = EncryptionService.generate_key()
    EncryptionService.reset_instance()

    service = AgentRegistrationService(session, api_key_repo, vpn_repo)

    # Generate and store API key
    plain_key, api_key_model = await service.create_api_key(description="Test key")

    # Register agent
    server = await service.register_agent(
        api_key=plain_key,
        hostname="test-host",
        ip_address="192.168.1.1",
        country_code="US",
        country_name="United States",
        agent_version="1.0.0",
        os_type="Linux",
        os_arch="x86_64",
        agent_url="https://test.com",
    )

    # Verify key is encrypted in database
    # (Query database directly to check _agent_api_key column)
    assert server._agent_api_key != plain_key
    assert len(server._agent_api_key) > len(plain_key)

    # Verify we can still access the decrypted key
    assert server.agent_api_key == plain_key
```

---

## Test 7: Integration Test - Full Flow

### Test: End-to-end encryption workflow

**Steps:**
1. Start application with ENCRYPTION_KEY set
2. Generate a new API key via the API
3. Register an agent
4. Query the database directly
5. Verify no plaintext keys exist

**Expected Result:**
- All API keys in database are encrypted
- Application can decrypt and use keys correctly
- Logs show no plaintext keys

**Manual Verification:**
```bash
# 1. Set encryption key
export ENCRYPTION_KEY=$(python -c 'from cryptography.fernet import Fernet; print(Fernet.generate_key().decode())')

# 2. Start the application
cd usipipo-backend
uvicorn src.main:app --reload

# 3. Generate API key (via API or script)
# 4. Register an agent

# 5. Query database directly
psql -d usipipo -c "SELECT id, name, LEFT(agent_api_key, 20) as key_preview FROM vpn_servers;"

# Expected: key_preview should show encrypted data (starts with base64 chars)
# NOT plaintext like "agent_xxxxx..."
```

---

## Test 8: Error Handling - Corrupted Keys

### Test: System handles corrupted/invalid encrypted keys

**Steps:**
1. Manually corrupt an encrypted key in the database
2. Try to read the key via the model property

**Expected Result:**
- `cryptography.fernet.InvalidToken` is raised
- Error is logged appropriately
- Application doesn't crash

**Test Code:**
```python
import os
import pytest
from cryptography.fernet import InvalidToken
from src.shared.encryption import EncryptionService, decrypt_sensitive_data

def test_corrupted_key_handling():
    """Test that corrupted keys raise appropriate errors."""
    os.environ["ENCRYPTION_KEY"] = EncryptionService.generate_key()
    EncryptionService.reset_instance()

    # Try to decrypt invalid data
    with pytest.raises(InvalidToken):
        decrypt_sensitive_data("invalid_corrupted_data")
```

---

## Test 9: Performance Test

### Test: Encryption overhead is acceptable

**Steps:**
1. Encrypt/decrypt 1000 keys
2. Measure time taken
3. Verify overhead is < 1ms per operation

**Expected Result:**
- Encryption/decryption is fast (< 1ms average)
- No memory leaks

**Test Code:**
```python
import os
import time
from src.shared.encryption import EncryptionService, encrypt_sensitive_data, decrypt_sensitive_data

def test_encryption_performance():
    """Test that encryption overhead is acceptable."""
    os.environ["ENCRYPTION_KEY"] = EncryptionService.generate_key()
    EncryptionService.reset_instance()

    iterations = 1000
    test_key = "agent_test123456789"

    start = time.time()

    for _ in range(iterations):
        encrypted = encrypt_sensitive_data(test_key)
        decrypted = decrypt_sensitive_data(encrypted)
        assert decrypted == test_key

    elapsed = time.time() - start
    avg_time_ms = (elapsed / iterations) * 1000

    print(f"Average encryption/decryption time: {avg_time_ms:.3f}ms")
    assert avg_time_ms < 1.0, f"Encryption too slow: {avg_time_ms:.3f}ms"
```

---

## Checklist

Before deploying to production, verify:

- [ ] ENCRYPTION_KEY is set in production environment
- [ ] ENCRYPTION_KEY is stored securely (e.g., AWS Secrets Manager, HashiCorp Vault)
- [ ] ENCRYPTION_KEY is backed up securely (loss = loss of all API keys)
- [ ] Migration has been run successfully
- [ ] All tests pass
- [ ] No plaintext keys exist in database (verify with SQL query)
- [ ] Application logs don't contain plaintext keys
- [ ] Key rotation procedure is documented

---

## Rollback Plan

If encryption causes issues:

1. **Before migration**: Simply don't run the migration
2. **After migration**:
   - Decryption requires ENCRYPTION_KEY
   - Without the key, agents cannot be authenticated
   - **CRITICAL**: Backup ENCRYPTION_KEY securely before migration

---

## Security Notes

1. **Never log plaintext API keys** - Review all logging statements
2. **ENCRYPTION_KEY must be unique per environment** - Don't share keys between dev/staging/prod
3. **Key rotation** - If ENCRYPTION_KEY is compromised:
   - Generate new key
   - Decrypt all keys with old key
   - Re-encrypt with new key
   - Update ENCRYPTION_KEY environment variable
4. **Database backups** - Encrypted keys in backups are safe, but ENCRYPTION_KEY must be protected
