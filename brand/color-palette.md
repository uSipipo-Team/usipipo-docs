# 🎨 Paleta de Colores - Cyberpunk Neon Night

> Sistema de colores oficial del ecosistema uSipipo

---

## 🌈 Visión General

La paleta **Cyberpunk Neon Night** evoca tecnología futurista, privacidad y velocidad. Diseñada exclusivamente para **modo oscuro**, combina fondos profundos con acentos neón vibrantes.

---

## 🎯 Colores Primarios

### **Primary - Cyan Neón**

```
#00F0FF
rgb(0, 240, 255)
hsl(184, 100%, 50%)
```

**Uso:**
- ✅ Botones primarios (CTA)
- ✅ Links y enlaces
- ✅ Iconos principales
- ✅ Estados activos
- ✅ Bordes y highlights

**No usar para:**
- ❌ Texto de cuerpo (muy brillante)
- ❌ Fondos grandes (muy intenso)
- ❌ Estados de error

**Ejemplo:**
```css
.btn-primary {
  background-color: #00F0FF;
  color: #0A0A0F;
}
```

---

### **Secondary - Magenta Neón**

```
#FF00AA
rgb(255, 0, 170)
hsl(334, 100%, 50%)
```

**Uso:**
- ✅ Botones secundarios
- ✅ Gradientes (con cyan)
- ✅ Acentos visuales
- ✅ Estados hover
- ✅ Decoración

**No usar para:**
- ❌ Texto importante (bajo contraste)
- ❌ Estados de éxito
- ❌ Alertas críticas

**Ejemplo:**
```css
.gradient-cyberpunk {
  background: linear-gradient(135deg, #00F0FF, #FF00AA);
}
```

---

### **Accent - Terminal Green**

```
#00FF41
rgb(0, 255, 65)
hsl(134, 100%, 50%)
```

**Uso:**
- ✅ Estados de éxito
- ✅ Indicadores online
- ✅ Datos positivos
- ✅ Acentos de código
- ✅ Confirmaciones

**No usar para:**
- ❌ Botones primarios
- ❌ Errores (confusión semántica)

**Ejemplo:**
```css
.status-online {
  color: #00FF41;
}
```

---

## 🌑 Colores de Fondo

### **Void Dark (Background Principal)**

```
#0A0A0F
rgb(10, 10, 15)
hsl(240, 20%, 2%)
```

**Uso:**
- ✅ Fondo de página
- ✅ Background principal
- ✅ Secciones dark

**Ejemplo:**
```css
body {
  background-color: #0A0A0F;
}
```

---

### **Card Dark (Surface)**

```
#12121A
rgb(18, 18, 26)
hsl(240, 18%, 9%)
```

**Uso:**
- ✅ Tarjetas y cards
- ✅ Contenedores
- ✅ Secciones elevadas
- ✅ Inputs y forms

**Ejemplo:**
```css
.card {
  background-color: #12121A;
}
```

---

### **Elevated Dark**

```
#1A1A24
rgb(26, 26, 36)
hsl(240, 16%, 12%)
```

**Uso:**
- ✅ Cards hover
- ✅ Elementos elevados
- ✅ Headers de tablas
- ✅ Bordes sutiles

**Ejemplo:**
```css
.card:hover {
  background-color: #1A1A24;
}
```

---

## 📝 Colores de Texto

### **Text Primary**

```
#E0E0E0
rgb(224, 224, 224)
hsl(0, 0%, 88%)
```

**Uso:**
- ✅ Texto de cuerpo
- ✅ Títulos
- ✅ Contenido principal

**Contraste sobre Void Dark:** 15.2:1 (AAA)

---

### **Text Secondary**

```
#888888
rgb(136, 136, 136)
hsl(0, 0%, 53%)
```

**Uso:**
- ✅ Texto secundario
- ✅ Placeholders
- ✅ Metadatos
- ✅ Timestamps

**Contraste sobre Void Dark:** 5.8:1 (AA)

---

### **Text Muted**

```
#555555
rgb(85, 85, 85)
hsl(0, 0%, 33%)
```

**Uso:**
- ✅ Texto deshabilitado
- ✅ Bordes sutiles
- ✅ Elementos inactivos

---

## 🚦 Colores de Estado

### **Success**

```
#00FF41
rgb(0, 255, 65)
```

**Uso:** Conexiones activas, pagos completados, estados positivos.

---

### **Warning**

```
#FFAA00
rgb(255, 170, 0)
```

**Uso:** Expiración próxima, límites alcanzados, advertencias.

---

### **Error**

```
#FF0044
rgb(255, 0, 68)
```

**Uso:** Errores, conexiones fallidas, pagos rechazados.

---

### **Info**

```
#00F0FF
rgb(0, 240, 255)
```

**Uso:** Información, tooltips, estados informativos.

---

## 🎨 Gradientes

### **Cyberpunk Gradient (Principal)**

```css
.cyberpunk-gradient {
  background: linear-gradient(135deg, 
    #00F0FF 0%, 
    #FF00AA 100%
  );
}
```

**Uso:**
- ✅ Hero sections
- ✅ Botones destacados
- ✅ Backgrounds promocionales

---

### **Neon Glow Gradient**

```css
.neon-glow {
  background: linear-gradient(135deg,
    rgba(0, 240, 255, 0.1) 0%,
    rgba(255, 0, 170, 0.1) 100%
  );
  border: 1px solid rgba(0, 240, 255, 0.3);
}
```

**Uso:**
- ✅ Cards con glow
- ✅ Bordes neón
- ✅ Efectos hover

---

### **Terminal Gradient**

```css
.terminal-gradient {
  background: linear-gradient(180deg,
    #0A0A0F 0%,
    #12121A 100%
  );
}
```

**Uso:**
- ✅ Fondos de código
- ✅ Terminales
- ✅ Logs

---

## ✨ Efectos de Glow

### **Cyan Glow**

```css
.cyan-glow {
  box-shadow: 0 0 20px rgba(0, 240, 255, 0.5),
              0 0 40px rgba(0, 240, 255, 0.3);
}
```

---

### **Magenta Glow**

```css
.magenta-glow {
  box-shadow: 0 0 20px rgba(255, 0, 170, 0.5),
              0 0 40px rgba(255, 0, 170, 0.3);
}
```

---

### **Green Glow**

```css
.green-glow {
  box-shadow: 0 0 20px rgba(0, 255, 65, 0.5),
              0 0 40px rgba(0, 255, 65, 0.3);
}
```

---

### **Text Glow**

```css
.neon-text {
  color: #00F0FF;
  text-shadow: 0 0 10px rgba(0, 240, 255, 0.8),
               0 0 20px rgba(0, 240, 255, 0.6);
}
```

---

## 📊 Tabla de Colores Completa

| Nombre | Hex | RGB | Uso Principal |
|--------|-----|-----|---------------|
| **Primary Cyan** | `#00F0FF` | `rgb(0, 240, 255)` | CTAs, links |
| **Secondary Magenta** | `#FF00AA` | `rgb(255, 0, 170)` | Acentos, hovers |
| **Accent Green** | `#00FF41` | `rgb(0, 255, 65)` | Éxito, online |
| **Void Dark** | `#0A0A0F` | `rgb(10, 10, 15)` | Fondo principal |
| **Card Dark** | `#12121A` | `rgb(18, 18, 26)` | Cards, surface |
| **Elevated Dark** | `#1A1A24` | `rgb(26, 26, 36)` | Hover, elevated |
| **Text Primary** | `#E0E0E0` | `rgb(224, 224, 224)` | Texto cuerpo |
| **Text Secondary** | `#888888` | `rgb(136, 136, 136)` | Metadatos |
| **Text Muted** | `#555555` | `rgb(85, 85, 85)` | Disabled |
| **Warning** | `#FFAA00` | `rgb(255, 170, 0)` | Advertencias |
| **Error** | `#FF0044` | `rgb(255, 0, 68)` | Errores |

---

## 🎯 Reglas de Uso

### ✅ DOs

1. **Siempre usar modo oscuro** - No existe modo claro en uSipipo
2. **Primary para CTAs** - Cyan es el color de acción principal
3. **Contraste mínimo AA** - Texto legible sobre fondos oscuros
4. **Glow con moderación** - Máximo 2-3 elementos con glow por pantalla
5. **Gradientes sutiles** - Opacidad máxima 30% en fondos

### ❌ DON'Ts

1. **Nunca usar blanco puro** (`#FFFFFF`) - Muy intenso, usar `#E0E0E0`
2. **Nunca usar modo claro** - Rompe la identidad cyberpunk
3. **No mezclar muchos neones** - Máximo 2 colores neón por elemento
4. **No usar en texto pequeño** - Neones solo en títulos y botones
5. **No ignorar accesibilidad** - Verificar contraste siempre

---

## ♿ Accesibilidad

### Contraste Mínimo Requerido

| Elemento | WCAG AA | WCAG AAA |
|----------|---------|----------|
| Texto normal | 4.5:1 | 7:1 |
| Texto grande | 3:1 | 4.5:1 |
| UI components | 3:1 | 3:1 |

### Combinaciones Aprobadas

| Fondo | Texto | Ratio | Nivel |
|-------|-------|-------|-------|
| `#0A0A0F` | `#E0E0E0` | 15.2:1 | ✅ AAA |
| `#12121A` | `#E0E0E0` | 12.8:1 | ✅ AAA |
| `#0A0A0F` | `#00F0FF` | 8.6:1 | ✅ AAA |
| `#0A0A0F` | `#888888` | 5.8:1 | ✅ AA |

### Herramientas de Verificación

- [WebAIM Contrast Checker](https://webaim.org/resources/contrastchecker/)
- [Contrast Grid](https://contrast-grid.eightshapes.com/)

---

## 💻 Implementación

### CSS Variables

```css
:root {
  /* Primary */
  --color-primary: #00F0FF;
  --color-secondary: #FF00AA;
  --color-accent: #00FF41;
  
  /* Backgrounds */
  --bg-void: #0A0A0F;
  --bg-card: #12121A;
  --bg-elevated: #1A1A24;
  
  /* Text */
  --text-primary: #E0E0E0;
  --text-secondary: #888888;
  --text-muted: #555555;
  
  /* States */
  --color-success: #00FF41;
  --color-warning: #FFAA00;
  --color-error: #FF0044;
  --color-info: #00F0FF;
}
```

### Tailwind CSS

```js
// tailwind.config.js
module.exports = {
  theme: {
    extend: {
      colors: {
        primary: '#00F0FF',
        secondary: '#FF00AA',
        accent: '#00FF41',
        void: '#0A0A0F',
        card: '#12121A',
        elevated: '#1A1A24',
      }
    }
  }
}
```

### Flutter/Dart

```dart
// lib/core/theme/cyberpunk_colors.dart
import 'package:flutter/material.dart';

class CyberpunkColors {
  static const Color primary = Color(0xFF00F0FF);
  static const Color secondary = Color(0xFFFF00AA);
  static const Color accent = Color(0xFF00FF41);
  
  static const Color bgVoid = Color(0xFF0A0A0F);
  static const Color bgCard = Color(0xFF12121A);
  static const Color bgElevated = Color(0xFF1A1A24);
  
  static const Color textPrimary = Color(0xFFE0E0E0);
  static const Color textSecondary = Color(0xFF888888);
  static const Color textMuted = Color(0xFF555555);
}
```

### Python (Backend)

```python
# Para generación de emails, PDFs, etc.
COLORS = {
    "primary": "#00F0FF",
    "secondary": "#FF00AA",
    "accent": "#00FF41",
    "bg_void": "#0A0A0F",
    "bg_card": "#12121A",
    "text_primary": "#E0E0E0",
}
```

---

## 📚 Recursos Relacionados

- [Identidad de Marca](identity.md)
- [Tipografía](typography.md)
- [Logo Assets](logo-assets.md)

---

**Última actualización:** 2026-03-27
