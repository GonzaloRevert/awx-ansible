# AWX Ansible Backups

Proyecto de automatización de copias de seguridad, validación y pruebas de restauración utilizando **Ansible** y **AWX**.

---

## 1. Objetivo

El objetivo del proyecto es automatizar el proceso de copia de seguridad de varios servidores hacia un servidor central de almacenamiento.

El sistema permite:

- Crear backups de los servidores origen.
- Comprimir los datos en archivos `.tar.gz`.
- Transferir los backups al servidor destino.
- Validar que los backups existen.
- Realizar una prueba real de restauración.
- Ejecutar todo el proceso mediante un Workflow de AWX.
- Programar la ejecución automática diaria.

La restauración se realiza en un directorio temporal para evitar modificar los datos originales.

---

## 2. Arquitectura

El laboratorio está compuesto por varios servidores origen y un servidor destino.

### Servidores origen

Grupo de Ansible:

```text
backup_sources


