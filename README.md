# PostgreSQL Stack

PostgreSQL 16 containerizado para desplegar con Portainer.

## Prerequisitos

```bash
docker network create backend_net
```

## Deployment

1. Abrir Portainer
2. Stacks → Add stack
3. Cargar `docker-compose.yml`
4. Configurar variables de entorno desde `.env`
5. Deploy

## Configuración

Editar `.env`:
```
POSTGRES_USER=admin
POSTGRES_PASSWORD=tu_password
```

La base de datos se crea automáticamente con el nombre del usuario.

## Acceso

```bash
# Ver logs
docker logs -f postgres

# Acceder a psql
docker exec -it postgres psql -U admin
```

## Puertos

- PostgreSQL: `5432`
