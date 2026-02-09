# Prompt para Infograma de Arquitectura

Usado para generar la imagen `images/postgres-stack.png` con ChatGPT/DALL-E.

---

Crea un infograma técnico profesional para un cluster de alta disponibilidad PostgreSQL con el título "PostgreSQL HA Cluster (Docker Compose + Patroni + Keepalived)".

## Estructura del infograma

### Fila superior: 4 columnas con los componentes principales

**Columna 1: VIP + KEEPALIVED**
- VIP: 192.168.37.30
- VRRP Protocol
- Priority: vm05(100) > vm06(99) > vm07(98)
- Failover: ~3 segundos

**Columna 2: HAPROXY**
- TCP Load Balancer
- Health Checks via Patroni API (/primary)
- Port 5000 (PostgreSQL)
- Stats Port: 7000
- Deteccion: 2 segundos (inter=1s, fall=2)

**Columna 3: PATRONI + SPILO**
- Automatic Failover
- Synchronous Replication
- Leader Election via etcd
- Failover: ~15 segundos (ttl=15, loop=5)

**Columna 4: ETCD**
- Distributed Consensus (Raft)
- Cluster State Store
- Ports: 2379 (client) / 2380 (peer)

### Fila central: Flujo de conexion

Mostrar el flujo de cliente:
```
Client → VIP 192.168.37.30:5000 → Keepalived MASTER → HAProxy → PostgreSQL Leader (Spilo)
```

Flujo de replicacion y consenso:
```
Leader ──streaming──► Sync Standby ──streaming──► Replica
         │                  │                        │
         └────── etcd (Raft consensus) ◄─────────────┘
```

### Fila inferior: 3 nodos identicos (network_mode: host)

| Nodo | IP | Patroni | Role | Priority |
|------|-----|---------|------|----------|
| vm05 | 192.168.37.15 | db01 | Leader | 100 |
| vm06 | 192.168.37.16 | db02 | Sync Standby | 99 |
| vm07 | 192.168.37.17 | db03 | Replica | 98 |

Stack de cada nodo (de arriba a abajo):
```
┌─────────────────────┐
│     Keepalived      │  ← VRRP failover
├─────────────────────┤
│      HAProxy        │  ← Load balancing
├─────────────────────┤
│    Spilo/Patroni    │  ← PostgreSQL + HA
├─────────────────────┤
│        etcd         │  ← Consensus
└─────────────────────┘
     network_mode: host
```

### Panel lateral: Tiempos de Failover

```
┌─────────────────────────────────────┐
│       FAILOVER TIMELINE             │
├─────────────────────────────────────┤
│                                     │
│  0s    Fallo del Leader             │
│  │                                  │
│  2s    HAProxy detecta (inter×fall) │
│  │                                  │
│  3s    Keepalived mueve VIP         │
│  │                                  │
│ 15s    Patroni elige nuevo Leader   │
│  │                                  │
│ 17s    HAProxy redirige trafico     │
│  │                                  │
│ ~20s   SERVICIO RESTAURADO          │
│                                     │
└─────────────────────────────────────┘
```

### Panel lateral: Parametros Optimizados

| Componente | Parametro | Valor |
|------------|-----------|-------|
| Patroni | ttl | 15s |
| Patroni | loop_wait | 5s |
| Patroni | retry_timeout | 5s |
| HAProxy | inter | 1s |
| HAProxy | fall | 2 |
| HAProxy | rise | 1 |
| Keepalived | advert_int | 1s |

### Pie del infograma: Red y Puertos

| Puerto | Servicio | Descripcion |
|--------|----------|-------------|
| 5000 | PostgreSQL | Conexion via HAProxy |
| 5432 | PostgreSQL | Conexion directa |
| 7000 | HAProxy | Dashboard de estadisticas |
| 8008 | Patroni | REST API (health checks) |
| 2379 | etcd | Client API |
| 2380 | etcd | Peer communication |

Red: 192.168.37.0/24 (host network, sin overlay)

## Estilo visual

- Colores profesionales:
  - Azul PostgreSQL (#336791) para Spilo/Patroni
  - Naranja (#E95420) para etcd
  - Verde (#4CAF50) para HAProxy
  - Rojo (#F44336) para Keepalived/VIP
  - Gris (#607D8B) para infraestructura
- Iconos tecnicos:
  - Elefante para PostgreSQL
  - Llave para etcd
  - Balanza para HAProxy
  - Corazon para Keepalived
- Flechas indicando:
  - Flujo de trafico (solidas)
  - Replicacion streaming (punteadas)
  - VRRP/Raft consensus (dobles)
- Destacar visualmente:
  - Leader con borde dorado
  - Sync Standby con borde azul
  - Replica con borde gris
- Indicar "network_mode: host" en cada nodo
- Mostrar "Synchronous Replication" entre Leader y Sync Standby
- Diseño limpio y minimalista
- Formato horizontal/landscape
- Incluir leyenda de colores y simbolos
