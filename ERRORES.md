# Guía de solución de errores habituales - AWX Ansible Backups

Esta guía recoge los problemas más habituales del proyecto y las comprobaciones recomendadas para resolverlos.

## 1. AWX no muestra un playbook nuevo

### Síntoma

El archivo existe localmente, pero no aparece en el selector **Playbook** de AWX.

### Comprobaciones

```bash
find playbooks -maxdepth 1 -type f -printf '%f\n' | sort
git ls-files playbooks/
git status
```

El playbook debe existir en el repositorio y estar incluido en un commit.

### Solución

```bash
git add playbooks/restore_test.yml
git commit -m "Añadir playbook de restauracion"
git push
```

Después, actualizar o sincronizar el proyecto en AWX:

```text
Proyectos
└── AWX Ansible Backups
    └── Actualizar / Sync
```

Volver a la plantilla y comprobar de nuevo el selector de playbooks.

## 2. AWX utiliza una versión antigua del proyecto

### Síntoma

El cambio existe en Git, pero AWX sigue ejecutando una versión anterior.

### Comprobaciones

```bash
git log -1 --oneline
git status
git remote -v
```

### Solución

```bash
git add .
git commit -m "Actualizar playbooks"
git push
```

Después, ejecutar manualmente **Update/Sync** en el proyecto de AWX antes de volver a lanzar la plantilla.

No se debe depender de rutas internas como `_19__...` dentro de AWX: el identificador puede cambiar. Es preferible actualizar el proyecto desde la interfaz o consultar la ruta real en el detalle del proyecto.

## 3. Error `couldn't resolve module/action`

### Síntoma

```text
ERROR! couldn't resolve module/action
```

### Causas habituales

- Nombre incorrecto del módulo.
- Colección no instalada en el Execution Environment.
- Diferencia entre la versión local de Ansible y la versión de AWX.
- Uso de una colección sin escribir su nombre completo.

### Comprobaciones

Revisar el módulo utilizado y el Execution Environment asignado a la plantilla. Si se utiliza una colección, indicar el nombre completo, por ejemplo:

```yaml
ansible.posix.synchronize:
```

Para comprimir datos también se puede utilizar una orden del sistema, siempre que esté justificada y validada:

```yaml
- name: Crear backup comprimido
  ansible.builtin.command:
    cmd: >-
      tar -czf
      /tmp/ansible-backup/{{ inventory_hostname }}-{{ ansible_date_time.date }}.tar.gz
      {{ item }}/
  loop: "{{ backup_paths }}"
  changed_when: true
```

Validar antes de subir cambios:

```bash
git diff --check
```

## 4. Warning `ANSIBLE_COLLECTIONS_PATHS`

### Síntoma

```text
[DEPRECATION WARNING]: ANSIBLE_COLLECTIONS_PATHS option
```

### Causa

Se está utilizando una variable de configuración antigua.

Buscarla con:

```bash
grep -R "ANSIBLE_COLLECTIONS_PATHS" -n .
```

Si la configuración lo requiere, utilizar el nombre recomendado por la versión de Ansible instalada. El warning no implica necesariamente que el trabajo haya fallado.

Comprobar siempre el resumen final del trabajo y el valor `failed=0`.

## 5. Error `Could not match supplied host pattern`

### Síntoma

```text
[WARNING]: Could not match supplied host pattern, ignoring: destino
```

### Causa

El playbook utiliza un grupo que no existe en el inventario o cuyo nombre no coincide exactamente.

Ejemplo:

```yaml
hosts: destino
```

El inventario debe contener exactamente el grupo `destino`.

### Comprobaciones

```bash
cat inventories/laboratorio/hosts.yml
ansible-inventory -i inventories/laboratorio/hosts.yml --graph
```

Recuerda que `Destino`, `destino` y `DESTINO` son nombres diferentes.

## 6. El playbook termina, pero no ejecuta tareas

### Síntoma

```text
skipping: no hosts matched
```

### Causas habituales

- El grupo no existe.
- La plantilla de AWX utiliza otro inventario.
- El filtro **Limit** excluye los hosts.
- El host está deshabilitado.

### Comprobaciones

```bash
ansible-inventory -i inventories/laboratorio/hosts.yml --graph
```

En AWX comprobar:

```text
Inventario: Laboratorio
Limit: vacío, salvo que sea necesario
Host habilitado: sí
```

Un trabajo puede terminar sin errores aunque una parte del playbook no haya encontrado hosts. Revisar siempre el `stdout`.

## 7. Variable indefinida

### Síntoma

```text
'backup_destination_path' is undefined
```

### Comprobaciones

Verificar la estructura:

```text
inventories/laboratorio/
├── hosts.yml
└── group_vars/
    └── all.yml
```

Buscar la variable:

```bash
grep -R "backup_destination_path\|backup_paths" -n inventories/
```

Ejemplo de configuración:

```yaml
backup_paths:
  - /datos_prueba

backup_destination_host: srv-destino-01
backup_destination_path: /backups
backup_retention_days: 7
backup_dry_run: true
```

Después de modificar variables:

```bash
git add inventories/
git commit -m "Corregir variables del inventario"
git push
```

Actualizar el proyecto en AWX antes de ejecutar la plantilla.

## 8. AWX no encuentra `group_vars`

### Síntoma

Las variables funcionan en local, pero aparecen como `undefined` en AWX.

### Causas habituales

- El archivo no está dentro del proyecto descargado por AWX.
- La estructura de inventario no es la esperada.
- El proyecto de AWX no se ha sincronizado.
- La plantilla utiliza otro inventario.

### Comprobaciones

```bash
find inventories -type f | sort
git ls-files inventories/
```

Debe aparecer:

```text
inventories/laboratorio/hosts.yml
inventories/laboratorio/group_vars/all.yml
```

Comprobar también que la plantilla utiliza el inventario correcto y que el proyecto se ha actualizado.

## 9. El backup se crea, pero no aparece en el destino

### Comprobaciones

En el servidor destino:

```bash
ls -lah /backups
find /backups -type f -name '*.tar.gz' -ls
```

Comprobar también:

- que `srv-destino-01` está encendido;
- que SSH está activo;
- que el usuario `ansible` tiene permisos;
- que existe `/backups`;
- que el destino configurado coincide con el inventario.

### Flujo recomendado

```text
Servidor origen
└── Crear backup
    └── Transferir mediante rsync/SSH
        └── srv-destino-01:/backups
            └── Validar archivo
```

La solución preferida es transferir desde el origen al destino mediante `ansible.posix.synchronize`. El uso de `fetch` y `copy` a través del nodo de control puede servir para una práctica inicial, pero hace que AWX transporte los archivos y no es la opción recomendada para backups grandes.

## 10. `fetch` falla

### Comprobaciones

En el servidor origen:

```bash
ls -lah /tmp/ansible-backup/
```

Comprobar que el nombre generado coincide exactamente con el nombre utilizado por `fetch`.

Un nombre válido puede ser:

```text
srv-origen-01-2026-08-28.tar.gz
```

### Causa habitual

La tarea crea el archivo en una ruta o con un nombre distinto del que posteriormente intenta descargar.

## 11. Error durante la restauración

### Comprobaciones

Buscar backups disponibles:

```bash
find /backups -type f -name '*.tar.gz' -ls
```

Comprobar que el archivo es válido:

```bash
tar -tzf /backups/archivo.tar.gz
```

Si el comando muestra el contenido, el archivo puede abrirse correctamente.

Cuando el archivo ya está en el servidor destino, la restauración debe utilizar `remote_src: true`:

```yaml
- name: Restaurar backup en directorio temporal
  ansible.builtin.unarchive:
    src: "{{ test_backup.path }}"
    dest: /tmp/restore-test
    remote_src: true
```

## 12. La restauración no encuentra archivos

### Causas habituales

- El backup está vacío.
- Solo se han guardado directorios.
- La ruta respaldada no contiene archivos.
- La extracción se ha realizado en una ruta distinta.

### Comprobaciones

```bash
tar -tzf /backups/archivo.tar.gz
find /tmp/restore-test -type f -ls
find /datos_prueba -type f -ls
```

Antes de generar el backup, comprobar que `/datos_prueba` contiene archivos de prueba.

## 13. Error de permisos

### Síntoma

```text
Permission denied
```

### Comprobaciones

```bash
ls -ld /datos_prueba
ls -ld /backups
id ansible
```

Si las tareas necesitan privilegios, utilizar `become: true` de forma controlada:

```yaml
- name: Ejecutar tareas con privilegios
  hosts: backup_sources
  become: true
```

Comprobar también que la credencial SSH de AWX utiliza el usuario correcto y que el usuario `ansible` tiene permisos sudo según corresponda.

## 14. AWX no puede conectarse por SSH

### Síntomas

```text
UNREACHABLE
Permission denied
Connection timed out
```

### Comprobaciones

```bash
docker ps -a
docker exec srv-origen-01 ps aux | grep '[s]shd'
docker exec srv-destino-01 ps aux | grep '[s]shd'
ssh -p 2201 ansible@192.168.50.10
ssh -p 2202 ansible@192.168.50.10
```

En AWX verificar:

- usuario SSH;
- clave privada;
- dirección `ansible_host`;
- puerto `ansible_port`;
- Credential asignada a la plantilla;
- estado del host;
- conectividad entre el Execution Environment y los contenedores.

## 15. Error de YAML

### Síntoma

Errores relacionados con `mapping`, `indentation` o `syntax`.

### Comprobaciones

```bash
git diff --check
ansible-playbook --syntax-check playbooks/restore_test.yml
```

Revisar especialmente:

- indentación con espacios;
- comillas;
- listas precedidas por `-`;
- dos puntos en valores;
- variables Jinja correctamente cerradas.

## 16. Git no permite hacer `push`

### Síntoma

Git solicita usuario y contraseña o rechaza la autenticación.

### Causa

GitHub no utiliza la contraseña normal de la cuenta para autenticar operaciones Git mediante HTTPS.

### Soluciones

Utilizar uno de estos mecanismos:

- Personal Access Token;
- clave SSH;
- Git Credential Manager.

No guardar tokens ni contraseñas en los playbooks o en este README.

## 17. Los cambios locales no aparecen en Git

### Comprobaciones

```bash
git status
git diff
git log -1 --oneline
git ls-files
```

Si aparece:

```text
nothing to commit, working tree clean
```

no hay cambios pendientes en el repositorio local.

## 18. El proyecto de AWX no se actualiza

### Síntoma

El repositorio tiene un commit nuevo, pero AWX continúa utilizando una revisión anterior.

### Solución

1. Confirmar que el cambio se ha subido:

   ```bash
   git log -1 --oneline
   git push
   ```

2. Abrir el proyecto en AWX.
3. Ejecutar **Actualizar / Sync**.
4. Esperar a que finalice correctamente.
5. Ejecutar de nuevo la plantilla.

Si el error continúa, revisar el historial de actualizaciones del proyecto y la credencial Git.

## 19. Procedimiento general de diagnóstico

Cuando falle un trabajo, seguir este orden:

1. Leer el error exacto en `stdout`.
2. Identificar el host y la tarea que han fallado.
3. Comprobar el inventario.
4. Comprobar las variables.
5. Comprobar la credencial SSH.
6. Probar la conectividad manualmente.
7. Ejecutar un `health_check`.
8. Validar la sintaxis del playbook.
9. Actualizar el proyecto en AWX.
10. Repetir la ejecución en modo seguro.

No eliminar datos ni ejecutar restauraciones sobre rutas reales hasta confirmar la causa del problema.
