# Incidencias del laboratorio HA PostgreSQL

Registro de los fallos reales encontrados durante el despliegue del clúster (Patroni + etcd + HAProxy + Keepalived), con causa raíz y solución aplicada. Complementa a [`HA_POSTGRESQL.md`](./HA_POSTGRESQL.md).

---

## 1. `pg_basebackup` falla al incorporar réplicas — `password authentication failed for user "replicator"`

**Fase:** inicialización del clúster / incorporación de réplicas (paso 4-5).

**Problema**

Los nodos réplica (`postgresql2` y `postgresql3`) no lograban unirse al clúster. `pg_basebackup` fallaba repetidamente con:

```
password authentication failed for user "replicator"
```

**Causa raíz**

La contraseña era correcta y coincidía en todos los nodos (`.pgpass` y `patroni.yml`), pero el **hash almacenado en el primario para el rol `replicator` estaba cifrado en `md5`**, un método de autenticación obsoleto e incompatible con lo que esperaba el resto de la configuración.

**Solución**

1. Cambiar el método de cifrado de contraseñas a `scram-sha-256`:
   ```sql
   ALTER SYSTEM SET password_encryption = 'scram-sha-256';
   SELECT pg_reload_conf();
   ```
2. Regenerar el hash del rol `replicator` con el nuevo cifrado (al re-ejecutar `ALTER ROLE` con `password_encryption` ya en `scram-sha-256`, el hash se recalcula automáticamente):
   ```sql
   ALTER ROLE replicator WITH PASSWORD '<REPLICATOR_PASSWORD>';
   ```
3. Reiniciar Patroni en los nodos réplica (`hadoop-worker2` y `hadoop-worker3`).

**Resultado:** `postgresql2` y `postgresql3` sincronizados correctamente como réplicas activas, siguiendo a `postgresql1` sin errores.

**Recomendación:** actualizar `pg_hba.conf` en todos los nodos a `scram-sha-256` de forma consistente, y auditar que los hashes de roles críticos (`replicator`, `rewind_user`, `postgres`) estén siempre en ese formato desde el principio, no solo tras un fallo.

---

## 2. HAProxy no arranca — `Permission denied` al enlazar los puertos

**Fase:** instalación y arranque de HAProxy.

**Problema**

Con una configuración de `haproxy.cfg` correcta, el servicio no arrancaba y mostraba:

```
Binding for frontend postgres_primary: cannot bind socket (Permission denied) for [0.0.0.0:5432]
Binding for proxy stats: cannot bind socket (Permission denied) for [0.0.0.0:7000]
```

**Causa raíz**

El problema no era HAProxy, sino **SELinux**. Estaba en modo `Enforcing` (comprobado con `getenforce`), y bloqueaba a HAProxy para escuchar en los puertos 5432 (PostgreSQL) y 7000 (estadísticas).

**Solución**

1. Instalar las herramientas de administración de SELinux (incluye `semanage`):
   ```bash
   dnf install -y policycoreutils-python-utils
   ```
2. En lugar de desactivar SELinux, autorizar explícitamente a HAProxy a realizar conexiones de red en cualquier puerto:
   ```bash
   semanage boolean --modify --on haproxy_connect_any
   ```
   Este cambio es persistente entre reinicios.
3. Reiniciar el servicio:
   ```bash
   systemctl restart haproxy
   ```

**Verificación**

```bash
systemctl status haproxy
ss -ltnp | grep -E ':5432|:7000'
```

**Resultado:** HAProxy abre correctamente ambos puertos y enruta conexiones con normalidad.

**Recomendación:** mantener SELinux en `Enforcing` (no desactivarlo) y usar siempre los booleanos/políticas específicas (`semanage boolean`, `semanage fcontext`) en vez de `setenforce 0` — es la diferencia entre resolver el síntoma y desactivar la seguridad del sistema.

---

## 3. Keepalived no ejecuta el script de comprobación — `Script check_haproxy now returning 126`

**Fase:** configuración de Keepalived / comprobación de salud de HAProxy.

**Problema**

Keepalived ejecuta periódicamente `/usr/local/bin/check_haproxy.sh` para comprobar si HAProxy está funcionando. Aunque el script tenía permisos de ejecución correctos (`chmod 750`), aparecía:

```
Script `check_haproxy` now returning 126
```

El código de salida 126 significa: *el archivo existe, pero el sistema no permite ejecutarlo*.

**Causa raíz**

De nuevo, **SELinux** — no un problema de permisos Linux ni del script en sí. SELinux no solo controla *quién* puede ejecutar un archivo, sino *qué procesos* pueden ejecutar *qué tipo* de archivos. Keepalived se ejecuta en un contexto SELinux que, por seguridad, no permite ejecutar scripts arbitrarios ubicados en `/usr/local/bin`.

**Solución**

1. Crear una regla de contexto SELinux para el script, asignándole el tipo que Keepalived sí tiene permitido ejecutar:
   ```bash
   semanage fcontext -a -t keepalived_unconfined_script_exec_t '/usr/local/bin/check_haproxy.sh'
   ```
2. Aplicar esa regla al archivo (`semanage fcontext` solo la registra; `restorecon` la aplica y cambia la etiqueta real del archivo):
   ```bash
   restorecon -v /usr/local/bin/check_haproxy.sh
   ```
   Esto cambia la etiqueta de seguridad del archivo:
   - Antes: `unconfined_u:object_r:usr_t:s0`
   - Después: `unconfined_u:object_r:keepalived_unconfined_script_exec_t:s0`

**Resultado:** Keepalived puede ejecutar el script de comprobación sin que SELinux lo bloquee, y el failover automático vuelve a funcionar según lo esperado.

**Recomendación:** cualquier script que un servicio systemd (o Keepalived) necesite ejecutar fuera de su contexto SELinux por defecto necesita su propia regla de `fcontext` — es un patrón que se repite con cualquier script personalizado en `/usr/local/bin`, no solo con `check_haproxy.sh`.

---

## Patrón común

Dos de los tres incidentes (#2 y #3) tenían el mismo origen real: **SELinux en modo `Enforcing`**, no un fallo de la herramienta que aparentaba estar rota. El primero (#1) fue un desajuste de método de cifrado de contraseñas entre lo esperado por el clúster y lo realmente almacenado.

La lección operativa: cuando un servicio con configuración aparentemente correcta falla al arrancar o ejecutar algo en RHEL/derivados, `getenforce` y los logs de auditoría de SELinux (`ausearch -m avc -ts recent` o `journalctl -xe`) deberían ser de las primeras cosas a comprobar, no de las últimas.
