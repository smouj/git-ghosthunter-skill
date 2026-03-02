name: git-ghosthunter
version: 1.0.0
description: Rastrea y elimina ramas zombi y PRs obsoletos
author: SMOUJBOT
tags: [git, limpieza, ramas, pr, repositorios]
type: devops
dependencies:
  - git >= 2.30
  - gh (GitHub CLI) >= 2.0
  - jq (para análisis JSON)
  - bash >= 4.0
env_vars:
  - GITHUB_TOKEN (opcional, para llamadas API)
  - GIT_GHOSTHUNTER_DRY_RUN (por defecto: "false")
  - GIT_GHOSTHUNTER_AGE_DAYS (por defecto: "90")
  - GIT_GHOSTHUNTER_PROTECTED_BRANCHES (por defecto: "main,master,develop,production")
```

# Git Ghosthunter

**Propósito**: Detectar, analizar y eliminar ramas zombi y PRs obsoletos que saturan repositorios, reducen el mantenimiento y crean conflictos de fusión. Esta habilidad automatiza la limpieza de código abandonado que nunca llegó a producción.

## Casos de Uso Reales

1. **Detección de Ramas Zombi**: Encontrar ramas de características sin uso por 90+ días que puedan eliminarse con seguridad
2. **Limpieza de PRs Obsoletos**: Identificar PRs sin actividad por 60+ días y cerrarlos/limpiarlos
3. **Higiene del Repositorio**: Trabajos de limpieza semanales para equipos con alta rotación de ramas
4. **Preparación Pre-Fusión**: Limpiar antes de lanzamientos importantes para reducir el caos de ramas
5. **Cumplimiento de Auditoría**: Generar informes de trabajo abandonado para dirección

## Alcance

Esta habilidad opera en repositorios git locales y remotos. NO:
- Modifica ramas protegidas
- Elimina ramas con PRs abiertos y activos (a menos que se especifique)
- Toca etiquetas o releases
- Reescribe historial

Comandos ejecutados:
- `git branch --list --all`
- `git for-each-ref` con filtrado por fecha
- `git log -1 --format=%ci <branch>`
- `gh pr list` con salida JSON
- `git branch -d` / `-D` para eliminación
- `gh pr close` para limpieza de PRs
- `git push origin --delete` para eliminación de rama remota

## Proceso de Trabajo

### Fase 1: Descubrimiento
```bash
# Recopilar todas las ramas locales con fechas de último commit
git for-each-ref --format='%(refname:short)|%(committerdate:iso8601)|%(committerdate:unix)' refs/heads/ > branches.txt

# Recopilar todas las ramas remotas
git for-each-ref --format='%(refname:short)|%(committerdate:iso8601)|%(committerdate:unix)' refs/remotes/origin/ > remote_branches.txt
```

### Fase 2: Identificar Zombis
```bash
CUTOFF=$(date -d "-${GIT_GHOSTHUNTER_AGE_DAYS:-90} days" +%s)
while IFS='|' read branch date timestamp; do
  if [ "$timestamp" -lt "$CUTOFF" ]; then
    echo "ZOMBIE: $branch (último commit: $date)"
  fi
done < branches.txt
```

### Fase 3: Correlación con PRs
```bash
# Verificar si la rama zombi tiene PR abierto
gh pr list --json headRefName,state,updatedAt,url | jq -r '.[] | select(.headRefName=="'\"$branch\"'") | "\(.url)|\(.updatedAt)\"'"
```

### Fase 4: Clasificación
Rama clasificada como:
- **SAFE_DELETE**: Sin PR abierto, sin actividad reciente, no protegida
- **STALE_PR**: PR existe pero sin actualizaciones > 60 días
- **PROTECTED**: En lista de ramas protegidas
- **ACTIVE**: Commits recientes (< umbral de edad)

### Fase 5: Ejecución de Acciones
```bash
# Para ramas SAFE_DELETE:
git branch -d "$branch" 2>/dev/null || git branch -D "$branch"
git push origin --delete "$branch"

# Para STALE_PR:
gh pr close "$pr_number" --comment "Cerrando PR obsoleto después de ${GIT_GHOSTHUNTER_AGE_DAYS} días de inactividad"
```

### Fase 6: Informes
Generar SKILL_REPORT.md con:
- Total de ramas escaneadas
- Zombis encontrados
- Eliminaciones realizadas
- PRs cerrados
- Errores encontrados

## Reglas de Oro

1. **NUNCA eliminar ramas protegidas**: Verificar contra `$GIT_GHOSTHUNTER_PROTECTED_BRANCHES` antes de cualquier eliminación
2. **Dry-run por defecto**: Configurar `GIT_GHOSTHUNTER_DRY_RUN=true` para previsualizar acciones
3. **PRs requieren precaución extra**: Solo cerrar PRs automáticamente si se pasa flag `--force-pr-cleanup`
4. **Respaldar primero**: Crear etiquetas de backup antes de eliminaciones masivas
5. **Remoto antes que local**: Eliminar rama remota primero, luego local
6. **Respetar worktrees**: No eliminar ramas actualmente checked out en cualquier worktree
7. **No forzar sin confirmación**: `-D` (eliminación forzada) requiere flag `--allow-force` explícito
8. **Registrar todo**: Cada acción escrita a `$WORKDIR/.git-ghosthunter.log`

## Ejemplos

**Ejemplo 1: Encontrar solo ramas zombi (dry-run)**
```
Input: /ghosthunter scan --dry-run --age 60
Output:
[+] Escaneando repositorio...
[+] Total de ramas: 47
[+] Ramas zombi detectadas: 8
[+] Ramas protegidas: 3 (main, develop, staging)
[+] Seguras para eliminar: 5
[+] DRY-RUN - Sin cambios realizados
[+] Ejecutar de nuevo sin --dry-run para eliminar
Informe: SKILL_REPORT_20240115_143022.md
```

**Ejemplo 2: Limpieza completa incluyendo PRs obsoletos**
```
Input: /ghosthunter full-cleanup --age 90 --force-pr-cleanup --allow-force
Output:
[+] Etiquetas de backup creadas: ghosthunter-backup-20240115-1430
[+] Eliminando remoto: feature/old-login-refactor (zombi: 156 días)
[+] Eliminando local: feature/old-login-refactor
[+] Cerrando PR #342: "Old login refactor" (obsoleto: 89 días)
[+] Limpieza completa: 12 ramas eliminadas, 4 PRs cerrados
Informe: SKILL_REPORT_20240115_143045.md
```

**Ejemplo 3: Generar solo informe de zombies**
```
Input: /ghosthunter report --format json --age 120
Output:
{
  "repository": "myapp/frontend",
  "scan_time": "2024-01-15T14:30:45Z",
  "total_branches": 52,
  "zombie_branches": [
    {
      "name": "feature/dark-mode-v1",
      "last_commit": "2023-08-20T10:15:00Z",
      "days_inactive": 148,
      "has_open_pr": false,
      "protected": false
    }
  ],
  "stale_prs": 3,
  "protected_branches": ["main", "develop"]
}
```

**Ejemplo 4: Modo limpieza interactiva**
```
Input: /ghosthunter interactive
Output:
? Seleccionar ramas a eliminar (usar espacio para alternar):
  [ ] feature/experiment-api (142 días) - Sin PR
  [ ] feature/old-dashboard (98 días) - PR #289
  [✓] feature/broken-refactor (180 días) - Sin PR
  [ ] feature/2023-q4-goals (200+ días) - PR #156
? Confirmar eliminación de 1 rama seleccionada? (y/N)
```

**Ejemplo 5: Auditoría de repositorio completo**
```
Input: /ghosthunter audit --all-repos --output ghosthunter-audit-$(date +%Y%m).csv
Output:
[+] Escaneando 23 repositorios...
[+] Completado. Resultados guardados en ghosthunter-audit-202401.csv
CSV Columns: repository,branch,last_commit_days,has_pr,pr_number,pr_days_inactive,protected,recommended_action
```

## Variables de Entorno

| Variable | Por Defecto | Propósito |
|----------|-------------|-----------|
| `GIT_GHOSTHUNTER_DRY_RUN` | `false` | Configurar a `true` para solo previsualizar |
| `GIT_GHOSTHUNTER_AGE_DAYS` | `90` | Umbral de inactividad para clasificación zombi |
| `GIT_GHOSTHUNTER_PR_AGE_DAYS` | `60` | Umbral de inactividad para cierre de PR |
| `GIT_GHOSTHUNTER_PROTECTED_BRANCHES` | `main,master,develop,production` | Nombres de ramas protegidas separados por comas |
| `GIT_GHOSTHUNTER_BACKUP_PREFIX` | `ghosthunter-backup` | Prefijo para etiquetas de backup |
| `GITHUB_TOKEN` | (no configurado) | Opcional: usar para aumentar límites de tasa de API |
| `GIT_GHOSTHUNTER_LOG_LEVEL` | `INFO` | Registro: DEBUG, INFO, WARN, ERROR |

## Dependencias

### Requeridas
- `git` >= 2.30 (para formato `for-each-ref`)
- `bash` >= 4.0 (para arrays asociativos)
- `jq` >= 1.6 (para análisis JSON de PRs)
- `gh` (GitHub CLI) >= 2.0 (para operaciones de PR)

### Opcionales
- `csvkit` (para formato de informe CSV)
- `column` (para tablas bonitas)

### Instalación en Ubuntu/Debian
```bash
apt-get update && apt-get install -y git jq gh
```

### Instalación en macOS
```bash
brew install git jq gh
```

## Verificación

Después de la limpieza, verificar:
```bash
# Confirmar ramas eliminadas localmente
git branch --list | grep -E "feature/|bugfix/|hotfix/"

# Confirmar ramas remotas eliminadas
git ls-remote --heads origin | grep -E "feature/|bugfix/"

# Buscar PRs obsoletos restantes
gh pr list --state open --json updatedAt,headRefName | jq -r '.[] | select(.updatedAt < "$(date -d "60 days ago" -Iseconds)")'

# Verificar ramas protegidas intactas
git branch --list | grep -E "^(main|master|develop|production)$"
```

## Rollback

Si la eliminación fue un error, inmediatamente:

```bash
# Listar etiquetas de backup creadas antes de eliminación
git tag -l "ghosthunter-backup-*" | sort -r

# Restaurar rama específica desde backup
git checkout -b feature/branch-name ghosthunter-backup-20240115-1430/feature/branch-name

# Push rama restaurada a remoto
git push origin feature/branch-name

# Para restauración de PR: no se pueden reabrir automáticamente PRs cerrados
# Paso manual: Crear nuevo PR con misma rama, referenciar PR anterior en descripción
gh pr create --fill --body "Restaurado desde backup de ghosthunter. PR original #342."
```

**Rollback Completado**: Rama restaurada, PR debe recrearse manualmente.

## Solución de Problemas

**Problema**: "error: branch not found" durante eliminación
- **Causa**: Rama ya eliminada (posiblemente por otro proceso)
- **Solución**: Continuar, registrar como advertencia. Inspeccionar `SKILL_REPORT.log` para condiciones de carrera

**Problema**: "Authentication failed" con gh CLI
- **Causa**: No logueado o token expirado
- **Solución**: `gh auth refresh` o `gh auth login`

**Problema**: Errores de "protected branch"
- **Causa**: Rama coincide con lista protegida pero no en configuración local
- **Solución**: Verificar que `GIT_GHOSTHUNTER_PROTECTED_BRANCHES` la incluya

**Problema**: Eliminando rama en worktree actual
- **Causa**: Rama checked out en otra ubicación de worktree
- **Solución**: `git worktree list` para encontrar y cambiar, o usar `--force` cuidadosamente

**Problema**: PR aún abierto después de limpieza
- **Causa**: PR actualizado muy recientemente, o diferente headRefName
- **Solución**: Revisión manual: `gh pr view <number>` y cerrar manualmente

## Notas de Rendimiento

- Repos grandes (>500 ramas): Usar `--batch-size 50` para procesamiento por lotes
- Ligado a red: Llamadas API de GH limitadas a 30 req/min (manejado automáticamente)
- Memoria: < 100MB incluso para 1000+ ramas (análisis JSON en streaming)
```