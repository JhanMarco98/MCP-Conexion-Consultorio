# MCP Clínica Foianini - Skills

Configuración de MCP Oracle y Skills para consultas de Clínica Foianini.

## Estructura

```
.claude/commands/
├── consultorio/
│   └── sk-consultorio.md    ← Área Consultorios
├── general/
│   └── (pendiente)          ← Consultas generales
└── (futuras áreas...)
```

## Skills Disponibles

| Skill | Comando | Descripción |
|-------|---------|-------------|
| Consultorio | `/sk-consultorio` | Turnos, producción, NPS del área consultorios |
| General | `/sk-general` | *(Pendiente)* |

## Instalación

1. Instalar el MCP Oracle: `docs/MCP_Oracle_Instalacion.md`
2. Clonar este repositorio
3. Usar los comandos `/sk-*` en Claude Code

## Ejemplos

```
/sk-consultorio
Dame los turnos presentes de enero a junio 2025
```
