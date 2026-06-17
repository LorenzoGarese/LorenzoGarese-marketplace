# Setup / reproducir el entorno

Cómo dejar una PC nueva (o recuperada) con el mismo entorno de Claude Code.

La fuente de verdad de "qué tengo instalado" es `~/.claude/settings.json`. Acá guardamos
una versión **saneada** de las claves portables en [`claude-settings.snippet.json`](claude-settings.snippet.json)
(sin paths absolutos ni secretos).

## Camino rápido

1. Instalá Claude Code.
2. Abrí `~/.claude/settings.json` y pegá/mergeá las claves de `claude-settings.snippet.json`
   (`model`, `enabledPlugins`, `extraKnownMarketplaces`, `mcpServers`).
3. Ajustá el path de `mcpServers` (ver nota abajo).
4. Abrí Claude Code. Detecta los marketplaces y baja los plugins habilitados solo.

## Camino manual (por UI)

```
/plugin marketplace add ruvnet/ruflo
/plugin marketplace add LorenzoGarese/LorenzoGarese-marketplace
```
(`claude-plugins-official` ya viene por defecto.)

Después instalá:
```
/plugin install superpowers@claude-plugins-official
/plugin install security-guidance@claude-plugins-official
/plugin install ruflo-core@ruflo
/plugin install ruflo-swarm@ruflo
/plugin install craft@LorenzoGarese-marketplace
```

## Plugins generales opcionales (NO instalados todavía)

Candidatos de propósito general para evaluar. **No están en mi `settings.json`** todavía, así que
NO van en el snippet de backup — quedan acá como lista curada. Si instalás alguno, después
refrescás el snippet para que entre al set reproducible.

```
# gsd (Get Shit Done) — workflow estructurado. Trae un MCP server.
/plugin marketplace add jnuyens/gsd-plugin
/plugin install gsd@gsd-plugin

# claude-mem — sistema de memoria/compresión de contexto para Claude Code
/plugin marketplace add thedotmack/claude-mem
/plugin install claude-mem@thedotmack

# context-mode — manejo de contexto (plugin MCP)
/plugin marketplace add mksglu/context-mode
/plugin install context-mode@context-mode
```

Repos verificados (tienen `.claude-plugin/marketplace.json`): `jnuyens/gsd-plugin`,
`thedotmack/claude-mem`, `mksglu/context-mode`.

## Qué NO está en el snippet (a propósito)

- **`permissions`**: son por-proyecto y con rutas absolutas de esta máquina. Se reconstruyen
  solas a medida que aprobás cosas. No tiene sentido versionarlas.
- **`mcpServers` → path local**: `agents-md-generator` apunta a una carpeta local
  (`<INSTALL_DIR>\agents-md-generator-0.5.3`). En el snippet va como placeholder;
  reemplazalo por la ruta real donde lo instales.

## Antes de commitear cambios a este snippet

`settings.json` puede tener secretos (tokens en `mcpServers.env`, etc.). **Revisá siempre**
antes de pegar una versión nueva acá. La que está commiteada hoy está limpia.

## Actualizar las skills de `craft` desde su upstream

Varias skills de `craft` son vendoreadas (copiadas de otros repos). Para traerlas a la última
sin perder los parches locales, ver [`../scripts/update-skills.mjs`](../scripts/update-skills.mjs)
y el manifiesto [`../plugins/craft/skills/SOURCES.json`](../plugins/craft/skills/SOURCES.json).
