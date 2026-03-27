# 🔤 Tipografía uSipipo

> Sistema tipográfico del ecosistema uSipipo

---

## 📐 Visión General

El sistema tipográfico de uSipipo combina fuentes **monoespaciadas** (tecnología, código) con fuentes **sans-serif** (legibilidad, modernidad) para crear una estética cyberpunk funcional.

---

## 🎨 Familias Tipográficas

### **Headings: JetBrains Mono**

```
Fuente: JetBrains Mono
Tipo: Monospace
Weights: 400 (Regular), 500 (Medium), 600 (SemiBold), 700 (Bold)
Foundry: JetBrains
License: Open Font License
```

**Uso:**
- ✅ Títulos principales (H1, H2)
- ✅ Subtítulos (H3, H4)
- ✅ Números y datos
- ✅ Código inline
- ✅ Labels técnicos

**No usar para:**
- ❌ Texto de cuerpo (muy ancho)
- ❌ Párrafos largos
- ❌ Texto en móvil pequeño

**Ejemplo:**
```css
h1, h2, h3, h4, h5, h6 {
  font-family: 'JetBrains Mono', monospace;
  font-weight: 600;
}
```

---

### **Body: Inter**

```
Fuente: Inter
Tipo: Sans-serif
Weights: 400 (Regular), 500 (Medium), 600 (SemiBold), 700 (Bold)
Foundry: Rasmus Andersson
License: Open Font License
```

**Uso:**
- ✅ Texto de cuerpo
- ✅ Párrafos
- ✅ Botones (texto)
- ✅ Forms (inputs, labels)
- ✅ UI general

**No usar para:**
- ❌ Títulos principales (muy neutral)
- ❌ Código
- ❌ Datos técnicos

**Ejemplo:**
```css
body, p, span, div {
  font-family: 'Inter', sans-serif;
}
```

---

## 📊 Escala Tipográfica

### Desktop (≥ 1024px)

| Elemento | Fuente | Size | Weight | Line Height | Letter Spacing |
|----------|--------|------|--------|-------------|----------------|
| **H1** | JetBrains Mono | 48px | 700 | 1.2 | -0.02em |
| **H2** | JetBrains Mono | 36px | 600 | 1.3 | -0.01em |
| **H3** | JetBrains Mono | 28px | 600 | 1.4 | 0 |
| **H4** | JetBrains Mono | 24px | 500 | 1.4 | 0 |
| **Body** | Inter | 16px | 400 | 1.6 | 0 |
| **Small** | Inter | 14px | 400 | 1.5 | 0.01em |
| **Caption** | Inter | 12px | 400 | 1.4 | 0.02em |
| **Code** | JetBrains Mono | 14px | 400 | 1.5 | 0 |

---

### Mobile (< 768px)

| Elemento | Fuente | Size | Weight | Line Height | Letter Spacing |
|----------|--------|------|--------|-------------|----------------|
| **H1** | JetBrains Mono | 32px | 700 | 1.2 | -0.02em |
| **H2** | JetBrains Mono | 28px | 600 | 1.3 | -0.01em |
| **H3** | JetBrains Mono | 24px | 600 | 1.4 | 0 |
| **H4** | JetBrains Mono | 20px | 500 | 1.4 | 0 |
| **Body** | Inter | 16px | 400 | 1.6 | 0 |
| **Small** | Inter | 14px | 400 | 1.5 | 0.01em |
| **Caption** | Inter | 12px | 400 | 1.4 | 0.02em |

---

## 🎯 Jerarquías de Uso

### **Nivel 1: Títulos Principales**

```css
h1 {
  font-family: 'JetBrains Mono', monospace;
  font-size: 48px;
  font-weight: 700;
  line-height: 1.2;
  letter-spacing: -0.02em;
  color: #E0E0E0;
}
```

**Ejemplo de uso:**
```html
<h1>VPN Instantánea vía Telegram</h1>
```

---

### **Nivel 2: Títulos de Sección**

```css
h2 {
  font-family: 'JetBrains Mono', monospace;
  font-size: 36px;
  font-weight: 600;
  line-height: 1.3;
  letter-spacing: -0.01em;
  color: #E0E0E0;
}
```

**Ejemplo de uso:**
```html
<h2>Características Principales</h2>
```

---

### **Nivel 3: Subtítulos**

```css
h3 {
  font-family: 'JetBrains Mono', monospace;
  font-size: 28px;
  font-weight: 600;
  line-height: 1.4;
  color: #E0E0E0;
}
```

**Ejemplo de uso:**
```html
<h3>Setup en 60 segundos</h3>
```

---

### **Nivel 4: Títulos de Card**

```css
h4 {
  font-family: 'JetBrains Mono', monospace;
  font-size: 24px;
  font-weight: 500;
  line-height: 1.4;
  color: #E0E0E0;
}
```

**Ejemplo de uso:**
```html
<h4>Plan Premium</h4>
```

---

### **Body Text**

```css
body, p {
  font-family: 'Inter', sans-serif;
  font-size: 16px;
  font-weight: 400;
  line-height: 1.6;
  color: #E0E0E0;
}
```

**Ejemplo de uso:**
```html
<p>
  uSipipo es un servicio VPN que funciona dentro de Telegram. 
  Sin apps complejas, sin configuraciones técnicas.
</p>
```

---

### **Texto Secundario**

```css
.text-secondary {
  font-family: 'Inter', sans-serif;
  font-size: 14px;
  font-weight: 400;
  line-height: 1.5;
  color: #888888;
}
```

**Ejemplo de uso:**
```html
<span class="text-secondary">Última actualización: 2026-03-27</span>
```

---

### **Código Inline**

```css
code {
  font-family: 'JetBrains Mono', monospace;
  font-size: 14px;
  font-weight: 400;
  line-height: 1.5;
  color: #00F0FF;
  background-color: rgba(0, 240, 255, 0.1);
  padding: 2px 6px;
  border-radius: 4px;
}
```

**Ejemplo de uso:**
```html
<p>Usa el comando <code>/start</code> para comenzar</p>
```

---

## ✨ Estilos Especiales

### **Neon Text (Títulos Destacados)**

```css
.neon-text {
  font-family: 'JetBrains Mono', monospace;
  color: #00F0FF;
  text-shadow: 
    0 0 10px rgba(0, 240, 255, 0.8),
    0 0 20px rgba(0, 240, 255, 0.6),
    0 0 30px rgba(0, 240, 255, 0.4);
}
```

**Uso:**
- ✅ Títulos de hero
- ✅ CTAs principales
- ✅ Texto promocional

---

### **Gradient Text**

```css
.gradient-text {
  font-family: 'JetBrains Mono', monospace;
  background: linear-gradient(135deg, #00F0FF, #FF00AA);
  -webkit-background-clip: text;
  -webkit-text-fill-color: transparent;
  background-clip: text;
}
```

**Uso:**
- ✅ Títulos promocionales
- ✅ Features destacados
- ✅ Landing pages

---

### **Terminal Text**

```css
.terminal-text {
  font-family: 'JetBrains Mono', monospace;
  color: #00FF41;
  background-color: #0A0A0F;
  padding: 16px;
  border-radius: 8px;
  border-left: 3px solid #00FF41;
}
```

**Uso:**
- ✅ Comandos
- ✅ Logs
- ✅ Outputs de terminal

---

## 📱 Implementación por Plataforma

### **Web (CSS)**

```css
/* Importar fuentes */
@import url('https://fonts.googleapis.com/css2?family=JetBrains+Mono:wght@400;500;600;700&family=Inter:wght@400;500;600;700&display=swap');

:root {
  --font-mono: 'JetBrains Mono', monospace;
  --font-sans: 'Inter', sans-serif;
  
  --text-xs: 12px;
  --text-sm: 14px;
  --text-base: 16px;
  --text-lg: 20px;
  --text-xl: 24px;
  --text-2xl: 28px;
  --text-3xl: 36px;
  --text-4xl: 48px;
}

body {
  font-family: var(--font-sans);
  font-size: var(--text-base);
  line-height: 1.6;
}

h1, h2, h3, h4, h5, h6 {
  font-family: var(--font-mono);
  line-height: 1.3;
}
```

---

### **Flutter/Dart**

```dart
// lib/core/theme/cyberpunk_theme.dart
import 'package:google_fonts/google_fonts.dart';
import 'package:flutter/material.dart';

class CyberpunkTheme {
  static ThemeData get lightTheme {
    return ThemeData(
      textTheme: TextTheme(
        displayLarge: GoogleFonts.jetBrainsMono(
          fontSize: 48,
          fontWeight: FontWeight.bold,
          height: 1.2,
        ),
        displayMedium: GoogleFonts.jetBrainsMono(
          fontSize: 36,
          fontWeight: FontWeight.w600,
          height: 1.3,
        ),
        headlineLarge: GoogleFonts.jetBrainsMono(
          fontSize: 28,
          fontWeight: FontWeight.w600,
          height: 1.4,
        ),
        headlineMedium: GoogleFonts.jetBrainsMono(
          fontSize: 24,
          fontWeight: FontWeight.w500,
          height: 1.4,
        ),
        bodyLarge: GoogleFonts.inter(
          fontSize: 16,
          fontWeight: FontWeight.normal,
          height: 1.6,
        ),
        bodyMedium: GoogleFonts.inter(
          fontSize: 14,
          fontWeight: FontWeight.normal,
          height: 1.5,
        ),
        labelSmall: GoogleFonts.inter(
          fontSize: 12,
          fontWeight: FontWeight.normal,
          height: 1.4,
        ),
      ),
    );
  }
}
```

---

### **React Native**

```javascript
// src/theme/typography.js
export const typography = {
  fonts: {
    mono: 'JetBrainsMono-Regular',
    sans: 'Inter-Regular',
  },
  sizes: {
    xs: 12,
    sm: 14,
    base: 16,
    lg: 20,
    xl: 24,
    '2xl': 28,
    '3xl': 36,
    '4xl': 48,
  },
  weights: {
    normal: '400',
    medium: '500',
    semibold: '600',
    bold: '700',
  },
};

// Uso en componentes
const styles = StyleSheet.create({
  h1: {
    fontFamily: typography.fonts.mono,
    fontSize: typography.sizes['4xl'],
    fontWeight: typography.weights.bold,
    lineHeight: 57.6, // 1.2 * 48
  },
  body: {
    fontFamily: typography.fonts.sans,
    fontSize: typography.sizes.base,
    fontWeight: typography.weights.normal,
    lineHeight: 25.6, // 1.6 * 16
  },
});
```

---

## ♿ Accesibilidad

### **Tamaño Mínimo de Texto**

| Contexto | Tamaño Mínimo |
|----------|---------------|
| Cuerpo de texto | 16px |
| Texto secundario | 14px |
| Captions | 12px (solo con contraste alto) |
| Botones | 16px |

### **Contraste Requerido**

| Elemento | WCAG AA | WCAG AAA |
|----------|---------|----------|
| Texto normal | 4.5:1 | 7:1 |
| Texto grande (>18px) | 3:1 | 4.5:1 |

### **Combinaciones Aprobadas**

| Fondo | Texto | Ratio | Nivel |
|-------|-------|-------|-------|
| `#0A0A0F` | `#E0E0E0` | 15.2:1 | ✅ AAA |
| `#12121A` | `#E0E0E0` | 12.8:1 | ✅ AAA |
| `#0A0A0F` | `#00F0FF` | 8.6:1 | ✅ AAA |
| `#0A0A0F` | `#888888` | 5.8:1 | ✅ AA |

---

## 🚫 Anti-patrones

### ❌ No Hacer

```css
/* Mal: Demasiadas fuentes diferentes */
h1 { font-family: 'Roboto'; }
h2 { font-family: 'Open Sans'; }
body { font-family: 'Lato'; }

/* Mal: Texto neón en cuerpo */
p {
  color: #00F0FF;
  text-shadow: 0 0 10px #00F0FF;
}

/* Mal: JetBrains Mono en párrafos largos */
p {
  font-family: 'JetBrains Mono';
  /* Muy ancho, difícil de leer */
}

/* Mal: Inter en títulos */
h1 {
  font-family: 'Inter';
  /* Pierde identidad cyberpunk */
}
```

### ✅ Hacer

```css
/* Bien: Consistencia */
h1, h2, h3, h4 {
  font-family: 'JetBrains Mono';
}

body, p, span {
  font-family: 'Inter';
}

/* Bien: Neón solo en títulos */
h1.neon {
  color: #00F0FF;
  text-shadow: 0 0 20px rgba(0, 240, 255, 0.8);
}

/* Bien: Contraste adecuado */
p {
  color: #E0E0E0; /* No blanco puro */
}
```

---

## 📚 Recursos

### **Descarga de Fuentes**

- [JetBrains Mono](https://www.jetbrains.com/lp/mono/)
- [Inter](https://rsms.me/inter/)
- [Google Fonts - JetBrains Mono](https://fonts.google.com/specimen/JetBrains+Mono)
- [Google Fonts - Inter](https://fonts.google.com/specimen/Inter)

### **Herramientas**

- [Type Scale Calculator](https://type-scale.com/)
- [Font Pair](https://www.fontpair.co/)
- [WebAIM Contrast Checker](https://webaim.org/resources/contrastchecker/)

---

## 📚 Recursos Relacionados

- [Identidad de Marca](identity.md)
- [Paleta de Colores](color-palette.md)
- [Logo Assets](logo-assets.md)

---

**Última actualización:** 2026-03-27
