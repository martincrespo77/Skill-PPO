# Evidencia del repositorio analizado

Contexto de dónde salieron los patrones. No se carga por defecto: leer solo si hace falta
justificar una regla con un caso real.

**Repositorio:** visual novel *Living with Temptation: American Sunset* sobre Ren'Py 8.3.7
(`renpy/vc_version.py`). El código propio vive compilado en `game/archive.rpa` (RPA-3.0).
Se analizó en modo solo lectura: índice del RPA y `.rpyc` des-serializados en memoria.
Las líneas `game/*.rpy:N` son las originales conservadas en el AST compilado.

## Capa de aplicación (`archive.rpa`)

| Métrica | Valor |
|---|---|
| Scripts `.rpyc` propios (sin `tl/`) | 27 |
| `class` definidas | **0** |
| `def` definidas | 10 (utilitarias) |
| Bloques Python embebidos | 6.783 (9.184 líneas, casi todos asignaciones) |
| Nodos `If` / ramas de condición | 2.061 / 2.791 |
| `default` / `define` globales | 742 / 272 |
| Flags `quest_*` (enteros mágicos) | 116 |
| `label` / `jump` / `call` / `menu` | 2.907 / 2.881 / 1.467 / 556 |

Estilo: Transaction Script puro sobre estado global. Valor como **catálogo de contraejemplos**.

## Capa de motor (`renpy/`)

| Métrica | Valor |
|---|---|
| Archivos `.py` | 156 |
| Clases / métodos | 452 / 2.387 |
| `abc.ABC` / `@abstractmethod` / `Protocol` | 0 / 0 / 0 |
| `raise NotImplementedError` como contrato | 4 |
| `@staticmethod` / `isinstance` / `global` | 34 / 620 / 213 |

Estilo: POO real con polimorfismo por jerarquías (`Node` 46 subclases, `Displayable` 33,
`Object` 28), pero con contratos implícitos y estado global de módulo. Valor como
**referencia de formas de patrón**, a blindar.

## Ubicación de cada patrón

| Patrón | Dónde | Debilidad documentada |
|---|---|---|
| Command | `renpy/ui.py:40-64` (`Action`); `renpy/common/00achievement.rpy:306-320` (`Sync`); `renpy/display/behavior.py:478-494` (`is_sensitive`) | Contrato no abstracto; `__call__` no rechequea `get_sensitive`; `isinstance` en el despacho |
| Strategy + Composite | `renpy/common/00achievement.rpy:29-120,193-207,279-290`; `renpy/savelocation.py:461-520` (`MultiLocation`) | Base no-op; `backends` global; nombres `str`; `Exception` genérico |
| Specification | `renpy/common/00gallery.rpy:24-48,75-84` | `eval(self.condition)` en `:35`; sin base común; AND fijo |
| Memento | `renpy/revertable.py:77-93,168-190,227-260` | Log global; snapshot mutable; `isinstance` para formatos viejos |
| Interpreter | `renpy/ast.py:254,350-357,1814,1850-1858` | `execute` falla en runtime; condiciones `py_eval(str)`; siguiente paso por efecto global |
| Contraejemplo (Transaction Script) | `game/buy.rpy:9-18` (y `:23,:44,:55…`); `game/actions.rpy:28-44,84-90,839-842`; `game/map.rpy:10-22`; globales en `game/data.rpy:58-130` | Ver `antipatrones.md` |

## Snippets verificados

```python
# renpy/ui.py:40-64
class Action(renpy.object.Object):
    def get_sensitive(self):
        return True
    def get_selected(self):
        return False
    def __call__(self):
        raise NotImplementedError
```

```python
# renpy/common/00gallery.rpy:30-36
class __GalleryArbitraryCondition(object):
    def __init__(self, condition):
        self.condition = condition
    def check(self, all_prior):
        return eval(self.condition)
```

```python
# renpy/ast.py:350-357
def execute(self):
    raise Exception("Node subclass forgot to define execute.")
```

```renpy
# game/actions.rpy:84-90 (reconstruido del AST compilado)
$ stamina += 5
if stamina >= stamina_max:
    $ stamina = stamina_max
$ inspira += 5
if inspira >= inspira_max:
    $ inspira = inspira_max
# game/actions.rpy:842 el mismo clamp con otro operador:  if inspira > inspira_max:
```

```renpy
# game/buy.rpy:9-18 (reconstruido del AST compilado)
if cash < 200:
    ...
else:
    $ item_next = 1
    $ item_game_wes = 1
    $ cash -= 200
    $ dice_txt = _("NextGen console purchased")
    $ quest_games = 3
```

## Hallazgo clave

No hay un patrón **State** explícito en ninguna de las dos capas: el ciclo de vida de las
misiones (`quest_*`) es exactamente el caso que este skill resuelve con objetos State.
