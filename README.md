# HA PostgreSQL con Patroni, etcd y HAProxy

Laboratorio propio de despliegue de un clúster PostgreSQL de alta disponibilidad, usando:

- **PostgreSQL** (3 nodos)
- **etcd** como almacén de coordinación distribuido (5 miembros)
- **Patroni** para failover automático y gestión del clúster
- **HAProxy** + **Keepalived (VIP)** para el enrutamiento de conexiones al líder

## Contenido

- [`HA_POSTGRESQL.md`](./HA_POSTGRESQL.md) — documento completo: instalación paso a paso, configuración y validación de la alta disponibilidad (failover de PostgreSQL y de HAProxy), incluyendo un incidente real resuelto durante el despliegue.
- [`media/diagrams/arquitectura_ha_postgresql.png`](./media/diagrams/arquitectura_ha_postgresql.png) — diagrama de la arquitectura: jerarquía de componentes y flujo desde el cliente (vía VIP) hasta el líder de Patroni.

## Arquitectura

![Arquitectura HA PostgreSQL](./media/diagrams/arquitectura_ha_postgresql.png)

## Notas

Las contraseñas del documento están anonimizadas (`<POSTGRES_PASSWORD>`, `<REPLICATOR_PASSWORD>`, `<REWIND_PASSWORD>`); sustitúyelas por tus propios valores si reproduces el laboratorio.
