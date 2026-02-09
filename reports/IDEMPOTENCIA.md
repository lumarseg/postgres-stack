# Idempotencia en Sistemas Distribuidos

## ¿Qué es Idempotencia?

**Idempotencia** = Una operación que puedes ejecutar múltiples veces y el resultado es el mismo que si la ejecutaras una sola vez.

### Ejemplo Simple

```sql
-- NO idempotente (cada ejecución agrega un registro)
INSERT INTO pedidos (producto) VALUES ('laptop');
-- Ejecutar 3 veces = 3 registros

-- SÍ idempotente (usa un ID único)
INSERT INTO pedidos (id, producto) VALUES ('uuid-123', 'laptop')
ON CONFLICT (id) DO NOTHING;
-- Ejecutar 3 veces = 1 registro
```

---

## ¿Por Qué es Importante en HA?

En un cluster de alta disponibilidad (HA), durante un failover pueden ocurrir situaciones ambiguas:

```
Cliente                          PostgreSQL
   |                                 |
   |-------- INSERT 'pedido' ------->|
   |                                 | ✓ COMMIT
   |<------ [CONEXIÓN MUERE] --------|
   |                                 |
   | "¿Falló? No sé..."              |
```

### El Problema

| Escenario | ¿Qué ocurre? | Resultado |
|-----------|--------------|-----------|
| INSERT committed antes del failover | Replicado a réplicas | ✅ Datos seguros |
| INSERT en vuelo cuando líder muere | No confirmado | ❌ PERDIDO |
| INSERT enviado a réplica (read-only) | Rechazado | ❌ PERDIDO |
| Conexión cerrada antes de respuesta | Ambiguo | ⚠️ Puede o no estar guardado |

### El Caso Ambiguo es el Más Peligroso

```
Sin idempotencia + Retry = DUPLICADOS

Cliente                          PostgreSQL
   |                                 |
   |-------- INSERT 'pedido' ------->|
   |                                 | ✓ COMMIT
   |<------ [CONEXIÓN MUERE] --------|
   |                                 |
   | "¿Falló? Reintento..."          |
   |                                 |
   |-------- INSERT 'pedido' ------->|
   |                                 | ✓ COMMIT (duplicado!)
   |<-------- OK --------------------|

Resultado: 2 pedidos duplicados
```

---

## La Solución: Idempotencia

```
Con idempotencia + Retry = SEGURO

Cliente                          PostgreSQL
   |                                 |
   | genera: request_id = "abc-123"  |
   |                                 |
   |-- INSERT id='abc-123' --------->|
   |                                 | ✓ COMMIT
   |<------ [CONEXIÓN MUERE] --------|
   |                                 |
   | "¿Falló? Reintento mismo ID..." |
   |                                 |
   |-- INSERT id='abc-123' --------->|
   |                                 | CONFLICT → DO NOTHING
   |<-------- OK --------------------|

Resultado: 1 pedido (correcto)
```

---

## Operaciones Idempotentes vs No Idempotentes

| Operación | ¿Idempotente? | Por qué |
|-----------|---------------|---------|
| `SELECT * FROM users` | ✅ Sí | Leer no cambia nada |
| `UPDATE users SET status='active' WHERE id=5` | ✅ Sí | Mismo resultado siempre |
| `DELETE FROM users WHERE id=5` | ✅ Sí | Borrar algo ya borrado = OK |
| `INSERT INTO logs (msg) VALUES ('x')` | ❌ No | Crea duplicados |
| `INSERT ... ON CONFLICT DO NOTHING` | ✅ Sí | Ignora duplicados |
| `UPDATE users SET balance = balance + 100` | ❌ No | Cada ejecución suma más |

---

## Soporte Necesario para Idempotencia

Para implementar idempotencia necesitas soporte en **la base de datos** o en **la aplicación** (o ambos).

### Opción A: Soporte a Nivel de Tabla (Recomendado)

La base de datos garantiza la unicidad mediante constraints.

#### Estructura de Tabla Requerida

```sql
-- ✅ Tabla CON soporte para idempotencia
CREATE TABLE pedidos (
    id SERIAL PRIMARY KEY,
    request_id VARCHAR(64) UNIQUE,  -- ← UNIQUE constraint obligatorio
    producto TEXT,
    cantidad INTEGER,
    created_at TIMESTAMP DEFAULT NOW()
);

-- Índice para mejorar rendimiento (opcional pero recomendado)
CREATE INDEX idx_pedidos_request_id ON pedidos(request_id);
```

#### Cómo Funciona

```sql
-- Primera ejecución: INSERT exitoso
INSERT INTO pedidos (request_id, producto, cantidad)
VALUES ('req-abc-123', 'laptop', 1)
ON CONFLICT (request_id) DO NOTHING;
-- Resultado: 1 fila insertada

-- Segunda ejecución (retry): No hace nada
INSERT INTO pedidos (request_id, producto, cantidad)
VALUES ('req-abc-123', 'laptop', 1)
ON CONFLICT (request_id) DO NOTHING;
-- Resultado: 0 filas insertadas (sin error, sin duplicado)
```

#### Ventajas

- **Atómico**: No hay race conditions
- **Rápido**: La verificación es a nivel de índice
- **Seguro**: Funciona con múltiples instancias de la aplicación
- **Simple**: Una sola query hace todo

#### Desventajas

- Requiere modificar el esquema de la tabla
- Necesitas generar IDs únicos en la aplicación

---

### Opción B: Soporte a Nivel de Aplicación

Si no puedes modificar la tabla, implementas la lógica en código.

#### Ejemplo en Python/Django

```python
from django.db import transaction, IntegrityError

def crear_pedido_idempotente(request_id, producto, cantidad):
    """
    Implementación de idempotencia en la aplicación.
    Menos eficiente que ON CONFLICT, pero funciona sin modificar la tabla.
    """
    # Verificar si ya existe
    existing = Pedido.objects.filter(request_id=request_id).first()
    if existing:
        return existing  # Ya existe, retornar el existente

    # Intentar crear
    try:
        with transaction.atomic():
            pedido = Pedido.objects.create(
                request_id=request_id,
                producto=producto,
                cantidad=cantidad
            )
            return pedido
    except IntegrityError:
        # Race condition: otro proceso lo creó entre el check y el create
        return Pedido.objects.get(request_id=request_id)
```

#### Ejemplo en Bash

```bash
# Verificar antes de insertar (menos eficiente)
check_and_insert() {
    local request_id=$1
    local data=$2

    # Verificar si existe
    exists=$(psql -t -A -c "SELECT 1 FROM pedidos WHERE request_id='$request_id' LIMIT 1;")

    if [ -z "$exists" ]; then
        # No existe, insertar
        psql -c "INSERT INTO pedidos (request_id, data) VALUES ('$request_id', '$data');"
    else
        echo "Ya existe, saltando..."
    fi
}
```

#### Ventajas

- No requiere modificar el esquema de la tabla
- Funciona con tablas legacy

#### Desventajas

- **Race conditions**: Entre el SELECT y el INSERT, otro proceso puede insertar
- **Más lento**: Dos queries en lugar de una
- **Más complejo**: Necesitas manejar errores de concurrencia

---

### Opción C: Híbrido (Mejor de Ambos Mundos)

Combina constraint en la tabla + lógica en la aplicación.

```python
# models.py - Tabla con constraint
class Pedido(models.Model):
    request_id = models.CharField(max_length=64, unique=True)  # ← Constraint
    producto = models.CharField(max_length=200)
    cantidad = models.IntegerField()

# views.py - Aplicación con retry
def crear_pedido(request):
    request_id = request.headers.get('X-Request-ID') or str(uuid.uuid4())

    for attempt in range(3):
        try:
            # Usa get_or_create que internamente hace ON CONFLICT
            pedido, created = Pedido.objects.get_or_create(
                request_id=request_id,
                defaults={
                    'producto': request.data['producto'],
                    'cantidad': request.data['cantidad']
                }
            )
            return pedido
        except OperationalError:
            # Error de conexión, reintentar con el MISMO request_id
            time.sleep(0.5)
            continue

    raise Exception("No se pudo crear el pedido después de 3 intentos")
```

---

### Comparación de Opciones

| Aspecto | Nivel de Tabla | Nivel de Aplicación | Híbrido |
|---------|----------------|---------------------|---------|
| Atomicidad | ✅ Garantizada | ⚠️ Race conditions | ✅ Garantizada |
| Rendimiento | ✅ Rápido (1 query) | ❌ Lento (2+ queries) | ✅ Rápido |
| Modificar esquema | ❌ Necesario | ✅ No necesario | ❌ Necesario |
| Complejidad código | ✅ Simple | ❌ Complejo | ✅ Simple |
| Múltiples instancias | ✅ Seguro | ⚠️ Requiere locks | ✅ Seguro |

**Recomendación**: Siempre que sea posible, usa **soporte a nivel de tabla** (Opción A o C).

---

## Implementación en SQL

### Opción 1: ON CONFLICT DO NOTHING

```sql
-- Requiere constraint UNIQUE en request_id
INSERT INTO pedidos (request_id, producto, cantidad)
VALUES ('uuid-único', 'laptop', 1)
ON CONFLICT (request_id) DO NOTHING;
```

### Opción 2: ON CONFLICT DO UPDATE (Upsert)

```sql
INSERT INTO configuracion (clave, valor)
VALUES ('theme', 'dark')
ON CONFLICT (clave) DO UPDATE SET valor = EXCLUDED.valor;
```

### Opción 3: INSERT WHERE NOT EXISTS

```sql
INSERT INTO pedidos (request_id, producto)
SELECT 'uuid-único', 'laptop'
WHERE NOT EXISTS (
    SELECT 1 FROM pedidos WHERE request_id = 'uuid-único'
);
```

---

## Implementación en Django

Django proporciona métodos que facilitan la idempotencia:

### Métodos Idempotentes

```python
# get_or_create - Crea solo si no existe
user, created = User.objects.get_or_create(
    email="test@example.com",
    defaults={"name": "Test User"}
)

# update_or_create - Actualiza si existe, crea si no
profile, created = Profile.objects.update_or_create(
    user=user,
    defaults={"bio": "Nueva bio"}
)

# filter().update() - Update masivo idempotente
User.objects.filter(id=5).update(status='active')
```

### Modelo con Unique Constraint

```python
from django.db import models
import uuid

class Pedido(models.Model):
    # UUID como PK = idempotente automáticamente
    id = models.UUIDField(primary_key=True, default=uuid.uuid4, editable=False)

    # O un campo de negocio único
    numero_pedido = models.CharField(max_length=50, unique=True)

    producto = models.CharField(max_length=200)
    cantidad = models.IntegerField()
    created_at = models.DateTimeField(auto_now_add=True)
```

### Vista con Manejo de Duplicados

```python
from django.db import IntegrityError, transaction
import uuid

def crear_pedido(request):
    # Generar ID único ANTES de intentar
    request_id = str(uuid.uuid4())

    try:
        with transaction.atomic():
            pedido = Pedido.objects.create(
                id=request_id,
                producto=request.data['producto'],
                cantidad=request.data['cantidad']
            )
    except IntegrityError:
        # Ya existe (fue un retry de una operación que sí se guardó)
        pedido = Pedido.objects.get(id=request_id)

    return pedido
```

---

## Configuración Django para Cluster HA

```python
# settings.py

DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'HOST': '192.168.37.15',  # VIP del cluster
        'PORT': '5000',           # Puerto HAProxy
        'NAME': 'mi_database',
        'USER': 'postgres',
        'PASSWORD': 'password',
        'CONN_MAX_AGE': 60,       # Conexiones persistentes
        'CONN_HEALTH_CHECKS': True,  # Django 4.1+ verifica conexión
        'OPTIONS': {
            'connect_timeout': 5,
            'options': '-c statement_timeout=30000',
        },
    }
}

# Transacciones atómicas por request
ATOMIC_REQUESTS = True
```

---

## Cuándo Necesitas Idempotencia

| Tipo de Dato | ¿Necesita Idempotencia? | Razón |
|--------------|-------------------------|-------|
| Logs, métricas, eventos | No necesariamente | Duplicados son tolerables |
| Pedidos, transacciones | **Sí, crítico** | Duplicados = dinero perdido |
| Pagos, facturas | **Sí, crítico** | Duplicados = problemas legales |
| Usuarios, cuentas | Parcial | UNIQUE en email ya protege |
| Configuración | No | UPDATE es idempotente |
| Cache | No | Se sobrescribe |

---

## Estrategias de Retry

### Sin Retry (Simple, pero pierde datos)

```python
try:
    db.execute("INSERT INTO orders ...")
except:
    log.error("Falló")  # Dato perdido
```

### Retry Sin Idempotencia (Riesgo de duplicados)

```python
for attempt in range(3):
    try:
        db.execute("INSERT INTO orders ...")
        break
    except:
        continue  # ⚠️ Puede duplicar
```

### Retry Con Idempotencia (Seguro)

```python
order_id = generate_uuid()
for attempt in range(3):
    try:
        db.execute("""
            INSERT INTO orders (id, ...) VALUES (%s, ...)
            ON CONFLICT (id) DO NOTHING
        """, order_id)
        break
    except:
        time.sleep(0.5)
        continue  # ✅ Mismo ID, no duplica
```

---

## Alternativas a Idempotencia

### 1. Replicación Síncrona

```yaml
# Patroni config
synchronous_mode: true
```

- **Pro**: Menos ambigüedad (el líder espera confirmación de réplica)
- **Contra**: Más lento, failover más complejo

### 2. Cola de Mensajes (Outbox Pattern)

```
App → Cola (Kafka/RabbitMQ) → Worker → PostgreSQL
```

- **Pro**: Garantiza entrega, desacopla
- **Contra**: Complejidad adicional

### 3. Two-Phase Commit

- **Pro**: Transacciones distribuidas atómicas
- **Contra**: Muy lento, complejo

---

## Pruebas de Idempotencia

El script `TrafficGenerator.sh` ahora implementa idempotencia:

```bash
# Ejecutar con logging para ver reintentos
./TrafficGenerator.sh --log

# La tabla ahora tiene request_id único
# INSERT usa: ON CONFLICT (request_id) DO NOTHING
# Reintentos usan el MISMO request_id
```

### Verificar que no hay duplicados

```sql
-- Buscar duplicados por request_id
SELECT request_id, COUNT(*)
FROM traffic_gen
GROUP BY request_id
HAVING COUNT(*) > 1;

-- Resultado esperado: 0 filas (sin duplicados)
```

---

## Resumen

1. **Idempotencia** = misma operación múltiples veces = mismo resultado
2. **Esencial en HA** porque failovers causan situaciones ambiguas
3. **Implementación**: usar IDs únicos + `ON CONFLICT DO NOTHING`
4. **Django**: usar `get_or_create()`, `update_or_create()`, unique constraints
5. **No todo necesita idempotencia**: solo operaciones críticas que se reintentan

---

## Referencias

- [PostgreSQL ON CONFLICT](https://www.postgresql.org/docs/current/sql-insert.html#SQL-ON-CONFLICT)
- [Django get_or_create](https://docs.djangoproject.com/en/stable/ref/models/querysets/#get-or-create)
- [Idempotency Patterns](https://stripe.com/docs/api/idempotent_requests)
- [Patroni Documentation](https://patroni.readthedocs.io/)
