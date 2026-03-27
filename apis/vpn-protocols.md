# 🔌 VPN Protocols - Protocolos VPN

> Documentación técnica de los protocolos VPN soportados por uSipipo

---

## 📊 Visión General

uSipipo soporta dos protocolos VPN principales: **WireGuard** y **Outline** (basado en Shadowsocks). Cada protocolo tiene sus propias características, ventajas y casos de uso.

| Característica | WireGuard | Outline |
|----------------|-----------|---------|
| **Tipo** | VPN moderno | Proxy SOCKS5 cifrado |
| **Protocolo** | UDP | TCP/UDP |
| **Cifrado** | ChaCha20-Poly1305 | AES-256-GCM |
| **Puerto** | 51820/udp | 8080/tcp (configurable) |
| **Rendimiento** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ |
| **Facilidad de setup** | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ |
| **Detección** | Más detectable | Menos detectable |

---

## 🔵 WireGuard

### **Introducción**

WireGuard es un protocolo VPN moderno de alto rendimiento que utiliza criptografía de última generación. Es más simple, más rápido y más seguro que IPsec y OpenVPN.

**Sitio oficial:** https://www.wireguard.com/

---

### **Arquitectura**

```
┌─────────────────────────────────────────────────────────┐
│                    WireGuard Tunnel                     │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  ┌─────────────┐         ┌─────────────┐               │
│  │   Client    │         │   Server    │               │
│  │  (Peer)     │         │  (Peer)     │               │
│  │             │         │             │               │
│  │ Private Key │◄───────►│ Public Key  │               │
│  │ 10.0.0.2/32 │         │ 10.0.0.1/24 │               │
│  │             │         │             │               │
│  │  ┌──────┐   │  UDP    │   ┌──────┐  │               │
│  │  │ Tun0 │   │  51820  │   │ Eth0 │  │               │
│  │  └──────┘   │────────►│   └──────┘  │               │
│  │             │         │             │               │
│  └─────────────┘         └─────────────┘               │
│                                                         │
│  Cifrado: ChaCha20-Poly1305                             │
│  Intercambio de keys: Curve25519                        │
│  Hash: BLAKE2s                                          │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

---

### **Configuración del Servidor**

**Instalación (Ubuntu 22.04):**
```bash
# Actualizar paquetes
sudo apt update && sudo apt upgrade -y

# Instalar WireGuard
sudo apt install -y wireguard wireguard-tools

# Generar keys
cd /etc/wireguard
wg genkey | tee privatekey | wg pubkey > publickey

# Ver keys
cat privatekey  # Guardar de forma segura
cat publickey   # Compartir con clientes
```

**Configuración del servidor (`/etc/wireguard/wg0.conf`):**
```ini
[Interface]
# Configuración del servidor
PrivateKey = <server_private_key>
Address = 10.0.0.1/24
ListenPort = 51820
SaveConfig = true

# PostUp/PostDown para NAT y forwarding
PostUp = iptables -A FORWARD -i %i -j ACCEPT; iptables -A FORWARD -o %i -j ACCEPT; iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
PostDown = iptables -D FORWARD -i %i -j ACCEPT; iptables -D FORWARD -o %i -j ACCEPT; iptables -t nat -D POSTROUTING -o eth0 -j MASQUERADE

# Clientes (se agregan dinámicamente)
# [Peer]
# PublicKey = <client_public_key>
# AllowedIPs = 10.0.0.2/32
```

**Habilitar IP forwarding:**
```bash
# Editar sysctl.conf
echo "net.ipv4.ip_forward=1" | sudo tee -a /etc/sysctl.conf

# Aplicar cambios
sudo sysctl -p
```

**Iniciar WireGuard:**
```bash
# Iniciar servicio
sudo systemctl enable wg-quick@wg0
sudo systemctl start wg-quick@wg0

# Ver estado
sudo wg show
```

---

### **Configuración del Cliente**

**Generar keys del cliente:**
```bash
# En el cliente (o servidor)
wg genkey | tee client_privatekey | wg pubkey > client_publickey

# Leer public key
cat client_publickey
```

**Agregar cliente al servidor:**
```bash
# En el servidor, agregar peer
wg set wg0 peer <client_public_key> allowed-ips 10.0.0.2/32

# Guardar configuración
wg-quick save wg0
```

**Configuración del cliente (`client.conf`):**
```ini
[Interface]
PrivateKey = <client_private_key>
Address = 10.0.0.2/24
DNS = 1.1.1.1
MTU = 1420

[Peer]
PublicKey = <server_public_key>
Endpoint = vpn.usipipo.com:51820
AllowedIPs = 0.0.0.0/0
PersistentKeepalive = 25
```

**Conectar cliente:**
```bash
# Linux
wg-quick up client

# Ver estado
wg show

# Desconectar
wg-quick down client
```

---

### **Configuración en Android (App uSipipo)**

**Platform Channel (Dart):**
```dart
class WireGuardService {
  static const platform = MethodChannel('com.usipipo.vpn/wireguard');

  Future<bool> connect(String config) async {
    try {
      final result = await platform.invokeMethod('connectWireGuard', {
        'config': config,
      });
      return result == true;
    } catch (e) {
      print('Error connecting WireGuard: $e');
      return false;
    }
  }

  Future<bool> disconnect() async {
    return await platform.invokeMethod('disconnectVPN') == true;
  }

  Future<Map<String, dynamic>> getStats() async {
    return await platform.invokeMethod('getWireGuardStats');
  }
}
```

**Native Android (Kotlin):**
```kotlin
// WireGuardVpnService.kt
class WireGuardVpnService : VpnService() {
    
    override fun onStartCommand(intent: Intent?, flags: Int, startId: Int): Int {
        val config = intent?.getStringExtra("config") ?: return START_NOT_STICKY
        
        // Configurar VPN
        val builder = Builder()
        builder.addAddress("10.0.0.2", 24)
        builder.addRoute("0.0.0.0", 0)
        builder.addDnsServer("1.1.1.1")
        builder.setSession("uSipipo WireGuard")
        
        // Establecer conexión
        val vpnInterface = builder.establish()
        
        // Iniciar WireGuard
        val wgConfig = WgConfig()
        wgConfig.parse(config)
        
        val wgInterface = WgInterface(wgConfig)
        vpnInterface.addTunInterface(wgInterface)
        
        return START_STICKY
    }
}
```

---

### **Métricas y Monitoreo**

**Comandos de monitoreo:**
```bash
# Ver estado de peers
wg show wg0

# Ver transferencia de datos
wg show wg0 transfer

# Ver último handshake
wg show wg0 latest-handshakes

# Ver configuración completa
wg showconf wg0
```

**Ejemplo de output:**
```
interface: wg0
  public key: ABC123...
  private key: (hidden)
  listening port: 51820

peer: XYZ789...
  endpoint: 192.168.1.100:54321
  allowed ips: 10.0.0.2/32
  latest handshake: 2 minutes, 30 seconds ago
  transfer: 1.5 GiB received, 500 MiB sent
  persistent keepalive: every 25 seconds
```

---

### **Solución de Problemas**

**Problema: No hay conectividad**

```bash
# Verificar que el servicio está corriendo
sudo systemctl status wg-quick@wg0

# Verificar firewall
sudo ufw status

# Agregar regla si es necesario
sudo ufw allow 51820/udp

# Verificar IP forwarding
cat /proc/sys/net/ipv4/ip_forward  # Debe ser 1

# Verificar NAT
sudo iptables -t nat -L -n -v | grep MASQUERADE
```

**Problema: Conexión lenta**

```bash
# Verificar MTU
ip link show wg0

# Ajustar MTU en configuración del cliente
# MTU = 1420 (recomendado para la mayoría de casos)

# Verificar latencia
ping -c 4 10.0.0.1

# Verificar pérdida de paquetes
mtr 10.0.0.1
```

---

## 🟠 Outline (Shadowsocks)

### **Introducción**

Outline es una aplicación VPN de código abierto desarrollada por Jigsaw (una subsidiaria de Google). Está basada en el protocolo Shadowsocks, un proxy SOCKS5 cifrado diseñado para evadir censura.

**Sitio oficial:** https://getoutline.org/

---

### **Arquitectura**

```
┌─────────────────────────────────────────────────────────┐
│                  Outline (Shadowsocks)                  │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  ┌─────────────┐         ┌─────────────┐               │
│  │   Client    │         │   Server    │               │
│  │             │         │             │               │
│  │  ss:// URL  │────────►│  Shadowsocks│               │
│  │             │  TCP    │   Server    │               │
│  │             │  8080   │  (ss-server)│               │
│  │             │         │             │               │
│  │             │         │   ┌──────┐  │               │
│  │             │         │   │ Eth0 │  │               │
│  │             │         │   └──────┘  │               │
│  └─────────────┘         └─────────────┘               │
│                                                         │
│  Cifrado: AES-256-GCM                                   │
│  Protocolo: SOCKS5                                      │
│  Ofuscación: Sí (tráfico parece HTTPS)                  │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

---

### **Instalación del Servidor**

**Método 1: Script oficial (recomendado)**
```bash
# Descargar y ejecutar script de instalación
bash -c "$(wget -qO- https://raw.githubusercontent.com/Jigsaw-Code/outline-server/master/src/server_manager/install_scripts/install_server.sh)"
```

**Output del script:**
```
✅ Outline Server installed!

Your access key is:
ss://YWVzLTI1Ni1nY206cGFzc3dvcmQ@hostname:8080#MyOutlineServer

Outline Manager URL:
https://manage.getoutline.org/#/access?key=ss://...

API Port: 8080
API URL: https://hostname:8080/abc123...
```

---

### **Outline Manager API**

**Autenticación:**
- mTLS (certificados cliente)
- SSL pinning requerido

**Endpoints principales:**

```bash
# Listar keys
curl --cert cert.pem --key key.pem \
  https://hostname:8080/abc123/access-keys

# Crear nueva key
curl --cert cert.pem --key key.pem \
  -X POST \
  -H "Content-Type: application/json" \
  -d '{"method":"aes-256-gcm","name":"user-123"}' \
  https://hostname:8080/abc123/access-keys

# Eliminar key
curl --cert cert.pem --key key.pem \
  -X DELETE \
  https://hostname:8080/abc123/access-keys/0
```

---

### **Integración con Backend uSipipo**

```python
# src/infrastructure/vpn_providers/outline_provider.py
import aiohttp
import ssl
from typing import Dict, Any, List

class OutlineProvider:
    def __init__(self, api_url: str, cert_path: str, key_path: str):
        self.api_url = api_url
        self.ssl_context = self._create_ssl_context(cert_path, key_path)
    
    def _create_ssl_context(self, cert_path: str, key_path: str) -> ssl.SSLContext:
        context = ssl.SSLContext(ssl.PROTOCOL_TLS_CLIENT)
        context.load_cert_chain(cert_path, key_path)
        context.check_hostname = False  # SSL pinning
        context.verify_mode = ssl.CERT_NONE
        return context
    
    async def create_access_key(self, name: str, method: str = "aes-256-gcm") -> Dict[str, Any]:
        """Crear nueva access key"""
        async with aiohttp.ClientSession() as session:
            async with session.post(
                f"{self.api_url}/access-keys",
                json={"method": method, "name": name},
                ssl=self.ssl_context,
            ) as response:
                return await response.json()
    
    async def list_access_keys(self) -> List[Dict[str, Any]]:
        """Listar todas las access keys"""
        async with aiohttp.ClientSession() as session:
            async with session.get(
                f"{self.api_url}/access-keys",
                ssl=self.ssl_context,
            ) as response:
                data = await response.json()
                return data.get("accessKeys", [])
    
    async def delete_access_key(self, key_id: str) -> bool:
        """Eliminar access key"""
        async with aiohttp.ClientSession() as session:
            async with session.delete(
                f"{self.api_url}/access-keys/{key_id}",
                ssl=self.ssl_context,
            ) as response:
                return response.status == 204
    
    async def get_transfer_metrics(self) -> Dict[str, Any]:
        """Obtener métricas de transferencia"""
        async with aiohttp.ClientSession() as session:
            async with session.get(
                f"{self.api_url}/metrics/transfer",
                ssl=self.ssl_context,
            ) as response:
                return await response.json()
```

---

### **Formato de URL de Outline**

```
ss://<method>:<password>@<hostname>:<port>#<name>
```

**Ejemplo:**
```
ss://YWVzLTI1Ni1nY206cGFzc3dvcmQ@vpn.usipipo.com:8080#HomeVPN
```

**Decodificar:**
```python
import base64
from urllib.parse import urlparse, unquote

def parse_outline_url(url: str) -> Dict[str, str]:
    """Parsear URL de Outline"""
    # Remover ss://
    url_without_scheme = url.replace("ss://", "")
    
    # Separar nombre
    if "#" in url_without_scheme:
        url_part, name = url_without_scheme.split("#")
        name = unquote(name)
    else:
        url_part = url_without_scheme
        name = ""
    
    # Separar auth@host:port
    auth_part, host_port = url_part.split("@")
    method_password = base64.urlsafe_b64decode(auth_part).decode()
    method, password = method_password.split(":")
    
    # Separar host:port
    host, port = host_port.split(":")
    
    return {
        "method": method,
        "password": password,
        "hostname": host,
        "port": int(port),
        "name": name,
    }
```

---

### **Configuración en Android (App uSipipo)**

**Platform Channel (Dart):**
```dart
class OutlineService {
  static const platform = MethodChannel('com.usipipo.vpn/outline');

  Future<bool> connect(String ssUrl) async {
    try {
      final result = await platform.invokeMethod('connectOutline', {
        'ssUrl': ssUrl,
      });
      return result == true;
    } catch (e) {
      print('Error connecting Outline: $e');
      return false;
    }
  }

  Future<bool> disconnect() async {
    return await platform.invokeMethod('disconnectVPN') == true;
  }

  Future<Map<String, dynamic>> getStats() async {
    return await platform.invokeMethod('getOutlineStats');
  }
}
```

**Native Android (Kotlin):**
```kotlin
// OutlineVpnService.kt
class OutlineVpnService : VpnService() {
    
    override fun onStartCommand(intent: Intent?, flags: Int, startId: Int): Int {
        val ssUrl = intent?.getStringExtra("ssUrl") ?: return START_NOT_STICKY
        
        // Parsear URL
        val config = OutlineConfig.parse(ssUrl)
        
        // Configurar VPN
        val builder = Builder()
        builder.addAddress("10.0.0.2", 24)
        builder.addRoute("0.0.0.0", 0)
        builder.addDnsServer("1.1.1.1")
        builder.setSession("uSipipo Outline")
        
        val vpnInterface = builder.establish()
        
        // Iniciar Shadowsocks
        val ssClient = ShadowsocksClient(config)
        ssClient.connect(vpnInterface)
        
        return START_STICKY
    }
}
```

---

### **Métricas y Monitoreo**

**API de métricas:**
```bash
# Obtener métricas de transferencia
curl --cert cert.pem --key key.pem \
  https://hostname:8080/abc123/metrics/transfer
```

**Response:**
```json
{
  "bytesTransferredByUserId": {
    "0": 1073741824,
    "1": 536870912,
    "2": 2147483648
  }
}
```

---

### **Solución de Problemas**

**Problema: Conexión rechazada**

```bash
# Verificar que el servidor está corriendo
sudo systemctl status outline-server

# Verificar puerto
sudo netstat -tlnp | grep 8080

# Verificar firewall
sudo ufw status
sudo ufw allow 8080/tcp
```

**Problema: Certificado SSL inválido**

```bash
# Regenerar certificados
sudo /opt/outline/persisted-state/update-cert.sh

# Reiniciar servidor
sudo systemctl restart outline-server
```

---

## 📊 Comparativa de Protocolos

### **Rendimiento**

| Métrica | WireGuard | Outline |
|---------|-----------|---------|
| **Velocidad máxima** | 500+ Mbps | 200+ Mbps |
| **Latencia** | 5-10ms | 10-20ms |
| **CPU usage** | Bajo | Medio |
| **Memoria** | 10 MB | 50 MB |

---

### **Seguridad**

| Característica | WireGuard | Outline |
|----------------|-----------|---------|
| **Cifrado** | ChaCha20-Poly1305 | AES-256-GCM |
| **Key exchange** | Curve25519 | Pre-shared key |
| **Perfect forward secrecy** | ✅ Sí | ❌ No |
| **Auditado** | ✅ Sí | ✅ Sí |

---

### **Detección y Ofuscación**

| Característica | WireGuard | Outline |
|----------------|-----------|---------|
| **Detectable** | ✅ Sí (firma UDP) | ❌ Menos (parece HTTPS) |
| **Ofuscación** | ❌ No (requiere obfsproxy) | ✅ Sí (built-in) |
| **Funciona en China** | ⚠️ A veces | ✅ Sí |

---

## 📚 Recursos Relacionados

- [Backend Stack](../technology/backend-stack.md)
- [Infrastructure Stack](../technology/infrastructure-stack.md)
- [VPN Connection Flow](../flows/vpn-connection-flow.md)

---

## 🔗 Enlaces Externos

- [WireGuard Documentation](https://www.wireguard.com/)
- [Outline Documentation](https://getoutline.org/get-started/)
- [Shadowsocks Protocol](https://shadowsocks.org/)
- [Jigsaw GitHub](https://github.com/Jigsaw-Code)

---

**Última actualización:** 2026-03-27
