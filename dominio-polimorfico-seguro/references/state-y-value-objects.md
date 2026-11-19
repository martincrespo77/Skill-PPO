# Value Objects y State: plantillas

Reemplazan el patrón "variables globales + flags enteros" (ver `evidencia-renpy.md`, ficha 6).

## Dinero y Billetera atómica

Antes: `if cash < 200: ... else: cash -= 200; item = 1` repetido, con el precio duplicado
en la guarda y en la resta, y nada que impida un saldo negativo.

```python
from __future__ import annotations
from dataclasses import dataclass
from decimal import Decimal

class ErrorDeDominio(Exception): ...
class OperacionNoPermitida(ErrorDeDominio): ...
class FondosInsuficientes(ErrorDeDominio): ...
class TransicionInvalida(ErrorDeDominio): ...

@dataclass(frozen=True, slots=True)
class Dinero:
    monto: Decimal
    def __post_init__(self) -> None:
        if self.monto < 0:
            raise ValueError("Dinero no puede ser negativo")
    def __le__(self, otro: Dinero) -> bool:
        return self.monto <= otro.monto
    def restar(self, otro: Dinero) -> Dinero:
        if not otro <= self:
            raise FondosInsuficientes(f"{self.monto} < {otro.monto}")
        return Dinero(self.monto - otro.monto)

class Billetera:
    def __init__(self, saldo: Dinero) -> None:
        self._saldo = saldo
    def pagar(self, precio: Dinero) -> None:        # validar y luego mutar
        self._saldo = self._saldo.restar(precio)    # lanza FondosInsuficientes
    def saldo(self) -> Dinero:
        return self._saldo
```

## Energia acotada

Antes: `stamina += 5; if stamina >= stamina_max: stamina = stamina_max` repetido en cada
llamador, con `>=` en unos y `>` en otros (inconsistente).

```python
from dataclasses import dataclass, replace

@dataclass(frozen=True, slots=True)
class Energia:
    actual: int
    maximo: int
    def __post_init__(self) -> None:
        if self.maximo <= 0 or not (0 <= self.actual <= self.maximo):
            raise ValueError("Energía fuera de rango")
    def recargar(self, n: int) -> Energia:          # el tope es interno
        if n < 0:
            raise ValueError("Recarga negativa")
        return replace(self, actual=min(self.maximo, self.actual + n))
    def gastar(self, n: int) -> Energia:
        if n > self.actual:
            raise OperacionNoPermitida("Energía insuficiente")
        return replace(self, actual=self.actual - n)
```

## State: ciclo de vida de una misión

Antes: `quest_lisa1stweek == 2 and day_count == 3` con 116 flags enteros; cualquier punto del
código podía asignar `quest_x = 7` y saltarse estados.

```python
from abc import ABC, abstractmethod
from dataclasses import dataclass

class EstadoMision(ABC):
    @abstractmethod
    def avanzar(self, dia: int) -> "EstadoMision": ...
    def esta_completa(self) -> bool:
        return False

class NoIniciada(EstadoMision):
    def avanzar(self, dia: int) -> EstadoMision:
        return EnCurso(dia_inicio=dia)

@dataclass(frozen=True)
class EnCurso(EstadoMision):
    dia_inicio: int
    def avanzar(self, dia: int) -> EstadoMision:
        if dia < self.dia_inicio + 2:
            raise TransicionInvalida("Aún no corresponde avanzar")
        return Completada()

class Completada(EstadoMision):
    def avanzar(self, dia: int) -> EstadoMision:
        raise TransicionInvalida("La misión ya está completada")
    def esta_completa(self) -> bool:
        return True

class Mision:
    def __init__(self) -> None:
        self._estado: EstadoMision = NoIniciada()
    def avanzar(self, dia: int) -> None:
        self._estado = self._estado.avanzar(dia)    # sin if sobre el estado
    def esta_completa(self) -> bool:
        return self._estado.esta_completa()
```

## Reglas prácticas

- Un estado nuevo = una clase nueva; el cliente no se modifica (Open/Closed).
- Los estados con datos propios (`EnCurso.dia_inicio`) son `frozen`.
- Las transiciones que no existen **lanzan**; nunca se ignoran en silencio.
- Los identificadores de ubicación u otros strings del dominio se modelan como VO validado
  o `Enum` cerrado de valores sin comportamiento. `" "` como "ninguno" se reemplaza por `Ubicacion | None`.
