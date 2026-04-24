# MCP Conexión Consultorio

Configuración de MCP Oracle y Skill para consultas del área de Consultorios de Clínica Foianini.

## Contenido

| Archivo | Descripción |
|---------|-------------|
| `.claude/commands/sk-consultorio.md` | Skill `/sk-consultorio` para Claude Code |
| `docs/MCP_Oracle_Instalacion.md` | Guía de instalación del MCP Oracle |

## Uso

1. Instalar el MCP Oracle siguiendo `docs/MCP_Oracle_Instalacion.md`
2. Clonar este repositorio en tu proyecto
3. Usar el comando `/sk-consultorio` en Claude Code

## Skill Consultorio

El skill permite consultar:
- **Turnos:** presentes, ausentes, ocupación, demanda, NPS
- **Producción:** fee, ingresos, servicios por médico

### Ejemplo
```
/sk-consultorio
Dame los turnos presentes de enero a junio 2025
```
