# 🔐 Política de Seguridad DevSecOps — test.com

**Elaborado por:** Sebastian
**Aplicable a:** Todos los repositorios bajo test.com
**Versión:** 1.0 | Fecha: 2026-03-07

---

## 1. Security Gate — Regla de Oro

> **Ningún código llega a producción sin pasar el pipeline de seguridad completo.**

El workflow `.github/workflows/devsecops-gate.yml` es un **required status check** en las ramas `main` y `master`. El merge está bloqueado hasta que todos los checks pasen.

---

## 2. Checks obligatorios y umbrales de bloqueo

> **Política:** el pipeline **bloquea únicamente ante vulnerabilidades CRITICAL**.
> HIGH, MEDIUM y LOW se reportan como avisos en los artefactos del workflow
> y deben ser revisados por el equipo en el siguiente sprint.

| Check | Herramienta | Bloquea si... | Solo aviso si... |
|-------|------------|---------------|------------------|
| SAST | **Semgrep** | Regla mapea a RCE/Injection directa (ERROR + rule en lista CRITICAL) | ERROR de otras reglas, todos los WARNING |
| SAST | **Bandit** | B605 shell injection, B301 pickle RCE, B302 marshal, B506 yaml RCE — con confianza HIGH/MEDIUM | Cualquier HIGH fuera de esos test IDs, todos los MEDIUM |
| SCA  | **pip-audit** | CVE con CVSS ≥ 9.0 | CVEs con CVSS < 9.0 |
| IaC  | **Hadolint** | **Nunca bloquea** — todos los hallazgos son avisos | Todo (DL3002, DL3007, etc.) |
| IaC  | **Trivy** | Misconfiguracion con `severity=CRITICAL` | Misconfigs HIGH |
| IaC  | **EOL images** | Siempre — imagen sin soporte = sin parches de seguridad | Tags `:latest` |

---

## 3. Reglas de código obligatorias

### Python / Flask
- ❌ `app.run(debug=True)` — **bloqueado** en cualquier rama
- ❌ `pickle.loads()` con datos de request/cookie
- ❌ `os.system()` / `os.popen()` con input de usuario
- ❌ `render_template_string(tpl % user_input)`
- ✅ `subprocess.run(['cmd', arg])` con lista de argumentos
- ✅ Variables de entorno para configuración sensible

### PHP
- ❌ SQL concatenado con `$_GET`/`$_POST` sin `prepare()`
- ❌ `echo $_GET[...]` sin `htmlspecialchars()`
- ✅ Prepared statements para todas las queries

### Dockerfiles
- ❌ Sin instrucción `USER nonroot` antes de `CMD`/`ENTRYPOINT`
- ❌ Imágenes EOL: `php:7.x`, `node:14`, `python:2`, `ubuntu:18.04`
- ❌ Tag `:latest` sin hash SHA256
- ✅ `USER appuser` con usuario no privilegiado creado explícitamente
- ✅ Imagen base con versión semántica fijada o SHA256

### Dependencias
- ✅ `requirements.txt` con versiones exactas (`==`) para Python
- ✅ `package-lock.json` commiteado para Node.js
- ❌ Dependencias con CVE conocido (bloqueado por pip-audit)

---

## 4. Configurar Branch Protection en GitHub

Para que el pipeline bloquee efectivamente los merges:

```
GitHub → Settings → Branches → Add branch protection rule

Branch name pattern: main
✅ Require a pull request before merging
✅ Require status checks to pass before merging
   → Buscar y añadir: "🔐 Security Gate — Decisión Final"
   → Buscar y añadir: "🔍 SAST — Semgrep"
   → Buscar y añadir: "🐍 SAST — Bandit (Python)"
   → Buscar y añadir: "📦 SCA — pip-audit"
   → Buscar y añadir: "🐳 IaC — Hadolint (Dockerfiles)"
   → Buscar y añadir: "🔭 IaC — Trivy Config & Image Scan"
✅ Require branches to be up to date before merging
✅ Do not allow bypassing the above settings
```

---

## 5. Flujo del Pipeline

```
┌──────────────────────────────────────────────────────────────┐
│                    DEVELOPER → PR → REVIEW                   │
└────────────────────────┬─────────────────────────────────────┘
                         │ push / pull_request
                         ▼
         ┌───────────────────────────────┐
         │   🔐 DevSecOps Security Gate  │
         ├───────────┬───────────────────┤
         │           │                   │
    ┌────▼───┐  ┌────▼───┐  ┌───────────▼──────────┐
    │ SAST   │  │  SCA   │  │         IaC          │
    │        │  │        │  │                      │
    │Semgrep │  │pip-    │  │ Hadolint + Trivy     │
    │Bandit  │  │audit   │  │ EOL image check      │
    └────┬───┘  └────┬───┘  └───────────┬──────────┘
         │           │                   │
         └───────────┴─────────┬─────────┘
                               ▼
                    ┌──────────────────┐
                    │ Security Gate    │
                    │ Decision Job     │
                    └────────┬─────────┘
                             │
              ┌──────────────┴──────────────┐
              ▼                             ▼
     ✅ ALL PASSED                  ❌ FAILURES FOUND
     Merge permitido                Merge BLOQUEADO
     Deploy a staging               PR comentado con
                                    hallazgos
```

---

## 6. Excepciones y falsos positivos

Si un hallazgo es un falso positivo documentado:

1. Agregar comentario `# nosemgrep: <rule-id>` en la línea afectada
2. Documentar la razón en el PR
3. Obtener aprobación del Security Lead (Sebastian)
4. Para Bandit: `# noqa: B<id>`

**No se permite suprimir hallazgos sin aprobación documentada.**

---

## 7. Herramientas y versiones fijadas

| Herramienta | Versión | Propósito |
|------------|---------|-----------|
| semgrep | 1.154.0 | SAST multi-lenguaje |
| bandit | 1.7.10 | SAST Python |
| pip-audit | 2.7.3 | SCA Python CVEs |
| hadolint | latest | IaC Dockerfiles |
| trivy | latest | IaC + Image scanning |

---

*test.com DevSecOps — Elaborado por Sebastian*
