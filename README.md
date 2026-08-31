# AWX Ansible: backups y despliegue web controlado
Proyecto de automatización de copias de seguridad, validación, pruebas de restauración y despliegue web controlado utilizando Ansible y AWX.

## 1. Objetivo

El proyecto automatiza el ciclo completo de protección de datos:

1. Comprobar la disponibilidad de los servidores.
2. Crear backups en los servidores origen.
3. Transferirlos a un servidor central de almacenamiento.
4. Validar que los backups existen y son utilizables.
5. Realizar una prueba de restauración en una ruta temporal.
6. Ejecutar el proceso desde un Workflow de AWX.
7. Programar ejecuciones automáticas.

La restauración se realiza en un directorio temporal para no modificar los datos originales.

## 2. Arquitectura del laboratorio

El laboratorio utiliza Ubuntu Server, Docker, SSH, rsync, Ansible, AWX y K3s.

### Componentes principales

| Componente | Dirección o nombre | Función |
|---|---|---|
| Ubuntu Server | `192.168.50.10` | Ansible, K3s, AWX y Docker |
| Ubuntu Cliente | `192.168.50.20` | Cliente de laboratorio |
| Servidor origen | `srv-origen-01` | Contenedor Docker de origen |
| Servidor destino | `srv-destino-01` | Contenedor Docker que almacena backups |
| Servidores web | `srv-web-01`, `srv-web-02` | Servidores incluidos en los backups |

Los contenedores utilizan el usuario `ansible` y autenticación SSH mediante clave.

## 3. Grupos de inventario

Los grupos principales son:

```text
backup_sources
├── srv-origen-01
├── srv-web-01
└── srv-web-02

destino
└── srv-destino-01
```

Los nombres de los grupos deben coincidir exactamente con los utilizados en los playbooks.

## 4. Directorios utilizados

```text
/datos_prueba       # Datos de prueba que se respaldan
/backups             # Almacenamiento de backups en el destino
/tmp/restore-test    # Restauraciones temporales
```

Una estructura de almacenamiento recomendada es:

```text
/backups/
├── srv-origen-01/
│   └── 2026-08-28/
├── srv-web-01/
│   └── 2026-08-28/
└── srv-web-02/
    └── 2026-08-28/
```

## 5. Estructura del proyecto

```text
awx-ansible/
├── README.md
├── inventories/
│   └── laboratorio/
│       ├── hosts.yml
│       └── group_vars/
│           └── all.yml
├── playbooks/
│   ├── backup.yml
│   ├── health_check.yml
│   ├── restore_test.yml
│   ├── validate_backup.yml
│   ├── web_health_check.yml
│   ├── web_deploy.yml
│   ├── web_validate.yml
│   └── web_rollback.yml
├── roles/
│   └── web_deploy/
│       ├── defaults/main.yml
│       ├── handlers/main.yml
│       ├── tasks/main.yml
│       └── templates/index.html.j2
└── docs/
    └── memoria-despliegue-awx.md
```

Los cuatro últimos playbooks corresponden al despliegue web controlado.

## 6. Variables principales

Las variables generales se encuentran en:

```text
inventories/laboratorio/group_vars/all.yml
```

Ejemplo:

```yaml
backup_paths:
  - /datos_prueba

backup_destination_host: srv-destino-01
backup_destination_path: /backups
backup_retention_days: 7
backup_dry_run: true
```

### Descripción de variables

| Variable | Descripción |
|---|---|
| `backup_paths` | Lista de rutas que se van a respaldar |
| `backup_destination_host` | Servidor que almacena los backups |
| `backup_destination_path` | Directorio raíz de almacenamiento |
| `backup_retention_days` | Días que se conservan los backups |
| `backup_dry_run` | Ejecuta el proceso sin realizar cambios definitivos |

Las rutas y variables sensibles no deben guardarse directamente en los playbooks. Las credenciales deben configurarse mediante AWX.

## 7. Playbooks

### `health_check.yml`

Comprueba que AWX puede conectarse a los servidores y muestra información básica:

- nombre del host;
- sistema operativo;
- dirección IP;
- espacio disponible;
- memoria;
- estado de SSH;
- existencia de las rutas de backup.

### `backup.yml`

Realiza el backup de cada servidor origen y lo transfiere al servidor destino.

El proceso recomendado es:

1. Crear el directorio de trabajo.
2. Comprimir las rutas seleccionadas.
3. Crear el archivo `.tar.gz`.
4. Transferirlo mediante SSH y rsync.
5. Crear un manifiesto.
6. Comprobar que el archivo existe en el destino.

AWX coordina el proceso, pero no debe utilizarse como repositorio de backups.

### `validate_backup.yml`

Valida que los backups existen en el destino y que contienen la información esperada.

Comprueba:

1. Que existen archivos `.tar.gz`.
2. Que el número de backups es mayor que cero.
3. Que existe el manifiesto.
4. Que el tamaño del backup es razonable.

### `restore_test.yml`

Realiza una restauración de prueba:

1. Busca el backup más reciente.
2. Crea `/tmp/restore-test`.
3. Extrae el backup en esa ruta.
4. Comprueba que se han recuperado archivos.
5. Muestra el resultado.
6. Elimina los datos temporales cuando finaliza la prueba.

Nunca debe restaurar directamente sobre `/datos_prueba` sin confirmación explícita.

## 8. Despliegue web controlado

El proyecto incluye un flujo independiente para desplegar una página web en `srv-web-01` y `srv-web-02`.

El despliegue web incluye:

- comprobación previa del estado de Nginx;
- instalación y actualización de la página mediante una plantilla Jinja2;
- validación HTTP posterior al despliegue;
- backups de la página anterior;
- aprobación manual desde AWX;
- rollback automático ante un error;
- posibilidad de provocar un fallo controlado para probar la recuperación.

El Workflow de despliegue web es:

```text
Web - Health Check
        ↓
Web - Deploy
        ↓
Web - Validate
        ├── Éxito → Aprobación despliegue
        └── Error → Web - Rollback
```

Los parámetros principales del Survey son:

| Variable | Función |
|---|---|
| `web_version` | Versión que se despliega. |
| `web_message` | Mensaje que aparece en la página. |
| `web_force_failure` | Permite provocar un fallo controlado. |

Para consultar la configuración completa y los resultados de las pruebas, véase la [memoria del despliegue web](docs/memoria-despliegue-awx.md).

## 9. Configuración de AWX

### Proyecto

Configurar un proyecto llamado:

```text
AWX Ansible Backups
```

Configuración recomendada:

```text
Organización: Default
Tipo de fuente: Git
Rama: main
Playbooks: playbooks/*.yml
```

El repositorio Git debe configurarse con la URL real del proyecto. No se debe depender de un servidor HTTP temporal para producción.

### Credenciales

Configurar en AWX:

- una credencial Git, si el repositorio es privado;
- una credencial de tipo Machine para la clave SSH;
- una credencial Vault si se utilizan secretos cifrados.

Las claves privadas, contraseñas y tokens no deben almacenarse en este README.

### Entorno de ejecución

Las plantillas deben utilizar un Execution Environment que incluya las colecciones necesarias, especialmente:

- `ansible.posix`;
- `community.general`, si algún playbook la necesita;
- la versión de `ansible-core` definida para el proyecto.

## 10. Plantillas de trabajo

Crear las siguientes plantillas:

```text
Health Check - Laboratorio
Backup - Laboratorio
Validate Backup - Laboratorio
Restore Test - Laboratorio
```

Todas deben utilizar el proyecto y el inventario adecuados. La plantilla de restauración debe empezar utilizando una ruta temporal y datos de prueba.

## 11. Workflow

El Workflow recomendado es:

```text
Health Check
     ↓
Backup
     ↓
Validate Backup
     ↓
Restore Test
```

La siguiente fase solo debe ejecutarse si la anterior finaliza correctamente. Las notificaciones de error deben configurarse en AWX cuando sea posible.

## 12. Survey

El Workflow o la plantilla de backup puede utilizar un Survey con estas variables:

```text
backup_paths
backup_dry_run
backup_retention_days
```

Durante las primeras pruebas se recomienda utilizar:

```yaml
backup_dry_run: true
```

La eliminación de backups antiguos debe probarse primero en modo simulación.

## 13. Programación

El Workflow puede programarse diariamente, por ejemplo, a las 08:00 en la zona horaria del laboratorio.

Antes de activar la programación:

- ejecutar el Workflow manualmente;
- revisar todos los resultados;
- comprobar el almacenamiento del destino;
- realizar una restauración de prueba;
- evitar ejecuciones simultáneas.

## 14. Pruebas de aceptación

El proyecto se considera correcto cuando:

- AWX conecta con todos los hosts;
- `health_check.yml` finaliza correctamente;
- se genera al menos un backup por servidor origen;
- los archivos aparecen en `/backups`;
- la validación detecta los backups;
- la restauración recupera archivos en `/tmp/restore-test`;
- ningún dato original se sobrescribe;
- el Workflow completo termina correctamente;
- el proceso puede ejecutarse de nuevo sin errores;
- los cambios están versionados en Git.

## 15. Gestión de cambios

Comprobar el estado del repositorio:

```bash
git status
git diff
git diff --check
```

Guardar y publicar cambios:

```bash
git add .
git commit -m "Descripcion del cambio"
git push
```

Después de cada `push`, actualizar el proyecto en AWX antes de ejecutar una plantilla.

## 16. Comandos útiles

Ver los playbooks:

```bash
find playbooks -maxdepth 1 -type f -printf '%f\n' | sort
```

Ver las variables utilizadas:

```bash
grep -R "backup_paths\|backup_destination_path\|backup_destination_host" -n inventories/
```

Ver el último commit:

```bash
git log -1 --oneline
```

## 17. Seguridad y buenas prácticas

- Utilizar claves SSH gestionadas por AWX.
- No guardar secretos en Git.
- Utilizar Ansible Vault para secretos necesarios.
- Probar primero con `check mode` o `dry run`.
- No utilizar `StrictHostKeyChecking=no` en entornos reales.
- No ejecutar eliminaciones sin una validación previa.
- Mantener AWX separado del almacenamiento de backups.
- Probar periódicamente la restauración.
- Documentar cada cambio en Git.

## 18. Estado del proyecto

El laboratorio dispone de una migración funcional entre servidores simulados mediante Docker, SSH, Ansible y AWX.

La siguiente evolución es completar el sistema de backups con:

- proyecto versionado en Git;
- Workflow de backup;
- validación automática;
- política de retención;
- prueba de restauración;
- programación diaria;
- notificaciones de éxito y error.

## 19. Tecnologías utilizadas

- Ansible
- AWX
- AWX Operator
- K3s
- Docker
- Ubuntu Server
- SSH
- rsync
- Git
- YAML
