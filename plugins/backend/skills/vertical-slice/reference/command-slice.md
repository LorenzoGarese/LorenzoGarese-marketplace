# Command slice (write)

Los `<Placeholders>` son **formas, no literales**. Reemplazá `<AppRoot>` y los `using` por los reales detectados en el repo destino (ver [conventions.md](conventions.md) → "Aprender del vecino"). Carpeta destino: `Features/<Area>/Commands/<Name>/`, con `namespace` = la ruta de esa carpeta.

## `<Name>Command.cs`

```csharp
using <AppRoot>.Application.Abstractions.Mediator; // namespace real del marker IWriteRequest (detectar)
using MediatR;

namespace <AppRoot>.Application.Features.<Area>.Commands.<Name>;

public sealed record <Name>Command(
    <Type1> <Field1>,
    <Type2> <Field2>)
    : IRequest<<Name>Response>, IWriteRequest;
```

## `<Name>CommandHandler.cs`

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

## `<Name>CommandValidator.cs`

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

## `<Name>Response.cs`

```csharp
namespace <AppRoot>.Application.Features.<Area>.Commands.<Name>;

public sealed record <Name>Response(
    <Type> <Field>);
```

> Si el command no devuelve datos, el Response puede ser un record vacío o seguir el patrón del repo (algunos endpoints devuelven solo mensaje vía `Success("...")` en el controller).
