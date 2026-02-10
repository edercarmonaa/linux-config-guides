# RHEL 10 — NetworkManager: levantar NIC y dejarla persistente con `nmcli`

En instalaciones recientes de RHEL 10 (especialmente en VMs), puede ocurrir que el sistema arranque sin dirección IP o con estados `disconnected` / `unmanaged`. Esto suele estar relacionado con una vinculación incorrecta entre el **perfil de conexión** de NetworkManager y el **dispositivo** (NIC), `autoconnect` deshabilitado o gestión incompleta del device por NetworkManager.

**Objetivo:** identificar la interfaz (ej. `ens160`), corregir la asociación del perfil, habilitar IPv4 por DHCP, desactivar IPv6 si no aplica, y asegurar conectividad **persistente** mediante `connection.autoconnect` y el arranque automático de NetworkManager.

---

## Alcance

- RHEL 10 / clones (Rocky/Alma con NetworkManager)
- NICs típicas en VM: `ens160`, `ens192`, `enp0s3`, etc.
- Método IPv4: DHCP (`ipv4.method auto`)

---

## Prerequisitos

- Acceso con privilegios `sudo`
- Conocer el nombre del **device** (ej. `ens160`)
- Si es VM: verificar que la NIC esté conectada en el hypervisor

---

## Conceptos clave

- **Device:** interfaz real del sistema (ej. `ens160`)
- **Connection (perfil):** configuración guardada en NetworkManager (ej. `Wired connection 1`, `ens160`)

> Importante: el **nombre del perfil** puede no coincidir con el **nombre del device**.  
> Antes de modificar, confirma el `NAME` real con `nmcli connection show`.

---

## Paso 1 — Identificar interfaces (kernel)

```bash
ip link
```

**Validar:**
- La interfaz existe (ej. `ens160`)
- Estado UP/DOWN

---

## Paso 2 — Validar estado en NetworkManager

```bash
nmcli device
```

**Interpretación rápida:**
- `connected`: OK
- `disconnected`: existe, pero sin conexión activa
- `unavailable`: problema de enlace/driver/hypervisor
- `unmanaged`: NetworkManager no gestiona ese device (requiere corrección)

---

## Paso 3 — Listar perfiles de conexión

```bash
nmcli connection show
```

**Recomendado:**
```bash
nmcli -f NAME,UUID,TYPE,DEVICE,STATE connection show
```

**Objetivo:** identificar el `NAME` del perfil asociado a tu device (columna `DEVICE`).

---

## Paso 4 — Confirmar que el device existe

Sustituye `ens160` por tu interfaz real.

```bash
ip link show ens160
```

Si el device no existe, revisar:
- NIC no agregada o no conectada en el hypervisor
- drivers / módulos
- configuración de hardware/VM

---

## Paso 5 — Vincular el perfil al device correcto (clave)

Si tu perfil no se llama `ens160`, usa el `NAME` real detectado en el Paso 3.

```bash
sudo nmcli connection modify ens160 connection.interface-name ens160
```

**Qué resuelve:** perfiles “flotantes” o asociados a otra NIC.

---

## Paso 6 — Configurar IPv4 por DHCP

```bash
sudo nmcli connection modify ens160 ipv4.method auto
```

---

## Paso 7 — Deshabilitar IPv6 (si no aplica)

```bash
sudo nmcli connection modify ens160 ipv6.method ignore
```

> Si tu red usa IPv6, considera `ipv6.method auto` en lugar de `ignore`.

---

## Paso 8 — Aplicar cambios reiniciando la conexión

```bash
sudo nmcli connection down ens160
sudo nmcli connection up ens160
```

### Verificación obligatoria 1 — IP asignada

```bash
ip a show ens160
```

**Validar:**
- Presencia de `inet X.X.X.X/...`
- Estado `UP`

### Verificación obligatoria 2 — Confirmar nombre exacto del perfil

```bash
nmcli connection show
```

Si tu conexión se llama distinto (ej. `Wired connection 1`), ajusta comandos usando ese `NAME`:

```bash
sudo nmcli connection modify "Wired connection 1" ipv4.method auto
```

---

## Paso 9 — Habilitar autoconnect (persistencia)

```bash
sudo nmcli connection modify ens160 connection.autoconnect yes
```

---

## Paso 10 — Asegurar que NetworkManager gestiona el device

```bash
sudo nmcli device set ens160 managed yes
```

Útil cuando `nmcli device` muestra `unmanaged`.

---

## Paso 11 — Habilitar NetworkManager al arranque y reiniciar servicio

```bash
sudo systemctl enable NetworkManager
sudo systemctl restart NetworkManager
```

### Verificación obligatoria 3 — Confirmar persistencia

```bash
nmcli connection show ens160 | grep autoconnect
```

Debe reflejar `yes`.

---

## Prueba final (post-install)

Reiniciar el sistema:

```bash
reboot
```

Al volver, validar IP, rutas y conectividad:

```bash
ip a
ip route
ping -c 3 8.8.8.8
ping -c 3 google.com
```

**Interpretación:**
- `ping 8.8.8.8` OK → conectividad IP/ruta
- `ping google.com` OK → DNS OK
- `8.8.8.8` falla → problema de red/gateway/hypervisor
- `google.com` falla pero `8.8.8.8` OK → problema de DNS

---

## Troubleshooting rápido

**Ver conexiones activas**
```bash
nmcli connection show --active
```

**Ver logs de NetworkManager (boot actual)**
```bash
journalctl -u NetworkManager -b --no-pager | tail -200
```

**Forzar reconexión (si aplica)**
```bash
sudo nmcli device disconnect ens160
sudo nmcli device connect ens160
```

---

## Notas

- En VMs, validar que la NIC esté “Connected” y el tipo de red (NAT/Bridge) sea el correcto.
- Si cambiaste de NIC o el nombre del device cambió, revisa `connection.interface-name`.
