# Docker — Documentación Completa
### Conceptos, Comandos, Seguridad y Flujo Real del Homelab

---

## ¿Qué es Docker y por qué existe?

Imagina que eres chef y tienes una receta perfecta. La haces en tu cocina y queda espectacular. Pero cuando la haces en la cocina de tu amigo — diferente estufa, diferentes ollas, faltan ingredientes — queda diferente o directamente no funciona.

**Docker resuelve eso.** Empacas no solo la receta sino **toda la cocina** — la estufa, las ollas, los ingredientes exactos — en una caja. Esa caja funciona igual en cualquier lado.

```
Sin Docker:  "en mi PC funciona" 😤
Con Docker:  funciona en todos lados, siempre, igual ✅
```

Docker también garantiza **aislamiento** — cada aplicación vive en su propio entorno virtual. No hay conflictos de dependencias, no hay crashes entre servicios. Si una app falla, las demás siguen corriendo sin enterarse.

---

## Los Conceptos Fundamentales

### 1. Dockerfile — La Receta

Es un archivo de texto con instrucciones para construir tu ambiente. Le dice a Docker exactamente qué instalar, qué copiar, qué configurar.

```dockerfile
FROM ubuntu:22.04        # parte de esta base
RUN apt install nginx    # instala esto
COPY ./app /var/www      # copia estos archivos
EXPOSE 80                # escucha en este puerto
CMD ["nginx"]            # arranca esto al iniciar
```

**Analogía:** es el plano arquitectónico de tu cocina. No es la cocina en sí, son las instrucciones para construirla.

---

### 2. Image — La Cocina Empacada

Cuando ejecutas el Dockerfile, Docker construye una **image** — una copia estática, lista para usar, de todo tu ambiente. Es inmutable — una vez construida no cambia.

```bash
docker build -t mi-app .    # construye la image desde el Dockerfile
docker images               # ver todas tus images locales
docker pull nginx           # descargar image de Docker Hub sin construir
docker rmi IMAGEN           # borrar una image
```

**Analogía:** la cocina ya construida y empacada en una caja. Lista para abrir y usar pero todavía no está funcionando — es una foto, no una cocina activa.

---

### 3. Container — La Cocina Funcionando

Un container es una image **corriendo**. Es la instancia activa y viva. Puedes tener 10 containers del mismo image corriendo al mismo tiempo, cada uno independiente.

```bash
docker run IMAGEN                    # crear y arrancar container
docker ps                            # ver containers activos
docker ps -a                         # ver TODOS (incluye parados)
docker stop CONTAINER                # parar container
docker start CONTAINER               # arrancar container parado
docker restart CONTAINER             # reiniciar container
docker rm CONTAINER                  # borrar container
docker exec -it CONTAINER bash       # entrar al container (shell interactivo)
docker logs CONTAINER                # ver logs del container
docker inspect CONTAINER             # ver toda la info en JSON
```

**Analogía:** abres la caja, instalas la cocina y la prendes. Ahora está activa y funcionando. Si la apagas y la prendes de nuevo — vuelve exactamente igual, sin memoria de lo que pasó antes.

> **Importante:** los containers son efímeros por naturaleza. Si lo borras, todo lo que había dentro desaparece. Para datos que deben persistir, se usan volúmenes.

---

### 4. Docker Hub — El Supermercado de Cocinas

Es el repositorio público donde viven miles de images listas para usar. No tienes que construir todo desde cero — casi cualquier servicio que necesites ya tiene una image oficial.

```bash
docker pull nginx           # descarga la image de nginx
docker pull postgres        # descarga postgres
docker pull grafana/grafana # descarga grafana
```

**Analogía:** en vez de construir tu cocina desde cero, vas al supermercado y compras una cocina prefabricada de la marca que quieras. La abres y ya funciona.

---

### 5. Volumen — La Caja Fuerte de los Datos

Los containers son **efímeros** — si lo borras, pierdes todo lo que había dentro. Los volúmenes son carpetas compartidas entre el container y tu servidor que **persisten** aunque el container muera o se recree.

```bash
docker volume ls                    # listar volúmenes
docker volume inspect VOLUMEN       # ver detalles

# En docker run:
docker run -v /ruta/host:/ruta/container IMAGEN

# Ejemplo real del homelab:
# -v /mnt/hdd/docker/grafana:/var/lib/grafana
```

**Analogía:** el container es un chef que vive en un hotel — si se va, se lleva todo. El volumen es una caja fuerte en tu casa donde el chef guarda las recetas importantes. El chef puede irse y volver, las recetas siempre están ahí.

**En el homelab:** todos los datos van a `/mnt/hdd/docker/` — el disco de 1TB. Si un container se rompe o se actualiza, los datos persisten intactos.

---

### 6. Docker Network — El Sistema de Intercomunicación

Por defecto cada container está aislado — no se habla con nadie. Docker Network les da un canal de comunicación entre containers.

```bash
docker network ls               # listar redes
docker network inspect RED      # ver detalles de una red

# En docker run:
docker run --network homelab IMAGEN
```

**Analogía:** tienes varios cuartos en el server. Sin network cada cuarto está sellado. Con network es como poner un sistema de intercomunicadores entre los cuartos — Grafana le puede hablar a Prometheus, Prometheus le puede hablar a Node Exporter, todos se entienden.

> **Lección aprendida en el homelab:** cuando Prometheus no podía conectarse a Node Exporter, era porque estaban en redes diferentes. Al ponerlos en la misma red `homelab`, se resolvió inmediatamente.

---

### 7. Docker Compose — El Director de Orquesta

Cuando tienes múltiples containers que trabajan juntos, en vez de arrancarlos uno por uno con `docker run`, usas un archivo `docker-compose.yml` que los define todos y los coordina.

```bash
docker compose up -d        # arranca todos los servicios en background
docker compose down         # para y elimina los containers
docker compose restart      # reinicia todos
docker compose restart SERVICIO  # reinicia solo uno
docker compose logs         # ver logs de todos
docker compose ps           # ver estado de los servicios
```

**Analogía:** en vez de contratar a cada músico por separado y decirle qué tocar, tienes un director de orquesta que coordina a todos al mismo tiempo con una sola partitura.

---

## Docker Run — Flags Importantes

Cuando usas `docker run` directamente (sin Compose), estos flags controlan el comportamiento del container:

| Flag | Descripción | Ejemplo |
|---|---|---|
| `-d` | Detached — corre en background | `docker run -d nginx` |
| `-p HOST:CONTAINER` | Mapeo de puertos | `-p 3000:3000` |
| `-v HOST:CONTAINER` | Mapeo de volumen | `-v /mnt/hdd/datos:/data` |
| `-e VAR=valor` | Variable de entorno | `-e POSTGRES_PASSWORD=secret` |
| `--name nombre` | Nombre del container | `--name mi-nginx` |
| `--restart unless-stopped` | Reinicio automático | reinicia siempre excepto si lo paras manualmente |
| `--network red` | Red específica | `--network homelab` |
| `-u USUARIO` | Correr como usuario no-root | `-u 1000:1000` |
| `--read-only` | Filesystem read-only | protege el container de escrituras |
| `--cap-drop ALL` | Quitar todas las capabilities | principio de mínimo privilegio |
| `--cap-add NET_BIND_SERVICE` | Agregar capability específica | solo agregar lo necesario |

---

## Docker Compose — Anatomía Completa

Cada campo del `docker-compose.yml` tiene un propósito específico:

```yaml
services:           # aquí empiezan todos los servicios
  nombre_servicio:  # identificador interno del servicio
    
    image:          # qué image usar de Docker Hub
    container_name: # nombre del container (sin esto Docker pone uno random)
    restart:        # política de reinicio
                    # unless-stopped → reinicia siempre excepto si lo paras tú
                    # always → siempre reinicia
                    # on-failure → solo si falla con error
                    # no → nunca reinicia
    
    security_opt:   # opciones de seguridad
      - no-new-privileges:true  # el container no puede escalar privilegios
    
    ports:          # mapeo de puertos
      - "HOST:CONTAINER"  # puerto de tu servidor : puerto del container
    
    volumes:        # mapeo de datos persistentes
      - /ruta/host:/ruta/container
    
    environment:    # variables de entorno
      - VARIABLE=valor
    
    networks:       # a qué redes pertenece
      - nombre_red
    
    command:        # sobreescribir el comando por defecto de la image

networks:           # definición de redes
  nombre_red:
    driver: bridge  # tipo de red (bridge es el estándar)
```

---

## Seguridad en Docker

### ¿Por qué `security_opt: no-new-privileges:true`?

Evita que el container o cualquier proceso dentro de él pueda escalar privilegios. Si alguien explota una vulnerabilidad dentro del container, no puede volverse root ni salirse al host.

```yaml
security_opt:
  - no-new-privileges:true
```

**Analogía:** es como decirle al empleado que entra al cuarto — puedes entrar y hacer tu trabajo, pero no puedes conseguirte una llave maestra mientras estás adentro.

### ¿Por qué el usuario del container importa?

Algunos containers corren internamente como usuarios específicos:
- Grafana usa el usuario `472`
- Prometheus usa el usuario `65534`
- Si la carpeta de datos pertenece a root, el container no puede escribir ahí

```bash
# Dar permisos correctos para Grafana
sudo chown -R 472:472 /mnt/hdd/docker/grafana

# Dar permisos correctos para Prometheus
sudo chown -R 65534:65534 /mnt/hdd/docker/prometheus
```

### El docker.sock — Poder Total

```yaml
volumes:
  - /var/run/docker.sock:/var/run/docker.sock
```

Este volumen le da al container acceso completo al daemon de Docker. **Es básicamente darle las llaves del servidor.** Solo se usa con containers de confianza como Portainer o Homepage que necesitan ver y manejar otros containers.

---

## Comandos de Mantenimiento

```bash
# Ver uso de recursos en tiempo real
docker stats

# Limpiar todo lo que no se usa (imágenes, containers parados, redes huérfanas)
docker system prune

# Limpiar incluyendo volúmenes (CUIDADO — borra datos)
docker system prune --volumes

# Ver cuánto espacio usa Docker
docker system df

# Actualizar un servicio a la versión más nueva
docker compose pull SERVICIO
docker compose up -d SERVICIO
```

---

## El Stack Completo del Homelab

### Arquitectura Actual

```
saga (Ubuntu 26.04 LTS)
│
├── Docker Engine
│   │
│   └── Red: homelab (bridge)
│       │
│       ├── Portainer      :9000  → manejo visual de Docker
│       ├── Homepage       :3000  → dashboard principal
│       ├── Prometheus     :9090  → almacena métricas
│       ├── Grafana        :3001  → visualiza métricas
│       └── Node Exporter  :9100  → recopila métricas del servidor
│
└── /mnt/hdd/docker/       → datos persistentes (disco 1TB)
    ├── portainer/
    ├── homepage/
    ├── prometheus/
    │   └── data/
    └── grafana/
```

### Flujo de Datos del Monitoreo

```
saga (el servidor físico)
        │
        │ métricas del sistema
        ▼
Node Exporter (:9100)
        │
        │ expone métricas en /metrics
        ▼
Prometheus (:9090)
        │
        │ scrape cada 15 segundos
        │ almacena en /mnt/hdd/docker/prometheus/data
        ▼
Grafana (:3001)
        │
        │ consulta métricas
        │ renderiza dashboards
        ▼
Tu navegador → Dashboard Node Exporter Full
```

### docker-compose.yml Completo y Comentado

```yaml
services:

  # ─── PORTAINER ─────────────────────────────────────────────
  # Interfaz web para manejar Docker visualmente
  portainer:
    image: portainer/portainer-ce:latest
    container_name: portainer
    restart: unless-stopped
    security_opt:
      - no-new-privileges:true
    ports:
      - "9000:9000"       # acceso web en puerto 9000
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock  # acceso al daemon Docker
      - /mnt/hdd/docker/portainer:/data            # config persistente
    networks:
      - homelab

  # ─── HOMEPAGE ──────────────────────────────────────────────
  # Dashboard principal — vista general de todos los servicios
  homepage:
    image: ghcr.io/gethomepage/homepage:latest
    container_name: homepage
    restart: unless-stopped
    security_opt:
      - no-new-privileges:true
    ports:
      - "3000:3000"
    environment:
      HOMEPAGE_ALLOWED_HOSTS: "<TAILSCALE_IP>:3000"  # IP de Tailscale
    volumes:
      - /mnt/hdd/docker/homepage:/app/config         # config persistente
      - /var/run/docker.sock:/var/run/docker.sock    # para ver estado de containers
    networks:
      - homelab

  # ─── PROMETHEUS ────────────────────────────────────────────
  # Base de datos de métricas — almacena todo lo que Node Exporter recopila
  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    restart: unless-stopped
    security_opt:
      - no-new-privileges:true
    ports:
      - "9090:9090"
    volumes:
      - /mnt/hdd/docker/prometheus:/etc/prometheus        # config (prometheus.yml)
      - /mnt/hdd/docker/prometheus/data:/prometheus       # datos de métricas
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'    # archivo de config a usar
    networks:
      - homelab

  # ─── GRAFANA ───────────────────────────────────────────────
  # Visualización de métricas — dashboards, gráficos, alertas
  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    restart: unless-stopped
    security_opt:
      - no-new-privileges:true
    ports:
      - "3001:3000"       # grafana usa 3000 internamente, lo exponemos en 3001
    volumes:
      - /mnt/hdd/docker/grafana:/var/lib/grafana   # dashboards y config persistentes
    environment:
      - GF_SECURITY_ADMIN_USER=<TU_USUARIO>
      - GF_SECURITY_ADMIN_PASSWORD=<TU_PASSWORD>           # cambiar después del primer login
    networks:
      - homelab

  # ─── NODE EXPORTER ─────────────────────────────────────────
  # Agente de métricas — recopila CPU, RAM, disco, red del servidor
  node-exporter:
    image: prom/node-exporter:latest
    container_name: node-exporter
    restart: unless-stopped
    security_opt:
      - no-new-privileges:true
    ports:
      - "9100:9100"
    volumes:
      - /proc:/host/proc:ro      # info de procesos (read-only)
      - /sys:/host/sys:ro        # info del kernel (read-only)
      - /:/rootfs:ro             # sistema de archivos (read-only)
    command:
      - '--path.procfs=/host/proc'
      - '--path.rootfs=/rootfs'
      - '--path.sysfs=/host/sys'
      - '--collector.filesystem.mount-points-exclude=^/(sys|proc|dev|host|etc)($$|/)'
    networks:
      - homelab

# ─── REDES ─────────────────────────────────────────────────────
# Red compartida para que todos los servicios se comuniquen entre sí
networks:
  homelab:
    driver: bridge
```

### prometheus.yml — Configuración de Scraping

```yaml
global:
  scrape_interval: 15s    # cada cuánto recopila métricas

scrape_configs:
  # Prometheus se monitorea a sí mismo
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  # Métricas del servidor via Node Exporter
  - job_name: 'node-exporter'
    static_configs:
      - targets: ['node-exporter:9100']  # nombre del container en la red homelab
```

---

## Problemas Reales Encontrados y Cómo se Resolvieron

### 1. Permisos de carpetas
**Problema:** Grafana y Prometheus no podían escribir en sus carpetas de datos.  
**Causa:** las carpetas fueron creadas por root pero los containers corren con usuarios específicos.  
**Solución:**
```bash
sudo chown -R 472:472 /mnt/hdd/docker/grafana
sudo chown -R 65534:65534 /mnt/hdd/docker/prometheus
sudo chown -R 65534:65534 /mnt/hdd/docker/prometheus/data
```

### 2. Puertos ocupados
**Problema:** Prometheus y Node Exporter no podían arrancar porque sus puertos ya estaban en uso.  
**Causa:** había instalaciones del sistema (no Docker) corriendo los mismos servicios.  
**Solución:**
```bash
sudo systemctl stop prometheus
sudo systemctl disable prometheus
sudo systemctl stop prometheus-node-exporter
sudo systemctl disable prometheus-node-exporter
```

### 3. Containers en redes diferentes
**Problema:** Prometheus no podía conectarse a Node Exporter aunque ambos corrían.  
**Causa:** cada `docker compose up` crea su propia red por defecto — los containers no se veían.  
**Solución:** definir una red compartida `homelab` en el `docker-compose.yml` y agregar todos los servicios a ella.

### 4. Indentación en YAML
**Problema:** `docker compose up` fallaba con errores de parsing.  
**Causa:** YAML es extremadamente sensible a la indentación — siempre 2 espacios, nunca tabs.  
**Solución:** revisar que cada nivel tenga exactamente 2 espacios más que el anterior.

---

## Referencia Rápida de Comandos

```bash
# ── IMÁGENES ────────────────────────────────────────────
docker pull IMAGEN              # descargar imagen del registry
docker images                   # listar imágenes locales
docker rmi IMAGEN               # borrar imagen
docker build -t nombre .        # construir imagen desde Dockerfile

# ── CONTAINERS ──────────────────────────────────────────
docker run [opciones] IMAGEN    # crear y arrancar container
docker ps                       # listar containers corriendo
docker ps -a                    # listar todos (incluye parados)
docker stop CONTAINER           # parar
docker start CONTAINER          # arrancar parado
docker restart CONTAINER        # reiniciar
docker rm CONTAINER             # borrar
docker exec -it CONTAINER bash  # entrar al container (shell)
docker logs CONTAINER           # ver logs
docker logs -f CONTAINER        # ver logs en tiempo real
docker inspect CONTAINER        # ver JSON completo con toda la info
docker stats                    # ver uso de recursos en tiempo real

# ── REDES ───────────────────────────────────────────────
docker network ls               # listar redes
docker network inspect RED      # ver detalles

# ── VOLÚMENES ───────────────────────────────────────────
docker volume ls                # listar volúmenes
docker volume inspect VOLUMEN  # ver detalles

# ── COMPOSE ─────────────────────────────────────────────
docker compose up -d            # arrancar todo en background
docker compose down             # parar y eliminar containers
docker compose restart          # reiniciar todo
docker compose restart SERVICIO # reiniciar solo uno
docker compose logs             # ver logs de todos
docker compose ps               # ver estado de servicios
docker compose pull             # actualizar todas las imágenes
docker compose up -d --force-recreate SERVICIO  # recrear un servicio específico

# ── MANTENIMIENTO ───────────────────────────────────────
docker system prune             # limpiar recursos huérfanos
docker system prune --volumes   # limpiar incluyendo volúmenes (CUIDADO)
docker system df                # ver uso de espacio
```

---

## Tecnologías del Stack de Monitoreo

| Servicio | Puerto | Rol | Image |
|---|---|---|---|
| Portainer | 9000 | Manejo visual de Docker | portainer/portainer-ce |
| Homepage | 3000 | Dashboard principal | ghcr.io/gethomepage/homepage |
| Prometheus | 9090 | Base de datos de métricas | prom/prometheus |
| Grafana | 3001 | Visualización de métricas | grafana/grafana |
| Node Exporter | 9100 | Agente de métricas del servidor | prom/node-exporter |

---

## Próximos Pasos

```
⏳ Loki + Promtail    → logging centralizado
⏳ LXC               → separar entornos producción/laboratorio
⏳ KVM               → laboratorio de pentesting
⏳ Homepage config   → personalizar el dashboard con todos los servicios
⏳ Alertas Grafana   → notificaciones cuando algo falla
```

---

*Parte de la serie de documentación del proyecto DevSecOps HomeLab — SamGab-ORG*
