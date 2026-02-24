# ansible-dhcp-dns

Playbook de Ansible **idempotente** para instalar y configurar **DNS (BIND/named)** + **DHCP (dhcpd)** en Rocky Linux 9.

---

## Árbol del repositorio

```
ansible-dhcp-dns/
├── ansible.cfg
├── inventory/
│   └── hosts.ini
├── group_vars/
│   └── all.yml
├── site.yml
├── roles/
│   ├── dns_bind/
│   │   ├── defaults/main.yml
│   │   ├── handlers/main.yml
│   │   ├── tasks/main.yml
│   │   └── templates/
│   │       ├── named.conf.j2
│   │       ├── zone.forward.j2
│   │       └── zone.reverse.j2
│   └── dhcpd/
│       ├── defaults/main.yml
│       ├── handlers/main.yml
│       ├── tasks/main.yml
│       └── templates/
│           ├── dhcpd.conf.j2
│           └── dhcpd_override.conf.j2
├── .gitignore
└── .github/
    └── workflows/
        └── ci.yml
```

---

## Requisitos

| Herramienta | Versión mínima |
|---|---|
| Ansible Core | ≥ 2.15 |
| Python | ≥ 3.9 |
| Colección `ansible.posix` | ≥ 1.5 |
| Vagrant | ≥ 2.3 |
| VirtualBox | ≥ 7.0 |

---

## Diseño de red del laboratorio

| Recurso | Valor | Motivo |
|---|---|---|
| Dominio | `jmelol.lab` | Dominio interno del lab |
| Red | `192.168.100.0/24` | Red de la `lab-red` de Vagrant |
| Gateway | `192.168.100.1` | Primera IP utilizable — convención estándar para routers |
| Servidor DNS/DHCP | `192.168.100.10` | Rango `.1–.10` reservado para infraestructura |
| Rango DHCP dinámico | `.100 – .200` | Deja espacio para IPs estáticas (`.11–.99`) y crecimiento |
| Forwarders | `8.8.8.8`, `8.8.4.4` | DNS públicos de Google |

### Convenciones del espacio de direcciones

```
192.168.100.0     → Dirección de red (no asignable)
192.168.100.1     → Gateway / router
192.168.100.2–9   → Infraestructura de red reservada
192.168.100.10    → Servidor DNS + DHCP (ns1.jmelol.lab)
192.168.100.11–99 → Servidores con IP fija manual
192.168.100.100–200 → Pool DHCP dinámico
192.168.100.201–254 → Reserva futura
192.168.100.255   → Broadcast
```

---

## Interfaz DHCP

**Problema:** El servidor tiene dos interfaces:

- `eth0` — NAT de Vagrant (para acceso a internet y SSH del propio Vagrant)
- `eth1` — Red interna `lab-red` (donde debe escuchar DHCP)

**Método elegido: systemd drop-in (`override.conf`)**

Se crea `/etc/systemd/system/dhcpd.service.d/override.conf` que pasa `eth1` como argumento al binario. Esto es el método idiomático en RHEL 8/9 y es estable con cualquier gestor de nombres de interfaz.

**¿Cómo sé que la interfaz se llama `eth1`?**

La box `generic/rocky9` de Roboxes deshabilita el naming predictable de systemd-networkd, por lo que las interfaces mantienen los nombres clásicos `eth0`, `eth1`. Verifica con:

```bash
vagrant ssh server
ip -br link show
# Salida esperada:
# lo     UNKNOWN
# eth0   UP    10.0.2.x/24    ← NAT Vagrant
# eth1   UP    192.168.100.10/24 ← lab-red
```

Si el nombre fuera diferente (ej. `enp0s8`), actualiza `dhcp_interface` en `group_vars/all.yml`.

---

## Consideraciones SELinux

Rocky Linux 9 ejecuta SELinux en modo **enforcing** por defecto. El playbook:

1. Instala `python3-libsemanage` (requerido por el módulo `ansible.posix.seboolean`)
2. Activa el booleano `named_write_master_zones` (permite a named escribir archivos de zona)

Los puertos estándar 53 (DNS) y 67 (DHCP) ya tienen etiquetas SELinux correctas en la política de stock (`dns_port_t`, `dhcp_port_t`). No se requieren modificaciones adicionales.

---

## Variables personalizables

Todas las variables se definen en `group_vars/all.yml`. Las más relevantes:

```yaml
# Dominio y red
lab_domain:    "jmelol.lab"
lab_network:   "192.168.100.0"
lab_server_ip: "192.168.100.10"
lab_gateway:   "192.168.100.1"

# DNS
dns_zone_serial: "2025062401"   # INCREMENTAR en cada cambio de zona
dns_forwarders:  ["8.8.8.8", "8.8.4.4"]
dns_a_records:   [...]          # Añadir nuevos hosts aquí

# DHCP
dhcp_subnets:      [...]
dhcp_reservations: []           # Reservas por MAC
dhcp_interface:    "eth1"
```

### Añadir una reserva por MAC

```yaml
dhcp_reservations:
  - hostname: "impresora"
    mac:      "08:00:27:aa:bb:cc"
    ip:       "192.168.100.60"
```

---

## Paso a paso de instalación (desde cero)

> Para alguien que tiene Vagrant y el Vagrantfile pero no ha usado Ansible.

### Paso 1 — Instalar dependencias en tu máquina física (host)

**Windows (PowerShell como Administrador):**
```powershell
# Instalar Chocolatey si no lo tienes
Set-ExecutionPolicy Bypass -Scope Process -Force
[System.Net.ServicePointManager]::SecurityProtocol = [System.Net.ServicePointManager]::SecurityProtocol -bor 3072
iex ((New-Object System.Net.WebClient).DownloadString('https://community.chocolatey.org/install.ps1'))

# Instalar Python y Git
choco install python git -y
```

**macOS:**
```bash
# Instalar Homebrew si no lo tienes
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
brew install python git
```

**Linux (Ubuntu/Debian):**
```bash
sudo apt update && sudo apt install python3 python3-pip git -y
```

### Paso 2 — Instalar Ansible en tu máquina física

```bash
pip3 install --user ansible ansible-lint
# Verificar
ansible --version
```

### Paso 3 — Instalar las colecciones Ansible requeridas

```bash
ansible-galaxy collection install ansible.posix community.general
```

### Paso 4 — Crear la estructura del proyecto

```bash
# Crear carpeta del proyecto (donde está o estará el Vagrantfile)
mkdir -p ~/lab-dns-dhcp
cd ~/lab-dns-dhcp

# El Vagrantfile ya debe estar aquí, o créalo/pégalo
```

### Paso 5 — Clonar / copiar este repositorio

```bash
# Opción A: clonar desde GitHub
git clone https://github.com/TU_USUARIO/ansible-dhcp-dns.git
cd ansible-dhcp-dns

# Opción B: copiar los archivos manualmente en la estructura indicada
```

**Estructura final esperada en tu disco:**
```
~/lab-dns-dhcp/
├── Vagrantfile          ← tu Vagrantfile original
└── ansible-dhcp-dns/   ← este repositorio
    ├── ansible.cfg
    ├── inventory/
    ├── group_vars/
    ├── site.yml
    └── roles/
```

### Paso 6 — Levantar las VMs con Vagrant

```bash
cd ~/lab-dns-dhcp

# Levantar las dos máquinas (tarda 3-5 minutos la primera vez)
vagrant up

# Verificar que ambas están corriendo
vagrant status
# Salida esperada:
# server   running (virtualbox)
# client   running (virtualbox)
```

### Paso 7 — Verificar la interfaz de red del servidor

```bash
vagrant ssh server
ip -br link show
# Anota el nombre de la interfaz con IP 192.168.100.10
# Normalmente: eth1
exit
```

Si el nombre es diferente a `eth1`, edita `group_vars/all.yml`:
```yaml
dhcp_interface: "enp0s8"   # ajusta al nombre real
```

### Paso 8 — Verificar que Ansible llega al servidor

```bash
cd ~/lab-dns-dhcp/ansible-dhcp-dns

# Ping de Ansible al servidor
ansible dns_dhcp_servers -m ping

# Salida esperada:
# ns1.jmelol.lab | SUCCESS => { "ping": "pong" }
```

Si falla, verifica la ruta de la clave SSH:
```bash
ls -la .vagrant/machines/server/virtualbox/private_key
```

### Paso 9 — Ejecutar en modo check (dry-run, sin cambios)

```bash
ansible-playbook site.yml --check --diff
```

Revisa la salida. Los cambios que se aplicarían aparecen en verde/amarillo.

### Paso 10 — Ejecutar el playbook

```bash
ansible-playbook site.yml
```

Espera una salida similar a:
```
PLAY RECAP
ns1.jmelol.lab : ok=22   changed=12   unreachable=0   failed=0
```

### Paso 11 — Idempotencia (volver a ejecutar)

```bash
# Segunda ejecución — debe mostrar changed=0
ansible-playbook site.yml
```

### Paso 12 — Verificar desde el cliente

```bash
vagrant ssh client

# Ver si recibió IP por DHCP en eth1
ip addr show eth1

# Probar DNS
dig ns1.jmelol.lab @192.168.100.10
nslookup ns1.jmelol.lab 192.168.100.10

# Probar resolución inversa
dig -x 192.168.100.10 @192.168.100.10
```

---

## Checklist de verificación

### En tu máquina física (host)

```bash
# 1. Lint del playbook
ansible-lint site.yml

# 2. Syntax check
ansible-playbook site.yml --syntax-check

# 3. Dry-run con diff
ansible-playbook site.yml --check --diff

# 4. Ejecución completa
ansible-playbook site.yml

# 5. Idempotencia (debe dar changed=0)
ansible-playbook site.yml
```

### En el servidor (vagrant ssh server)

```bash
# Validar named.conf
named-checkconf /etc/named.conf

# Validar zona forward
named-checkzone jmelol.lab /var/named/jmelol.lab.zone

# Validar zona reverse
named-checkzone 100.168.192.in-addr.arpa /var/named/100.168.192.in-addr.arpa.zone

# Validar dhcpd.conf
dhcpd -t -cf /etc/dhcp/dhcpd.conf

# Estado de servicios
systemctl status named dhcpd

# Puertos abiertos
ss -tulnp | grep -E '(53|67)'
firewall-cmd --list-ports

# Leases activos (después de que el cliente pida IP)
cat /var/lib/dhcpd/dhcpd.leases
```

### En el cliente (vagrant ssh client)

```bash
# IP asignada por DHCP
ip addr show eth1
ip route show

# Resolución DNS directa
dig ns1.jmelol.lab @192.168.100.10

# Resolución inversa
dig -x 192.168.100.10 @192.168.100.10

# Con systemd-resolved (si está activo)
resolvectl query ns1.jmelol.lab
```

---

## Despliegue en Entornos Híbridos (WSL2 + Windows + VirtualBox)

Este laboratorio fue adaptado exitosamente para sortear las limitaciones de red y permisos entre WSL2 y Windows Host.

### 1. Entorno de Python (PEP 668)
En Ubuntu 24.04 (WSL), no es posible instalar Ansible globalmente con `pip`. Se debe aislar el proyecto:
```bash
python3 -m venv .venv
source .venv/bin/activate
pip install ansible ansible-lint
ansible-galaxy collection install ansible.posix community.general```

### 2. Localhost y el Inventario SSH

Vagrant reenvía los puertos de la VM (2222 para server, 2223 para client) a la interfaz 127.0.0.1 de Windows. Sin embargo, WSL2 tiene su propio 127.0.0.1 aislado.
Para que Ansible (en Linux) conecte con las VMs, el archivo inventory/hosts.ini debe apuntar a la IP del adaptador vEthernet (WSL) en Windows, no al localhost:

```
[dns_dhcp_servers]
ns1.jmelol.lab ansible_host=TU_IP_WINDOWS ansible_port=2222 ansible_user=vagrant ansible_ssh_private_key_file=../.vagrant/machines/server/virtualbox/private_key
```

### 3. Actualización de Ansible Facts (v2.24+)

El código de este repositorio utiliza la sintaxis moderna de Ansible `ansible_facts['variable']` en lugar de las variables inyectadas estáticamente (ej. `ansible_distribution`), garantizando compatibilidad futura y cero advertencias de depreciación (Deprecation Warnings) en la ejecución.


---

## Licencia

MIT
