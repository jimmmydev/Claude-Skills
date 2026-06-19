# db-safety — Claude Code Skill

> **Un SKILL que bloquea operaciones destructivas de base de datos.**
> Claude nunca ejecuta `DROP`, `DELETE` ni `TRUNCATE` directamente.
> Siempre proporciona la query para que la revises y ejecutes tú.

---

## Instalación

Copia el bloque de abajo a tu proyecto en:

```
.claude/skills/db-safety/SKILL.md
```

Luego referencia el skill desde tu `CLAUDE.md`:

```markdown
## Skills (Auto-load based on context)

| Context                        | Read this file                          |
|--------------------------------|-----------------------------------------|
| Queries SQL, migraciones, BD   | `.claude/skills/db-safety/SKILL.md`     |
```

---

## SKILL.md

```markdown
---
name: db-safety
description: >
  Prevents Claude from executing destructive database operations.
  Applies whenever working with SQL, migrations, ORMs, or database tooling.
---

## Regla principal

NUNCA ejecutes directamente ninguna de estas operaciones:

- `DROP DATABASE`
- `DROP TABLE`
- `DROP SCHEMA`
- `DELETE FROM` (con o sin WHERE)
- `TRUNCATE TABLE`
- `ALTER TABLE … DROP COLUMN`
- `UPDATE` masivo sin WHERE

## Comportamiento requerido

Cuando una tarea requiera una operación de la lista anterior:

1. **No ejecutes el comando.** No uses `psql`, `mysql`, `sqlite3` ni ningún ORM para lanzarla.
2. **Proporciona la query** como bloque de código SQL con sintaxis resaltada.
3. **Añade un aviso** que indique qué datos se verán afectados y que la operación es irreversible.
4. **Pide confirmación explícita**: indica al usuario que la revise y la ejecute manualmente.

## Formato de respuesta

```
⚠️  Operación destructiva — no la ejecuto directamente.

Revisa esta query antes de lanzarla:

```sql
DELETE FROM users
WHERE active = false
  AND last_login < NOW() - INTERVAL '90 days';
```

Esto eliminará permanentemente los registros que cumplan la condición.
Sin backup previo no hay vuelta atrás. Ejecútala tú cuando estés seguro.
```

## Operaciones que SÍ puedes ejecutar

- `SELECT` (lectura, siempre seguro)
- `INSERT` (siempre que sea explícito y acotado)
- `UPDATE` con `WHERE` acotado y revisado
- `CREATE TABLE`, `ALTER TABLE … ADD COLUMN`
- Migraciones de adición (nunca de borrado sin revisión)
- `EXPLAIN` / `EXPLAIN ANALYZE`

## Por qué existe esta regla

Claude Code pide confirmación antes de ejecutar comandos destructivos,
pero el humano puede aprobar sin leer. Esta skill elimina la posibilidad
de que un descuido en el prompt de confirmación borre datos de producción.
La última línea de defensa siempre debe ser un humano que lee la query.
```

---

## Complemento recomendado: usuario de BD con permisos mínimos

El SKILL protege a nivel de comportamiento de Claude.
Para una segunda capa de seguridad, crea un usuario de BD dedicado para Claude
**sin permisos de DROP ni DELETE**:

```sql
-- PostgreSQL
CREATE USER claude_agent WITH PASSWORD '…';
GRANT CONNECT ON DATABASE myapp TO claude_agent;
GRANT USAGE ON SCHEMA public TO claude_agent;
GRANT SELECT, INSERT, UPDATE ON ALL TABLES IN SCHEMA public TO claude_agent;
-- Sin GRANT DELETE, DROP ni TRUNCATE

-- MySQL / MariaDB
CREATE USER 'claude_agent'@'localhost' IDENTIFIED BY '…';
GRANT SELECT, INSERT, UPDATE ON myapp.* TO 'claude_agent'@'localhost';
-- Sin DELETE, DROP ni ALTER
```

Así, aunque Claude intentase ejecutar un `DROP` (o el usuario lo apruebe por error),
**el motor de BD lo rechazará a nivel de permisos**.

---

## Dos capas, cero sorpresas

| Capa | Qué hace | Cuándo falla |
|------|----------|--------------|
| **SKILL db-safety** | Claude no genera ni ejecuta la op destructiva | Si el SKILL no está activo en esa sesión |
| **Usuario BD sin DROP** | El motor rechaza la operación a nivel de permisos | Nunca — es infraestructura |

Usa las dos juntas. Una sola capa no es suficiente.
