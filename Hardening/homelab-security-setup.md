# Homelab Security Setup
-- SSH Key Authentication + Fail2ban | Defense in Depth

---

## Resumen del Proyecto -- HARDENING

Este proyecto documenta la configuración de seguridad de un servidor Linux (Ubuntu 26.04) en un homelab personal. El objetivo fue implementar múltiples capas de 
seguridad siguiendo el principio de **defense in depth** — cada capa asume que la anterior puede fallar, por lo que nunca dependemos de un solo mecanismo de protección.

### Stack de Seguridad Implementado

```
Internet
    │
    ▼
Tailscale        ← VPN privada: solo entran miembros de la red
    │
    ▼
Firewall         ← filtra puertos y tráfico no autorizado
    │
    ▼
SSH Key Auth     ← autenticación por clave criptográfica (sin passwords)
    │
    ▼
Fail2ban         ← banea IPs con intentos fallidos automáticamente
```

---

## Parte 1: Generación de SSH Keys

### ¿Qué es una SSH Key?

Una SSH key es un par de claves criptográficas que reemplaza el login por contraseña. Funciona así:

- **Clave privada** (`homelab_ed25519`): vive en tu PC, nunca sale de ahí. Es como tu documento de identidad.
- **Clave pública** (`homelab_ed25519.pub`): se copia al servidor. Es como el registro de quién puede entrar.

Cuando te conectas, el servidor verifica que tu clave privada corresponde a la pública que tiene registrada — sin necesidad de enviar ninguna contraseña por la red.

### ¿Por qué ED25519?

ED25519 es el algoritmo de curva elíptica más moderno disponible en OpenSSH. Comparado con RSA:

- Claves mucho más cortas (256 bits vs 2048-4096 bits)
- Más rápido para firmar y verificar
- Igual o más seguro que RSA-4096
- Resistente a ataques de tiempo

### Comandos Ejecutados

**En la PC cliente (`sama/yo`):**

```bash
# Crear el directorio .ssh con permisos correctos
mkdir -p ~/.ssh
chmod 700 ~/.ssh

# Generar el par de claves ED25519
# -t ed25519: algoritmo
# -a 100: número de rondas de KDF (Key Derivation Function), más rondas = más lento para atacantes
# -C: comentario identificador
# -f: nombre del archivo de salida
ssh-keygen -t ed25519 -a 100 -C "xxxxxxx@homelab" -f ~/.ssh/homelab_ed25519
```

> **Nota sobre `-a 100`:** Este parámetro controla cuántas rondas de hashing se usan para derivar la clave de encriptación de la passphrase. 
Más rondas hace que un atacante que robe el archivo tarde mucho más en hacer fuerza bruta.

**Resultado:**
```
~/.ssh/homelab_ed25519      ← clave privada (permisos 600, solo tú)
~/.ssh/homelab_ed25519.pub  ← clave pública (se copia al servidor)
```

---

## Parte 2: Copiar la Clave Pública al Servidor

### ¿Qué es authorized_keys?

El archivo `~/.ssh/authorized_keys` en el servidor contiene todas las claves públicas autorizadas para conectarse. Cuando alguien intenta conectarse, 
SSH verifica si su clave privada corresponde a alguna de las entradas de este archivo.

### Comando Ejecutado

```bash
# Copia automáticamente la clave pública al servidor
# -i: especifica qué clave pública copiar
ssh-copy-id -i ~/.ssh/homelab_ed25519.pub xxxxxxx@100.xxx.xx.xxx
```

`ssh-copy-id` se conecta al servidor con password, agrega la clave pública a `~/.ssh/authorized_keys`, y configura los permisos correctos automáticamente.

**Verificación en el servidor:**
```bash
ls -la ~/.ssh/
# authorized_keys debe tener permisos 600 (-rw-------)
```

---

## Parte 3: Configuración del Cliente SSH

### ¿Qué es ~/.ssh/config?

El archivo `~/.ssh/config` en tu PC le dice al cliente SSH cómo conectarse a cada host. En lugar de escribir `ssh -i ~/.ssh/homelab_ed25519 samgab@100.xxx.xxx.xxx` 
cada vez, defines un alias con todas las opciones.

### Configuración Aplicada

```bash
nano ~/.ssh/config
chmod 600 ~/.ssh/config
```

```ini
Host homelab
    HostName 100.xxx.xx.xxx       # IP de Tailscale del servidor
    User xxxxxx                    # usuario en el servidor
    IdentityFile ~/.ssh/homelab_ed25519  # clave privada a usar
    IdentitiesOnly yes             # usar SOLO esta clave, no otras
    ServerAliveInterval 60         # envía keepalive cada 60 segundos
    ServerAliveCountMax 3          # corta la conexión tras 3 keepalives sin respuesta
```

**Resultado:** conectar al homelab con un simple:
```bash
ssh homelab
```

### ¿Por qué IdentitiesOnly yes?

Sin esta opción, SSH puede intentar múltiples claves de tu agente antes de usar la correcta. Con `IdentitiesOnly yes` le decimos explícitamente que solo use la clave 
especificada, evitando intentos innecesarios que podrían ser registrados como sospechosos por el servidor.

---

## Parte 4: SSH Agent y Passphrase

### ¿Qué es ssh-agent?

`ssh-agent` es un proceso que corre en tu PC y cachea las claves privadas en memoria RAM. Funciona así:

1. **Primera conexión:** te pide la passphrase de tu clave privada
2. **Conexiones siguientes:** el agente ya tiene la clave en RAM y la usa sin pedirte nada
3. **Al reiniciar:** el agente se limpia y vuelve a pedir la passphrase

Esto te da seguridad (el archivo en disco está protegido por passphrase) y comodidad (no la escribes en cada conexión).

### Agregar Passphrase a una Clave Existente

```bash
# Cambiar o agregar passphrase sin regenerar la clave
ssh-keygen -p -f ~/.ssh/homelab_ed25519
```

### Verificar Claves en el Agente

```bash
# Ver claves cargadas en el agente
ssh-add -l

# Remover una clave del agente (fuerza que pida passphrase de nuevo)
ssh-add -d ~/.ssh/homelab_ed25519
```

---

## Parte 5: Instalación y Configuración de Fail2ban

### ¿Qué es Fail2ban?

Fail2ban es un servicio que monitorea los logs del sistema en tiempo real. Cuando detecta demasiados intentos fallidos de login desde una IP, 
la banea automáticamente usando las reglas del firewall (`iptables`/`nftables`).

**Sin Fail2ban:** un atacante puede intentar millones de contraseñas contra tu servidor sin consecuencias.  
**Con Fail2ban:** después de X intentos fallidos, esa IP queda bloqueada por el tiempo que configures.

### ¿Cómo es diferente a Prometheus/Grafana?

| Herramienta | Tipo | Función |
|---|---|---|
| **Fail2ban** | Seguridad activa | Detecta y bloquea atacantes automáticamente |
| **Prometheus** | Observabilidad | Recolecta métricas del sistema (CPU, RAM, red) |
| **Grafana** | Visualización | Muestra las métricas de Prometheus en dashboards |

Fail2ban *actúa*. Prometheus/Grafana *observan*.

### Instalación

```bash
sudo apt update && sudo apt install fail2ban -y
```

Fail2ban instala y arranca automáticamente como servicio de systemd.

```bash
# Verificar que está corriendo
sudo systemctl status fail2ban
```

### Arquitectura de Configuración

Fail2ban usa dos archivos:

- `/etc/fail2ban/jail.conf` — configuración por defecto. **Nunca se modifica directamente** porque se sobreescribe en actualizaciones.
- `/etc/fail2ban/jail.local` — tu configuración personalizada. Sobreescribe `jail.conf`. Aquí trabajamos.

### Configuración Aplicada

```bash
sudo nano /etc/fail2ban/jail.local
```

```ini
[DEFAULT]
# Tiempo que dura el ban
bantime  = 1h

# Ventana de tiempo en la que se cuentan los intentos fallidos
findtime = 10m

# Intentos fallidos antes de banear (configuración global)
maxretry = 5

# IPs que NUNCA se banean (tu red Tailscale y localhost)
ignoreip = 127.0.0.1/8 100.xxx.xx.xxx[IP DEL SERVER MUY IMPORTANTE] [IPs_Tailscale_del_equipo]

[sshd]
# Activar la jail para SSH
enabled  = true
port     = ssh

# Dónde están los logs de SSH
logpath  = %(sshd_log)s

# Backend de lectura de logs
backend  = %(sshd_backend)s

# Para SSH somos más estrictos: solo 3 intentos
maxretry = 3
```

### Concepto de "Jail"

Una **jail** en Fail2ban es una regla para un servicio específico. Define:
- Qué logs monitorear
- Cuántos intentos antes de banear
- Qué acción tomar (ban, notificación, etc.)

Puedes tener jails para SSH, Nginx, Apache, Postfix, etc. — cada servicio con sus propias reglas.

### Aplicar Cambios y Verificar

```bash
# Reiniciar para aplicar la nueva configuración
sudo systemctl restart fail2ban

# Ver el estado de la jail de SSH
sudo fail2ban-client status sshd
```

**Output esperado:**
```
Status for the jail: sshd
|- Filter
|  |- Currently failed:  0
|  |- Total failed:      0
|  `- Journal matches:   _SYSTEMD_UNIT=ssh.service + _COMM=sshd
`- Actions
   |- Currently banned:  0
   |- Total banned:      0
   `- Banned IP list:
```

- **Currently failed:** intentos fallidos activos en la ventana de tiempo
- **Currently banned:** IPs actualmente baneadas
- **Journal matches:** confirma que está leyendo los logs correctos de SSH

---

## Arquitectura Final: Defense in Depth

El concepto de **defense in depth** (defensa en profundidad) viene de la seguridad militar: nunca dependas de una sola línea de defensa. Si una capa falla, la siguiente te protege.

```
┌─────────────────────────────────────────┐
│              INTERNET                   │
└─────────────────┬───────────────────────┘
                  │ tráfico externo
                  ▼
┌─────────────────────────────────────────┐
│              TAILSCALE                  │
│  VPN mesh P2P con WireGuard             │
│  Solo dispositivos autorizados entran   │
│  IPs privadas 100.x.x.x                 │
└─────────────────┬───────────────────────┘
                  │ solo red Tailscale
                  ▼
┌─────────────────────────────────────────┐
│              FIREWALL                   │
│  Filtra puertos y protocolos            │
│  Bloquea tráfico no autorizado          │
└─────────────────┬───────────────────────┘
                  │ solo puerto SSH
                  ▼
┌─────────────────────────────────────────┐
│           SSH KEY AUTH                  │
│  Autenticación criptográfica ED25519    │
│  Sin clave privada = sin acceso         │
│  Password auth desactivado              │
└─────────────────┬───────────────────────┘
                  │ solo con clave válida
                  ▼
┌─────────────────────────────────────────┐
│             FAIL2BAN                    │
│  Monitorea intentos fallidos            │
│  3 intentos → ban de 1 hora             │
│  IPs de confianza en whitelist          │
└─────────────────┬───────────────────────┘
                  │ acceso concedido
                  ▼
┌─────────────────────────────────────────┐
│           SERVIDOR (saga)               │
│  Ubuntu 24.04 LTS                       │
└─────────────────────────────────────────┘
```

### ¿Por qué esta arquitectura importa?

- **Tailscale** reduce la superficie de ataque drasticamente — el servidor ni siquiera es visible en internet público
- **Firewall** es la segunda barrera si Tailscale tuviera alguna vulnerabilidad
- **SSH Keys** eliminan ataques de fuerza bruta contra contraseñas
- **Fail2ban** protege contra intentos de uso de claves comprometidas o escaneos automatizados

Cada capa tiene un costo de implementación bajo y un beneficio de seguridad alto. Este es el estándar en entornos de producción reales.

---

## Comandos de Referencia Rápida

```bash
# Conectar al homelab
ssh homelab

# Estado de Fail2ban
sudo fail2ban-client status
sudo fail2ban-client status sshd

# Ver IPs baneadas
sudo fail2ban-client status sshd | grep "Banned IP"

# Desbanear una IP manualmente
sudo fail2ban-client set sshd unbanip <IP>

# Ver logs de Fail2ban en tiempo real
sudo tail -f /var/log/fail2ban.log

# Recargar configuración sin reiniciar
sudo fail2ban-client reload

# Ver claves SSH en el agente
ssh-add -l
```

---

## Tecnologías Utilizadas

| Tecnología | Versión | Rol |
|---|---|---|
| Ubuntu Server | 24.04 LTS | Sistema operativo del servidor |
| OpenSSH | 10.2p1 | Protocolo de acceso remoto seguro |
| ED25519 | — | Algoritmo de clave criptográfica |
| Fail2ban | 1.1.0 | Protección contra fuerza bruta |
| Tailscale | — | VPN mesh sobre WireGuard |

---

*Documentación generada como parte del proyecto de homelab personal — seguridad y administración de sistemas Linux.*
