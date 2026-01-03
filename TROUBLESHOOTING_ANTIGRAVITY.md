# Documentación: Problema de "Tool Bloat" en roam-research-mcp + Antigravity

> **Propósito:** Este documento describe un problema conocido al usar el servidor MCP `roam-research-mcp` con Antigravity. Contiene toda la información necesaria para que un asistente AI nuevo pueda diagnosticar y resolver el problema.

---

## Síntomas del Problema

1. **Error "Agent Terminated"**: Al iniciar Antigravity, el servidor MCP se conecta brevemente pero luego el proceso termina abruptamente.
2. **El servidor funciona en Claude Desktop**: El mismo servidor MCP funciona perfectamente con Claude Desktop de Anthropic.
3. **Funciona con pocas herramientas**: Si se deshabilitan la mayoría de las herramientas en `mcp_config.json`, el servidor conecta correctamente.
4. **No hay errores en logs**: El servidor no reporta errores internos; simplemente muere.

---

## Causa Raíz: Tool Bloat

### ¿Qué es Tool Bloat?
El protocolo MCP requiere que el servidor envíe la lista completa de herramientas disponibles (`tools/list`) durante la inicialización. El servidor `roam-research-mcp` expone **18+ herramientas** con descripciones muy detalladas.

### Tamaño del Payload
```
Cada herramienta tiene:
- Nombre: ~30 caracteres
- Descripción: 200-800 caracteres (algunas tienen párrafos completos)
- inputSchema: 500-2000 caracteres con propiedades anidadas

Total para 18 herramientas: ~25,000-35,000 caracteres de JSON
```

### ¿Por qué Claude Desktop funciona y Antigravity no?
| Aspecto | Claude Desktop | Antigravity |
|---------|----------------|-------------|
| Buffer de respuesta | Grande, optimizado | Posiblemente limitado |
| Timeout de inicialización | Largo | Más corto |
| Madurez MCP | Creadores del protocolo | Implementación más nueva |

### Evidencia de la Comunidad
- [RFC LTAP: Lazy Tool Loading](https://github.com/modelcontextprotocol/modelcontextprotocol/discussions/1945)
- [Handling tool bloat - hundreds of tools](https://github.com/modelcontextprotocol/modelcontextprotocol/discussions/2036)

---

## Solución Implementada

### Enfoque: Reducción de Herramientas en el Código Fuente

En lugar de depender solo de `disabledTools` en la configuración, **comentamos las herramientas no esenciales directamente en el código** del servidor. Esto reduce el payload de `tools/list` desde el origen.

### Archivo a Modificar
```
src/tools/schemas.ts
```

### Herramientas Recomendadas (10 herramientas - FUNCIONA)
```typescript
// ACTIVAS (no comentar):
roam_add_todo
roam_fetch_page_by_title
roam_create_outline
roam_import_markdown
roam_search_by_text
roam_markdown_cheatsheet
roam_remember
roam_recall
roam_process_batch_actions
roam_fetch_block_with_children

// COMENTADAS (envueltas en /* */):
roam_create_page
roam_search_for_tag
roam_search_by_status
roam_search_block_refs
roam_search_hierarchy
roam_search_by_date
roam_find_pages_modified_today
roam_datomic_query
roam_create_table
roam_update_page_markdown
```

### Cómo Comentar una Herramienta
```typescript
// ANTES (activa):
roam_create_page: {
  name: 'roam_create_page',
  description: '...',
  inputSchema: { ... }
},

// DESPUÉS (comentada):
/*
roam_create_page: {
  name: 'roam_create_page',
  description: '...',
  inputSchema: { ... }
},
*/
```

### Después de Modificar, Recompilar
```powershell
cd c:\Users\redk8\OneDrive\Documentos\proyectosVibeCoding\roam-research-mcp
npm run build
```

---

## Configuración de mcp_config.json

### Ubicación
```
c:\Users\redk8\.gemini\antigravity\mcp_config.json
```

### Plantilla para Nuevo Grafo
```json
{
  "mcpServers": {
    "roam-research-NOMBRE_GRAFO": {
      "command": "node",
      "args": [
        "c:\\Users\\redk8\\OneDrive\\Documentos\\proyectosVibeCoding\\roam-research-mcp\\build\\index.js"
      ],
      "env": {
        "ROAM_API_TOKEN": "roam-graph-token-XXXXXXX",
        "ROAM_GRAPH_NAME": "NOMBRE_GRAFO"
      },
      "disabledTools": []
    }
  }
}
```

### Si Necesitas Deshabilitar Herramientas Adicionales
```json
"disabledTools": [
  "roam_remember",
  "roam_recall"
]
```

---

## Troubleshooting

### Problema: Sigue crasheando con 10 herramientas
**Posible causa:** Descripciones aún muy largas.
**Solución:** Reducir a 6-8 herramientas o acortar descripciones en `schemas.ts`.

### Problema: Error "MODULE_NOT_FOUND"
**Causa:** PowerShell está en directorio incorrecto.
**Solución:** 
```powershell
cd c:\Users\redk8\OneDrive\Documentos\proyectosVibeCoding\roam-research-mcp
```

### Problema: Error "Missing environment variables"
**Causa:** Falta archivo `.env` para pruebas manuales.
**Solución:** Crear `.env` en raíz del proyecto:
```
ROAM_API_TOKEN=tu-token-aqui
ROAM_GRAPH_NAME=nombre-grafo
```

### Problema: Build falla con "cat not found"
**Causa:** Script de build usa comandos Unix.
**Solución:** En `package.json`, cambiar:
```json
"build": "tsc"
```

---

## Comandos de Verificación

### Probar servidor manualmente
```powershell
Write-Output '{"jsonrpc": "2.0", "method": "tools/list", "id": 1}' | node build/index.js
```
**Resultado esperado:** JSON con lista de herramientas.

### Ver estado de Git
```powershell
git remote -v
git branch
git status
```

---

## Resumen de Límites Conocidos

| Escenario | Herramientas | Resultado |
|-----------|--------------|-----------|
| 18 herramientas (original) | Todas | ❌ Crash |
| 10 herramientas | Balanceado | ✅ Funciona |
| 6 herramientas | Mínimo | ✅ Funciona |
| 1 herramienta | Solo cheatsheet | ✅ Funciona |

**Recomendación segura:** No exceder 10-12 herramientas activas en `schemas.ts`.

---

## Estructura del Proyecto

```
roam-research-mcp/
├── src/
│   ├── tools/
│   │   └── schemas.ts      ← AQUÍ se comentan herramientas
│   ├── server/
│   │   └── roam-server.ts
│   └── index.ts
├── build/                   ← Código compilado (no editar)
├── package.json            ← Scripts de build
├── .env                    ← Credenciales (crear manualmente)
└── .git/config             ← Remotos (origin/upstream)
```

---

## Contacto y Referencias

- **Fork del usuario:** https://github.com/camiloluvino/roam-research-mcp
- **Repo original:** https://github.com/2b3pro/roam-research-mcp
- **Rama con fix:** `fix/antigravity-tools`

---

## Actualización: Solución Refinada (Enero 2026)

Las siguientes secciones documentan lecciones aprendidas y una estrategia más robusta para trabajar con Antigravity.

---

## Estrategia de Archivos Separados por Grafo

### Problema
Consolidar múltiples grafos de Roam en un solo `mcp_config.json` con varias herramientas cada uno causa inestabilidad en Antigravity. Incluso con 4-5 herramientas por grafo, el payload combinado puede exceder los límites.

### Solución
Mantener **un archivo de configuración por grafo**. Cuando necesites trabajar con un grafo específico:
1. Reemplazar el contenido de `mcp_config.json` con la configuración de ese grafo
2. Reiniciar Antigravity

### Ubicación Sugerida para Backups
```
c:\Users\redk8\.gemini\antigravity\configs\
├── mcp_config_campodepruebas.json
├── mcp_config_mundano.json
├── mcp_config_cosasAcademicas.json
└── mcp_config_kulturama.json
```

---

## Template Mínimo Recomendado (3 herramientas)

Basado en pruebas extensivas, estas **3 herramientas** cubren las necesidades más comunes sin causar "tool bloat":

| Herramienta | Función |
|-------------|---------|
| `roam_create_outline` | Crear contenido estructurado en páginas |
| `roam_search_by_text` | Buscar texto en el grafo |
| `roam_datomic_query` | Búsquedas avanzadas (requiere descomentar en código) |

### JSON Template
```json
{
  "mcpServers": {
    "roam-research-NOMBRE_GRAFO": {
      "command": "node",
      "args": [
        "c:\\Users\\redk8\\OneDrive\\Documentos\\proyectosVibeCoding\\roam-research-mcp\\build\\index.js"
      ],
      "env": {
        "ROAM_API_TOKEN": "roam-graph-token-XXXXXXXXX",
        "ROAM_GRAPH_NAME": "NOMBRE_GRAFO"
      },
      "disabledTools": [
        "roam_add_todo",
        "roam_fetch_page_by_title",
        "roam_import_markdown",
        "roam_markdown_cheatsheet",
        "roam_remember",
        "roam_recall",
        "roam_process_batch_actions",
        "roam_fetch_block_with_children"
      ],
      "disabled": false
    }
  }
}
```

---

## Dos Niveles de Control (Importante)

### ¿Por qué `disabledTools` no es suficiente?

Existen **dos niveles** donde se controlan las herramientas:

```
┌─────────────────────────────────────────────────────────────────┐
│  NIVEL 1: Código Fuente (src/tools/schemas.ts)                  │
│  ────────────────────────────────────────────────────────────── │
│  Herramientas comentadas con /* */ NO existen para el servidor  │
│  Reduce el payload ANTES de enviarlo                            │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼ El servidor envía TODAS las herramientas no comentadas
                              
┌─────────────────────────────────────────────────────────────────┐
│  NIVEL 2: Configuración Cliente (mcp_config.json)               │
│  ────────────────────────────────────────────────────────────── │
│  disabledTools filtra DESPUÉS de recibir el payload             │
│  Si el crash ocurre durante recepción, este filtro llega tarde  │
└─────────────────────────────────────────────────────────────────┘
```

### Implicación Práctica
- Si una herramienta está **comentada en el código**, no aparecerá aunque la quites de `disabledTools`
- Para habilitar `roam_datomic_query`, debes:
  1. Descomentarla en `src/tools/schemas.ts`
  2. Recompilar con `npm run build`
  3. Quitarla de `disabledTools` en `mcp_config.json`

---

## Templates por Grafo

### campodepruebas
```json
{
  "mcpServers": {
    "roam-research-campodepruebas": {
      "command": "node",
      "args": ["c:\\Users\\redk8\\OneDrive\\Documentos\\proyectosVibeCoding\\roam-research-mcp\\build\\index.js"],
      "env": {
        "ROAM_API_TOKEN": "roam-graph-token-N1QX6YFwWSf9MmiacikRr-pKb847-",
        "ROAM_GRAPH_NAME": "campodepruebas"
      },
      "disabledTools": ["roam_add_todo","roam_fetch_page_by_title","roam_import_markdown","roam_markdown_cheatsheet","roam_remember","roam_recall","roam_process_batch_actions","roam_fetch_block_with_children"],
      "disabled": false
    }
  }
}
```

### mundano
```json
{
  "mcpServers": {
    "roam-research-mundano": {
      "command": "node",
      "args": ["c:\\Users\\redk8\\OneDrive\\Documentos\\proyectosVibeCoding\\roam-research-mcp\\build\\index.js"],
      "env": {
        "ROAM_API_TOKEN": "roam-graph-token-zhodLiFyErPgqEo7oM_NCYAiDqg7J",
        "ROAM_GRAPH_NAME": "mundano"
      },
      "disabledTools": ["roam_add_todo","roam_fetch_page_by_title","roam_import_markdown","roam_markdown_cheatsheet","roam_remember","roam_recall","roam_process_batch_actions","roam_fetch_block_with_children"],
      "disabled": false
    }
  }
}
```

### cosasAcademicas
```json
{
  "mcpServers": {
    "roam-research-cosasAcademicas": {
      "command": "node",
      "args": ["c:\\Users\\redk8\\OneDrive\\Documentos\\proyectosVibeCoding\\roam-research-mcp\\build\\index.js"],
      "env": {
        "ROAM_API_TOKEN": "roam-graph-token-DfKgeLhv5yxLfTxQ1uDOwRadZGEz8",
        "ROAM_GRAPH_NAME": "cosasAcademicas"
      },
      "disabledTools": ["roam_add_todo","roam_fetch_page_by_title","roam_import_markdown","roam_markdown_cheatsheet","roam_remember","roam_recall","roam_process_batch_actions","roam_fetch_block_with_children"],
      "disabled": false
    }
  }
}
```

### kulturama
```json
{
  "mcpServers": {
    "roam-research-kulturama": {
      "command": "node",
      "args": ["c:\\Users\\redk8\\OneDrive\\Documentos\\proyectosVibeCoding\\roam-research-mcp\\build\\index.js"],
      "env": {
        "ROAM_API_TOKEN": "roam-graph-token-zxXSjclQoxfsjHmlyLrI0PNjlrbTh",
        "ROAM_GRAPH_NAME": "kulturama"
      },
      "disabledTools": ["roam_add_todo","roam_fetch_page_by_title","roam_import_markdown","roam_markdown_cheatsheet","roam_remember","roam_recall","roam_process_batch_actions","roam_fetch_block_with_children"],
      "disabled": false
    }
  }
}
```

---

## Notas Finales

Esta documentación refleja el estado de la solución a **Enero 2026**. La limitación de "tool bloat" es un problema conocido en la comunidad MCP y puede ser resuelto en futuras versiones de Antigravity o del protocolo MCP.

**Solución ideal futura:** Implementar filtrado de herramientas a nivel de servidor mediante variables de entorno, eliminando la necesidad de comentar código.
