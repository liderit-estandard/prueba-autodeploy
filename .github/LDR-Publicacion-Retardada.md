# LDR: Publicación en Producción Retardada

## Descripción general

Este mecanismo permite programar automáticamente la publicación en producción cada vez que se crea un release. En lugar de publicar de forma inmediata, el sistema:

1. Al finalizar el release, calcula la hora de publicación configurada y la fija como un cron schedule en el workflow `LDR_PublicarEnProduccionRetardado.yaml`.
2. A la hora programada, ese workflow se activa, lanza el deploy al entorno de producción y elimina el cron de sí mismo, dejando el workflow limpio hasta el siguiente release.

---

## Flujo completo

```
CreateRelease.yaml
  └── Job: CustomJob-UpdatePublishSchedule
        ├── Lee HoraPublicacion de LDR-Settings.json
        ├── Calcula cron con la fecha actual y la hora configurada
        └── Escribe el cron en LDR_PublicarEnProduccionRetardado.yaml y hace commit

LDR_PublicarEnProduccionRetardado.yaml  (activado por el cron escritoo)
  ├── Job: TriggerPublish
  │     ├── Valida que el retraso de ejecución no supere 3 horas
  │     ├── Lee NombreEntorno de LDR-Settings.json
  │     └── Dispara PublishToEnvironment.yaml para ese entorno
  └── Job: RemoveCronAfterExecution
        ├── Elimina la sección schedule del propio workflow
        └── Hace commit del archivo limpio
```

---

## Archivos implicados

| Archivo | Propósito |
|---|---|
| `.github/LDR-Settings.json` | Configuración del mecanismo (ver sección siguiente) |
| `.github/workflows/CreateRelease.yaml` | Contiene el job `CustomJob-UpdatePublishSchedule` que programa el cron |
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

### Pasos

#### 1. Generate GitHub App token
Genera un token de acceso de corta duración a partir del secreto `GHTOKENWORKFLOW` (que debe contener `GitHubAppClientId` y `PrivateKey` en formato JSON). Este token con permisos de escritura en el repositorio es necesario para que el commit posterior desencadene otros workflows (un `GITHUB_TOKEN` estándar no lo hace).

#### 2. Checkout
Clona el repositorio usando el token generado en el paso anterior.

#### 3. Check PublicarOnRelease setting
Lee el campo `PublicarOnRelease` de `LDR-Settings.json` y lo expone como output `publicar`. Si el valor no es `true`, el siguiente paso se omite por completo y el job finaliza sin error ni cambios.

#### 4. Update cron schedule and commit
> Este paso solo se ejecuta si `PublicarOnRelease` es `true`.
- Lee `HoraPublicacion` de `.github/LDR-Settings.json`.
- Calcula la expresión cron usando el día y mes actuales (del momento en que se crea el release) y la hora configurada.  
  Ejemplo: si se crea el release el 14 de abril a cualquier hora y `HoraPublicacion` es `23:45`, el cron generado será:  
  ```
  45 23 14 4 *
  ```
- Localiza la sección `on: > schedule: > - cron:` dentro de `LDR_PublicarEnProduccionRetardado.yaml` con un script Python y sustituye (o crea) el valor del cron.
- Hace `git commit` y `git push` solo si hubo cambios reales.

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

Se ejecuta siempre que `TriggerPublish` termine (con éxito o fallo). Genera un token de App, hace checkout y ejecuta un script Python que elimina toda la sección `schedule:` del bloque `on:` del propio archivo `LDR_PublicarEnProduccionRetardado.yaml`, dejándolo solo con `workflow_dispatch`. Luego hace commit y push.

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
- **Contents**: Read & Write (para hacer commit/push)
- **Actions**: Write (para disparar `workflow_dispatch`)

### Workflow `PublishToEnvironment.yaml`

Debe existir en `.github/workflows/PublishToEnvironment.yaml` y aceptar los inputs `environmentName`, `appVersion` y `createEnvIfNotExists`. Este workflow es generado y mantenido por AL-Go for GitHub.

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
CustomJob-UpdatePublishSchedule
      │  escribe cron (ej: "45 23 14 4 *")
      ▼
LDR_PublicarEnProduccionRetardado.yaml  ◄── (commit activa el cron en GitHub)
      │
      │  (a las 23:45 UTC del día del release)
      ▼
Job: TriggerPublish
      │  dispara PublishToEnvironment para NombreEntorno
      ▼
Job: RemoveCronAfterExecution
      │  elimina la sección schedule del workflow
      ▼
   (workflow queda limpio, sin cron)
```
