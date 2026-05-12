# Homelab — Initial Server Setup
### Network Configuration + Git + Tailscale + Disk Mount | Ubuntu Server

---

## Resumen del Proyecto

Este documento cubre la configuración inicial del servidor `saga` — desde dejarlo con internet funcionando hasta conectarlo a Tailscale, configurar Git para documentación continua, y 
montar correctamente un disco secundario. Todo esto forma la base sobre la que se construye el resto del homelab.

---

## Parte 1: Configuración de Red con Netplan

### El Problema

Al instalar Ubuntu Server, el servidor no tenía conexión a internet. Sin internet no se podía instalar nada ni continuar con el setup. 
La solución fue conectar un celular por USB como fuente de internet temporal (tethering) para poder editar la configuración de red.

> **Lección aprendida:** En Ubuntu Server, la red se configura con **Netplan** — un sistema de configuración declarativa en YAML. 
Los errores de indentación rompen todo porque YAML es extremadamente sensible a los espacios.

### ¿Qué es Netplan?

Netplan es el sistema de configuración de red en Ubuntu moderno. En lugar de editar archivos de red directamente, escribes un archivo YAML que describe cómo 
quieres que estén configuradas tus interfaces, y Netplan lo aplica.

El archivo vive en `/etc/netplan/` y tiene extensión `.yaml`.

### ¿Por qué importa la indentación?

YAML usa espacios (no tabs) para definir la jerarquía. **2 espacios por nivel** es el estándar. Un espacio de más o de menos y el archivo no parsea correctamente — Netplan lo rechaza o interpreta mal 
la configuración.

```yaml
# CORRECTO — 2 espacios por nivel
network:
  version: 2
  wifis:
    wlp0s20f3:
      dhcp4: true
      access-points:
        "NombreWifi":
          password: "contraseña"

# INCORRECTO — tabs o espacios inconsistentes rompen todo
network:
    version: 2   # ← 4 espacios, va a fallar
```

### Comandos Ejecutados

```bash
# Editar la configuración de red
sudo nano /etc/netplan/00-installer-config.yaml

# Aplicar los cambios (se ejecutó múltiples veces hasta encontrar la config correcta)
sudo netplan apply

# Verificar que la interfaz de red levantó con IP
ip a

# Probar conectividad
ping www.google.com
```

> **Nota:** `sudo netplan apply` se ejecutó muchas veces durante este proceso. Cada intento era ajustar un espacio, aplicar, ver si funcionaba. Así es como se aprende — con iteración real.

### Comandos de Diagnóstico Usados

```bash
# Ver todas las interfaces de red y sus IPs
ip link
ip a

# Reiniciar el subsistema de dispositivos (se usó cuando el USB no era reconocido)
sudo modprobe -r usbcore && sudo modprobe usbcore
sudo systemctl restart systemd-udevd
```

---

## Parte 2: Conexión a Tailscale

### ¿Qué es Tailscale?

Tailscale es una VPN mesh basada en WireGuard. En lugar de una VPN centralizada donde todo el tráfico pasa por un servidor central.
Tailscale crea conexiones directas entre dispositivos (peer-to-peer). Cada dispositivo en tu red Tailscale recibe una IP estable en el rango `100.x.x.x`.

Para el homelab, Tailscale es fundamental porque:
- El servidor no necesita estar expuesto en internet público
- Todos los dispositivos del equipo pueden acceder al servidor desde cualquier lugar
- Las IPs son estables — no cambian cuando cambia la IP del router

### Verificación de Estado

```bash
# Ver el estado de Tailscale y los dispositivos conectados
tailscale status
```

Una vez conectado, el servidor `saga` tiene su IP de Tailscale (`100.xxx.xx.xxx`) que se usa para todo el acceso remoto.

---

## Parte 3: Configuración de Git en el Servidor

### ¿Por qué Git en el servidor?

El servidor tiene un repositorio clonado (`DevSecOps-HomeLab`) donde se sube toda la documentación del proyecto. 
Esto permite mantener un historial de todo lo que se hace y tenerlo disponible en GitHub como portafolio.

### El Problema con las Credenciales

Git en el servidor no recordaba las credenciales de GitHub, lo que hacía que cada `git pull` o `git push` fallara o pidiera usuario y contraseña.

### Solución: Credential Helper

```bash
# Configurar Git para guardar credenciales en disco
git config --global credential.helper store

# Configurar identidad del usuario
git config --global user.name "tu_usuario"
git config --global user.email "tu_email@gmail.com"

# Marcar el directorio como seguro (necesario cuando se corre como root)
git config --global --add safe.directory /home/samgab/DevSecOps-HomeLab/

# Verificar la configuración
git config --global --list | grep credential
git config --global --list | grep user

# Sincronizar con el repositorio remoto
git pull
```

> **Nota sobre `safe.directory`:** Cuando Git detecta que el directorio pertenece a un usuario diferente al que lo está ejecutando (por ejemplo, cuando usas `sudo`), lo bloquea por seguridad. 
Agregar el directorio como seguro resuelve esto.

### Ver Sesiones Activas en el Servidor

Durante la configuración se usó `w` para ver quién estaba conectado al servidor en ese momento:

```bash
# Ver usuarios conectados, desde dónde y qué están haciendo
w

# Ver todos los usuarios incluyendo sesiones locales (TTY)
who -a
```

Esto es útil para confirmar que tu sesión SSH está activa y ver si hay otras sesiones abiertas.

---

## Parte 4: Montaje de Disco Secundario (HDD)

### El Problema

El servidor entró en **modo emergencia** al arrancar. Esto sucede cuando Linux no puede montar un disco que está definido en `/etc/fstab` — como medida de protección, 
el sistema se detiene antes de arrancar completamente para evitar corrupción de datos.

> **¿Qué es /etc/fstab?** Es el archivo que le dice a Linux qué discos montar automáticamente al arrancar y en qué punto del sistema de archivos (`/mnt/hdd`, `/data`, etc.).

### La Solución

Se tuvo que reformatear el disco y montarlo correctamente desde cero:

```bash
# Paso 1: Particionar el disco, formatearlo en ext4 y montarlo
# Todo en un solo comando encadenado:
sudo mount /dev/sda1 2>/dev/null
sudo parted /dev/sda --script mklabel gpt mkpart primary ext4 0% 100%
sudo mkfs.ext4 /dev/sda1
sudo mkdir -p /mnt/hdd
sudo mount /dev/sda1 /mnt/hdd
df -h /mnt/hdd

# Paso 2: Obtener el UUID del disco (más confiable que usar /dev/sda1)
sudo blkid /dev/sda1

# Paso 3: Configurar el montaje automático en fstab
sudo nano /etc/fstab
# Se agrega una línea con el UUID del disco:
# UUID=xxxx-xxxx  /mnt/hdd  ext4  defaults  0  2

# Paso 4: Recargar systemd y probar que fstab está correcto
sudo systemctl daemon-reload
sudo mount -a          # monta todo lo que está en fstab
df -h /mnt/hdd         # verificar que el disco está montado

# Paso 5: Verificar todos los discos del sistema
lsblk
```

### Conceptos Importantes

**¿Por qué UUID en lugar de `/dev/sda1`?**
El nombre `/dev/sda1` puede cambiar si se agregan o reordenan los discos. El UUID es un identificador único que no cambia nunca — es la forma correcta de referenciar discos en `fstab`.

**¿Qué es ext4?**
Es el sistema de archivos estándar de Linux. Estable, maduro y compatible con todo el ecosistema Linux.

**¿Qué significa `0  2` al final de la línea en fstab?**
- `0` — no hacer backup con `dump` (generalmente 0 para discos de datos)
- `2` — verificar el disco en el arranque, pero después del disco raíz (`/`)

---

## Resumen: Estado del Servidor Después del Setup Inicial

```
saga (Ubuntu 24.04 LTS)
├── Red configurada via Netplan (WiFi)
├── Tailscale activo (IP: 100.xxx.xx.xxx)
├── Git configurado con credenciales
│   └── Repositorio: DevSecOps-HomeLab sincronizado
└── Disco secundario montado en /mnt/hdd
```

---

## Lecciones Aprendidas

**YAML es estricto con la indentación** — siempre 2 espacios, nunca tabs. Un error de formato y Netplan rechaza la configuración.

**Modo emergencia en Linux** no es un fallo catastrófico — es el sistema protegiéndose. Se entra con la contraseña de root, se corrige el problema en `fstab` o el disco, y se reinicia.

**UUID > nombre de dispositivo** — siempre usar UUID en `fstab` para evitar problemas si los discos cambian de orden.

**`sudo git`** causa problemas de permisos — mejor configurar Git como usuario normal y usar `safe.directory` cuando sea necesario.

---

## Comandos de Referencia Rápida

```bash
# Red
sudo netplan apply                    # aplicar cambios de red
ip a                                  # ver interfaces y IPs
ping 8.8.8.8                         # probar conectividad

# Tailscale
tailscale status                      # ver dispositivos conectados

# Git
git config --global --list            # ver configuración global
git pull                              # sincronizar con GitHub
git add . && git commit -m "msg" && git push  # subir cambios

# Discos
lsblk                                 # ver todos los discos
df -h                                 # ver uso de disco
sudo blkid /dev/sda1                  # obtener UUID de un disco
sudo mount -a                         # montar todo lo de fstab

# Sesiones
w                                     # ver usuarios conectados
who -a                                # ver todas las sesiones
```

---

*Parte de la serie de documentación del proyecto DevSecOps HomeLab.*
