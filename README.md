# Skill-PPO

Skill para Claude Code que guía el diseño de **lógica de dominio en Python 3.11+** con
**polimorfismo real** y **seguridad de diseño** (*design safety*): estados inválidos
irrepresentables, invariantes protegidas y reglas encapsuladas.

El contenido se destiló de una auditoría estática de un proyecto Ren'Py: su capa de aplicación
resultó un catálogo de antipatrones (estado global, flags enteros, invariantes que se corrigen
después de romperse) y su motor (`renpy/`) una fuente de formas de patrón (Command, Strategy,
Composite, Specification, Memento, Interpreter) a las que este skill les agrega contratos
explícitos y validación en construcción.

## Qué incluye

```
Skill-PPO/
├── README.md
└── dominio-polimorfico-seguro/
    ├── SKILL.md                      # propósito, 13 invariantes, flujo de 9 pasos, checklist
    └── references/
        ├── patrones.md               # Command, Strategy+Composite, Specification, Memento, Interpreter
        ├── state-y-value-objects.md  # Dinero, Billetera, Energia, State de ciclo de vida
        ├── antipatrones.md           # síntoma → riesgo → corrección, y greps de auditoría
        └── evidencia-renpy.md        # métricas y rutas/líneas reales de las que salió cada patrón
```

## Qué hace el skill

- Impone **13 invariantes** verificables (contratos `ABC`/`Protocol`, State en vez de flags,
  validar antes de mutar, Value Objects `frozen`, comandos que se autoprotegen, dominio sin
  infraestructura, sin `eval`, excepciones semánticas, etc.).
- Propone un **flujo de trabajo** ordenado: Value Objects → contratos polimórficos →
  State/Command/Strategy → aislamiento de infraestructura → auditoría.
- Entrega **plantillas en Python puro** (sin dependencias de infraestructura) para cada patrón.
- Incluye una **matriz de antipatrones** y un **checklist** Sí/No para aprobar un módulo.

## Instalación

Copiá la carpeta del skill a tu directorio de skills de Claude Code.

Instalación global (Windows):

```powershell
Copy-Item -Recurse dominio-polimorfico-seguro "$env:USERPROFILE\.claude\skills\"
```

Instalación global (Linux/macOS):

```bash
cp -r dominio-polimorfico-seguro ~/.claude/skills/
```

Instalación solo para un proyecto:

```bash
cp -r dominio-polimorfico-seguro <tu-proyecto>/.claude/skills/
```

El skill se activa solo cuando la tarea coincide con su `description` (diseño o revisión de
entidades, ciclos de vida, permisos, reglas de elegibilidad o precios, o cuando aparecen
`if/elif` por tipo/estado, flags enteros, estado global, `eval` de condiciones o *primitive obsession*).

## Relación con `python-poo-dominio-rico`

Complementa a un skill de Dominio Rico existente: aporta la parte de *design safety* (validar
antes de mutar, Value Objects acotados, Specification sin `eval`, Memento opaco, comandos que se
autoprotegen). Si ya usás ese skill, este puede sumarse como material de referencia sin duplicar
Factory Method ni Double Dispatch.

## Alcance

Cubre diseño y auditoría de dominio. No cubre UI, persistencia concreta ni frameworks: esos
entran solo como adaptadores detrás de puertos `Protocol`.

## Origen del análisis

La auditoría fue de solo lectura: el código propio se leyó del índice de `game/archive.rpa` y se
des-serializó en memoria, sin escribir ni ejecutar nada en el repositorio analizado. Detalle y
métricas en [`dominio-polimorfico-seguro/references/evidencia-renpy.md`](dominio-polimorfico-seguro/references/evidencia-renpy.md).
