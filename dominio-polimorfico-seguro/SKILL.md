---
name: dominio-polimorfico-seguro
description: >
  Diseña, revisa y refactoriza lógica de dominio en Python 3.11+ con polimorfismo
  real (Command, State, Strategy, Specification, Composite, Memento, Interpreter,
  doble despacho) y seguridad de diseño (estados inválidos irrepresentables,
  Value Objects inmutables, validar-antes-de-mutar, dominio sin infraestructura).
  Activar al modelar entidades, ciclos de vida, permisos, reglas de elegibilidad
  o precios, o al detectar if/elif por tipo/estado, flags enteros de progreso,
  estado global mutable, clamps dispersos, eval de condiciones o primitive obsession.
---

# Dominio polimórfico y seguro en Python

## Propósito

Diseñar lógica de dominio donde el comportamiento variable se resuelve con **polimorfismo real**
y donde los **estados inválidos no se pueden representar** (*design safety*).

## Cuándo activarlo

- Se crean o modifican entidades, Value Objects, agregados, máquinas de estado, permisos,
  reglas de elegibilidad, precios o ciclos de vida (pedidos, misiones, tickets, suscripciones).
- Aparece alguno de estos síntomas:
  - cadenas `if/elif`, `match` o `isinstance` sobre tipo, rol o estado;
  - flags enteros de progreso (`quest_x == 3`);
  - variables globales mutables;
  - `saldo -= n` sin chequear antes, o clamps repetidos (`if x > max: x = max`);
  - strings como identificadores de dominio; `dict`/`**kwargs` opacos; `eval` de condiciones.
- Se audita un módulo de dominio antes de aprobarlo (ver el checklist al final).

**Fuera de alcance:** UI, render, persistencia concreta y frameworks. Entran solo como
**adaptadores** detrás de puertos (`Protocol`).

## Invariantes no negociables

1. **INV-01 Contratos explícitos.** Toda variación se declara con `abc.ABC` + `@abstractmethod`
   (herencia) o `typing.Protocol` (puertos). Prohibidas las bases con métodos no-op silenciosos
   y el `raise Exception("forgot to define")` en tiempo de ejecución.
2. **INV-02 Cero ramas por tipo/rol/estado fuera de fábricas.** `if/elif`, `match` o `isinstance`
   solo en una fábrica o adaptador de borde. En el dominio se envía un mensaje polimórfico.
3. **INV-03 Ciclo de vida = objeto State.** Prohibido `estado == 3` o strings mágicos. Cada estado
   es una clase; la transición devuelve el siguiente estado o lanza `TransicionInvalida`.
4. **INV-04 Validar antes de mutar.** Primero todas las precondiciones, después la mutación.
   Prohibido "mutar y corregir". Los límites viven dentro del Value Object.
5. **INV-05 Value Objects inmutables.** Dinero, cantidades, IDs, rangos, horas y ubicaciones son
   `@dataclass(frozen=True, slots=True)` con validación en `__post_init__`. Prohibido `int`/`str`/`dict` crudos.
6. **INV-06 Comandos que se autoprotegen.** Todo comando expone `motivo_de_bloqueo()` y su
   `ejecutar()` vuelve a verificarlo (Template Method). La UI nunca es la única barrera.
7. **INV-07 Tell, Don't Ask / Doble Despacho.** Las operaciones reciben el objeto completo
   (`actor.puede_aprobar(pedido)`). Ninguna entidad inspecciona atributos internos de otra.
8. **INV-08 Dominio sin infraestructura.** Nada de UI, BD, red, reloj ni azar global. Se inyectan
   como `Protocol`. Prohibido el estado global mutable de módulo.
9. **INV-09 Reglas como datos tipados, nunca strings evaluables.** Prohibido `eval`/`exec`.
   Usar `Specification` componible (`&`, `|`, `~`) o árboles Interpreter.
10. **INV-10 Colecciones encapsuladas.** Nunca devolver la `list`/`dict`/`set` interna mutable;
    exponer `tuple`, `frozenset` o métodos de consulta.
11. **INV-11 Excepciones de dominio semánticas.** Jerarquía `ErrorDeDominio`
    (`FondosInsuficientes`, `TransicionInvalida`, `OperacionNoPermitida`). Nunca `Exception`
    genérico ni `None` silencioso.
12. **INV-12 Snapshots opacos.** El Memento es inmutable y solo lo restaura quien lo creó.
13. **INV-13 Sin `@staticmethod` de validación en entidades.** Las reglas de formato viven en
    Value Objects o en un módulo de funciones puras.

## Flujo de trabajo

1. **Lenguaje ubicuo e invariantes.** Listar conceptos, reglas y transiciones; cada regla será un test.
2. **Value Objects primero.** Datos atómicos con reglas, inmutables, operaciones cerradas
   que devuelven un VO nuevo. Plantillas en `references/state-y-value-objects.md`.
3. **Contratos polimórficos.** Por cada `if` anticipado sobre tipo/rol/estado, un ABC o `Protocol`.
   El único `match` permitido va en una `Fabrica`.
4. **Ciclos de vida con State.** La entidad delega en `self._estado`.
5. **Comandos, Specifications y Strategies.** Intenciones = Command; elegibilidad = Specification;
   políticas variables (precio, recompensa) = Strategy. Ver `references/patrones.md`.
6. **Composición y extensión.** Composite para uno-o-muchos, Decorator para extender sin modificar.
7. **Eventos y snapshots.** Eventos de dominio (Observer desacoplado); Memento si hay que deshacer.
8. **Aislar la infraestructura.** Puertos `Protocol` y adaptadores fuera de `dominio/`. Los servicios
   de aplicación solo orquestan: cargar, invocar dominio, persistir, publicar eventos. Cero `if` de negocio.
9. **Auditar.** Pasar el checklist y correr los greps de `references/antipatrones.md`.

## Dónde leer más

| Necesidad | Archivo |
|---|---|
| Plantillas de Command, Strategy+Composite, Specification, Memento, Interpreter | `references/patrones.md` |
| Value Objects (`Dinero`, `Energia`), State, Billetera atómica | `references/state-y-value-objects.md` |
| Tabla síntoma → riesgo → corrección y regex de auditoría | `references/antipatrones.md` |
| Evidencia real (rutas y líneas) de la que se extrajeron los patrones | `references/evidencia-renpy.md` |

## Checklist de auditoría (Sí/No; un "No" bloquea la aprobación)

**Contratos y polimorfismo**
- [ ] ¿Cada variación tiene contrato `ABC` + `@abstractmethod` o `Protocol`?
- [ ] ¿Ninguna base tiene métodos vacíos/no-op que oculten una implementación faltante?
- [ ] ¿No hay `if/elif`, `match` ni `isinstance` sobre tipo/rol/estado fuera de fábricas y adaptadores?
- [ ] ¿Toda subclase especializa comportamiento (no solo agrega campos)?

**Estado y ciclo de vida**
- [ ] ¿Los estados son clases State (no enteros, strings ni booleanos combinados)?
- [ ] ¿Toda transición ilegal lanza `TransicionInvalida`?
- [ ] ¿Las operaciones validan antes de mutar?

**Value Objects y tipos**
- [ ] ¿Dinero, cantidades, IDs, rangos y ubicaciones son VOs `frozen` validados en `__post_init__`?
- [ ] ¿Es imposible construir un VO o entidad inválidos?
- [ ] ¿No hay `dict[str, Any]`, `**kwargs` opacos ni `Any` en firmas del dominio?
- [ ] ¿No hay `@staticmethod _validar_*` en entidades?

**Encapsulamiento y colaboración**
- [ ] ¿Ninguna entidad expone colecciones internas mutables?
- [ ] ¿Las decisiones se delegan a quien tiene la información (Tell, Don't Ask / doble despacho)?
- [ ] ¿Los comandos verifican su precondición dentro de `ejecutar()`?
- [ ] ¿Las reglas de elegibilidad son Specifications tipadas, sin `eval`/`exec`?

**Aislamiento**
- [ ] ¿`dominio/` no importa UI, persistencia, red, reloj ni azar?
- [ ] ¿No hay globales mutables ni `global` en el dominio?
- [ ] ¿Reloj, azar y almacenamiento se inyectan como `Protocol`?
- [ ] ¿Los servicios de aplicación no tienen `if` sobre roles o estados?

**Errores y reversión**
- [ ] ¿Todas las fallas usan la jerarquía `ErrorDeDominio`?
- [ ] ¿Los snapshots son inmutables y los restaura solo su creador?
- [ ] ¿Cada invariante tiene un test que intenta romperla?
