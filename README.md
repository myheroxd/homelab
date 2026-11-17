# Homelab doméstico: red, Tailscale y SSH

Documento donde recojo cómo monté la red de casa, la red privada de Tailscale y el acceso SSH entre el PC de casa, el móvil y el PC del aula. Está escrito en primera persona, con las decisiones que tomé y los problemas que encontré.

> **Convenciones:** las direcciones, usuarios y nombres de equipo están sustituidos por marcadores (`<IP_PC>`, `<usuario>`, `<dominio>`…). La leyenda está en la sección 11.

---

## 1. Qué quería conseguir

Tres cosas:

1. Entrar al PC de casa desde el móvil y desde el PC del aula, sin abrir puertos en el router.
2. Que solo entrara yo: nada de contraseñas, claves y un cortafuegos que limite el acceso.
3. Que funcionara también en la red del aula, que es la más restrictiva.

La respuesta fueron dos caminos: **Tailscale** para casa y el móvil, y **Cloudflare Tunnel** para el aula (documentado aparte).

---

## 2. La red de casa

```
Internet (<operador>, 600 Mb simétricos, GPON)
        │
   Livebox 6  ── <IP_LIVEBOX> (DHCP 10-150)
        │  Ethernet
   Mesh ZTE H196Q (principal)  ── WiFi 5 GHz
        │  enlace inalámbrico
   Mesh ZTE H196Q (habitación)
        │  Ethernet
   Beryl MT3000  ── WAN <IP_BERYL_WAN> (reservada)
        │            LAN <IP_BERYL_LAN>
   PC de casa  ── <IP_PC> (Ethernet USB-C)
```

- **Livebox 6**: router del operador. No lo sustituí; no quiero tocar la instalación mientras funcione. Tiene los puertos LAN libres y el servidor del homelab irá ahí por Ethernet.
- **Mesh ZTE**: dos nodos, uno conectado por cable al Livebox y otro inalámbrico en mi habitación. Mantengo la red tal cual.
- **Beryl MT3000**: router de viaje GL.iNet con OpenWrt. Lo uso como router de la habitación: reparte la red <RED_BERYL> y da DNS con AdGuard Home.
- **PC de casa**: CachyOS, conectado al Beryl por Ethernet USB-C.

### Direcciones

| Equipo | Red | Dirección |
|---|---|---|
| Livebox | <RED_LIVEBOX> | <IP_LIVEBOX> |
| Beryl (WAN) | <RED_LIVEBOX> | <IP_BERYL_WAN> (reserva DHCP) |
| Beryl (LAN) | <RED_BERYL> | <IP_BERYL_LAN> |
| PC de casa | <RED_BERYL> | <IP_PC> |
| PC de casa (Tailscale) | 100.64.0.0/10 | <IP_TAILSCALE_PC> |
| Móvil (Tailscale) | 100.64.0.0/10 | <IP_TAILSCALE_MOVIL> |

---

## 3. Tailscale

### 3.1 En el PC de casa

```sh
sudo pacman -S tailscale
sudo systemctl enable --now tailscaled
sudo tailscale up
tailscale ip -4
tailscale status
```

`tailscale up` abre el navegador para autenticar la cuenta. La IP que asigna (`100.x`) solo existe dentro de la red privada.

### 3.2 En el móvil

Instalé la aplicación Tailscale, inicié sesión con la misma cuenta y activé la conexión. En Termux se puede consultar la IP con `tailscale ip -4`.

### 3.3 Política de acceso

En la consola de Tailscale definí una ACL con sintaxis `grants`. La idea:

- Mi usuario puede entrar al PC de casa y al móvil.
- El aula (`tag:classroom`) solo puede abrir el puerto 22 del PC de casa y el 8022 del móvil.
- Nada de acceso del aula al resto de la red.

```jsonc
// Esquema simplificado; la política real se edita en la consola de Tailscale.
{
  "tagOwners": { "tag:classroom": ["autogroup:admin"] },
  "grants": [
    { "src": ["autogroup:member"], "dst": ["tag:classroom"], "ip": ["22"] },
    { "src": ["tag:classroom"], "dst": ["<IP_TAILSCALE_PC>"], "ip": ["22"] },
    { "src": ["tag:classroom"], "dst": ["<IP_TAILSCALE_MOVIL>"], "ip": ["8022"] }
  ]
}
```

> Las claves de autenticación del aula son **efímeras** y con `tag:classroom`: cuando el equipo se apaga o se retira el nodo, desaparece de la red.

### 3.4 El problema del aula

La red del centro **bloquea Tailscale**. Desde el aula, `tailscale up` no completa y el tráfico `100.x` no sale. Por eso el aula va por Cloudflare Tunnel, que usa el puerto 443.

---

## 4. SSH

### 4.1 PC de casa

`sshd` acepta conexiones y lo endurecí para que solo entren claves:

```sh
sudo systemctl enable --now sshd
```

`/etc/ssh/sshd_config.d/00-hardening.conf`:

```
PasswordAuthentication no
KbdInteractiveAuthentication no
PermitRootLogin no
AllowUsers <usuario>
```

Y en el cortafuegos, solo por Tailscale:

```sh
sudo ufw allow in on tailscale0 to any port 22 proto tcp
```

Comprobación:

```sh
ss -tlnp | grep :22
systemctl status sshd --no-pager
```

### 4.2 Móvil con Termux

```sh
pkg install openssh
ssh-keygen -t ed25519 -C "movil"
sshd
termux-wake-lock
```

`sshd` en Termux escucha en el **8022** y se inicia a mano; `termux-wake-lock` evita que Android lo suspenda.

### 4.3 PC del aula

Ubuntu, con `openssh-server` instalado y la clave `aula` en un USB. La clave privada no se queda en el aula: se copia a `/tmp`, se ajustan permisos y se borra al salir.

```sh
cp /media/$USER/USB/aula /tmp/aula
chmod 600 /tmp/aula
ssh -i /tmp/aula <usuario>@<IP_TAILSCALE_PC>
```

El USB es exFAT y no guarda permisos (`0755`), y OpenSSH se niega a usar una clave con permisos abiertos; de ahí la copia a `/tmp`. Al terminar:

```sh
rm -f /tmp/aula
ssh-keygen -R ssh.<dominio>
rm -f ~/.ssh/config
rm -rf ~/.cloudflared
history -c && rm -f ~/.bash_history
```

### 4.4 Atajos

`~/.ssh/config` en cada equipo:

```
# PC de casa
Host termux
    HostName <IP_TAILSCALE_MOVIL>
    User <usuario_termux>
    Port 8022
```

```
# Móvil
Host casa
    HostName <IP_TAILSCALE_PC>
    User <usuario>
```

```
# Aula
Host casa-cf
    HostName ssh.<dominio>
    User <usuario>
    ProxyCommand cloudflared access ssh --hostname %h
```

### 4.5 Claves

| Clave | Dónde está la privada | Para qué |
|---|---|---|
| `id_ed25519` | PC de casa (`~/.ssh`) | Entrar al móvil |
| `id_ed25519` (móvil) | Termux | Entrar al PC |
| `aula` | USB y copia en `~/.ssh/aula` | Entrar desde el aula |
| `deploy_web` | PC de casa (`~/.ssh`) | Desplegar `jesusefx-web` en GitHub |
| `deploy_msx` | PC de casa (`~/.ssh`) | Desplegar `MSX-Hosting` en GitHub |
| `deploy_homelab` | PC de casa (`~/.ssh`) | Desplegar este repositorio |

Las públicas están en el `authorized_keys` del equipo de destino. La del aula va con restricciones (`no-port-forwarding`, `no-agent-forwarding`, `no-X11-forwarding`).

Las tres `deploy_*` son **deploy keys de GitHub**, sin passphrase y limitadas a un repositorio cada una. Una deploy key no se puede reutilizar en dos repositorios: GitHub responde «Key already in use». Si se compromete el PC, se revocan desde GitHub → Settings → Deploy keys.

---

## 5. El Beryl MT3000

- Firmware GL.iNet 4.11.0 sobre OpenWrt 21.02.
- **AdGuard Home** activo como DNS de la red <RED_BERYL>, con dos listas (AdGuard DNS y OISD Small).
- **WPA3-SAE** en 5 GHz, canal 100, 80 MHz (el canal 149 no está disponible).
- Filtro `filter_aaaa` activado para evitar problemas con IPv6 en algunos servicios.
- Reserva DHCP para el PC (<IP_PC>).

Comandos útiles:

```sh
ssh root@<IP_BERYL_LAN>
/etc/init.d/dnsmasq restart
/etc/init.d/adguardhome restart
```

> Al reiniciar AdGuard aparecen avisos de `fw3` y `ETHERNET_TTL`. Son del firmware GL.iNet, no un fallo del cortafuegos.

---

## 6. El aula y Cloudflare

El aula bloquea Tailscale, así que monté un túnel de Cloudflare con el dominio `<dominio>`:

- `ssh.<dominio>` → túnel `ssh-casa` → `ssh://localhost:22`.
- Access delante, con PIN por correo.
- En el aula, `cloudflared access ssh` establece un WebSocket contra el túnel.

El montaje completo está en el otro documento del repositorio del sitio.

---

## 7. Incidentes y cómo los resolví

| Problema | Causa | Solución |
|---|---|---|
| `ssh termux` daba «Connection refused» | Android había dormido Termux | `sshd` + `termux-wake-lock` |
| `ssh casa` no conectaba | La app Tailscale del móvil estaba desconectada | Activarla |
| OpenSSH rechazaba la clave del USB | exFAT monta con 0755 | Copiar a `/tmp` y `chmod 600` |
| `~/.ssh/config` no funcionaba | Era un **directorio** con permisos 600 | Borrarlo y crearlo como archivo |
| `scp` fallaba con `sftp-server: not found` | El servidor no tenía el binario | `scp -O` o `ssh ... "cat" > local` |
| `sshd` parado en el PC | Lo detuve en unas pruebas | `systemctl start sshd` |
| El aula no resolvía `ssh.<dominio>` | Caché del operador | Esperar o fijar la IP en `/etc/hosts` |
| `ssh casa-cf` con «bad handshake» | Un comodín DNS del Worker `web` interceptaba `ssh.` | Quitar el dominio `*.<dominio>` |

---

## 8. Seguridad

- SSH solo con claves; contraseñas deshabilitadas y `root` sin acceso.
- El puerto 22 del PC solo se acepta por `tailscale0`.
- Clave distinta para el aula, con passphrase y restricciones, y sin copiarla a disco.
- Los nodos del aula son efímeros y con etiqueta.
- En el aula, al salir: borrar clave, configuración, token e historial.
- No abro puertos en el router; el túnel es saliente.

---

## 9. Inventario

### Equipos

| Equipo | Sistema | Usuario | SSH |
|---|---|---|---|
| PC de casa | CachyOS | `<usuario>` | 22 (solo Tailscale) |
| Móvil | Android + Termux | `<usuario_termux>` | 8022 |
| Aula | Ubuntu | `<usuario_aula>` | cliente |

### Servicios

| Servicio | Dónde | Arranque |
|---|---|---|
| `sshd` | PC de casa | `sudo systemctl start sshd` |
| `tailscaled` | PC de casa | `sudo systemctl start tailscaled` |
| `cloudflared-ssh` | PC de casa | `sudo systemctl start cloudflared-ssh` |
| `sshd` | Termux | `sshd` / `pkill sshd` |
| `ssh` | Aula | `sudo systemctl start ssh` |

---

## 10. Mantenimiento

```sh
# Estado
systemctl is-active sshd tailscaled cloudflared-ssh
tailscale status
ss -tlnp | grep :22

# IPs
tailscale ip -4
ip addr show

# Túnel
cloudflared tunnel info ssh-casa
journalctl -u cloudflared-ssh -n 30 --no-pager

# Limpiar caché de DNS
sudo resolvectl flush-caches
```

### Publicar este documento

```sh
cd ~/Documentos/deepseek-harness/default-workspace
./deploy-homelab.sh "Actualizar la red de casa"
```

Hace commit y push con la deploy key `deploy_homelab`, sin depender del agente SSH.

### Rotar claves

1. Generar otra (`ssh-keygen -t ed25519`).
2. Copiar la pública al `authorized_keys` del destino.
3. Probar.
4. Borrar la antigua del `authorized_keys`.

Para las deploy keys: borrar la clave en GitHub → Deploy keys, generar otra, añadirla al repositorio y actualizar el script que la use.

---

## 11. Leyenda de marcadores

| Marcador | Qué representa |
|---|---|
| `<dominio>` | Mi dominio (`ssh.<dominio>` es el hostname del túnel) |
| `<hostname_pc>` | Nombre del PC de casa |
| `<usuario>` | Mi usuario del PC |
| `<usuario_termux>` | Usuario de Termux |
| `<usuario_aula>` | Usuario del PC del aula |
| `<IP_PC>` | IP del PC en la red del Beryl |
| `<IP_BERYL_LAN>` | IP del Beryl en su red |
| `<IP_BERYL_WAN>` | IP del Beryl en la red del Livebox |
| `<IP_LIVEBOX>` | IP del Livebox |
| `<IP_TAILSCALE_PC>` | IP del PC en Tailscale |
| `<IP_TAILSCALE_MOVIL>` | IP del móvil en Tailscale |
| `<RED_BERYL>` | Subred del Beryl (`192.168.8.0/24`) |
| `<RED_LIVEBOX>` | Subred del Livebox (`192.168.1.0/24`) |
| `<operador>` | Compañía de internet |

Las dos subredes aparecen como rango porque son direcciones privadas estándar (RFC 1918): sirven para entender la topología sin revelar direcciones concretas.

---

## 12. Pendientes

- Montar el homelab con Proxmox en un mini PC reacondicionado.
- Pasar Home Assistant, Docker y el laboratorio de redes a ese equipo.
- Copias automáticas al disco USB.
- Valorar una ONT propia y un router neutro más adelante, sin prisa.
- Montar un equipo de servicios en casa, cuando haya un mini PC encendido todo el día.

---

El montaje del dominio y Cloudflare está en [jesusefx-web](https://github.com/myheroxd/jesusefx-web).
