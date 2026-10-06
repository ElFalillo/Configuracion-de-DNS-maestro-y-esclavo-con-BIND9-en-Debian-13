# Configuración de DNS maestro y esclavo con BIND9 en Debian 13

Guía práctica para configurar un servidor DNS **maestro/primario** y un servidor DNS **esclavo/secundario** con BIND9 en Debian GNU/Linux 13 (Trixie).

El objetivo es disponer de dos servidores autoritativos para la zona `haven.local`:

- El servidor maestro mantiene los archivos originales de zona.
- El servidor esclavo descarga y sincroniza automáticamente las zonas del maestro.
- Ambos servidores pueden resolver nombres directos e inversos.
- Las transferencias de zona se limitan al esclavo autorizado.

> [!WARNING]
> Esta guía usa `haven.local` porque es el dominio empleado en la práctica. El sufijo `.local` está reservado para Multicast DNS (mDNS), por lo que no es recomendable para producción. Para un laboratorio funciona si se consulta BIND explícitamente usando la IP del servidor DNS.

---

## Índice

- [Topología](#topología)
- [Conceptos básicos](#conceptos-básicos)
- [Requisitos previos](#requisitos-previos)
- [Parte A: servidor maestro](#parte-a-servidor-maestro)
- [Parte B: servidor esclavo](#parte-b-servidor-esclavo)
- [Prueba de sincronización](#prueba-de-sincronización)
- [Fallos comunes](#fallos-comunes)
- [Checklist final](#checklist-final)

---

## Topología

| Equipo | Rol | Nombre DNS | Dirección IP |
|---|---|---|---:|
| VM 1 | DNS maestro | `raul.haven.local` | `192.168.1.100` |
| VM 2 | DNS esclavo | `dns2.haven.local` | `192.168.1.101` |

Configuración utilizada:

```text
Zona directa: haven.local
Zona inversa: 1.168.192.in-addr.arpa
Red de laboratorio: 192.168.1.0/24
Puerto DNS: 53 TCP/UDP
```

La zona inversa se genera invirtiendo los tres primeros octetos de una red IPv4 `/24`:

```text
192.168.1.0/24 → 1.168.192.in-addr.arpa
```

---

## Conceptos básicos

| Concepto | Explicación |
|---|---|
| DNS maestro o primario | Servidor que mantiene los archivos originales de la zona. Las modificaciones se hacen aquí. |
| DNS esclavo o secundario | Servidor que descarga una copia de las zonas desde el maestro y responde consultas con ella. |
| Zona directa | Permite resolver un nombre hacia una IP, por ejemplo: `raul.haven.local → 192.168.1.100`. |
| Zona inversa | Permite resolver una IP hacia un nombre, por ejemplo: `192.168.1.100 → raul.haven.local`. |
| Registro `A` | Relaciona un nombre con una dirección IPv4. |
| Registro `PTR` | Relaciona una dirección IP con un nombre de dominio. Se usa en la resolución inversa. |
| Registro `NS` | Declara los servidores DNS autoritativos de una zona. |
| Registro `SOA` | Contiene la información principal de la zona, incluido el número serial. |
| Transferencia de zona | Proceso mediante el cual el esclavo descarga una copia de una zona desde el maestro. |
| Serial | Número de versión de la zona. Debe aumentar cuando se modifica un archivo de zona. |

---

## Requisitos previos

Necesitas:

- Dos máquinas virtuales con Debian GNU/Linux 13.
- BIND9 instalado en ambas máquinas.
- Acceso mediante un usuario con `sudo`.
- Dos adaptadores de red por VM:
  - Adaptador 1: NAT para acceso a Internet.
  - Adaptador 2: la misma red privada para conectar maestro y esclavo.

Ejemplo en VirtualBox:

| Adaptador | Maestro | Esclavo | Propósito |
|---|---|---|---|
| Adaptador 1 | NAT | NAT | Acceso a Internet e instalación de paquetes |
| Adaptador 2 | Misma red interna, NAT Network o host-only | Misma red interna, NAT Network o host-only | Comunicación entre los DNS |

> [!IMPORTANT]
> Ambas VMs deben usar **el mismo tipo de red y el mismo nombre de red** en el segundo adaptador. Si una usa una red interna llamada `red-dns` y otra usa `red-practica`, no podrán comunicarse aunque sus IP pertenezcan al mismo rango.

---

# Parte A: servidor maestro

## 1. Configurar la red del maestro

Archivo:

```text
/etc/network/interfaces
```

Configuración de ejemplo:

```conf
source /etc/network/interfaces.d/*

auto lo
iface lo inet loopback

# Adaptador NAT: acceso a Internet
allow-hotplug enp0s3
iface enp0s3 inet dhcp

# Red privada del laboratorio DNS
auto enp0s8
iface enp0s8 inet static
    address 192.168.1.100
    netmask 255.255.255.0
    broadcast 192.168.1.255
```

Aplicar los cambios:

```bash
sudo ifdown --force enp0s8
sudo ifup --force enp0s8
```

Comprobar la IP:

```bash
ip addr show enp0s8
```

Resultado esperado:

```text
inet 192.168.1.100/24
```

### ¿Qué se está haciendo?

Se asigna una IP fija al maestro. Es importante que un DNS tenga una dirección estable, porque el esclavo y los clientes deben saber siempre dónde consultar las zonas.

---

## 2. Instalar BIND9 en el maestro

```bash
sudo apt update
sudo apt install bind9 bind9-utils bind9-dnsutils
```

Comprobar que el servicio existe:

```bash
sudo systemctl status bind9
```

### ¿Qué se está instalando?

| Paquete | Función |
|---|---|
| `bind9` | Servicio DNS principal. |
| `bind9-utils` | Utilidades de validación como `named-checkconf` y `named-checkzone`. |
| `bind9-dnsutils` | Herramientas de consulta como `dig` y `nslookup`. |

---

## 3. Configurar opciones generales

Archivo:

```text
/etc/bind/named.conf.options
```

Contenido:

```conf
acl "safeclients" {
    localhost;
    192.168.1.0/24;
};

options {
    directory "/var/cache/bind";

    recursion yes;
    allow-recursion { safeclients; };
    allow-query { safeclients; };
    allow-query-cache { safeclients; };

    listen-on port 53 { 127.0.0.1; 192.168.1.100; };
    listen-on-v6 { none; };

    allow-transfer { 192.168.1.101; };

    forwarders {
        1.1.1.1;
        9.9.9.9;
    };
};
```

### Explicación

| Directiva | Función |
|---|---|
| `acl "safeclients"` | Define los equipos permitidos para realizar consultas. |
| `recursion yes` | Permite resolver dominios externos. |
| `allow-recursion` | Limita quién puede usar la resolución recursiva. |
| `allow-query` | Limita los clientes que pueden consultar el DNS. |
| `listen-on` | Indica las direcciones IP donde BIND escucha consultas DNS. |
| `allow-transfer` | Permite que el esclavo `192.168.1.101` descargue las zonas. |
| `forwarders` | DNS públicos a los que se reenvían consultas externas. |

> [!CAUTION]
> No uses `allow-transfer { any; };`. Esto permitiría que otros equipos descarguen la información de tus zonas DNS.

---

## 4. Declarar las zonas en el maestro

Archivo:

```text
/etc/bind/named.conf.local
```

Contenido:

```conf
zone "haven.local" {
    type primary;
    file "/etc/bind/zones/db.haven.local";
};

zone "1.168.192.in-addr.arpa" {
    type primary;
    file "/etc/bind/zones/db.1.168.192";
};
```

Crear el directorio de zonas:

```bash
sudo mkdir -p /etc/bind/zones
```

### ¿Qué se está haciendo?

Se definen dos zonas:

- `haven.local`: zona directa.
- `1.168.192.in-addr.arpa`: zona inversa para `192.168.1.0/24`.

El tipo `primary` indica que esta máquina mantiene los archivos originales. También puede aparecer como `master` en configuraciones antiguas o compatibles.

---

## 5. Crear la zona directa

Archivo:

```text
/etc/bind/zones/db.haven.local
```

Contenido:

```dns
$TTL 604800

@   IN  SOA raul.haven.local. hostmaster.haven.local. (
        2026100601 ; Serial
        12h        ; Refresh
        15m        ; Retry
        3w         ; Expire
        2h         ; Negative Cache TTL
)

; Servidores DNS autoritativos
@       IN      NS      raul.haven.local.
@       IN      NS      dns2.haven.local.

; Registros A de servidores y hosts
raul    IN      A       192.168.1.100
ns      IN      A       192.168.1.100
dns2    IN      A       192.168.1.101
```

### Explicación de los registros

| Registro | Significado |
|---|---|
| `@` | Representa el dominio principal de la zona: `haven.local`. |
| `SOA` | Define el servidor principal, el contacto técnico y los temporizadores de la zona. |
| `NS` | Declara que `raul.haven.local` y `dns2.haven.local` son DNS autoritativos. |
| `raul IN A` | Crea `raul.haven.local → 192.168.1.100`. |
| `ns IN A` | Crea un alias opcional: `ns.haven.local → 192.168.1.100`. |
| `dns2 IN A` | Crea `dns2.haven.local → 192.168.1.101`. |

> [!IMPORTANT]
> Los puntos finales en `raul.haven.local.` y `dns2.haven.local.` son importantes. Indican que son nombres completos y evitan que BIND añada automáticamente `.haven.local`.

### Sobre el serial

El serial identifica la versión de la zona:

```dns
2026100601
```

Cada vez que modifiques los registros de esta zona, aumenta el serial:

```dns
2026100601
```

por:

```dns
2026100602
```

El esclavo usa este número para detectar que hay cambios y descargar una copia actualizada.

---

## 6. Crear la zona inversa

Archivo:

```text
/etc/bind/zones/db.1.168.192
```

Contenido:

```dns
$TTL 604800

@   IN  SOA raul.haven.local. hostmaster.haven.local. (
        2026100601 ; Serial
        12h        ; Refresh
        15m        ; Retry
        3w         ; Expire
        2h         ; Negative Cache TTL
)

; Servidores DNS autoritativos
@       IN      NS      raul.haven.local.
@       IN      NS      dns2.haven.local.

; Resolución inversa
100     IN      PTR     raul.haven.local.
101     IN      PTR     dns2.haven.local.
```

### ¿Qué se está haciendo?

Los registros `PTR` permiten la resolución inversa:

```text
192.168.1.100 → raul.haven.local
192.168.1.101 → dns2.haven.local
```

Como la zona ya representa `192.168.1`, solo se escribe el último octeto:

```dns
100 IN PTR raul.haven.local.
```

---

## 7. Validar y arrancar el maestro

Validar cada zona:

```bash
sudo named-checkzone haven.local /etc/bind/zones/db.haven.local
sudo named-checkzone 1.168.192.in-addr.arpa /etc/bind/zones/db.1.168.192
```

Validar la configuración global y hacer una carga de prueba:

```bash
sudo named-checkconf -z
```

Reiniciar BIND:

```bash
sudo systemctl restart bind9
sudo systemctl status bind9
```

Comprobar que escucha en el puerto DNS:

```bash
sudo ss -lntup | grep ':53'
```

Resultado esperado:

```text
192.168.1.100:53
```

### ¿Por qué se valida primero?

`named-checkzone` detecta errores en los archivos de zona antes de que BIND intente cargarlos. `named-checkconf -z` comprueba la configuración y realiza una carga de prueba de las zonas primarias.

---

## 8. Probar el maestro

```bash
nslookup raul.haven.local 192.168.1.100
nslookup dns2.haven.local 192.168.1.100
nslookup 192.168.1.100 192.168.1.100
nslookup 192.168.1.101 192.168.1.100
```

Resultados esperados:

| Consulta | Resultado |
|---|---|
| `raul.haven.local` | `192.168.1.100` |
| `dns2.haven.local` | `192.168.1.101` |
| `192.168.1.100` | `raul.haven.local` |
| `192.168.1.101` | `dns2.haven.local` |

---

# Parte B: servidor esclavo

## 9. Consideraciones al clonar el maestro

Si utilizas un clon de la VM maestra, BIND9 ya estará instalado, pero también se habrán copiado:

- La IP del maestro.
- El hostname del maestro.
- La configuración de BIND.
- Los archivos de zona primarios.

Antes de arrancar las dos máquinas al mismo tiempo debes convertir el clon en esclavo.

> [!WARNING]
> No enciendas maestro y clon simultáneamente mientras ambos tengan `192.168.1.100`. Dos equipos con la misma IP provocan conflictos de red.

---

## 10. Configurar la red del esclavo

Archivo:

```text
/etc/network/interfaces
```

Contenido:

```conf
source /etc/network/interfaces.d/*

auto lo
iface lo inet loopback

# Adaptador NAT: acceso a Internet
allow-hotplug enp0s3
iface enp0s3 inet dhcp

# Red privada del laboratorio DNS
auto enp0s8
iface enp0s8 inet static
    address 192.168.1.101
    netmask 255.255.255.0
    broadcast 192.168.1.255
    dns-nameservers 192.168.1.100
```

Aplicar:

```bash
sudo ifdown --force enp0s8
sudo ifup --force enp0s8
```

Comprobar:

```bash
ip addr show enp0s8
ping -c 4 192.168.1.100
```

Resultado esperado:

```text
inet 192.168.1.101/24
```

### ¿Qué se está haciendo?

Se da al esclavo una IP distinta y se configura inicialmente al maestro como DNS. Si el ping falla, no continúes: primero revisa la red de VirtualBox, las IP y el cable virtual.

---

## 11. Cambiar el hostname del esclavo

```bash
sudo hostnamectl set-hostname dns2
```

Editar:

```bash
sudo nano /etc/hosts
```

Debe incluir:

```text
127.0.0.1       localhost
127.0.1.1       dns2
```

Comprobar:

```bash
hostnamectl
```

### ¿Qué se está haciendo?

El hostname identifica el sistema operativo. No crea registros DNS por sí mismo, pero evita que maestro y esclavo tengan el mismo nombre local tras clonar la máquina.

---

## 12. Configurar opciones de BIND en el esclavo

Archivo:

```text
/etc/bind/named.conf.options
```

Contenido:

```conf
acl "safeclients" {
    localhost;
    192.168.1.0/24;
};

options {
    directory "/var/cache/bind";

    recursion yes;
    allow-recursion { safeclients; };
    allow-query { safeclients; };
    allow-query-cache { safeclients; };

    listen-on port 53 { 127.0.0.1; 192.168.1.101; };
    listen-on-v6 { none; };

    allow-transfer { none; };

    forwarders {
        1.1.1.1;
        9.9.9.9;
    };
};
```

### Diferencias respecto al maestro

| Ajuste | Maestro | Esclavo |
|---|---|---|
| IP en `listen-on` | `192.168.1.100` | `192.168.1.101` |
| `allow-transfer` | Permite al esclavo | Denegado a otros equipos |
| Archivos de zona | Originales en `/etc/bind/zones/` | Copias descargadas en `/var/cache/bind/slaves/` |

---

## 13. Declarar zonas secundarias

Archivo:

```text
/etc/bind/named.conf.local
```

Contenido completo:

```conf
zone "haven.local" {
    type secondary;
    file "/var/cache/bind/slaves/db.haven.local";
    primaries { 192.168.1.100; };
};

zone "1.168.192.in-addr.arpa" {
    type secondary;
    file "/var/cache/bind/slaves/db.1.168.192";
    primaries { 192.168.1.100; };
};
```

### Explicación

| Directiva | Función |
|---|---|
| `type secondary;` | Indica que esta zona es una réplica. |
| `file` | Ruta local donde BIND guardará la zona transferida. |
| `primaries` | Dirección del maestro desde el que se descarga la zona. |

> [!CAUTION]
> No combines `masters { ... };` y `primaries { ... };` en la misma zona. Son sinónimos, pero BIND mostrará un error si aparecen los dos. Usa solo `primaries`.

---

## 14. Preparar directorio de zonas esclavas

```bash
sudo mkdir -p /var/cache/bind/slaves
sudo chown -R bind:bind /var/cache/bind/slaves
sudo chmod 755 /var/cache/bind/slaves
```

### ¿Qué se está haciendo?

BIND se ejecuta con el usuario `bind`. Este usuario debe poder escribir en `/var/cache/bind/slaves/`, donde guardará las copias transferidas desde el maestro.

---

## 15. Mover archivos heredados del maestro

Este paso solo se realiza en el clon convertido en esclavo.

```bash
sudo mkdir -p /root/backup-zonas-maestro

sudo mv /etc/bind/zones/db.haven.local \
    /root/backup-zonas-maestro/ 2>/dev/null

sudo mv /etc/bind/zones/db.1.168.192 \
    /root/backup-zonas-maestro/ 2>/dev/null
```

### ¿Por qué se mueven?

El esclavo no debe usar los archivos originales copiados durante la clonación. Sus zonas deben descargarse desde el maestro y almacenarse en:

```text
/var/cache/bind/slaves/
```

> [!IMPORTANT]
> No ejecutes estos comandos en el maestro. El maestro necesita conservar sus archivos en `/etc/bind/zones/`.

---

## 16. Validar y arrancar el esclavo

Validar la sintaxis:

```bash
sudo named-checkconf
```

Si no aparece ningún mensaje, la sintaxis es correcta.

Reiniciar el servicio:

```bash
sudo systemctl reset-failed named
sudo systemctl restart named
sudo systemctl status named
```

También puedes utilizar:

```bash
sudo systemctl restart bind9
sudo systemctl status bind9
```

Comprobar el puerto DNS:

```bash
sudo ss -lntup | grep ':53'
```

Resultado esperado:

```text
192.168.1.101:53
```

---

## 17. Comprobar la transferencia de zonas

Ver los últimos eventos de BIND:

```bash
sudo journalctl -u named.service --no-pager -n 80
```

Busca mensajes similares a:

```text
transfer of 'haven.local/IN' from 192.168.1.100#53
transfer completed
```

Comprobar las zonas descargadas:

```bash
sudo ls -lah /var/cache/bind/slaves/
```

Deben aparecer:

```text
db.haven.local
db.1.168.192
```

### ¿Qué se está haciendo?

El esclavo contacta al maestro por el puerto DNS TCP 53 y descarga una copia de las zonas. La primera transferencia suele ser completa; después BIND puede realizar transferencias incrementales cuando cambian los registros.

---

## 18. Probar el servidor esclavo

Forzar consultas contra la IP del esclavo:

```bash
nslookup raul.haven.local 192.168.1.101
nslookup dns2.haven.local 192.168.1.101
nslookup 192.168.1.100 192.168.1.101
nslookup 192.168.1.101 192.168.1.101
```

Consultar los servidores DNS autoritativos:

```bash
dig @192.168.1.101 haven.local NS +short
```

Resultado esperado:

```text
raul.haven.local.
dns2.haven.local.
```

---

# Prueba de sincronización

## 19. Añadir un registro al maestro

En el maestro, editar:

```bash
sudo nano /etc/bind/zones/db.haven.local
```

Añadir el registro al final de los registros `A`:

```dns
prueba  IN  A  192.168.1.50
```

Ejemplo:

```dns
raul    IN  A  192.168.1.100
ns      IN  A  192.168.1.100
dns2    IN  A  192.168.1.101
prueba  IN  A  192.168.1.50
```

Aumentar el serial del SOA:

```dns
2026100601
```

por:

```dns
2026100602
```

Validar:

```bash
sudo named-checkzone haven.local /etc/bind/zones/db.haven.local
```

Recargar la zona:

```bash
sudo rndc reload haven.local
```

Si `rndc` falla:

```bash
sudo systemctl reload bind9
```

---

## 20. Confirmar que el esclavo se actualiza

En el esclavo, observar el log:

```bash
sudo journalctl -u named.service -f
```

En otra terminal, consultar al esclavo:

```bash
nslookup prueba.haven.local 192.168.1.101
```

Resultado esperado:

```text
Name:    prueba.haven.local
Address: 192.168.1.50
```

### ¿Qué demuestra esta prueba?

- El registro se ha creado solo en el maestro.
- El serial se ha incrementado.
- El esclavo ha detectado el cambio.
- BIND ha transferido la zona actualizada.
- El esclavo responde con el nuevo registro.

---

# Fallos comunes

| Problema | Causa habitual | Solución |
|---|---|---|
| Maestro y esclavo no responden a `ping` | Las VMs no usan la misma red privada, hay IP incorrectas o cable virtual desconectado | Revisar la configuración de VirtualBox, `ip addr`, IPs `.100` y `.101` y máscara `/24` |
| BIND no inicia | Error de sintaxis en `.conf` o zona | Ejecutar `sudo named-checkconf` y `sudo named-checkzone` |
| `missing 'primaries' entry` | La zona secundaria no sabe de qué maestro descargar | Añadir `primaries { 192.168.1.100; };` |
| `primaries and masters cannot both be used` | Se han escrito ambas directivas en la misma zona | Borrar `masters` y dejar solo `primaries` |
| `missing ';' before '}'` | Falta `;` en una directiva | Revisar especialmente `primaries { 192.168.1.100; };` |
| `zone transfer denied` | El maestro no autoriza al esclavo | Añadir `allow-transfer { 192.168.1.101; };` en el maestro y recargar |
| No hay archivos en `/var/cache/bind/slaves/` | Sin conectividad, permisos incorrectos o transferencias bloqueadas | Revisar `ping`, logs, `allow-transfer` y propietario `bind:bind` |
| El esclavo no recibe registros nuevos | No se incrementó el serial | Aumentar el serial del SOA, validar y recargar el maestro |
| El esclavo escucha en `192.168.1.100` | El clon mantiene la IP o `listen-on` del maestro | Cambiar IP a `.101` y `listen-on` a `192.168.1.101` |
| Las dos VMs tienen el mismo hostname | El clon conserva la identidad del maestro | Ejecutar `sudo hostnamectl set-hostname dns2` |
| `nslookup ns` devuelve `NXDOMAIN` | No existe un registro `ns` o no existe dominio de búsqueda | Añadir `ns IN A 192.168.1.100` y consultar `ns.haven.local` |
| Algunas aplicaciones fallan con `haven.local` | `.local` se usa para mDNS | Consultar explícitamente los DNS BIND en el laboratorio o usar otro sufijo en producción |

---

# Checklist final

## Maestro

```bash
ip addr show enp0s8

sudo named-checkconf -z

sudo named-checkzone haven.local \
    /etc/bind/zones/db.haven.local

sudo named-checkzone 1.168.192.in-addr.arpa \
    /etc/bind/zones/db.1.168.192

sudo systemctl status bind9
```

## Esclavo

```bash
ip addr show enp0s8

ping -c 4 192.168.1.100

sudo named-checkconf

sudo systemctl status named

sudo ls -lah /var/cache/bind/slaves/
```

## Consultas contra el esclavo

```bash
nslookup raul.haven.local 192.168.1.101

nslookup 192.168.1.100 192.168.1.101

nslookup prueba.haven.local 192.168.1.101
```

Si los últimos comandos devuelven las IP y los nombres esperados, la configuración maestro-esclavo está terminada correctamente.
