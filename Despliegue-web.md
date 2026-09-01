# Memoria del proyecto: despliegue web controlado con AWX y Ansible

## 1. Introducción

El objetivo de este proyecto es implementar un sistema de despliegue web automatizado mediante **Ansible y AWX**, permitiendo realizar despliegues controlados sobre varios servidores web.

El sistema incorpora:

- Comprobación previa del estado de los servidores.
- Despliegue automatizado de una página web.
- Validación posterior del despliegue.
- Aprobación manual antes de finalizar el proceso.
- Rollback automático en caso de fallo.
- Backups de la página web anterior.
- Parámetros configurables mediante encuestas de AWX.
- Control de versiones mediante Git.

El entorno de laboratorio está compuesto por dos servidores web:

- `srv-web-01`
- `srv-web-02`

## 2. Arquitectura del entorno

AWX actúa como plataforma de automatización y Ansible como herramienta de configuración y despliegue.

El flujo general es:

```text
AWX
└── Workflow: Despliegue Web Controlado - Laboratorio
    ├── Web - Health Check
    ├── Web - Deploy
    ├── Web - Validate
    ├── Aprobación del despliegue
    └── Web - Rollback en caso de error
```

Los servidores web están agrupados en Ansible bajo el grupo `origenes`. En AWX, este grupo aparece como `Origenes`.

## 3. Inventario

El inventario de laboratorio se encuentra en:

```text
inventories/laboratorio/hosts.yml
```

La comprobación del inventario se realizó mediante:

```bash
ansible-inventory -i inventories/laboratorio/hosts.yml --graph
```

Resultado relevante:

```text
@origenes:
  |--srv-web-01
  |--srv-web-02
```

También se comprobó el contenido del inventario mediante:

```bash
ansible-inventory -i inventories/laboratorio/hosts.yml --list
```

## 4. Estructura del proyecto

```text
awx-ansible/
├── inventories/
│   └── laboratorio/
│       └── hosts.yml
├── playbooks/
│   ├── web_health_check.yml
│   ├── web_deploy.yml
│   ├── web_validate.yml
│   └── web_rollback.yml
├── roles/
│   └── web_deploy/
│       ├── defaults/
│       │   └── main.yml
│       ├── handlers/
│       │   └── main.yml
│       ├── tasks/
│       │   └── main.yml
│       └── templates/
│           └── index.html.j2
└── docs/
    └── memoria-despliegue-awx.md
```

## 5. Playbooks

### 5.1 Web Health Check

Archivo: `playbooks/web_health_check.yml`

Comprueba que los servidores están preparados para recibir el despliegue. Entre las comprobaciones realizadas se encuentra el estado de Nginx.

El playbook utiliza:

```yaml
hosts: Origenes
```

En AWX, el inventario `Laboratorio` contiene el grupo correspondiente a los servidores web.

### 5.2 Web Deploy

Archivo: `playbooks/web_deploy.yml`

Este playbook ejecuta el despliegue utilizando el role `web_deploy`.

El role realiza las siguientes operaciones:

1. Instala Nginx si es necesario.
2. Crea el directorio de backups.
3. Crea el directorio web.
4. Comprueba si existe una página anterior.
5. Genera una nueva página mediante una plantilla Jinja2.
6. Compara la página nueva con la existente.
7. Crea un backup de la versión anterior cuando corresponde.
8. Instala la nueva página.
9. Valida la configuración de Nginx.
10. Permite provocar un fallo controlado para probar el rollback.
11. Garantiza que Nginx queda habilitado y arrancado.

### 5.3 Web Validate

Archivo: `playbooks/web_validate.yml`

Realiza las comprobaciones posteriores al despliegue:

- Nginx está activo.
- Existe `/var/www/html/index.html`.
- El archivo es regular.
- La página responde correctamente mediante HTTP.
- Contiene la versión esperada.
- Contiene el mensaje esperado.

Resultado de ejemplo:

```text
srv-web-01: Nginx activo, HTTP=200, versión=1.0, validación correcta
srv-web-02: Nginx activo, HTTP=200, versión=1.0, validación correcta
```

### 5.4 Web Rollback

Archivo: `playbooks/web_rollback.yml`

El rollback permite restaurar la página anterior almacenada en:

```text
/var/backups/awx-web/index.html.backup
```

El proceso:

1. Comprueba que existe el backup.
2. Restaura el archivo anterior.
3. Valida la configuración de Nginx mediante `nginx -t`.
4. Garantiza que Nginx está activo.
5. Reinicia Nginx cuando corresponde.
6. Muestra el resultado de la restauración.

El playbook utiliza `serial: 1` para realizar el rollback de forma controlada, servidor por servidor.

## 6. Variables del role

Las variables por defecto se encuentran en `roles/web_deploy/defaults/main.yml`.

```yaml
---
web_version: "1.0"
web_message: "Página desplegada mediante AWX"
web_document_root: /var/www/html
web_index_file: /var/www/html/index.html
web_backup_path: /var/backups/awx-web
web_force_failure: false
```

| Variable | Función |
|---|---|
| `web_version` | Versión que se despliega. |
| `web_message` | Mensaje que aparece en la página. |
| `web_document_root` | Directorio raíz de la web. |
| `web_index_file` | Archivo HTML principal. |
| `web_backup_path` | Ubicación del backup. |
| `web_force_failure` | Permite provocar un fallo controlado. |

Los valores de `defaults/main.yml` actúan como valores predeterminados y pueden ser sobrescritos desde AWX.

## 7. Plantilla web

La página se genera mediante `roles/web_deploy/templates/index.html.j2`.

La plantilla utiliza variables de Ansible, entre ellas:

```jinja2
{{ inventory_hostname }}
{{ ansible_hostname }}
{{ web_version }}
{{ web_message }}
{{ ansible_date_time.iso8601 }}
```

Ejemplo:

```text
Servidor: srv-web-01
Hostname: srv-web-01
Versión desplegada: 3.0
Mensaje: Despliegue 3.0 mediante AWX
```

## 8. Survey de AWX

El Workflow dispone de una encuesta para solicitar los parámetros del despliegue:

| Pregunta | Variable | Tipo | Valor predeterminado |
|---|---|---|---|
| Versión a desplegar | `web_version` | Texto | `1.0` |
| Mensaje de la página | `web_message` | Texto | `Primera versión desplegada mediante AWX` |
| Forzar fallo controlado | `web_force_failure` | Selección múltiple | `false` |

Para `web_force_failure`, las opciones disponibles son `false` y `true`. Esto permite ejecutar despliegues normales y pruebas de recuperación.

## 9. Configuración del Workflow

El Workflow utilizado es `Despliegue Web Controlado - Laboratorio`.

El flujo principal es:

```text
Web - Health Check
        │
        ▼
Web - Deploy
        │
        ▼
Web - Validate
        │
        ├── Éxito → Aprobación despliegue → Correcto
        │
        └── Error → Web - Rollback
```

También se ha configurado un camino de fallo controlado desde el despliegue hacia el rollback.

## 10. Aprobación manual

Después de una validación correcta, el Workflow llega al nodo `Aprobación despliegue` y queda esperando una decisión humana.

La aprobación se realiza desde:

```text
AWX → Aprobaciones del flujo de trabajo
```

Una vez aprobada, el Workflow continúa y finaliza correctamente.

## 11. Prueba 1: despliegue normal

Valores utilizados:

```text
Versión: 3.0
Mensaje: Despliegue 3.0 mediante AWX
Forzar fallo: false
```

Resultado:

```text
Health Check       Correcto
        ↓
Web - Deploy       Correcto
        ↓
Web - Validate     Correcto
        ↓
Aprobación         Aprobado
```

La prueba demostró que el flujo normal de despliegue funciona correctamente.

## 12. Comprobación del despliegue

Después del despliegue se comprobó la página mediante:

```bash
curl -s http://localhost:8081 | grep -E -A2 "Versión desplegada|Mensaje:"
curl -s http://localhost:8082 | grep -E -A2 "Versión desplegada|Mensaje:"
```

Resultados:

```text
srv-web-01
Versión desplegada: 3.0
Mensaje: Despliegue 3.0 mediante AWX

srv-web-02
Versión desplegada: 3.0
Mensaje: Despliegue 3.0 mediante AWX
```

También se comprobó que ambos servidores respondían mediante HTTP con `HTTP/1.1 200 OK`.

## 13. Prueba 2: fallo controlado y rollback

Valores utilizados:

```text
Versión: 4.0
Mensaje: Prueba de rollback
Forzar fallo: true
```

El objetivo era provocar deliberadamente un error durante el despliegue.

Flujo esperado:

```text
Health Check
     ↓
Web - Deploy
     ↓
Fallo controlado
     ↓
Web - Rollback
```

La prueba permitió comprobar que el Workflow detecta el fallo y ejecuta el rollback.

## 14. Resultado del rollback

Antes de la prueba, los servidores tenían desplegada la versión `3.0`, con el mensaje `Despliegue 3.0 mediante AWX`.

El despliegue de `4.0` provocó el fallo controlado. Posteriormente se ejecutó automáticamente el rollback.

Resultado final:

```text
srv-web-01 → versión 3.0
srv-web-02 → versión 3.0
```

La comprobación se realizó mediante:

```bash
curl -s http://localhost:8081 | grep -E -A2 "Versión desplegada|Mensaje:"
curl -s http://localhost:8082 | grep -E -A2 "Versión desplegada|Mensaje:"
```

En ambos servidores se obtuvo:

```text
Versión desplegada: 3.0
Mensaje: Despliegue 3.0 mediante AWX
```

Esto demuestra que la versión anterior fue restaurada correctamente.

## 15. Backups

Los backups se almacenan en:

```text
/var/backups/awx-web/
```

En el entorno de laboratorio se comprobó su existencia mediante:

```bash
docker exec srv-web-01 ls -lh /var/backups/awx-web/
docker exec srv-web-02 ls -lh /var/backups/awx-web/
```

El archivo de backup es:

```text
index.html.backup
```

Este backup permitió restaurar la versión `3.0` después del fallo controlado de la versión `4.0`.

## 16. Validación de los playbooks

Se realizó una comprobación de sintaxis:

```bash
for f in \\
  playbooks/web_health_check.yml \\
  playbooks/web_deploy.yml \\
  playbooks/web_validate.yml \\
  playbooks/web_rollback.yml
do
  echo "===== $f ====="
  ansible-playbook -i inventories/laboratorio/hosts.yml \\
    --syntax-check "$f" || exit 1
done
```

Los cuatro playbooks superaron correctamente la comprobación de sintaxis.

## 17. Validación del repositorio Git

También se comprobó el formato del repositorio mediante:

```bash
git diff --check
```

El resultado no mostró errores.

Commits asociados a la validación y configuración de AWX:

```text
942bf14 Completar validación y rollback web
04acedf Ajustar grupo Origenes para AWX
```

> Comprueba que estos identificadores de commit existen realmente en el repositorio antes de publicar esta memoria.

El repositorio quedó sincronizado con GitHub.

## 18. Estado final del proyecto

Al finalizar las pruebas:

```text
srv-web-01                 correcto
srv-web-02                 correcto
```

| Componente | Estado |
|---|---|
| Web - Health Check | ✅ Correcto |
| Web - Deploy | ✅ Correcto |
| Web - Validate | ✅ Correcto |
| Aprobación manual | ✅ Correcta |
| Web - Rollback | ✅ Correcto |
| Survey del Workflow | ✅ Correcto |
| Backups | ✅ Correctos |
| Validación HTTP | ✅ Correcta |
| Control de versiones Git | ✅ Correcto |

## 19. Flujos finales demostrados

### Flujo normal

```text
Health Check
     │ Éxito
     ▼
Web - Deploy
     │ Éxito
     ▼
Web - Validate
     │ Éxito
     ▼
Aprobación
     │ Aprobado
     ▼
CORRECTO
```

### Flujo de recuperación

```text
Health Check
     │ Éxito
     ▼
Web - Deploy
     │ Error
     ▼
Web - Rollback
     ▼
VERSIÓN ANTERIOR RESTAURADA
```

## 20. Conclusiones

El proyecto ha permitido implementar y validar un sistema de despliegue web automatizado utilizando Ansible y AWX.

Se ha conseguido:

- Automatizar el despliegue de una página web.
- Gestionar dos servidores web mediante un inventario común.
- Utilizar roles y plantillas Jinja2.
- Parametrizar los despliegues mediante Surveys de AWX.
- Validar automáticamente el resultado del despliegue.
- Incorporar una aprobación manual.
- Crear backups de las versiones anteriores.
- Ejecutar un rollback automático ante errores.
- Simular fallos controlados para comprobar el mecanismo de recuperación.
- Verificar que la versión anterior se restaura correctamente.
- Mantener los playbooks bajo control de versiones mediante Git.

Las dos pruebas principales fueron satisfactorias:

```text
DESPLIEGUE NORMAL
3.0 → Deploy → Validate → Aprobación → Correcto

FALLO CONTROLADO
4.0 → Deploy → Error → Rollback → 3.0 restaurado
```

