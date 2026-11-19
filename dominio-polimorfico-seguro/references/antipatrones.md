# Matriz de antipatrones vs. refactorización segura

| Síntoma observado | Riesgo de diseño | Corrección polimórfica y segura |
|---|---|---|
| `if saldo < N: … else: saldo -= N` repetido muchas veces | Precio duplicado, saldo negativo posible, compra no atómica | `Billetera.pagar(Dinero)` lanza `FondosInsuficientes`; comando `Comprar(producto)` con `motivo_de_bloqueo()` |
| `x += 5; if x >= max: x = max` (con `>=` y `>` mezclados) | La invariante depende de cada llamador; ya hay inconsistencia | VO `Energia` acotada: `recargar()`/`gastar()` mantienen el rango por construcción |
| Decenas de flags `quest_*` comparados con enteros | Transiciones ilegales silenciosas, saltos de estado | Patrón State: `avanzar()` devuelve el siguiente estado o lanza `TransicionInvalida` |
| Cientos de `default`/globales mutables | Acoplamiento oculto, imposible testear de forma aislada | Agregados con estado privado; servicios que cargan y persisten vía repositorio |
| Ubicaciones/IDs como strings (`"loca_bathroom"`, `" "` = nada) | Erratas no detectadas, centinelas ambiguos | VO validado o `Enum` cerrado; `X | None` explícito |
| Agenda por `dia == N and hora == H` | Explosión combinatoria de `if` | `Agenda` como Strategy o tabla tipada `Franja → Ubicacion` |
| Base con métodos no-op | Implementación incompleta que "funciona" en silencio | `Protocol` o `ABC` + `@abstractmethod` |
| `raise Exception("forgot to define X")` en la base | Error recién en runtime | `@abstractmethod`: falla al instanciar |
| `eval(cond_str)` / `py_eval(cond)` | Inyección, sin tipos, no testeable | `Especificacion[T]` componible con `& \| ~` |
| Validación solo `if config.developer` | Datos inválidos en producción | VO que valida siempre en `__post_init__` |
| Lista global de backends/handlers | Registro mutable desde cualquier lado | Composite inmutable inyectado por constructor |
| `isinstance` para despachar comportamiento | Hay que tocar el despachador con cada tipo nuevo | Composite o método polimórfico |
| `__call__` sin rechequear la precondición | La regla se puede saltear desde otra vía | Template Method: `ejecutar()` verifica `motivo_de_bloqueo()` |
| `raise Exception(...)` genérico | El cliente no puede reaccionar con semántica | Jerarquía `ErrorDeDominio` |
| Snapshot como lista mutable + log global | Snapshots manipulables, estado compartido | Memento `frozen` y opaco; `restaurar()` verifica el dueño |
| Subclases que solo agregan campos | Herencia estructural sin polimorfismo | Composición o un VO; heredar solo si hay comportamiento que especializar |
| Servicio con `if usuario.rol == "admin"` | Lógica de negocio fuera del dominio | `actor.puede_hacer_x(recurso)` (doble despacho) |
| `@staticmethod _validar_*` en entidades | Validación dispersa, no reusable | VO con validación por construcción o módulo `validaciones` |

## Greps de auditoría (Python)

Ejecutar sobre el paquete de dominio; todo hallazgo debe justificarse o corregirse.

```text
isinstance\(                 # ramas por tipo
\belif\b.*==                 # ramas por estado/rol
^\s*global\b                 # estado global
\beval\(|\bexec\(            # reglas como strings
dict\[str,\s*Any\]|\*\*kwargs # contratos opacos
@staticmethod                # validación en entidades
raise Exception\(            # error genérico
-=|\+=                       # mutación de primitivos del dominio (revisar caso a caso)
NotImplementedError          # contrato débil: migrar a @abstractmethod
```

Ejemplo:

```bash
grep -rnE "isinstance\(|^\s*global\b|\beval\(|@staticmethod|raise Exception\(" dominio/
```
