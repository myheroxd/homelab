# El servidor doméstico: un mini PC para todo el año

Documento hermano del [README](README.md), que explica la red, Tailscale y SSH. Aquí va el
equipo que se queda encendido 24/7: qué compré, por qué esas decisiones y cómo está montado.
Igual que el resto, en primera persona y con marcadores en vez de direcciones reales.

---

## 1. El equipo

Un **HP ProDesk 400 G4 Mini** de segunda mano: i5-8500T, 16 GB de RAM, NVMe de 256 GB.

Es el formato que buscaba: bajo consumo, silencioso y con sitio para dos discos. Lo que no
trae es fuente: la alimentación es un adaptador externo de **65 W (19,5 V / 3,33 A)** con
punta de **4,5 × 3,0 mm**, y hay unidades con punta de **7,4 × 5,0 mm**, así que hay que
mirar el conector antes de comprar el cargador.

Detalles que condicionan el diseño:

- **B360**, TPM 2.0, Secure Boot y dos ranuras SO-DIMM DDR4-2666 (hasta 32 GB).
- **Dos DisplayPort 1.2**, ni HDMI ni VGA: para instalarlo hace falta un monitor con DP o un
  adaptador.
- **Sin WiFi**: el módulo era opcional. Va por cable, que es lo que quería.
- Bahía de 2,5" además del M.2, para añadir un SSD SATA si algún día hace falta espacio.

Recepción: diagnóstico **F2**, BIOS **F10** y, antes de nada, comprobar que no haya
contraseña de administrador ni **Absolute Persistence** activo (si el vendedor lo dejó activo
y no se puede desactivar, el equipo puede borrarse en remoto). Después, disco a cero.

---

## 2. Por qué Debian y no Proxmox

Mi primer impulso fue Proxmox: virtualizar todo y tener máquinas separadas. Lo descarté por
tres motivos:

1. Un hipervisor necesita RAM para él y para cada VM; con 16 GB, los contenedores de Docker
   dan más servicios por euro.
2. Todo lo que quiero correr son servicios Linux: no hay nada que necesite una máquina
   entera para él.
3. Menos capas, menos cosas que actualizar y menos que se rompa un domingo.

Lo mismo con Home Assistant: no tengo dispositivos que lo justifiquen, y añadirlo «por si
acaso» es la forma más rápida de acabar con un sistema que no entiendes del todo.

Así que **Debian 13 + Docker**, y cada servicio en su contenedor.

### El disco

256 GB dan para mucho si no se guardan películas:

| Punto | Tamaño | Por qué |
|---|---|---|
| EFI | 1 GB | margen de sobra para actualizaciones del firmware |
| `/` | 50 GB | sistema y Docker; los volúmenes van aparte |
| `/srv` | ~200 GB | datos: mundos, fotos, documentos, copias |
| swap | 4 GB | suficiente; si hiciera falta más, se añade un archivo |

Dejo unos 4 GB sin asignar por el over-provisioning del SSD: es gratis y alarga la vida del
disco.

---

## 3. Dónde va conectado

El reparto de la red está en el README (§2). Resumen del problema: el Livebox está en la
entrada, el Beryl en la habitación y el PC colgando del Beryl.

```
Livebox ── Ethernet ── Mesh (entrada) ── WiFi ── Mesh (habitación)
                                                     ├─ Ethernet ── Beryl ── Ethernet ── PC
                                                     └─ Ethernet ── <SERVIDOR>   ← aquí
```

Lo pongo en el **segundo puerto LAN del nodo mesh de la habitación**:

- Queda en la red del Livebox (`<RED_LIVEBOX>`), con reserva DHCP por MAC. Si no se reserva,
  el primer día que el router reparta esa IP a otro equipo, el reenvío de puertos deja de
  funcionar sin avisar.
- El PC y el servidor comparten el tramo Ethernet del Beryl: el tráfico entre ellos es local.
- Solo el tráfico del servidor hacia internet cruza el enlace inalámbrico entre los dos mesh.

La alternativa era colgarlo del Beryl, pero ese router tiene **un solo puerto LAN** y ya lo
ocupa el PC. Meterlo por WiFi era peor: un servidor de juego con WiFi añade jitter justo
donde más se nota.

---

## 4. El aislamiento, que es la parte difícil

La pregunta incómoda: si alguien consigue ejecutar código en el servidor, ¿puede llegar a mi
PC? Con la red plana, **sí**. Y eso no lo arregla el cortafuegos del router, porque NAT solo
filtra en el sentido contrario.

Lo que hay disponible:

- **Beryl MT3000** (OpenWrt, firmware 4.11): soporta VLANs de verdad. Red principal (VLAN 1,
  `<RED_BERYL>`), invitados (VLAN 9, 192.168.9.0/24) e IoT (VLAN 10, 192.168.10.0/24). El
  puerto LAN admite trunking (*modo VLAN: Standard* y lista de VLANs permitidas).
- **Livebox 6**: red WiFi de invitados, aislada de la red local. Pero es solo WiFi y su radio
  está apagada.

Ninguno de los dos sirve para un servidor **por cable**: el Beryl necesita una VLAN en un
puerto que no tiene libre, y el Livebox solo aísla a quien entra por WiFi.

Así que la contención baja al propio servidor: `DOCKER-USER` corta la salida del contenedor
hacia `10/8`, `172.16/12`, `192.168/16` y `169.254/16`. Sale a internet, no entra en casa.
No detiene una fuga de contenedor con root en el host, pero sí el caso realista: un plugin o
el propio servidor ejecutando código ajeno.

Si algún día quiero que la barrera no dependa del servidor, la vía limpia es un switch
gestionado (~30–40 €) que permita hacer trunk con el Beryl y poner el servidor en su propia
VLAN. Está apuntado en los pendientes, sin prisa.

---

## 5. Qué va a correr

De momento, el servidor de Minecraft y poco más. La idea es no llenarlo el primer día: cada
contenedor es una cosa más que actualizar.

Candidatos, por orden de utilidad:

- **Minecraft** (Purpur sobre el contenedor `itzg/minecraft-server`): es el que motiva el
  proyecto. Con whitelist, `online-mode` y 6 GB de RAM.
- **AdGuard Home**: ya está en el Beryl, no lo duplico.
- **Vaultwarden, Forgejo o Syncthing**: cuando haga falta.
- **Beszel o Uptime Kuma**: para enterarme de que algo se ha caído antes de que me avise un
  amigo.

Los que descarté por hardware: Jellyfin con películas (no hay disco ni GPU), Immich (necesita
mucho espacio y CPU), Nextcloud con Collabora, y cualquier cosa que pida GPU.

---

## 6. El juego, en concreto

- **Whitelist activa** (`white-list` y `enforce-whitelist`): sin ella, un servidor abierto se
  escanea en horas.
- **`online-mode`**: solo cuentas premium. Sin esto, cualquiera puede entrar con cualquier
  nick.
- **RCON activo pero sin publicar**: se usa para congelar el mundo durante las copias. No
  acepto un 25575 abierto a internet.
- **Copias diarias** del mundo, con rotación, y una copia fuera del servidor de vez en cuando.
  Una copia en el mismo disco no protege de un fallo del disco.
- **fail2ban no se usa aquí**: no hay login que proteger. Va para SSH y los paneles web.

Acceso para los amigos: escriben `mc.<dominio>` y entran. Nada de descargas, nada de
inicios de sesión. El registro DNS es **«DNS only»** (nube gris) porque el tráfico del juego
no puede pasar por el proxy de Cloudflare; el nombre se mantiene al día con un script de
DDNS que consulta la IP pública y actualiza el registro A.

---

## 7. Lo que aprendí montándolo

- **El puerto del router es lo fácil; lo difícil es el aislamiento.** Abrir el 25565 son
  cinco minutos. Decidir qué puede hacer un atacante una vez dentro es la conversación de
  verdad.
- **Una reserva DHCP no es burocracia.** Sin ella, el reenvío de puertos funciona hasta que
  deja de funcionar, y el día que falla no hay ningún mensaje de error que lo explique.
- **El firmware del Beryl no expone el canal 149** aunque sea legal en España: el rango que
  ofrece llega hasta el 128. El canal 100 funciona bien mientras no haya radares.
