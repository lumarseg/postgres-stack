# Plan de Pruebas - Cluster PostgreSQL HA

## Arquitectura

| VM | IP | Rol | Componentes | Prioridad Keepalived |
|----|-----|-----|-------------|----------------------|
| vm06 | 192.168.37.16 | PostgreSQL + HA | Spilo, etcd, HAProxy, Keepalived | 100 (máxima) |
| vm07 | 192.168.37.17 | PostgreSQL + HA | Spilo, etcd, HAProxy, Keepalived | 99 |
| vm08 | 192.168.37.18 | PostgreSQL + HA | Spilo, etcd, HAProxy, Keepalived | 98 (mínima) |
| **VIP** | **192.168.37.15** | IP Flotante | Gestionada por Keepalived | - |

### Diagrama

```
                    ┌─────────────────────┐
                    │   VIP: 192.168.37.15│
                    │   (Keepalived)      │
                    └──────────┬──────────┘
                               │
           ┌───────────────────┼───────────────────┐
           │                   │                   │
           ▼                   ▼                   ▼
    ┌─────────────┐     ┌─────────────┐     ┌─────────────┐
    │    vm06     │     │    vm07     │     │    vm08     │
    │ HAProxy     │     │ HAProxy     │     │ HAProxy     │
    │ Keepalived  │     │ Keepalived  │     │ Keepalived  │
    │ PostgreSQL  │     │ PostgreSQL  │     │ PostgreSQL  │
    │ etcd        │     │ etcd        │     │ etcd        │
    └─────────────┘     └─────────────┘     └─────────────┘
```

## Credenciales

```
Host: 192.168.37.15 (VIP)
Port: 5000
User: postgres
Password: cjtz8CHhVxF70hbTjpBu
```

---

## Test 1: Verificar estado del cluster

```bash
ssh debian@192.168.37.16 "sudo docker exec spilo patronictl list"
```

**Resultado esperado:** Un nodo como Leader, los otros como Replica streaming

---

## Test 2: Conectividad a través de VIP

```bash
PGPASSWORD=cjtz8CHhVxF70hbTjpBu psql -h 192.168.37.15 -p 5000 -U postgres -c "SELECT inet_server_addr() AS servidor;"
```

**Resultado esperado:** Muestra IP del líder actual

---

## Test 3: Crear datos de prueba

```bash
PGPASSWORD=cjtz8CHhVxF70hbTjpBu psql -h 192.168.37.15 -p 5000 -U postgres -c "
CREATE TABLE IF NOT EXISTS test_ha (
    id SERIAL PRIMARY KEY,
    mensaje TEXT,
    created_at TIMESTAMP DEFAULT NOW()
);
INSERT INTO test_ha (mensaje) VALUES ('Prueba antes de failover');
SELECT * FROM test_ha;"
```

**Resultado esperado:** Tabla creada y datos insertados

---

## Test 4: Verificar replicación

```bash
# Consultar directamente en vm07 (réplica)
PGPASSWORD=cjtz8CHhVxF70hbTjpBu psql -h 192.168.37.17 -U postgres -c "SELECT * FROM test_ha;"

# Consultar directamente en vm08 (réplica)
PGPASSWORD=cjtz8CHhVxF70hbTjpBu psql -h 192.168.37.18 -U postgres -c "SELECT * FROM test_ha;"
```

**Resultado esperado:** Los datos aparecen en ambas réplicas

---

## Test 5: Failover automático (detener líder)

```bash
# Obtener líder actual
ssh debian@192.168.37.16 "sudo docker exec spilo patronictl list"

# Detener el contenedor spilo en el líder (ejemplo: vm06)
ssh debian@192.168.37.16 "sudo docker stop spilo"

# Esperar 15-30 segundos y verificar nuevo líder
sleep 20
ssh debian@192.168.37.17 "sudo docker exec spilo patronictl list"
```

**Resultado esperado:** Otro nodo se convierte en nuevo Leader

---

## Test 6: Verificar HAProxy redirige al nuevo líder

```bash
PGPASSWORD=cjtz8CHhVxF70hbTjpBu psql -h 192.168.37.15 -p 5000 -U postgres -c "SELECT inet_server_addr() AS servidor;"
```

**Resultado esperado:** Muestra IP del nuevo líder

---

## Test 7: Verificar datos después del failover

```bash
PGPASSWORD=cjtz8CHhVxF70hbTjpBu psql -h 192.168.37.15 -p 5000 -U postgres -c "SELECT * FROM test_ha;"
```

**Resultado esperado:** Los datos insertados antes del failover existen

---

## Test 8: Insertar datos con nuevo líder

```bash
PGPASSWORD=cjtz8CHhVxF70hbTjpBu psql -h 192.168.37.15 -p 5000 -U postgres -c "INSERT INTO test_ha (mensaje) VALUES ('Prueba después de failover'); SELECT * FROM test_ha;"
```

**Resultado esperado:** Nuevo registro insertado correctamente

---

## Test 9: Recuperar nodo caído

```bash
# Reiniciar spilo en el nodo caído (ejemplo: vm06)
ssh debian@192.168.37.16 "sudo docker start spilo"

# Esperar y verificar que se une como réplica
sleep 20
ssh debian@192.168.37.16 "sudo docker exec spilo patronictl list"
```

**Resultado esperado:** El nodo aparece como Replica streaming

---

## Test 10: Verificar sincronización del nodo recuperado

```bash
# Los datos insertados mientras el nodo estaba caído deben existir
PGPASSWORD=cjtz8CHhVxF70hbTjpBu psql -h 192.168.37.16 -U postgres -c "SELECT * FROM test_ha;"
```

**Resultado esperado:** Todos los registros (antes y después del failover) visibles

---

## Test 11: Switchover manual

```bash
# Obtener líder actual
ssh debian@192.168.37.16 "sudo docker exec spilo patronictl list"

# Switchover a vm06 (reemplazar CURRENT_LEADER con el líder actual)
ssh debian@192.168.37.16 "sudo docker exec spilo patronictl switchover postgres-cluster --leader CURRENT_LEADER --candidate vm06 --force"
```

**Resultado esperado:** vm06 se convierte en Leader

---

## Test 12: Prueba final de conexión

```bash
# Verificar cluster
ssh debian@192.168.37.16 "sudo docker exec spilo patronictl list"

# Verificar conexión via VIP
PGPASSWORD=cjtz8CHhVxF70hbTjpBu psql -h 192.168.37.15 -p 5000 -U postgres -c "SELECT inet_server_addr() AS servidor;"
```

**Resultado esperado:** Leader correcto, HAProxy redirige correctamente

---

## Test 13: Limpiar datos de prueba (opcional)

```bash
PGPASSWORD=cjtz8CHhVxF70hbTjpBu psql -h 192.168.37.15 -p 5000 -U postgres -c "DROP TABLE test_ha;"
```

---

## Test 14: Failover de VIP (Keepalived)

```bash
# Ver quién tiene la VIP
for ip in 16 17 18; do echo "=== vm0$((ip-10)) ===" && ssh debian@192.168.37.$ip "ip a show eth0 | grep 192.168.37.15" 2>/dev/null; done

# Detener keepalived en el nodo con la VIP (ejemplo: vm06)
ssh debian@192.168.37.16 "sudo docker stop keepalived"

# Verificar que la VIP se movió a otro nodo
sleep 5
for ip in 16 17 18; do echo "=== vm0$((ip-10)) ===" && ssh debian@192.168.37.$ip "ip a show eth0 | grep 192.168.37.15" 2>/dev/null; done

# La conexión debe seguir funcionando
PGPASSWORD=cjtz8CHhVxF70hbTjpBu psql -h 192.168.37.15 -p 5000 -U postgres -c "SELECT 1;"

# Restaurar keepalived
ssh debian@192.168.37.16 "sudo docker start keepalived"
```

**Resultado esperado:** La VIP se mueve a otro nodo y la conexión sigue funcionando

---

## Comandos útiles

### Ver estado del cluster PostgreSQL
```bash
ssh debian@192.168.37.16 "sudo docker exec spilo patronictl list"
```

### Ver logs de Patroni
```bash
ssh debian@192.168.37.16 "sudo docker logs spilo --tail 50"
```

### Ver estado de HAProxy
```bash
curl -s http://192.168.37.15:7000/
```

### Ver quién tiene la VIP
```bash
for ip in 16 17 18; do echo "=== 192.168.37.$ip ===" && ssh debian@192.168.37.$ip "ip a show eth0 | grep 192.168.37.15"; done
```

### Ver estado de etcd
```bash
ssh debian@192.168.37.16 "sudo docker exec etcd etcdctl endpoint health --endpoints=http://192.168.37.16:2379,http://192.168.37.17:2379,http://192.168.37.18:2379"
```

### Reiniciar servicios en una VM
```bash
ssh debian@192.168.37.16 "cd /opt/postgres-stack && sudo docker compose restart"
```

### Ver espacio en /var
```bash
for ip in 16 17 18; do echo "=== VM: 192.168.37.$ip ===" && ssh debian@192.168.37.$ip "df -h /var"; done
```

### Ver logs de Keepalived
```bash
ssh debian@192.168.37.16 "sudo docker logs keepalived --tail 20"
```
