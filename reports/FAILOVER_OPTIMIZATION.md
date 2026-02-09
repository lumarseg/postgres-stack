# Optimización de Tiempos de Failover

Este documento describe los parámetros que afectan el tiempo de failover en el cluster PostgreSQL HA y cómo optimizarlos.

## Componentes del Failover

El tiempo total de failover es la suma de:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         TIEMPO TOTAL DE FAILOVER                            │
├─────────────────────┬─────────────────────┬─────────────────────────────────┤
│  Detección de fallo │ Elección nuevo líder│ Redirección de tráfico          │
│      (Patroni)      │     (Patroni)       │   (HAProxy + Keepalived)        │
└─────────────────────┴─────────────────────┴─────────────────────────────────┘
```

## 1. Patroni - Detección y Elección

### Parámetros clave

| Parámetro | Descripción | Default | Optimizado |
|-----------|-------------|---------|------------|
| `ttl` | Tiempo máximo sin heartbeat antes de considerar al líder muerto | 30s | 15s |
| `loop_wait` | Intervalo entre verificaciones del estado del cluster | 10s | 5s |
| `retry_timeout` | Timeout para operaciones de PostgreSQL y DCS | 10s | 5s |

### Cálculo del tiempo de detección

```
Tiempo máximo de detección = ttl
Tiempo típico de detección = ttl / 2 + loop_wait
```

| Configuración | Tiempo máximo | Tiempo típico |
|---------------|---------------|---------------|
| Default (ttl=30, loop=10) | 30s | 25s |
| Optimizada (ttl=15, loop=5) | 15s | 12.5s |

### Configuración en docker-compose.yml.j2

```yaml
SPILO_CONFIGURATION: |
  bootstrap:
    dcs:
      ttl: 15              # Reducido de 30s
      loop_wait: 5         # Reducido de 10s
      retry_timeout: 5     # Reducido de 10s
```

### Modificar en caliente (sin redeploy)

```bash
# Ver configuración actual
docker exec spilo patronictl show-config

# Modificar parámetros
docker exec spilo patronictl edit-config --force \
  -p 'ttl=15' \
  -p 'loop_wait=5' \
  -p 'retry_timeout=5'
```

## 2. HAProxy - Detección del Líder

### Parámetros clave

| Parámetro | Descripción | Default | Optimizado |
|-----------|-------------|---------|------------|
| `inter` | Intervalo entre health checks | 3s | 1s |
| `fall` | Número de fallos para marcar servidor caído | 3 | 2 |
| `rise` | Número de éxitos para marcar servidor disponible | 2 | 1 |

### Cálculo del tiempo de detección

```
Tiempo de detección = inter × fall
```

| Configuración | Tiempo de detección |
|---------------|---------------------|
| Default (inter=3s, fall=3) | 9s |
| Optimizada (inter=1s, fall=2) | 2s |

### Configuración en haproxy.cfg.j2

```
listen postgres
    bind *:5000
    option httpchk GET /primary
    http-check expect status 200
    default-server inter 1s fall 2 rise 1 on-marked-down shutdown-sessions
```

### Opciones adicionales

| Opción | Efecto |
|--------|--------|
| `on-marked-down shutdown-sessions` | Cierra conexiones inmediatamente cuando el servidor cae |
| `fastinter` | Intervalo más rápido durante transiciones |

Ejemplo con fastinter:
```
default-server inter 1s fastinter 500ms fall 2 rise 1
```

## 3. Keepalived - Failover de VIP

### Parámetros clave

| Parámetro | Descripción | Valor actual |
|-----------|-------------|--------------|
| `advert_int` | Intervalo de anuncios VRRP | 1s |
| `priority` | Prioridad del nodo (mayor = preferido) | 100/99/98 |

### Cálculo del tiempo de failover VIP

```
Tiempo de failover = (advert_int × 3) + skew_time
Tiempo típico = 3-4 segundos
```

### Configuración en keepalived.conf.j2

```
vrrp_instance VI_1 {
    state {{ keepalived_state }}
    interface {{ keepalived_interface }}
    virtual_router_id {{ keepalived_vrrp_id }}
    priority {{ node_priority }}
    advert_int 1

    authentication {
        auth_type PASS
        auth_pass {{ keepalived_auth_pass }}
    }

    virtual_ipaddress {
        {{ vip }}/24
    }
}
```

## Resumen de Tiempos

### Antes de optimización

| Componente | Tiempo |
|------------|--------|
| Patroni (detección + elección) | ~30s |
| HAProxy (detección) | ~9s |
| Keepalived (VIP) | ~3s |
| **Total máximo** | **~40s** |

### Después de optimización

| Componente | Tiempo |
|------------|--------|
| Patroni (detección + elección) | ~15s |
| HAProxy (detección) | ~2s |
| Keepalived (VIP) | ~3s |
| **Total máximo** | **~20s** |

## Configuración Agresiva (no recomendada para producción)

Para entornos donde la latencia de red es muy baja y estable:

```yaml
# Patroni
ttl: 10
loop_wait: 3
retry_timeout: 3

# HAProxy
default-server inter 500ms fall 2 rise 1
```

**Tiempo total:** ~10-12 segundos

### Riesgos de configuración agresiva

1. **Falsos positivos**: Picos de latencia pueden causar failovers innecesarios
2. **Split-brain**: Particiones de red pueden causar múltiples líderes
3. **Carga en etcd**: Más verificaciones = más carga en el DCS

## Trade-offs

```
         RÁPIDO                              ESTABLE
           ◄──────────────────────────────────────►

   ttl=10, loop=3                        ttl=30, loop=10

   + Failover rápido                     + Tolerante a glitches
   + Menor downtime                      + Menos falsos positivos
   - Sensible a latencia                 - Mayor downtime
   - Posibles falsos positivos           - Más tolerante a red
```

## Monitoreo de Failovers

### Ver historial de timelines

```bash
docker exec spilo patronictl list
```

La columna `TL` (Timeline) incrementa con cada failover.

### Logs de Patroni

```bash
docker logs spilo 2>&1 | grep -E "promoted|demoted|failover"
```

### Métricas de HAProxy

```bash
curl http://192.168.37.30:7000/stats
```

## Recomendaciones por Entorno

| Entorno | ttl | loop_wait | HAProxy inter | Tiempo total |
|---------|-----|-----------|---------------|--------------|
| Desarrollo | 10s | 3s | 500ms | ~12s |
| Testing | 15s | 5s | 1s | ~18s |
| Producción | 20s | 5s | 2s | ~25s |
| Crítico (con red estable) | 15s | 5s | 1s | ~18s |

## Aplicar Cambios

### Opción 1: Redeploy completo

```bash
ansible-playbook playbooks/deploy.yml
```

### Opción 2: Actualizar en caliente

```bash
# Patroni (se propaga a todos los nodos)
docker exec spilo patronictl edit-config --force \
  -p 'ttl=15' -p 'loop_wait=5' -p 'retry_timeout=5'

# HAProxy (en cada nodo)
ansible-playbook playbooks/deploy.yml --tags haproxy
```

## Referencias

- [Patroni Configuration](https://patroni.readthedocs.io/en/latest/SETTINGS.html)
- [HAProxy Configuration](https://www.haproxy.com/documentation/haproxy-configuration-manual/)
- [Keepalived Configuration](https://keepalived.readthedocs.io/en/latest/configuration_synopsis.html)
