# Backend `vertical-slice` Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Crear el plugin `backend` con su primera skill `vertical-slice`, que genera una vertical slice CQRS end-to-end en backends .NET (Clean Architecture + MediatR + FluentValidation) siguiendo las convenciones de casa.

**Architecture:** Skill de instrucciones (markdown) + templates de referencia, sin scripts ni dependencias de runtime. El `SKILL.md` orquesta: detectar proyectos por convención → interview corto → generar los 6 archivos → `dotnet build`. La carpeta `reference/` guarda las reglas y los templates anotados.

**Tech Stack:** Claude Code plugin/skill (markdown + JSON). Target del código generado: C# / .NET, MediatR, FluentValidation, EF Core, ASP.NET. El plugin en sí NO ejecuta C#.

## Global Constraints

- El plugin es **markdown + JSON only**: sin scripts node/python, sin dependencias (como `craft`, distinto de `ui-ux-pro-max`).
- Frontmatter de `SKILL.md`: `name` y `description` obligatorios. La `description` debe disparar al pedir crear feature/endpoint/command/query/slice en backend .NET CQRS+MediatR.
- `allowed-tools` de la skill: `Bash(dotnet build*)` (para la verificación final).
- Estilo del código generado (canónico, "moderno"): **primary constructor** para DI, `sealed record` para Command/Query/Response, `sealed class` para Handler/Validator, subcarpeta `Commands/` o `Queries/`, **namespace = carpeta**, mensajes de dominio en **español**, excepciones de dominio desde `*.Application.Exceptions._4XX`, marker `IWriteRequest` en commands, wrapper de respuesta vía `BaseApiController` (`Success`/`CreatedSuccess`).
- **Genericidad**: detectar proyectos por convención (carpeta con `Features/` + MediatR = app layer, sea `.Application` o `.Core`; carpeta con `Controllers/` = web). **Nunca** hardcodear `Company.*`; aprender usings/markers/wrapper reales leyendo un feature vecino del repo destino.
- **Nunca pisar** un archivo existente: si el nombre ya existe, parar y preguntar.
- Referencias internas de la skill: rutas relativas (`reference/x.md`), que Claude resuelve solo. No hace falta `${CLAUDE_PLUGIN_ROOT}` (no hay scripts ejecutables).

---

## File Structure

- `plugins/backend/.claude-plugin/plugin.json` — manifiesto del plugin.
- `.claude-plugin/marketplace.json` — (modificar) agregar entrada `backend`.
- `plugins/backend/skills/vertical-slice/SKILL.md` — orquestación del flujo.
- `plugins/backend/skills/vertical-slice/reference/conventions.md` — reglas de casa + detección.
- `plugins/backend/skills/vertical-slice/reference/command-slice.md` — templates write.
- `plugins/backend/skills/vertical-slice/reference/query-slice.md` — templates read.
- `plugins/backend/skills/vertical-slice/reference/web-wiring.md` — Request DTO + endpoint.
- `README.md` — (modificar) listar el plugin `backend`.

---

### Task 1: Scaffold del plugin `backend` + entrada en el marketplace

**Files:**
- Create: `plugins/backend/.claude-plugin/plugin.json`
- Modify: `.claude-plugin/marketplace.json`

**Interfaces:**
- Produces: marketplace name `LorenzoGarese-marketplace` ahora lista un plugin `backend` con `source: ./plugins/backend`.

- [ ] **Step 1: Crear `plugins/backend/.claude-plugin/plugin.json`**

```json
{
  "name": "backend",
  "version": "0.1.0",
  "description": "Scaffolding de backend .NET con Clean Architecture + CQRS (MediatR) + FluentValidation. La skill vertical-slice genera una vertical slice end-to-end (Command/Query + Handler + Validator + Response + Request DTO + endpoint) siguiendo las convenciones del repo destino.",
  "author": { "name": "Garese" }
}
```

- [ ] **Step 2: Agregar la entrada `backend` en `.claude-plugin/marketplace.json`**

En el array `plugins`, después de la entrada `craft`, agregar:

```json
,
    { "name": "backend", "source": "./plugins/backend", "version": "0.1.0", "description": "Scaffolding de backend .NET (Clean Architecture + CQRS): vertical slices end-to-end con tus convenciones." }
```

- [ ] **Step 3: Validar que ambos JSON parsean**

Run: `node -e "JSON.parse(require('fs').readFileSync('.claude-plugin/marketplace.json','utf8')); JSON.parse(require('fs').readFileSync('plugins/backend/.claude-plugin/plugin.json','utf8')); console.log('OK')"`
Expected: `OK`

- [ ] **Step 4: Commit**

```bash
git add plugins/backend/.claude-plugin/plugin.json .claude-plugin/marketplace.json
git commit -m "backend: scaffold del plugin + entrada en marketplace"
```

---

### Task 2: `reference/conventions.md` (reglas de casa + detección)

**Files:**
- Create: `plugins/backend/skills/vertical-slice/reference/conventions.md`

**Interfaces:**
- Produces: el documento canónico de convenciones que `SKILL.md` y los otros reference linkean.

- [ ] **Step 1: Escribir `conventions.md`** con estas secciones, contenido completo:

  1. **Detección de proyectos** (determinística, con Glob/Grep):
     - App layer = el `.csproj` cuya carpeta contiene `Features/` y referencia `MediatR` (matchea `.Application` o `.Core` u otro sufijo). Glob `**/*.csproj`, luego Grep `MediatR` + existencia de `Features/`.
     - Web = carpeta con `Controllers/` y `Program.cs`.
     - Domain = `*.Domain` o el proyecto de entidades.
     - Si hay ambigüedad (varios candidatos), preguntar una vez.
  2. **Aprender del vecino**: antes de generar, leer 1 feature existente (un `*CommandHandler.cs`, su `*Command.cs`, su validator, y un `*Controller.cs`) para captar: namespace raíz real, marker de escritura (`IWriteRequest` o equivalente) y su `using`, wrapper de respuesta (`ApiResponse`/`Success`/`CreatedSuccess`), helper de repos (`GetByIdOrThrowAsync`), extensiones de validación (`.PasswordRules()`), y el tipo de excepciones (`Application.Exceptions._4XX`: `NotFoundException`, `BadRequestException`, `ConflictException`, `UnauthorizedException`).
  3. **Estilo canónico** (con un ejemplo correcto y uno incorrecto al lado):
     - Primary constructor para DI (no ctor explícito + campos `readonly`).
     - `sealed record` para Command/Query/Response; `sealed class` para Handler/Validator.
     - Carpeta `Features/<Area>/Commands/<Name>/` o `Queries/<Name>/`; **namespace = ruta de carpeta**.
     - `Handle(<X> request, CancellationToken ct)`.
     - Mensajes y comentarios de dominio en español.
  4. **Marker / transacción**: commands implementan `IRequest<TResponse>, IWriteRequest`; el `TransactionBehavior` hace el `SaveChanges`, así que el handler **no** llama `SaveChanges` salvo que el repo no use ese behavior (verificar leyendo `Behaviours/`).
  5. **Guardrails**: nunca pisar archivos; usar los namespaces reales detectados; insertar endpoints sin romper la clase.

- [ ] **Step 2: Verificar que el archivo no quedó vacío y linkea bien**

Run: `test -s plugins/backend/skills/vertical-slice/reference/conventions.md && echo OK`
Expected: `OK`

- [ ] **Step 3: Commit**

```bash
git add plugins/backend/skills/vertical-slice/reference/conventions.md
git commit -m "backend/vertical-slice: reference de convenciones y deteccion"
```

---

### Task 3: `reference/command-slice.md` (templates write)

**Files:**
- Create: `plugins/backend/skills/vertical-slice/reference/command-slice.md`

**Interfaces:**
- Consumes: convenciones de `conventions.md`.
- Produces: los 4 templates de Application para el camino command.

- [ ] **Step 1: Escribir `command-slice.md`** con los 4 templates anotados. Encabezar el archivo con: *"Los `<Placeholders>` son formas, no literales. Reemplazá `<AppRoot>` y los `using` por los reales detectados en el repo destino (ver conventions.md → Aprender del vecino)."* Templates:

  **`<Name>Command.cs`**
  ```csharp
  using <AppRoot>.Application.Abstractions.Mediator; // namespace real del marker IWriteRequest (detectar)
  using MediatR;

  namespace <AppRoot>.Application.Features.<Area>.Commands.<Name>;

  public sealed record <Name>Command(
      <Type1> <Field1>,
      <Type2> <Field2>)
      : IRequest<<Name>Response>, IWriteRequest;
  ```

  **`<Name>CommandHandler.cs`**
  ```csharp
  using <AppRoot>.Application.Abstractions.Persistence;
  using <AppRoot>.Application.Exceptions._4XX;
  using MediatR;
  using Microsoft.EntityFrameworkCore;

  namespace <AppRoot>.Application.Features.<Area>.Commands.<Name>;

  public sealed class <Name>CommandHandler(
      IAppDbContext db /*, + repos/abstractions detectados segun los campos */)
      : IRequestHandler<<Name>Command, <Name>Response>
  {
      public async Task<<Name>Response> Handle(<Name>Command request, CancellationToken ct)
      {
          // 1. Cargar lo que toque (repo.GetByIdOrThrowAsync(...) o db.<Set>.FirstOrDefaultAsync + throw NotFoundException).
          // 2. Validar invariantes de dominio que el Validator no cubre -> throw BadRequestException/ConflictException con mensaje en espanol.
          // 3. Mutar la entidad / db.<Set>.Add(nueva).
          // 4. NO llamar SaveChanges: el TransactionBehavior lo hace por el marker IWriteRequest
          //    (si el repo no usa ese behavior, agregar await db.SaveChangesAsync(ct)).
          return new <Name>Response(/* ... */);
      }
  }
  ```

  **`<Name>CommandValidator.cs`**
  ```csharp
  using FluentValidation;

  namespace <AppRoot>.Application.Features.<Area>.Commands.<Name>;

  public sealed class <Name>CommandValidator : AbstractValidator<<Name>Command>
  {
      public <Name>CommandValidator()
      {
          RuleFor(x => x.<Field1>)
              .NotEmpty().WithMessage("<mensaje en espanol>");
          // Reusar extensiones del repo cuando apliquen (ej: .PasswordRules()).
      }
  }
  ```

  **`<Name>Response.cs`**
  ```csharp
  namespace <AppRoot>.Application.Features.<Area>.Commands.<Name>;

  public sealed record <Name>Response(
      <Type> <Field>);
  ```

  Agregar una nota: si el command no devuelve datos, el Response puede ser un record vacío o usar el patrón del repo (algunos endpoints devuelven solo mensaje vía `Success("...")`).

- [ ] **Step 2: Verificar archivo no vacío**

Run: `test -s plugins/backend/skills/vertical-slice/reference/command-slice.md && echo OK`
Expected: `OK`

- [ ] **Step 3: Commit**

```bash
git add plugins/backend/skills/vertical-slice/reference/command-slice.md
git commit -m "backend/vertical-slice: templates de slice command (write)"
```

---

### Task 4: `reference/query-slice.md` (templates read)

**Files:**
- Create: `plugins/backend/skills/vertical-slice/reference/query-slice.md`

**Interfaces:**
- Consumes: convenciones de `conventions.md`.
- Produces: templates de Application para el camino query.

- [ ] **Step 1: Escribir `query-slice.md`** con el mismo encabezado de placeholders y estos templates:

  **`<Name>Query.cs`** (sin marker de escritura)
  ```csharp
  using MediatR;

  namespace <AppRoot>.Application.Features.<Area>.Queries.<Name>;

  public sealed record <Name>Query(
      <Type> <Param>)            // params de filtro/id; para "listar" incluir page/pageSize
      : IRequest<<Name>Response>;
  ```

  **`<Name>QueryHandler.cs`** (read: `AsNoTracking` + proyección)
  ```csharp
  using <AppRoot>.Application.Abstractions.Persistence;
  using <AppRoot>.Application.Exceptions._4XX;
  using MediatR;
  using Microsoft.EntityFrameworkCore;

  namespace <AppRoot>.Application.Features.<Area>.Queries.<Name>;

  public sealed class <Name>QueryHandler(IAppDbContext db)
      : IRequestHandler<<Name>Query, <Name>Response>
  {
      public async Task<<Name>Response> Handle(<Name>Query request, CancellationToken ct)
      {
          // GET por id: proyectar y throw NotFoundException si no existe.
          var item = await db.<Set>
              .AsNoTracking()
              .Where(x => x.Id == request.<Param>)
              .Select(x => new <Name>Response(x.<Field1>, x.<Field2>))
              .FirstOrDefaultAsync(ct)
              ?? throw new NotFoundException("<Entidad>");
          return item;
      }
  }
  ```

  **`<Name>Response.cs`**
  ```csharp
  namespace <AppRoot>.Application.Features.<Area>.Queries.<Name>;

  public sealed record <Name>Response(
      <Type> <Field>);
  ```

  Nota de paginación: para "listar", el Response es un record con `Items` + total/`page`/`pageSize`, y el handler aplica `.Skip((page-1)*pageSize).Take(pageSize)` (seguir el patrón de `GetLoginAttempts` si existe en el repo). Las queries normalmente **no** llevan Validator salvo que los params lo requieran.

- [ ] **Step 2: Verificar archivo no vacío**

Run: `test -s plugins/backend/skills/vertical-slice/reference/query-slice.md && echo OK`
Expected: `OK`

- [ ] **Step 3: Commit**

```bash
git add plugins/backend/skills/vertical-slice/reference/query-slice.md
git commit -m "backend/vertical-slice: templates de slice query (read)"
```

---

### Task 5: `reference/web-wiring.md` (Request DTO + endpoint)

**Files:**
- Create: `plugins/backend/skills/vertical-slice/reference/web-wiring.md`

**Interfaces:**
- Consumes: nombres de Command/Query/Response de los tasks 3 y 4.
- Produces: template del Request DTO y del endpoint, y las reglas de inserción en el controller.

- [ ] **Step 1: Escribir `web-wiring.md`** con:

  **`<Name>Request.cs`** (en `Web/Contracts/<Area>/`)
  ```csharp
  namespace <WebRoot>.Web.Contracts.<Area>;

  public sealed record <Name>Request(
      <Type1> <Field1>,
      <Type2> <Field2>);   // espejo de los campos del Command/Query
  ```

  **Endpoint** (método a insertar en `<Area>Controller : BaseApiController`)
  ```csharp
  [Authorize]                                  // o [AllowAnonymous]
  [RequiresPermission("<area>", "<Action>")]   // si el repo usa permisos y aplica
  [EnableRateLimiting("<policy>")]             // si aplica
  [HttpPost("<route>")]                        // GET para query; PUT/DELETE segun corresponda
  public async Task<IActionResult> <Name>([FromBody] <Name>Request request, CancellationToken ct)
  {
      var result = await Sender.Send(new <Name>Command(request.<Field1>, request.<Field2>), ct);
      return Success(result, "<mensaje en espanol>");   // o CreatedSuccess(...) en creaciones
  }
  ```

  **Reglas de inserción / creación del controller:**
  - Si `<Area>Controller.cs` existe: insertar el método **antes** de la llave de cierre de la clase; agregar los `using` del Command/Request si faltan; no tocar lo demás.
  - Si no existe: crearlo siguiendo el patrón (`[Route("api/<area>")] public sealed class <Area>Controller : BaseApiController`), con los `using` necesarios.
  - Para query/GET: params `[FromQuery]` en vez de `[FromBody]`; sin Request DTO si son pocos params primitivos.
  - El `Sender`, `Success`/`CreatedSuccess` vienen de `BaseApiController` (no redeclararlos).

- [ ] **Step 2: Verificar archivo no vacío**

Run: `test -s plugins/backend/skills/vertical-slice/reference/web-wiring.md && echo OK`
Expected: `OK`

- [ ] **Step 3: Commit**

```bash
git add plugins/backend/skills/vertical-slice/reference/web-wiring.md
git commit -m "backend/vertical-slice: web wiring (request DTO + endpoint)"
```

---

### Task 6: `SKILL.md` (orquestación)

**Files:**
- Create: `plugins/backend/skills/vertical-slice/SKILL.md`

**Interfaces:**
- Consumes: los 4 reference (tasks 2-5).
- Produces: la skill disparable.

- [ ] **Step 1: Escribir el frontmatter**

```markdown
---
name: vertical-slice
description: Use cuando el usuario quiere crear un endpoint, feature, command, query o "slice" en un backend .NET con Clean Architecture + CQRS (MediatR) + FluentValidation. Genera la vertical slice completa end-to-end (Command/Query + Handler + Validator + Response + Request DTO + endpoint en el controller) detectando y siguiendo las convenciones del repo. Para .NET/C# con MediatR; no para frontend ni otros stacks.
allowed-tools:
  - Bash(dotnet build*)
---
```

- [ ] **Step 2: Escribir el cuerpo del SKILL.md** con estas secciones (cada una con instrucciones accionables, linkeando a los reference):

  1. **Setup / detección** — seguir `reference/conventions.md` → Detección de proyectos + Aprender del vecino. No continuar sin haber identificado app layer, web y los markers/wrapper reales.
  2. **Interview corto** — inferir del prompt; preguntar SOLO lo faltante (área, operación command/query autodetectada por el verbo, campos+tipos, auth/permiso, rate limiting, forma del response). Máx 2-3 por ronda, usando la herramienta de preguntas estructurada cuando exista.
  3. **Generación** — elegir camino: command → `reference/command-slice.md`; query → `reference/query-slice.md`. Después el web wiring → `reference/web-wiring.md`. Escribir todos los archivos con los namespaces/usings reales detectados. Respetar carpeta `Commands/`|`Queries/`.
  4. **Guardrails** — nunca pisar archivos existentes (si existe, parar y preguntar); insertar el endpoint sin romper la clase; verificar que MediatR/FluentValidation auto-registran (leer la registración del repo) y solo registrar a mano si hace falta.
  5. **Verificación** — correr `dotnet build` de la solución; si falla, leer los errores y corregir; reportar el resultado. Listar al usuario los archivos creados/modificados.

- [ ] **Step 3: Validar frontmatter (name + description presentes)**

Run: `awk '/^---/{c++} c==1{print} c==2{exit}' plugins/backend/skills/vertical-slice/SKILL.md | grep -E '^(name|description):'`
Expected: aparecen las líneas `name:` y `description:`.

- [ ] **Step 4: Verificar que los links a reference existen**

Run: `for f in conventions command-slice query-slice web-wiring; do test -s "plugins/backend/skills/vertical-slice/reference/$f.md" && echo "$f OK"; done`
Expected: las 4 líneas `... OK`.

- [ ] **Step 5: Commit**

```bash
git add plugins/backend/skills/vertical-slice/SKILL.md
git commit -m "backend/vertical-slice: SKILL.md (orquestacion del flujo)"
```

---

### Task 7: Test de aceptación contra Company-API + README

**Files:**
- Modify: `README.md`
- (Sin cambios de skill; este task valida end-to-end y documenta.)

**Interfaces:**
- Consumes: la skill completa (tasks 1-6).

- [ ] **Step 1: Dry-run manual — command slice**

En el repo `Company-Repository` (working dir adicional), simular la skill: detectar proyectos (`Company.API.Application` tiene `Features/` + MediatR; `Company.API.Web` tiene `Controllers/`), leer el feature vecino `Auth/Commands/ChangePassword`, y generar una slice de prueba `Features/Diagnostics/Commands/PingWrite/` (Command vacío + Handler trivial que devuelve un Response + Validator + Response + Request + endpoint en un `DiagnosticsController` nuevo).

- [ ] **Step 2: Compilar**

Run: `dotnet build` en `Company-Repository/src/Company-API`
Expected: `Build succeeded` (0 errores).

- [ ] **Step 3: Comparar estilo**

Verificar que los archivos generados son indistinguibles del estilo de `ChangePassword` (primary ctor, record sellado, namespace=carpeta, marker, `Success(...)`).

- [ ] **Step 4: Descartar los archivos de prueba**

Borrar la carpeta `Features/Diagnostics/` y el `DiagnosticsController.cs` generados; confirmar que el árbol de Company quedó limpio (`git status` en ese repo sin cambios).

- [ ] **Step 5: Repetir para query slice** (generar `Features/Diagnostics/Queries/PingRead/` + endpoint GET, `dotnet build`, descartar).

- [ ] **Step 6: Actualizar `README.md`** — agregar el plugin `backend` debajo de `craft`:

```markdown
### backend
Scaffolding de backend .NET (Clean Architecture + CQRS). Genera vertical slices end-to-end con tus convenciones.

```
/plugin install backend@LorenzoGarese-marketplace
```

| Skill | Eje |
|---|---|
| vertical-slice | Genera una slice CQRS completa (Command/Query + Handler + Validator + Response + endpoint) |
```

- [ ] **Step 7: Commit + push**

```bash
git add README.md
git commit -m "backend: README + acceptance test verde contra Company-API"
git push origin main
```

---

## Self-Review

**Spec coverage:** todas las decisiones del spec tienen task — alcance end-to-end (T3-T5), command/query (T3/T4), interview (T6 §2), estilo moderno (T2/T3), genericidad/detección (T2, T6 §1), enfoque sin scripts (estructura), verificación dotnet build (T6 §5, T7), fuera de alcance respetado (no hay tasks de migrations/tests/openapi).

**Placeholder scan:** los `<Placeholders>` en los templates son intencionales (formas que la skill reemplaza en runtime), explicados en el encabezado de cada reference; no son TODOs del plan.

**Type consistency:** nombres consistentes — `<Name>Command`/`<Name>CommandHandler`/`<Name>CommandValidator`/`<Name>Response`/`<Name>Request`; marker `IWriteRequest`; helpers `Success`/`CreatedSuccess`/`Sender` de `BaseApiController`; excepciones `NotFoundException`/`BadRequestException`/`ConflictException` de `Application.Exceptions._4XX`.
