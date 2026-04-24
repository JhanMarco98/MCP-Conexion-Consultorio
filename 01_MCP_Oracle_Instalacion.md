# Guía de Instalación: MCP Server Oracle DWH

Esta guía explica cómo configurar el servidor MCP (Model Context Protocol) para conectar Claude Code con el DWH Oracle del Grupo Foianini.

## Requisitos Previos

1. **Python 3.10+** instalado
2. **Oracle Instant Client** (versión 19c o superior)
3. **Claude Code** instalado (`npm install -g @anthropic-ai/claude-code`)

---

## Paso 1: Instalar Oracle Instant Client

### Windows

1. Descargar Oracle Instant Client Basic de: https://www.oracle.com/database/technologies/instant-client/winx64-64-downloads.html

2. Extraer en una carpeta (ej: `C:\oracle\instantclient_19_23`)

3. Agregar al PATH del sistema:
   - Abrir "Variables de entorno del sistema"
   - En "Variables del sistema", editar `Path`
   - Agregar la ruta del Instant Client

4. Verificar instalación:
   ```cmd
   sqlplus -V
   ```

---

## Paso 2: Crear el Servidor MCP

### 2.1 Crear estructura de carpetas

```cmd
mkdir C:\Users\%USERNAME%\.claude\mcp-servers\oracle-dwh
```

### 2.2 Crear el archivo `server.py`

Crear el archivo `C:\Users\%USERNAME%\.claude\mcp-servers\oracle-dwh\server.py` con el siguiente contenido:

```python
#!/usr/bin/env python3
"""MCP Server for Oracle DWH queries."""

import json
import sys
import oracledb

# Configuración de conexión
ORACLE_CONFIG = {
    "host": "192.168.30.21",
    "port": 1521,
    "sid": "plan2",
    "user": "POWER_STR",
    "password": "TU_PASSWORD_AQUI"  # Reemplazar con password real
}

def get_connection():
    """Crear conexión a Oracle."""
    dsn = oracledb.makedsn(
        ORACLE_CONFIG["host"],
        ORACLE_CONFIG["port"],
        sid=ORACLE_CONFIG["sid"]
    )
    return oracledb.connect(
        user=ORACLE_CONFIG["user"],
        password=ORACLE_CONFIG["password"],
        dsn=dsn
    )

def handle_oracle_query(sql: str, max_rows: int = 1000) -> dict:
    """Ejecutar SELECT y retornar resultados."""
    try:
        conn = get_connection()
        cursor = conn.cursor()
        cursor.execute(sql)

        columns = [col[0] for col in cursor.description]
        rows = cursor.fetchmany(max_rows)

        result = {
            "columns": columns,
            "rows": [list(row) for row in rows],
            "row_count": len(rows),
            "truncated": len(rows) == max_rows
        }

        cursor.close()
        conn.close()
        return {"success": True, "data": result}
    except Exception as e:
        return {"success": False, "error": str(e)}

def handle_oracle_execute(sql: str) -> dict:
    """Ejecutar DDL/DML (CREATE, INSERT, UPDATE, DELETE)."""
    try:
        conn = get_connection()
        cursor = conn.cursor()
        cursor.execute(sql)
        conn.commit()

        result = {"rows_affected": cursor.rowcount}

        cursor.close()
        conn.close()
        return {"success": True, "data": result}
    except Exception as e:
        return {"success": False, "error": str(e)}

def handle_oracle_list_tables(schema: str = None) -> dict:
    """Listar tablas disponibles."""
    try:
        conn = get_connection()
        cursor = conn.cursor()

        if schema:
            sql = f"SELECT table_name FROM all_tables WHERE owner = '{schema.upper()}' ORDER BY table_name"
        else:
            sql = "SELECT table_name FROM user_tables ORDER BY table_name"

        cursor.execute(sql)
        tables = [row[0] for row in cursor.fetchall()]

        cursor.close()
        conn.close()
        return {"success": True, "data": {"tables": tables}}
    except Exception as e:
        return {"success": False, "error": str(e)}

def handle_oracle_describe(table_name: str, schema: str = None) -> dict:
    """Describir estructura de una tabla."""
    try:
        conn = get_connection()
        cursor = conn.cursor()

        full_table = f"{schema}.{table_name}" if schema else table_name

        sql = f"""
        SELECT column_name, data_type, data_length, nullable
        FROM all_tab_columns
        WHERE table_name = '{table_name.upper()}'
        {f"AND owner = '{schema.upper()}'" if schema else ""}
        ORDER BY column_id
        """

        cursor.execute(sql)
        columns = []
        for row in cursor.fetchall():
            columns.append({
                "name": row[0],
                "type": row[1],
                "length": row[2],
                "nullable": row[3] == 'Y'
            })

        cursor.close()
        conn.close()
        return {"success": True, "data": {"table": table_name, "columns": columns}}
    except Exception as e:
        return {"success": False, "error": str(e)}

def handle_oracle_test_connection() -> dict:
    """Probar conexión al DWH."""
    try:
        conn = get_connection()
        cursor = conn.cursor()
        cursor.execute("SELECT 1 FROM DUAL")
        cursor.close()
        conn.close()
        return {"success": True, "message": "Conexión exitosa al DWH Oracle"}
    except Exception as e:
        return {"success": False, "error": str(e)}

# MCP Protocol Implementation
def read_message():
    """Leer mensaje JSON-RPC del stdin."""
    line = sys.stdin.readline()
    if not line:
        return None
    return json.loads(line)

def write_message(msg):
    """Escribir mensaje JSON-RPC al stdout."""
    sys.stdout.write(json.dumps(msg) + "\n")
    sys.stdout.flush()

def main():
    """Loop principal del servidor MCP."""
    # Inicialización
    tools = [
        {
            "name": "oracle_query",
            "description": "Ejecuta una query SELECT en el DWH Oracle y retorna resultados.",
            "inputSchema": {
                "type": "object",
                "properties": {
                    "sql": {"type": "string", "description": "Query SQL SELECT a ejecutar"},
                    "max_rows": {"type": "integer", "default": 1000, "description": "Máximo de filas a retornar"}
                },
                "required": ["sql"]
            }
        },
        {
            "name": "oracle_execute",
            "description": "Ejecuta DDL/DML (CREATE, INSERT, UPDATE, DELETE) en el DWH Oracle.",
            "inputSchema": {
                "type": "object",
                "properties": {
                    "sql": {"type": "string", "description": "Statement SQL a ejecutar"}
                },
                "required": ["sql"]
            }
        },
        {
            "name": "oracle_list_tables",
            "description": "Lista todas las tablas disponibles en el DWH Oracle.",
            "inputSchema": {
                "type": "object",
                "properties": {
                    "schema": {"type": "string", "description": "Filtrar por schema/owner específico (opcional)"}
                }
            }
        },
        {
            "name": "oracle_describe",
            "description": "Describe la estructura de una tabla (columnas, tipos, etc.).",
            "inputSchema": {
                "type": "object",
                "properties": {
                    "table_name": {"type": "string", "description": "Nombre de la tabla"},
                    "schema": {"type": "string", "description": "Schema/owner de la tabla (opcional)"}
                },
                "required": ["table_name"]
            }
        },
        {
            "name": "oracle_test_connection",
            "description": "Prueba la conexión al DWH Oracle.",
            "inputSchema": {
                "type": "object",
                "properties": {}
            }
        }
    ]

    while True:
        msg = read_message()
        if msg is None:
            break

        method = msg.get("method")
        msg_id = msg.get("id")
        params = msg.get("params", {})

        if method == "initialize":
            write_message({
                "jsonrpc": "2.0",
                "id": msg_id,
                "result": {
                    "protocolVersion": "2024-11-05",
                    "capabilities": {"tools": {}},
                    "serverInfo": {"name": "oracle-dwh", "version": "1.0.0"}
                }
            })
        elif method == "tools/list":
            write_message({
                "jsonrpc": "2.0",
                "id": msg_id,
                "result": {"tools": tools}
            })
        elif method == "tools/call":
            tool_name = params.get("name")
            tool_args = params.get("arguments", {})

            if tool_name == "oracle_query":
                result = handle_oracle_query(tool_args.get("sql"), tool_args.get("max_rows", 1000))
            elif tool_name == "oracle_execute":
                result = handle_oracle_execute(tool_args.get("sql"))
            elif tool_name == "oracle_list_tables":
                result = handle_oracle_list_tables(tool_args.get("schema"))
            elif tool_name == "oracle_describe":
                result = handle_oracle_describe(tool_args.get("table_name"), tool_args.get("schema"))
            elif tool_name == "oracle_test_connection":
                result = handle_oracle_test_connection()
            else:
                result = {"error": f"Tool '{tool_name}' not found"}

            write_message({
                "jsonrpc": "2.0",
                "id": msg_id,
                "result": {"content": [{"type": "text", "text": json.dumps(result, default=str)}]}
            })
        elif method == "notifications/initialized":
            pass  # No response needed

if __name__ == "__main__":
    main()
```

### 2.3 Instalar dependencias Python

```cmd
pip install oracledb
```

---

## Paso 3: Configurar Claude Code

### 3.1 Editar settings.json

Abrir el archivo `C:\Users\%USERNAME%\.claude\settings.json` y agregar la configuración del MCP server:

```json
{
  "mcpServers": {
    "oracle-dwh": {
      "command": "python",
      "args": ["C:/Users/TU_USUARIO/.claude/mcp-servers/oracle-dwh/server.py"]
    }
  }
}
```

**Nota:** Reemplazar `TU_USUARIO` con tu nombre de usuario de Windows.

### 3.2 Reiniciar Claude Code

Cerrar y volver a abrir Claude Code para que cargue la nueva configuración.

---

## Paso 4: Verificar Instalación

Dentro de Claude Code, ejecutar:

```
Prueba la conexión al DWH Oracle
```

Si todo está correcto, Claude debería poder usar las herramientas:
- `oracle_query` - Ejecutar SELECTs
- `oracle_execute` - Ejecutar DDL/DML
- `oracle_list_tables` - Listar tablas
- `oracle_describe` - Ver estructura de tablas
- `oracle_test_connection` - Probar conexión

---

## Solución de Problemas

### Error: ORA-12541 (TNS: no listener)
- El servidor de base de datos no está disponible
- Verificar conectividad de red al servidor 192.168.30.21

### Error: DPI-1047 (cannot locate Oracle Client library)
- Oracle Instant Client no está en el PATH
- Reinstalar y agregar al PATH del sistema

### Error: ORA-01017 (invalid username/password)
- Verificar credenciales en server.py
- Confirmar que el usuario tiene permisos en el DWH

---

## Configuración de Conexión Actual

| Parámetro | Valor |
|-----------|-------|
| Host | 192.168.30.21 |
| Puerto | 1521 |
| SID | plan2 |
| Usuario | POWER_STR |
| Esquema | POWER_STR (usuario = esquema por defecto) |

---

## Contacto

Para problemas de conexión al DWH, contactar al área de Sistemas.
Para problemas con Claude Code, revisar documentación en: https://claude.ai/code
