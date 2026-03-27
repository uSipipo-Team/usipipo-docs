# Reglas de Protección de Rama - uSipipo Docs

## 📋 Configuración Actual

### Rama Protegida: `main`

| Configuración | Valor | Descripción |
|---------------|-------|-------------|
| **Enforce admins** | ✅ Activado | Las reglas aplican también a administradores |
| **Required PR reviews** | ✅ 1 review | Se requiere al menos 1 aprobación |
| **Dismiss stale reviews** | ✅ Activado | Las aprobaciones se descartan si hay nuevos commits |
| **Allow force pushes** | ❌ Desactivado | No se permiten force pushes |
| **Allow deletions** | ❌ Desactivado | No se permite eliminar la rama |

---

## 🔄 Flujo de Trabajo para Contribuir

### 1. Crear rama de feature
```bash
git checkout main
git pull origin main
git checkout -b feature/nueva-documentacion
```

### 2. Hacer cambios y commit
```bash
git add .
git commit -m "docs: agregar nueva documentación de X"
git push origin feature/nueva-documentacion
```

### 3. Crear Pull Request
```bash
gh pr create \
  --title "docs: agregar nueva documentación de X" \
  --body "## Cambios\n- Agregada documentación de X\n\n## Checklist\n- [ ] Documentación revisada\n- [ ] Ortografía verificada" \
  --base main \
  --head feature/nueva-documentacion
```

### 4. Esperar review
- Al menos 1 aprobador requerido
- Los CODEOWNERS son notificados automáticamente
- Las aprobaciones anteriores se descartan si se agregan nuevos commits

### 5. Merge del PR
- Merge desde GitHub UI
- O usar `gh pr merge <number> --merge`
- La rama feature se puede eliminar después del merge

---

## 👥 CODEOWNERS

Los siguientes usuarios/equipos son notificados para review:

| Path | Owners |
|------|--------|
| `*` | @uSipipo-Team/developers |
| `/brand/` | @uSipipo-Team/developers |
| `/context/` | @uSipipo-Team/developers |
| `/prds/` | @uSipipo-Team/developers |
| `/flows/` | @uSipipo-Team/developers |
| `/technology/` | @uSipipo-Team/developers |
| `/apis/` | @uSipipo-Team/developers |

---

## 🚫 Restricciones

- ❌ No push directo a `main`
- ❌ No force push
- ❌ No eliminar la rama `main`
- ✅ PRs requieren 1+ aprobación
- ✅ CI/CD debe pasar (cuando se configure)

---

## 📊 Ver Estado de Protección

```bash
# Ver configuración de protección
gh api repos/uSipipo-Team/usipipo-docs/branches/main/protection

# Ver estado de la rama
gh api repos/uSipipo-Team/usipipo-docs/branches/main
```

---

**Última actualización:** 2026-03-27
