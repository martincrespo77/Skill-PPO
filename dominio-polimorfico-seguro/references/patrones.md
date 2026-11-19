# Catálogo de patrones en dominio puro (Python 3.11+)

Base común de todas las plantillas:

```python
class ErrorDeDominio(Exception): ...
class OperacionNoPermitida(ErrorDeDominio): ...
class TransicionInvalida(ErrorDeDominio): ...
class FondosInsuficientes(ErrorDeDominio): ...
```

Las plantillas reutilizan `Dinero` de `state-y-value-objects.md`.

---

## 1. Command con precondición de habilitación (+ Double Dispatch)

**Garantía:** la operación conoce sus precondiciones y se puede consultar antes de invocarla;
`ejecutar()` las vuelve a verificar, así que saltearse la UI no salta la regla.

**Debilidad que blinda (fuente):** `ui.Action.__call__` no rechequea `get_sensitive()` y el
contrato no es abstracto.

```python
from __future__ import annotations
from abc import ABC, abstractmethod
from dataclasses import dataclass

class Usuario(ABC):
    @abstractmethod
    def motivo_para_no_reembolsar(self, monto: Dinero) -> str | None: ...

class Cliente(Usuario):
    def motivo_para_no_reembolsar(self, monto: Dinero) -> str | None:
        return "Un cliente no aprueba reembolsos"

@dataclass(frozen=True)
class Supervisor(Usuario):
    tope: Dinero
    def motivo_para_no_reembolsar(self, monto: Dinero) -> str | None:
        return None if monto <= self.tope else "Supera el tope del supervisor"

class Comando(ABC):
    @abstractmethod
    def motivo_de_bloqueo(self) -> str | None: ...
    @abstractmethod
    def _hacer(self) -> None: ...

    def esta_habilitado(self) -> bool:          # para la UI
        return self.motivo_de_bloqueo() is None

    def ejecutar(self) -> None:                  # barrera real (Template Method)
        if (motivo := self.motivo_de_bloqueo()) is not None:
            raise OperacionNoPermitida(motivo)
        self._hacer()

class Pedido:
    def __init__(self, total: Dinero) -> None:
        self._total, self._reembolsado = total, False
    def motivo_para_no_reembolsar(self, actor: Usuario) -> str | None:
        if self._reembolsado:
            return "Ya reembolsado"
        return actor.motivo_para_no_reembolsar(self._total)   # doble despacho
    def reembolsar(self, por: Usuario) -> None:
        if (m := self.motivo_para_no_reembolsar(por)) is not None:
            raise OperacionNoPermitida(m)
        self._reembolsado = True

@dataclass(frozen=True)
class ReembolsarPedido(Comando):
    pedido: Pedido
    actor: Usuario
    def motivo_de_bloqueo(self) -> str | None:
        return self.pedido.motivo_para_no_reembolsar(self.actor)
    def _hacer(self) -> None:
        self.pedido.reembolsar(por=self.actor)
```

---

## 2. Strategy / Puerto + Composite

**Garantía:** el dominio no conoce el almacenamiento concreto; uno o muchos destinos se tratan igual.

**Debilidades que blinda (fuente):** `achievement.Backend` hace no-op silencioso, los nombres son
`str` crudos, la lista `backends` es global mutable y el error es un `Exception` genérico.

```python
import re
from dataclasses import dataclass
from typing import Protocol

@dataclass(frozen=True, slots=True)
class IdLogro:
    valor: str
    def __post_init__(self) -> None:
        if not re.fullmatch(r"[a-z][a-z0-9_]{1,40}", self.valor):
            raise ValueError(f"IdLogro inválido: {self.valor!r}")

class RegistroDeLogros(Protocol):
    def otorgar(self, logro: IdLogro) -> None: ...
    def tiene(self, logro: IdLogro) -> bool: ...

class RegistroCompuesto:                      # Composite: mismo Protocol
    def __init__(self, *registros: RegistroDeLogros) -> None:
        if not registros:
            raise ValueError("Se requiere al menos un registro")
        self._registros: tuple[RegistroDeLogros, ...] = registros
    def otorgar(self, logro: IdLogro) -> None:
        for r in self._registros:
            if not r.tiene(logro):
                r.otorgar(logro)
    def tiene(self, logro: IdLogro) -> bool:
        return any(r.tiene(logro) for r in self._registros)

class RegistroEnMemoria:                      # adaptador (fuera de dominio/ en prod)
    def __init__(self) -> None:
        self._otorgados: set[IdLogro] = set()
    def otorgar(self, logro: IdLogro) -> None:
        self._otorgados.add(logro)
    def tiene(self, logro: IdLogro) -> bool:
        return logro in self._otorgados
```

---

## 3. Specification

**Garantía:** cada criterio es un objeto puro evaluable; la entidad no conoce las reglas concretas.

**Debilidades que blinda (fuente):** `__GalleryArbitraryCondition` usa `eval(str)`, no hay base
común y solo se compone con AND fijo.

```python
from abc import ABC, abstractmethod
from dataclasses import dataclass
from decimal import Decimal
from typing import Generic, TypeVar

T = TypeVar("T")

class Especificacion(ABC, Generic[T]):
    @abstractmethod
    def es_satisfecha_por(self, candidato: T) -> bool: ...
    def __and__(self, o: "Especificacion[T]") -> "Especificacion[T]": return _Y(self, o)
    def __or__(self, o: "Especificacion[T]") -> "Especificacion[T]": return _O(self, o)
    def __invert__(self) -> "Especificacion[T]": return _No(self)

@dataclass(frozen=True)
class _Y(Especificacion[T]):
    a: Especificacion[T]; b: Especificacion[T]
    def es_satisfecha_por(self, c: T) -> bool:
        return self.a.es_satisfecha_por(c) and self.b.es_satisfecha_por(c)

@dataclass(frozen=True)
class _O(Especificacion[T]):
    a: Especificacion[T]; b: Especificacion[T]
    def es_satisfecha_por(self, c: T) -> bool:
        return self.a.es_satisfecha_por(c) or self.b.es_satisfecha_por(c)

@dataclass(frozen=True)
class _No(Especificacion[T]):
    a: Especificacion[T]
    def es_satisfecha_por(self, c: T) -> bool:
        return not self.a.es_satisfecha_por(c)

@dataclass(frozen=True)
class Compra:
    total: Dinero
    es_primera_compra: bool

@dataclass(frozen=True)
class TotalMinimo(Especificacion[Compra]):
    minimo: Dinero
    def es_satisfecha_por(self, c: Compra) -> bool:
        return self.minimo <= c.total

class PrimeraCompra(Especificacion[Compra]):
    def es_satisfecha_por(self, c: Compra) -> bool:
        return c.es_primera_compra

cupon_bienvenida = PrimeraCompra() & TotalMinimo(Dinero(Decimal("20")))
```

---

## 4. Memento (snapshot opaco)

**Garantía:** la reversión es exacta y el snapshot no se puede manipular desde afuera.

**Debilidades que blinda (fuente):** `revertable.mutator` depende de un log global
(`renpy.game.log`, `global mutate_flag`) y el snapshot es una lista mutable común.

```python
from dataclasses import dataclass

@dataclass(frozen=True, slots=True)
class _InstantaneaCarrito:            # opaca: el prefijo _ marca uso interno
    dueno_id: int
    items: tuple[tuple[str, int], ...]

class Carrito:
    def __init__(self) -> None:
        self._items: dict[str, int] = {}
    def agregar(self, sku: str, cantidad: int) -> None:
        if cantidad <= 0:
            raise ErrorDeDominio("Cantidad debe ser positiva")
        self._items[sku] = self._items.get(sku, 0) + cantidad
    def items(self) -> tuple[tuple[str, int], ...]:
        return tuple(self._items.items())           # nunca el dict interno
    def instantanea(self) -> _InstantaneaCarrito:
        return _InstantaneaCarrito(id(self), self.items())
    def restaurar(self, inst: _InstantaneaCarrito) -> None:
        if inst.dueno_id != id(self):
            raise OperacionNoPermitida("Instantánea de otro carrito")
        self._items = dict(inst.items)
```

En producción `sku` también va como VO `Sku`.

---

## 5. Interpreter / árbol de reglas

**Garantía:** cada nodo sabe evaluarse a sí mismo, sin un `switch` central; la omisión de
`evaluar` falla al instanciar, no al ejecutar.

**Debilidades que blinda (fuente):** `ast.Node.execute` lanza `Exception` en runtime,
`If.execute` evalúa strings con `py_eval` y el siguiente paso se comunica por efecto global.

```python
from abc import ABC, abstractmethod
from dataclasses import dataclass
from decimal import Decimal

@dataclass(frozen=True)
class ContextoPrecio:
    base: Dinero
    compra: Compra

class ReglaPrecio(ABC):
    @abstractmethod
    def evaluar(self, ctx: ContextoPrecio) -> Dinero: ...

class PrecioBase(ReglaPrecio):
    def evaluar(self, ctx: ContextoPrecio) -> Dinero:
        return ctx.base

@dataclass(frozen=True)
class DescuentoPorcentual(ReglaPrecio):
    interna: ReglaPrecio
    porcentaje: Decimal
    def __post_init__(self) -> None:
        if not (Decimal(0) < self.porcentaje < Decimal(100)):
            raise ValueError("Porcentaje fuera de rango")
    def evaluar(self, ctx: ContextoPrecio) -> Dinero:
        p = self.interna.evaluar(ctx)
        return Dinero(p.monto * (100 - self.porcentaje) / 100)

@dataclass(frozen=True)
class Condicional(ReglaPrecio):       # reemplaza If + py_eval(str)
    cuando: Especificacion[Compra]
    entonces: ReglaPrecio
    si_no: ReglaPrecio
    def evaluar(self, ctx: ContextoPrecio) -> Dinero:
        rama = self.entonces if self.cuando.es_satisfecha_por(ctx.compra) else self.si_no
        return rama.evaluar(ctx)
```
