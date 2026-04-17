# LDR: Publicación en Producción Retardada

## Descripción general

Todo el proceso se desencadena ejecutando el workflow **Create release** desde GitHub Actions (ver [Cómo crear un release](#cómo-crear-un-release)). En lugar de publicar en producción de forma inmediata, el sistema:

1. Al finalizar el release, calcula la hora de publicación configurada y la fija como un cron schedule en el workflow `LDR_PublicarEnProduccionRetardado.yaml`.
2. A la hora programada, ese workflow se activa, lanza el deploy al entorno de producción y elimina el cron de sí mismo, dejando el workflow limpio hasta el siguiente release.

En repositorios con protección de rama, los cambios sobre los workflows se realizan a través de **Pull Requests** que se aprueban y fusionan automáticamente.

---

## Flujo completo

```
CreateRelease.yaml
  └── Job: CustomJob-UpdatePublishSchedule
        └── Invoca LDR_ProgramarPublicacion.yaml (workflow_call)

LDR_ProgramarPublicacion.yaml
  └── Job: UpdatePublishSchedule
        ├── Comprueba PublicarOnRelease en LDR-Settings.json
        ├── Lee HoraPublicacion de LDR-Settings.json
        ├── Calcula cron con la fecha actual y la hora configurada
        ├── Modifica el cron en LDR_PublicarEnProduccionRetardado.yaml
        ├── Crea un Pull Request con el cambio (rama: ldr/update-schedule)
        ├── Habilita auto-merge en el PR (si el repositorio tiene checks requeridos)
        └── Job Auto-aprobacion: aprueba el PR con GHTOKEN_AUTO y lo fusiona directamente

LDR_PublicarEnProduccionRetardado.yaml  (activado por el cron escrito)
  ├── Job: TriggerPublish
  │     ├── Valida que el retraso de ejecución no supere 3 horas
  │     ├── Lee NombreEntorno de LDR-Settings.json
  │     └── Dispara PublishToEnvironment.yaml para ese entorno
  └── Job: RemoveCronAfterExecution
        ├── Elimina la sección schedule del propio workflow
        ├── Crea un Pull Request con el cambio (rama: ldr/remove-cron)
        ├── Habilita auto-merge en el PR (si el repositorio tiene checks requeridos)
        └── Job Auto-aprobacion: aprueba el PR con GHTOKEN_AUTO y lo fusiona directamente
```

---

## Archivos implicados

| Archivo | Propósito |
|---|---|
| `.github/LDR-Settings.json` | Configuración del mecanismo (ver sección siguiente) |
| `.github/workflows/CreateRelease.yaml` | Contiene el job `CustomJob-UpdatePublishSchedule` que invoca `LDR_ProgramarPublicacion.yaml` |
| `.github/workflows/LDR_ProgramarPublicacion.yaml` | Contiene toda la lógica de programación del cron; se puede invocar también manualmente |
| `.github/workflows/LDR_PublicarEnProduccionRetardado.yaml` | Workflow que ejecuta el deploy y se auto-limpia |

---

## Configuración: `LDR-Settings.json`

```json
{
  "PublicarOnRelease": true,
  "NombreEntorno": "PRODUCTION",
  "HoraPublicacion": "23:45"
}
```

| Campo | Tipo | Descripción |
|---|---|---|
| `PublicarOnRelease` | boolean | Controla si se debe programar la publicación al crear un release. Si es `false`, el job `CustomJob-UpdatePublishSchedule` se ejecuta pero **no programa ningún cron** ni modifica ningún archivo. |
| `NombreEntorno` | string | Nombre exacto del entorno de Business Central al que se publicará. Debe coincidir con el nombre configurado en el repositorio de GitHub. |
| `HoraPublicacion` | string `HH:MM` | Hora UTC a la que se programará la publicación el mismo día en que se cree el release. Formato 24 horas. |

> **Importante:** La hora es UTC. Tenga en cuenta el desfase horario de su zona.  
> Ejemplo: si su zona es UTC+2 y quiere publicar a las 01:45 hora local, configure `"HoraPublicacion": "23:45"`.

---

## Job `CustomJob-UpdatePublishSchedule` en `CreateRelease.yaml`

Este job se ejecuta en paralelo con `UpdateVersionNumber` y `CreateReleaseBranch`, después de que `CreateRelease` y `UploadArtifacts` hayan finalizado con éxito.

Actúa como punto de entrada delegando toda la lógica al workflow reutilizable `LDR_ProgramarPublicacion.yaml` mediante `workflow_call`:

```yaml
uses: ./.github/workflows/LDR_ProgramarPublicacion.yaml
with:
  releaseVersion: ${{ needs.CreateRelease.outputs.releaseVersion }}
secrets: inherit
```

Gracias a `secrets: inherit`, el secreto `GHTOKENWORKFLOW` se transmite automáticamente al workflow llamado.

---

## Workflow `LDR_ProgramarPublicacion.yaml`

Este workflow contiene toda la lógica de programación del cron. Se puede invocar de dos formas:
- **Automáticamente** desde `CreateRelease.yaml` mediante `workflow_call`.
- **Manualmente** desde la pestaña Actions de GitHub mediante `workflow_dispatch`, pasando opcionalmente el input `releaseVersion` (se usa solo en el mensaje del commit).

### Job `UpdatePublishSchedule`

#### 1. Generate GitHub App token
Genera un token de acceso de corta duración a partir del secreto `GHTOKENWORKFLOW` (que debe contener `GitHubAppClientId` y `PrivateKey` en formato JSON). Este token con permisos de escritura en el repositorio es necesario para que el commit posterior desencadene otros workflows (un `GITHUB_TOKEN` estándar no lo hace).

#### 2. Checkout
Clona el repositorio usando el token generado en el paso anterior.

#### 3. Check PublicarOnRelease setting
Lee el campo `PublicarOnRelease` de `LDR-Settings.json` y lo expone como output `publicar`. Si el valor no es `true`, el siguiente paso se omite por completo y el job finaliza sin error ni cambios.

#### 4. Update cron schedule
> Este paso solo se ejecuta si `PublicarOnRelease` es `true`.
- Lee `HoraPublicacion` de `.github/LDR-Settings.json`.
- Calcula la expresión cron usando el día y mes actuales (del momento en que se crea el release) y la hora configurada.  
  Ejemplo: si se crea el release el 14 de abril a cualquier hora y `HoraPublicacion` es `23:45`, el cron generado será:  
  ```
  45 23 14 4 *
  ```
- Localiza la sección `on: > schedule: > - cron:` dentro de `LDR_PublicarEnProduccionRetardado.yaml` con un script Python y sustituye (o crea) el valor del cron.

#### 5. Create Pull Request
> Este paso solo se ejecuta si `PublicarOnRelease` es `true`.

Crea un Pull Request en la rama `ldr/update-schedule` con el archivo `LDR_PublicarEnProduccionRetardado.yaml` modificado. A continuación intenta habilitar el **auto-merge** en el PR (squash), aunque este paso es opcional y no interrumpe el flujo si el repositorio no tiene checks requeridos configurados. El job `Auto-aprobacion` aprueba el PR usando el secreto `GHTOKEN_AUTO` y a continuación lo fusiona directamente con `gh pr merge --squash`, garantizando el merge independientemente de si el auto-merge estaba disponible.

---

## Workflow `LDR_PublicarEnProduccionRetardado.yaml`

### Job `TriggerPublish`

| Paso | Descripción |
|---|---|
| Block manual execution without scheduled cron | Si se lanza manualmente y no hay ningún cron activo en el archivo, la ejecución falla con error explicativo. Esto evita publicaciones accidentales fuera del ciclo de release. |
| Check scheduled execution delay | Si el trigger es `schedule`, comprueba que el workflow no se ha ejecutado con más de 3 horas de retraso respecto a la hora programada. Si se supera ese límite (por ejemplo, por una cola de GitHub Actions muy cargada), la ejecución se cancela como medida de precaución. |
| Read NombreEntorno | Lee el campo `NombreEntorno` de `LDR-Settings.json`. |
| Trigger Publish To Environment | Envía un `workflow_dispatch` a `PublishToEnvironment.yaml` pasando el entorno y `appVersion: current`. |

### Job `RemoveCronAfterExecution`

Se ejecuta siempre que `TriggerPublish` termine (con éxito o fallo). Genera un token de App, hace checkout y ejecuta un script Python que elimina toda la sección `schedule:` del bloque `on:` del propio archivo `LDR_PublicarEnProduccionRetardado.yaml`, dejándolo solo con `workflow_dispatch`. A continuación crea un Pull Request en la rama `ldr/remove-cron`, intenta habilitar auto-merge (paso opcional con `continue-on-error`) y el job `Auto-aprobacion` aprueba el PR con `GHTOKEN_AUTO` y lo fusiona directamente con `gh pr merge --squash`.

Esto garantiza que el workflow **no se repita** en fechas futuras con el mismo cron, ya que GitHub Actions ejecutaría de nuevo un cron `45 23 14 4 *` cualquier 14 de abril de años sucesivos.

---

## Requisitos previos

### Secreto `GHTOKENWORKFLOW`

Debe existir un secreto de repositorio (u organización) llamado `GHTOKENWORKFLOW` con el siguiente contenido JSON:

```json
{
  "GitHubAppClientId": "Iv1.xxxxxxxxxxxx",
  "PrivateKey": "-----BEGIN RSA PRIVATE KEY-----\n...\n-----END RSA PRIVATE KEY-----"
}
```

La GitHub App asociada debe tener instalación en el repositorio con los permisos:
- **Contents**: Read & Write (para hacer commit en la rama del PR)
- **Actions**: Write (para disparar `workflow_dispatch`)
- **Pull requests**: Read & Write (para crear PRs y habilitar auto-merge)

### Secreto `GHTOKEN_AUTO`

Debe existir un secreto de repositorio (u organización) llamado `GHTOKEN_AUTO` con un **Personal Access Token** del usuario **Lider-IT-Consulting**.

Este token se usa exclusivamente para aprobar los Pull Requests creados por los workflows LDR, actuando como revisor independiente (GitHub no permite que el mismo actor que crea el PR lo apruebe).

Requisitos del token:
- **Tipo**: Classic PAT con scope `repo`, o Fine-grained PAT con permisos `Pull requests: Read and write` y `Contents: Read and write` sobre el repositorio.
- **Rol en el repositorio**: el usuario propietario del token debe tener rol **Write** (o superior: Maintain, Admin) en el repositorio. Sin este rol, el merge del PR fallará con el error `does not have the correct permissions to execute MergePullRequest`.
- **Usuario**: debe ser distinto al autor del PR (la GitHub App configurada en `GHTOKENWORKFLOW`).
- Si el repositorio pertenece a una organización con SSO, el token debe estar autorizado para esa organización.

### Configuración del repositorio: Allow auto-merge

Para que el paso `Habilitar Pull Request Automerge` funcione (opción adicional cuando hay checks requeridos), debe estar habilitada la opción **Allow auto-merge** en la configuración del repositorio:

1. Ir a **Settings** → **General**.
2. En la sección **Pull Requests**, marcar **Allow auto-merge**.

Esta opción es **opcional**: los workflows LDR siempre realizan el merge directamente con `gh pr merge --squash` en el job `Auto-aprobacion`, por lo que el flujo funciona aunque la rama no tenga checks requeridos ni auto-merge habilitado.

### Workflow `PublishToEnvironment.yaml`

Debe existir en `.github/workflows/PublishToEnvironment.yaml` y aceptar los inputs `environmentName`, `appVersion` y `createEnvIfNotExists`. Este workflow es generado y mantenido por AL-Go for GitHub.

### Configuración de AL-Go: `AL-Go-Settings.json`

Para que AL-Go sepa a qué entorno desplegar y cómo, deben configurarse dos entradas en `.github/AL-Go-Settings.json`:

**1. `environments`** — lista de identificadores de entorno que AL-Go gestionará:

```json
"environments": [
    "<environmentId>"
]
```

**2. `DeployTo<environmentId>`** — configuración específica del entorno. El nombre del campo debe coincidir exactamente con el identificador usado en `environments`:

```json
"DeployTo<environmentId>": [
  {
    "EnvironmentName": "<NombreEntornoEnBusinessCentral>",
    "ContinuousDeployment": false
  }
]
```

> El valor de `EnvironmentName` es el nombre del entorno tal como aparece en el portal de Business Central / GitHub Environments. El valor de `<environmentId>` en el nombre del campo es el identificador interno de AL-Go (puede contener guiones bajos pero no espacios).

Consulte la referencia completa de `DeployTo<Target>` en la [documentación oficial de AL-Go](https://github.com/microsoft/AL-Go/blob/main/Scenarios/settings.md#DeployTo).

### Secreto `<environmentId>_AUTHCONTEXT`

Para que AL-Go pueda autenticarse contra el entorno de Business Central al publicar, debe existir un secreto de repositorio (u organización) con el nombre `<environmentId>_AUTHCONTEXT`, donde `<environmentId>` es el mismo identificador usado en `environments` y `DeployTo<environmentId>`.

El contenido del secreto es un objeto JSON con las credenciales de la service connection. Por ejemplo, usando autenticación de aplicación:

```json
{
  "TenantId": "<tenant-id>",
  "AppId": "<app-id>",
  "AppSecret": "<app-secret>"
}
```

Consulte los formatos de autenticación soportados en la [documentación oficial de AL-Go](https://github.com/microsoft/AL-Go/blob/main/Scenarios/settings.md#DeployTo).

---

## Cómo crear un release

El proceso completo se inicia ejecutando el workflow **Create release** desde la pestaña **Actions** de GitHub con los siguientes inputs:

| Input | Valor recomendado | Notas |
|---|---|---|
| Build version to promote to release | `latest` | Usa el artefacto de la última build |
| Name of this release | `v1.1` | Nombre legible del release (ej. `v<Major>.<Minor>`) |
| Tag of this release | `1.1.0` | Versión semántica ([semver.org](https://semver.org)), debe coincidir con la versión de la app |
| Release, prerelease or draft? | `Release` | Usar `Release` para publicar en producción |
| Create Release Branch? | _(marcado)_ | Crea una rama de mantenimiento para el release |
| Prefix for release branch | `release/` | Solo relevante si se marca la opción anterior |
| New Version Number in main branch | `+0.1` | Incrementa el número de minor en la rama main tras el release; usar `+1` para salto de major |
| Skip updating dependency version numbers | _(desmarcado)_ | Marcar solo si se gestionan dependencias manualmente |
| Direct Commit? | _(desmarcado)_ | Si se desmarca, el incremento de versión se hace mediante PR |
| Use GhTokenWorkflow for PR/Commit? | _(marcado)_ | Necesario para que los commits de sistema desencadenen otros workflows |

> **Nota:** Al completarse el release con éxito, el job `CustomJob-UpdatePublishSchedule` invoca automáticamente `LDR_ProgramarPublicacion.yaml`, que programa la publicación en producción a la hora configurada en `LDR-Settings.json`.

---

## Ejecución manual de `LDR_PublicarEnProduccionRetardado`

Es posible lanzar el workflow manualmente desde la pestaña **Actions** de GitHub, pero **solo si hay un cron activo** en el archivo (es decir, si se ha creado un release y aún no se ha ejecutado el cron). Si no hay cron, la ejecución manual fallará en el primer paso con un mensaje de error claro.

Para forzar una publicación manual fuera del ciclo de release, use directamente el workflow `PublishToEnvironment.yaml`.

---

## Diagrama de secuencia

```
CreateRelease workflow
      │
      ▼
CustomJob-UpdatePublishSchedule  (workflow_call)
      │
      ▼
LDR_ProgramarPublicacion.yaml
      │  crea PR con cron (ej: "45 23 14 4 *") → auto-aprobado → auto-merged
      ▼
LDR_PublicarEnProduccionRetardado.yaml  ◄── (merge activa el cron en GitHub)
      │
      │  (a las 23:45 UTC del día del release)
      ▼
Job: TriggerPublish
      │  dispara PublishToEnvironment para NombreEntorno
      ▼
Job: RemoveCronAfterExecution
      │  crea PR eliminando schedule → auto-aprobado → auto-merged
      ▼
   (workflow queda limpio, sin cron)
```
