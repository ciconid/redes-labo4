# Guía Completa: DNS con BIND9 — Laboratorio 4
### Redes de Computadoras · UNS · Debian 12

---

## ÍNDICE
1. [¿Qué es DNS y cómo funciona?](#1-qué-es-dns-y-cómo-funciona)
2. [¿Qué es un servidor DNS local?](#2-qué-es-un-servidor-dns-local)
3. [Tipos de registros DNS](#3-tipos-de-registros-dns)
4. [VLSM: Cálculo de subredes](#4-vlsm-cálculo-de-subredes)
5. [Instalación de BIND9 en router4](#5-instalación-de-bind9-en-router4)
6. [Estructura de archivos de BIND9](#6-estructura-de-archivos-de-bind9)
7. [named.conf.options — Opciones globales](#7-namedconfoptions--opciones-globales)
8. [named.conf.local — Declaración de zonas](#8-namedconflocal--declaración-de-zonas)
9. [Zona directa: redes.cs.intra](#9-zona-directa-redescsintra)
10. [Zonas inversas (PTR)](#10-zonas-inversas-ptr)
11. [Verificación de sintaxis](#11-verificación-de-sintaxis)
12. [Gestión del servicio](#12-gestión-del-servicio)
13. [Pruebas con dig y nslookup](#13-pruebas-con-dig-y-nslookup)
14. [Configurar DNS en los demás routers](#14-configurar-dns-en-los-demás-routers)
15. [DHCP con dnsmasq en router1](#15-dhcp-con-dnsmasq-en-router1)
16. [Resumen de comandos rápidos](#16-resumen-de-comandos-rápidos)

---

## 1. ¿Qué es DNS y cómo funciona?

### El problema que DNS resuelve

Las computadoras se comunican usando direcciones IP (ej: `192.168.151.1`). Los humanos preferimos nombres (ej: `router4.redes.cs.intra`). DNS es el sistema que traduce nombres en IPs (y también IPs en nombres, que se llama **resolución inversa**).

Pensalo como la guía telefónica de Internet: vos buscás el nombre "google.com" y DNS te devuelve la IP `142.250.80.46`.

### La jerarquía de DNS

DNS es una base de datos distribuida y jerárquica. Tiene forma de árbol:

```
.  (raíz)
├── com
│   ├── google
│   │   └── www  → 142.250.80.46
│   └── amazon
├── ar
│   ├── com
│   │   └── unc
│   └── edu
│       └── uns  → ns1.uns.edu.ar
└── intra         ← dominio "inventado" / privado
    └── cs
        └── redes ← el dominio del labo
            ├── router1 → 192.168.148.1
            └── router4 → 192.168.152.1
```

### Cómo funciona una consulta DNS (paso a paso)

Cuando escribís `ping router1.redes.cs.intra` en un router, pasa lo siguiente:

```
Tu equipo
  │
  ├─(1) ¿Tengo esto en /etc/hosts? No.
  ├─(2) ¿Tengo esto en caché local? No.
  └─(3) Pregunto al servidor DNS configurado en /etc/resolv.conf
           │
           └─► Servidor DNS local (router4)
                 │
                 ├─ ¿Soy autoritativo para "redes.cs.intra"? SÍ.
                 └─► Leo el archivo de zona → 192.168.148.1
```

Si la consulta fuera para un dominio externo (ej: `google.com`):
```
Tu equipo → router4
                │
                ├─ ¿Soy autoritativo para "google.com"? NO.
                ├─ ¿Tengo en caché? No (primera vez).
                └─► Forwarder (10.3.0.1 — DNS de la UNS)
                         │
                         └─► Resuelve y devuelve la respuesta
                                   │
                              router4 guarda en caché y responde
```

### ¿Qué se guarda en /etc/hosts?

Es un archivo de texto plano que mapea nombres a IPs directamente, sin consultar ningún servidor. Es el mecanismo más primitivo de resolución, anterior a DNS. El sistema operativo lo consulta primero porque `/etc/nsswitch.conf` define el orden:

```
hosts:  files dns
```

Primero `files` (que es `/etc/hosts`), después `dns`. Si encontrás el nombre ahí, no llegás nunca a preguntar al DNS.

### Las cachés

Hay dos niveles de caché:

**Caché del resolver (router4 con BIND9)**: la importante. Cuando router4 resuelve `google.com` preguntando a `8.8.8.8`, guarda la respuesta en memoria respetando el TTL. La próxima vez que cualquier equipo de la red pregunte por `google.com`, router4 responde desde su caché sin salir a Internet. Para las zonas propias (`redes.cs.intra`) no hay caché porque router4 es autoritativo: lee directo del archivo.

**Caché local del cliente**: en Debian 12 básico no existe por defecto. Si corre `systemd-resolved` sí hay una caché local pequeña. Verificar con `systemctl status systemd-resolved`.

### Tipos de consultas

- **Recursiva**: "Resolvé esto por mí, yo espero la respuesta final." Es lo que hace tu equipo al consultar a router4.
- **Iterativa**: "Dame la mejor pista que tengas." Es lo que hace router4 cuando consulta a los forwarders, preguntando de servidor en servidor hasta encontrar la respuesta.

---

## 2. ¿Qué es un servidor DNS local?

Un **servidor DNS local** es un servidor DNS que:

1. **Conoce los nombres de tu red interna**: sabe que `router4.redes.cs.intra` → `192.168.152.1`, información que ningún servidor DNS de Internet conoce.
2. **Reenvía las consultas externas**: cuando le preguntan por `google.com`, reenvía la consulta a un DNS externo (**forwarder**). En tu caso: `10.3.0.1` (DNS de la UNS).
3. **Es autoritativo**: para el dominio `redes.cs.intra`, router4 es la autoridad máxima.

**En tu laboratorio**: `router4` hace los dos roles simultáneamente: es autoritativo para `redes.cs.intra` y es el resolver al que todos los demás routers le preguntan.

### La diferencia entre los tres conceptos

**Servidor autoritativo**: tiene los datos reales de una zona escritos en un archivo. Responde con el flag `AA` (Authoritative Answer). El autoritativo de `google.com` vive en los servidores de Google. En tu labo, router4 es el autoritativo de `redes.cs.intra`.

**Servidor DNS local (resolver)**: el servidor al que tus equipos le preguntan directamente. No necesariamente sabe las respuestas, pero sabe cómo encontrarlas: con sus propios datos, con caché, o preguntando a otros (forwarders). En tu labo, router4 también hace este rol.

**Registro NS (nameserver)**: no es un tipo de servidor, es un registro dentro del DNS que dice quién es el autoritativo de una zona. Cuando configurás `redes.cs.intra. IN NS ns.redes.cs.intra.` estás escribiendo ese cartel dentro de tu zona.

### Si no tuvieras zona propia

Configurarías `/etc/resolv.conf` con `nameserver 8.8.8.8` y listo. 8.8.8.8 resuelve todo lo público. El problema es que 8.8.8.8 no sabe que `router1.redes.cs.intra` existe. Por eso necesitás un DNS propio.

---

## 3. Tipos de registros DNS

Cada entrada en DNS se llama **Resource Record (RR)**. Formato general:

```
nombre   [TTL]   IN   TIPO   valor
```

Los que usás en este labo:

| Tipo | Nombre | Para qué sirve | Ejemplo |
|------|--------|----------------|---------|
| **A** | Address | Nombre → IPv4 | `router4.redes.cs.intra. IN A 192.168.152.1` |
| **PTR** | Pointer | IPv4 → Nombre (reversa) | `1.152.168.192.in-addr.arpa. IN PTR router4.redes.cs.intra.` |
| **NS** | Name Server | Quién es autoritativo | `redes.cs.intra. IN NS ns.redes.cs.intra.` |
| **SOA** | Start of Authority | Parámetros de la zona | Ver más abajo |
| **CNAME** | Canonical Name | Alias | `ns IN CNAME router4.redes.cs.intra.` |

### El registro SOA explicado campo a campo

```
@  IN  SOA  ns.redes.cs.intra.  hostmaster.redes.cs.intra. (
              2026052201  ; Serial   ← versión del archivo (año+mes+día+nn)
              3600        ; Refresh  ← cada cuánto el secundario chequea cambios (seg)
              900         ; Retry    ← si falla el refresh, cuánto espera para reintentar (seg)
              1209600     ; Expire   ← si no puede refrescar, cuánto tiempo la info sigue válida
              86400       ; Negative TTL ← cuánto guarda en caché la respuesta "no existe"
)
```

- `@` significa "este mismo dominio" (redes.cs.intra)
- `ns.redes.cs.intra.` es el servidor primario (con punto final = FQDN)
- `hostmaster.redes.cs.intra.` es el email del admin con `.` en lugar de `@` (y punto final)
- **El Serial es crítico**: si cambiás el archivo y no aumentás el serial, el servidor secundario no se entera. Formato: `YYYYMMDDnn` donde `nn` va de 00 a 99 por cambio del día.

### ¿Para qué sirve el TTL?

Es el "tiempo de vida" de una respuesta en la caché de otros servidores. Cuando router4 responde que `router1.redes.cs.intra = 192.168.148.1`, incluye el TTL (86400 seg = 24 hs) para decirle al que preguntó: "guardá esta respuesta 24 horas, no me vuelvas a preguntar antes". TTL alto = menos tráfico, cambios lentos de propagar. TTL bajo = más tráfico, cambios rápidos.

El `Negative TTL` en el SOA hace lo mismo pero para las respuestas "ese nombre no existe" (NXDOMAIN).

### ¿Para qué sirve DNS inverso?

La resolución inversa es IP → nombre. Cuando hacés `traceroute`, la salida muestra nombres en lugar de IPs crudas porque por cada IP que encuentra hace una consulta inversa. Sin zonas inversas configuradas:

```
3  192.168.152.66  1.4 ms    ← solo IP, sin nombre
```

Con zonas inversas:
```
3  router4-red7.redes.cs.intra (192.168.152.66)  1.4 ms
```

También lo usan logs de servidores, firewalls, y el comando `dig -x`. El enunciado te pide verificar reversas con `dig -x` para todas las IPs y hacer `traceroute` por nombre a Internet, así que sin zonas inversas no completás el labo.

### ¿Por qué el sufijo .in-addr.arpa?

DNS es jerárquico de derecha a izquierda: en `router1.redes.cs.intra`, lo más general está a la derecha. Las IPs funcionan al revés: en `192.168.148.1`, lo más general (`192`) está a la izquierda. Para meterlas en el árbol DNS de forma consistente se invierten los octetos y se colocan bajo el dominio especial `.in-addr.arpa`. La IP `192.168.148.1` se convierte en `1.148.168.192.in-addr.arpa`, donde ahora lo más específico (`1`) está a la izquierda, igual que cualquier nombre DNS. Es un estándar del RFC 1035 (1987) que todos los servidores DNS entienden, incluso dentro de redes privadas que nunca salen a Internet.

### Flujo de resolución inversa

Desde router2, `traceroute` ve la IP `192.168.152.66` y quiere saber su nombre:

```
router2
 │  El SO convierte 192.168.152.66 → 66.152.168.192.in-addr.arpa
 │
 └─► router4 (192.168.152.1)
       │  Recibe: "¿PTR de 66.152.168.192.in-addr.arpa?"
       │  Tengo zone "152.168.192.in-addr.arpa"
       │  Leo 152.168.192.zone → 66 IN PTR router4-red7.redes.cs.intra.
       └─► Responde: router4-red7.redes.cs.intra.

traceroute muestra:
  3  router4-red7.redes.cs.intra (192.168.152.66)  1.4 ms
```

---

## 4. VLSM: Cálculo de subredes

### Bloque disponible
```
192.168.148.0/24
192.168.149.0/24
192.168.150.0/24
192.168.151.0/24
192.168.152.0/24
```
Total: 5 bloques /24 contiguos = espacio de `192.168.148.0` a `192.168.152.255`

### Regla de VLSM: ordenar de mayor a menor y asignar en orden

| Orden | Red | Hosts requeridos | Hosts útiles con /mask | Máscara | Dirección de red |
|-------|-----|-----------------|------------------------|---------|-----------------|
| 1 | Red1 | 356 | 510 | /23 | 192.168.148.0 |
| 2 | Red2 | 230 | 254 | /24 | 192.168.150.0 |
| 3 | Red4 | 110 | 126 | /25 | 192.168.151.0 |
| 4 | Red3 | 61  | 62  | /26 | 192.168.151.128 |
| 5 | Red5 | 55  | 62  | /26 | 192.168.151.192 |
| 6 | Red6 | 50  | 62  | /26 | 192.168.152.0 |
| 7 | Red7 | 2   | 2   | /30 | 192.168.152.64 |
| 8 | Red8 | 2   | 2   | /30 | 192.168.152.68 |

> **¿Por qué /23 para 356 hosts?** Con /24 tenés 254 direcciones útiles (no alcanza). Con /23 tenés 2⁹ − 2 = 510 (alcanza). Un /23 ocupa dos bloques /24 consecutivos: 148.x y 149.x. Por eso Red1 necesita **dos zonas inversas** separadas en BIND.

### Detalle de cada subred con IPs asignadas

---

**Red1 — 192.168.148.0/23** — 356 equipos (sw_red1)

| Campo | Valor |
|-------|-------|
| Máscara | `255.255.254.0` |
| Broadcast | `192.168.149.255` |
| Rango útil | `192.168.148.1` → `192.168.149.254` |
| Gateway / router1 | `192.168.148.1` |
| Workstation 1 | `192.168.148.2` |

---

**Red2 — 192.168.150.0/24** — 230 equipos (sw_red2)

| Campo | Valor |
|-------|-------|
| Máscara | `255.255.255.0` |
| Broadcast | `192.168.150.255` |
| Rango útil | `192.168.150.1` → `192.168.150.254` |
| Gateway / router1 | `192.168.150.1` |

---

**Red4 — 192.168.151.0/25** — 110 equipos (sw_red4)

| Campo | Valor |
|-------|-------|
| Máscara | `255.255.255.128` |
| Broadcast | `192.168.151.127` |
| Rango útil | `192.168.151.1` → `192.168.151.126` |
| Gateway / router2 | `192.168.151.1` |

---

**Red3 — 192.168.151.128/26** — 61 equipos (sw_red3)

| Campo | Valor |
|-------|-------|
| Máscara | `255.255.255.192` |
| Broadcast | `192.168.151.191` |
| Rango útil | `192.168.151.129` → `192.168.151.190` |
| Gateway / router1 | `192.168.151.129` |
| router2 en Red3 | `192.168.151.130` |

---

**Red5 — 192.168.151.192/26** — 55 equipos (sw_red5)

| Campo | Valor |
|-------|-------|
| Máscara | `255.255.255.192` |
| Broadcast | `192.168.151.255` |
| Rango útil | `192.168.151.193` → `192.168.151.254` |
| Gateway / router3 | `192.168.151.193` |

---

**Red6 — 192.168.152.0/26** — 50 equipos (sw_red6)

| Campo | Valor |
|-------|-------|
| Máscara | `255.255.255.192` |
| Broadcast | `192.168.152.63` |
| Rango útil | `192.168.152.1` → `192.168.152.62` |
| Gateway / router4 | `192.168.152.1` |

---

**Red7 — 192.168.152.64/30** — enlace punto a punto router2 ↔ router4 (sw_red7)

| Campo | Valor |
|-------|-------|
| Máscara | `255.255.255.252` |
| Broadcast | `192.168.152.67` |
| router2 | `192.168.152.65` |
| router4 | `192.168.152.66` |

---

**Red8 — 192.168.152.68/30** — enlace punto a punto router2 ↔ router3 (sw_red8)

| Campo | Valor |
|-------|-------|
| Máscara | `255.255.255.252` |
| Broadcast | `192.168.152.71` |
| router2 | `192.168.152.69` |
| router3 | `192.168.152.70` |

---

**Red31 — 172.31.0.0/24** — enlace a router_internet (sw_red31)

| Campo | Valor |
|-------|-------|
| Máscara | `255.255.255.0` |
| router3 | `172.31.0.1` |
| router_internet | `172.31.0.254` |

---

### Resumen de IPs por equipo

| Equipo | Red | Switch | IP | Máscara |
|--------|-----|--------|----|---------|
| router1 | Red1 | sw_red1 | `192.168.148.1` | /23 |
| router1 | Red2 | sw_red2 | `192.168.150.1` | /24 |
| router1 | Red3 | sw_red3 | `192.168.151.129` | /26 |
| router2 | Red3 | sw_red3 | `192.168.151.130` | /26 |
| router2 | Red4 | sw_red4 | `192.168.151.1` | /25 |
| router2 | Red7 | sw_red7 | `192.168.152.65` | /30 |
| router2 | Red8 | sw_red8 | `192.168.152.69` | /30 |
| router3 | Red5 | sw_red5 | `192.168.151.193` | /26 |
| router3 | Red8 | sw_red8 | `192.168.152.70` | /30 |
| router3 | Red31 | sw_red31 | `172.31.0.1` | /24 |
| router4 | Red6 | sw_red6 | `192.168.152.1` | /26 |
| router4 | Red7 | sw_red7 | `192.168.152.66` | /30 |
| router_internet | Red31 | sw_red31 | `172.31.0.254` | /24 |
| ws1 | Red1 | sw_red1 | `192.168.148.2` | /23 |

---

## 5. Instalación de BIND9 en router4

**Todo lo que sigue se ejecuta EN router4.**

```bash
apt update
apt install -y bind9 bind9utils bind9-doc dnsutils
```

- `bind9`: el servidor DNS (el daemon `named`)
- `bind9utils`: herramientas `named-checkconf`, `named-checkzone`, `rndc`
- `bind9-doc`: documentación
- `dnsutils`: `dig` y `nslookup`
- `-y`: no preguntar confirmación

```bash
systemctl status bind9
```
> Debe mostrar `active (running)` después de la instalación.

---

## 6. Estructura de archivos de BIND9

Después de instalar, el directorio `/etc/bind/` contiene:

```
/etc/bind/
├── named.conf              ← archivo principal (NO editar, solo incluye a los otros)
├── named.conf.options      ← opciones globales (forwarders, recursión, etc.) ← EDITAR
├── named.conf.local        ← definición de nuestras zonas ← EDITAR
├── named.conf.default-zones← zonas por defecto (localhost, etc.) ← NO tocar
├── db.local                ← zona del localhost (plantilla)
├── db.127                  ← reversa del localhost (plantilla)
├── db.empty                ← plantilla de zona vacía
└── zones/                  ← directorio que vamos a CREAR para nuestros archivos de zona
```

`named.conf` solo hace `include` de los otros tres:
```
named.conf
├── include named.conf.options
├── include named.conf.local
└── include named.conf.default-zones
```

### Crear el directorio de zonas

```bash
mkdir -p /etc/bind/zones
chown -R bind:bind /etc/bind/zones
```

> BIND9 corre como el usuario `bind`. Si los archivos de zona no le pertenecen, no los puede leer.

---

## 7. named.conf.options — Opciones globales

### Backup y edición

```bash
cp /etc/bind/named.conf.options /etc/bind/named.conf.options.bak
nano /etc/bind/named.conf.options
```

### Contenido — copiar y pegar

```
options {
    directory "/var/cache/bind";

    forwarders {
        10.3.0.1;
        8.8.8.8;
        1.1.1.1;
    };

    recursion yes;
    allow-recursion { any; };
    allow-query { any; };
    auth-nxdomain no;

    // dnssec-validation auto;

    listen-on { any; };
    listen-on-v6 { any; };
};
```

### ¿Qué hace cada opción?

**`directory "/var/cache/bind"`**: directorio de trabajo de BIND donde guarda caché y archivos temporales. No modificar.

**`forwarders { ... }`**: cuando BIND no es autoritativo para un dominio (ej: `google.com`), reenvía la consulta a estos servidores. `10.3.0.1` es el DNS de la UNS para usar en el laboratorio. `8.8.8.8` y `1.1.1.1` son backup para usar desde casa.

**`recursion yes`**: permite que los clientes (los demás routers) pidan a este servidor que resuelva recursivamente. Sin esto solo respondería para sus propias zonas.

**`allow-recursion { any }`**: cualquier IP puede hacer consultas recursivas. En producción pondrías solo tu red interna, pero para el labo `any` está bien.

**`allow-query { any }`**: cualquier IP puede hacer consultas.

**`auth-nxdomain no`**: cuando BIND responde "ese nombre no existe" (NXDOMAIN), no marca la respuesta como autoritativa. El valor `no` es el correcto según los RFC modernos.

**`dnssec-validation auto`** (comentada): DNSSEC verifica autenticidad de respuestas DNS con firmas criptográficas. Nuestro dominio `redes.cs.intra` es privado y no tiene firmas DNSSEC. Si se deja activa, BIND intenta validar firmas que no existen y puede fallar al resolver nombres externos. Se comenta con `//`.

---

## 8. named.conf.local — Declaración de zonas

Este archivo le dice a BIND: "soy autoritativo para estas zonas, y los datos están en estos archivos".

### Backup y edición

```bash
cp /etc/bind/named.conf.local /etc/bind/named.conf.local.bak
nano /etc/bind/named.conf.local
```

### Contenido — copiar y pegar

```
// Zona DIRECTA: nombre → IP
zone "redes.cs.intra" {
    type master;
    file "/etc/bind/zones/redes.cs.intra.zone";
};

// Zonas INVERSAS: IP → nombre

// Red1 parte 1 (192.168.148.x)
zone "148.168.192.in-addr.arpa" {
    type master;
    file "/etc/bind/zones/148.168.192.zone";
};

// Red1 parte 2 (192.168.149.x) — segunda mitad del /23
zone "149.168.192.in-addr.arpa" {
    type master;
    file "/etc/bind/zones/149.168.192.zone";
};

// Red2 (192.168.150.x)
zone "150.168.192.in-addr.arpa" {
    type master;
    file "/etc/bind/zones/150.168.192.zone";
};

// Red4 + Red3 + Red5 (192.168.151.x)
zone "151.168.192.in-addr.arpa" {
    type master;
    file "/etc/bind/zones/151.168.192.zone";
};

// Red6 + Red7 + Red8 (192.168.152.x)
zone "152.168.192.in-addr.arpa" {
    type master;
    file "/etc/bind/zones/152.168.192.zone";
};

// Red31 (172.31.0.x)
zone "0.31.172.in-addr.arpa" {
    type master;
    file "/etc/bind/zones/0.31.172.zone";
};
```

### Explicación de los conceptos clave

**`type master`**: este servidor es el primario (autoritativo) para esta zona. En el labo no hay servidor secundario, así que todos son `master`.

**`file "..."`**: ruta al archivo que contiene los registros de la zona. BIND lo lee al iniciarse o cuando se recarga.

**¿Por qué la zona inversa se llama así?** La zona inversa de `192.168.148.0/24` se llama `148.168.192.in-addr.arpa` porque se toman los primeros tres octetos (`192`, `168`, `148`), se invierten (`148`, `168`, `192`) y se agrega `.in-addr.arpa`.

**¿Por qué Red1 tiene dos zonas inversas?** Red1 usa `192.168.148.0/23`, que abarca dos bloques /24: `148.x` y `149.x`. Un /23 cruza el límite entre dos /24, entonces necesitás una zona inversa por cada uno.

**¿Por qué Red4, Red3 y Red5 comparten zona inversa?** Las tres están dentro del bloque `192.168.151.0/24`. Todas comparten el mismo tercer octeto (151), entonces comparten la zona `151.168.192.in-addr.arpa`. En el archivo de zona ponés un PTR por cada IP individualmente.

---

## 9. Zona directa: redes.cs.intra

```bash
nano /etc/bind/zones/redes.cs.intra.zone
```

### Contenido — copiar y pegar

```
; Zona directa: redes.cs.intra
; IMPORTANTE: el ";" es el carácter de comentario en archivos de zona BIND.
; Los nombres SIN punto al final son RELATIVOS: "router1" → "router1.redes.cs.intra."
; Los nombres CON punto al final son ABSOLUTOS (FQDN): "router1.redes.cs.intra."
; Si olvidás el punto en un FQDN, BIND lo convierte en "router1.redes.cs.intra.redes.cs.intra." ← bug clásico

$TTL 86400

@   IN  SOA  ns.redes.cs.intra.  hostmaster.redes.cs.intra. (
                2026052201  ; Serial: YYYYMMDDNN — incrementar en cada cambio
                3600        ; Refresh
                900         ; Retry
                1209600     ; Expire
                86400       ; Negative TTL
)

@               IN  NS   ns.redes.cs.intra.

; --- router4 (servidor DNS) ---
; "ns" es el glue record: BIND exige que el nombre del NS tenga un A en la misma zona
ns              IN  A    192.168.152.1
router4-red6    IN  A    192.168.152.1
router4-red7    IN  A    192.168.152.66
router4         IN  A    192.168.152.1

; --- router1 ---
router1-red1    IN  A    192.168.148.1
router1-red2    IN  A    192.168.150.1
router1-red3    IN  A    192.168.151.129
router1         IN  A    192.168.148.1

; --- router2 ---
router2-red3    IN  A    192.168.151.130
router2-red4    IN  A    192.168.151.1
router2-red7    IN  A    192.168.152.65
router2-red8    IN  A    192.168.152.69
router2         IN  A    192.168.151.1

; --- router3 ---
router3-red5    IN  A    192.168.151.193
router3-red8    IN  A    192.168.152.70
router3-red31   IN  A    172.31.0.1
router3         IN  A    192.168.151.193

; --- router_internet ---
router-internet IN  A    172.31.0.254

; --- Workstation 1 ---
ws1             IN  A    192.168.148.2
```

### Puntos clave

**`$TTL 86400`**: directiva que define el TTL por defecto para todos los registros de la tabla.

**El `@`**: significa "la zona en sí misma", es decir, `redes.cs.intra.`.

**¿Por qué `ns` y también `router4-red6` con la misma IP?** `ns` es el nombre que aparece en el registro NS. BIND exige que ese nombre tenga un registro A en la misma zona (se llama *glue record*). `router4-red6` es el nombre descriptivo de esa interfaz.

**Serial `2026052201`**: año `2026`, mes `05`, día `22`, versión del día `01`. Si editás el archivo dos veces el mismo día: `2026052202`.

---

## 10. Zonas inversas (PTR)

Las zonas inversas permiten resolver IP → nombre. Se usan en `traceroute`, logs, y `dig -x`.

### Cómo funciona un PTR

Para buscar el nombre de `192.168.151.1`, DNS convierte esa IP a `1.151.168.192.in-addr.arpa.` y busca el registro PTR. El registro se escribe usando **solo el último octeto** (porque la zona ya tiene el contexto de los primeros tres):

```
; En la zona 151.168.192.in-addr.arpa:
1   IN  PTR  router2-red4.redes.cs.intra.
```

> **Importante**: en los PTR, el nombre destino SIEMPRE lleva punto al final. Si no ponés el punto, BIND agrega el dominio de la zona y el PTR queda mal.

---

### /etc/bind/zones/148.168.192.zone
#### Reversa de 192.168.148.x — Red1 parte 1

```bash
nano /etc/bind/zones/148.168.192.zone
```

```
$TTL 86400

@   IN  SOA  ns.redes.cs.intra.  hostmaster.redes.cs.intra. (
                2026052201
                3600
                900
                1209600
                86400
)

@   IN  NS   ns.redes.cs.intra.

1   IN  PTR  router1-red1.redes.cs.intra.
2   IN  PTR  ws1.redes.cs.intra.
```

---

### /etc/bind/zones/149.168.192.zone
#### Reversa de 192.168.149.x — Red1 parte 2

```bash
nano /etc/bind/zones/149.168.192.zone
```

```
$TTL 86400

@   IN  SOA  ns.redes.cs.intra.  hostmaster.redes.cs.intra. (
                2026052201
                3600
                900
                1209600
                86400
)

@   IN  NS   ns.redes.cs.intra.

; No hay equipos con IPs en 192.168.149.x en la topología actual.
; La zona existe para que BIND responda NXDOMAIN en lugar de SERVFAIL
; ante cualquier consulta inversa de ese rango.
```

---

### /etc/bind/zones/150.168.192.zone
#### Reversa de 192.168.150.x — Red2

```bash
nano /etc/bind/zones/150.168.192.zone
```

```
$TTL 86400

@   IN  SOA  ns.redes.cs.intra.  hostmaster.redes.cs.intra. (
                2026052201
                3600
                900
                1209600
                86400
)

@   IN  NS   ns.redes.cs.intra.

1   IN  PTR  router1-red2.redes.cs.intra.
```

---

### /etc/bind/zones/151.168.192.zone
#### Reversa de 192.168.151.x — Red4 + Red3 + Red5

```bash
nano /etc/bind/zones/151.168.192.zone
```

```
$TTL 86400

@   IN  SOA  ns.redes.cs.intra.  hostmaster.redes.cs.intra. (
                2026052201
                3600
                900
                1209600
                86400
)

@   IN  NS   ns.redes.cs.intra.

; Red4 — 192.168.151.0/25 (octetos finales: 1–126)
1   IN  PTR  router2-red4.redes.cs.intra.

; Red3 — 192.168.151.128/26 (octetos finales: 129–190)
129 IN  PTR  router1-red3.redes.cs.intra.
130 IN  PTR  router2-red3.redes.cs.intra.

; Red5 — 192.168.151.192/26 (octetos finales: 193–254)
193 IN  PTR  router3-red5.redes.cs.intra.
```

---

### /etc/bind/zones/152.168.192.zone
#### Reversa de 192.168.152.x — Red6 + Red7 + Red8

```bash
nano /etc/bind/zones/152.168.192.zone
```

```
$TTL 86400

@   IN  SOA  ns.redes.cs.intra.  hostmaster.redes.cs.intra. (
                2026052201
                3600
                900
                1209600
                86400
)

@   IN  NS   ns.redes.cs.intra.

; Red6 — 192.168.152.0/26 (octetos finales: 1–62)
1   IN  PTR  router4-red6.redes.cs.intra.

; Red7 — 192.168.152.64/30 (octetos finales: 65–66)
65  IN  PTR  router2-red7.redes.cs.intra.
66  IN  PTR  router4-red7.redes.cs.intra.

; Red8 — 192.168.152.68/30 (octetos finales: 69–70)
69  IN  PTR  router2-red8.redes.cs.intra.
70  IN  PTR  router3-red8.redes.cs.intra.
```

---

### /etc/bind/zones/0.31.172.zone
#### Reversa de 172.31.0.x — Red31

```bash
nano /etc/bind/zones/0.31.172.zone
```

```
$TTL 86400

@   IN  SOA  ns.redes.cs.intra.  hostmaster.redes.cs.intra. (
                2026052201
                3600
                900
                1209600
                86400
)

@   IN  NS   ns.redes.cs.intra.

1   IN  PTR  router3-red31.redes.cs.intra.
254 IN  PTR  router-internet.redes.cs.intra.
```

---

## 11. Verificación de sintaxis

Antes de reiniciar BIND, siempre verificar la sintaxis. Un error hace que el servidor no arranque.

### Verificar named.conf

```bash
named-checkconf
```
> Si no muestra nada, no hay errores. Solo muestra output si encuentra problemas.
> Ejemplo de error: `/etc/bind/named.conf.local:5: missing ';'`

### Verificar cada archivo de zona

```bash
named-checkzone redes.cs.intra           /etc/bind/zones/redes.cs.intra.zone
named-checkzone 148.168.192.in-addr.arpa /etc/bind/zones/148.168.192.zone
named-checkzone 149.168.192.in-addr.arpa /etc/bind/zones/149.168.192.zone
named-checkzone 150.168.192.in-addr.arpa /etc/bind/zones/150.168.192.zone
named-checkzone 151.168.192.in-addr.arpa /etc/bind/zones/151.168.192.zone
named-checkzone 152.168.192.in-addr.arpa /etc/bind/zones/152.168.192.zone
named-checkzone 0.31.172.in-addr.arpa    /etc/bind/zones/0.31.172.zone
```

> Output exitoso: `zone redes.cs.intra/IN: loaded serial 2026052201` seguido de `OK`

### Errores comunes y cómo arreglarlos

| Error | Causa probable | Solución |
|-------|---------------|----------|
| `missing ';'` | Faltó un punto y coma | Revisá la línea indicada |
| `has no address records` | El NS apunta a un nombre sin registro A | Asegurarte que `ns` tiene un A |
| `PTR record not defined` | Una IP no tiene PTR en la zona inversa | Agregar el PTR correspondiente |
| `CNAME and other data` | CNAME y otro registro para el mismo nombre | Borrar el duplicado |
| Silencio en `named-checkconf` | ✅ Sin errores | Todo bien |

---

## 12. Gestión del servicio

### Reiniciar BIND (después de cambios en named.conf)

```bash
systemctl restart bind9
```
> Para toda la instancia y la vuelve a iniciar. Usar cuando cambiás `named.conf.options` o `named.conf.local`.

### Recargar zonas (después de cambios en archivos de zona)

```bash
rndc reload
```
> Recarga los archivos de zona sin detener el servicio. Usar cuando solo modificás archivos `.zone`. Más rápido y no interrumpe consultas en curso.

### Recarga de una sola zona

```bash
rndc reload redes.cs.intra
```

### Ver logs si algo falla

```bash
journalctl -u bind9 -n 50
journalctl -u bind9 -f        # en tiempo real
```

### Verificar que BIND escucha en el puerto 53

```bash
ss -tulnp | grep named
```
> Debe mostrar `named` escuchando en puerto `53` UDP y TCP.

---

## 13. Pruebas con dig y nslookup

### dig — la herramienta principal

#### Consulta directa (nombre → IP)

```bash
dig @192.168.152.1 router1.redes.cs.intra
```

Salida esperada:
```
;; ANSWER SECTION:
router1.redes.cs.intra.  86400  IN  A  192.168.148.1
```

#### Consulta inversa (IP → nombre)

```bash
dig @192.168.152.1 -x 192.168.148.1
dig @192.168.152.1 -x 192.168.151.130
dig @192.168.152.1 -x 192.168.152.65
dig @192.168.152.1 -x 172.31.0.1
```

Salida esperada:
```
;; ANSWER SECTION:
1.148.168.192.in-addr.arpa.  86400  IN  PTR  router1-red1.redes.cs.intra.
```

#### Consulta de tipo NS

```bash
dig @192.168.152.1 redes.cs.intra NS
```

#### Probar forwarders (resolución externa)

```bash
dig @192.168.152.1 google.com
```
> Si los forwarders están bien, devuelve la IP de google.com.

### Interpretar las flags en la respuesta de dig

```
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 1
```
- `qr`: es una respuesta (Query Response)
- `aa`: **Authoritative Answer** — el servidor responde con autoridad. Debe aparecer para dominios internos. Si no aparece en una consulta a `redes.cs.intra`, algo falló al cargar la zona.
- `rd`: el cliente pidió recursión (Recursion Desired)
- `ra`: el servidor soporta recursión (Recursion Available)

### nslookup

```bash
nslookup router1.redes.cs.intra 192.168.152.1
```

---

## 14. Configurar DNS en los demás routers

Los routers router1, router2, router3 y router4 necesitan apuntar a router4 como su servidor DNS. Lo mismo la workstation1.

### /etc/resolv.conf — copiar y pegar en router1, router2, router3, router4, wk1

```bash
nano /etc/resolv.conf
```

```
nameserver 192.168.152.1
search redes.cs.intra
```

> `search redes.cs.intra` permite usar nombres cortos: `ping router1` en lugar de `ping router1.redes.cs.intra`.

### Hacerlo persistente (evitar que se sobreescriba)

```bash
chattr +i /etc/resolv.conf
```

> En Debian 12 con systemd-resolved corriendo, `/etc/resolv.conf` puede ser regenerado automáticamente. `chattr +i` lo hace inmutable. Para deshacerlo: `chattr -i /etc/resolv.conf`.

### Verificar desde cada router

```bash
ping -c2 router4.redes.cs.intra       # resolución directa
dig -x 192.168.152.1                  # resolución inversa
traceroute google.com                 # forwarder + resolución externa
```

---

## 15. DHCP con dnsmasq en router1

`dnsmasq` es un servidor DHCP y DNS liviano. En este labo lo usamos **solo como DHCP** en `router1` para Red1. Lo configuramos para que no levante DNS (eso ya lo hace BIND9 en router4).

### Instalar dnsmasq (en router1)

```bash
apt update
apt install -y dnsmasq
```

### Obtener la MAC de Workstation 1

En Workstation 1:
```bash
ip link show
```
Verás algo como `link/ether aa:bb:cc:dd:ee:ff`. Esa es la MAC.

### Obtener el nombre de la interfaz de router1 conectada a Red1

```bash
ip link show
```
Buscá la interfaz que tenga la IP `192.168.148.1`. Puede ser `eth0`, `eth1`, `ens3`, etc.

### /etc/dnsmasq.conf — copiar y pegar en router1

```bash
nano /etc/dnsmasq.conf
```

```
# No actuar como servidor DNS (ese rol lo tiene BIND9 en router4)
port=0

# No leer /etc/resolv.conf
no-resolv

# Escuchar solo en la interfaz de Red1
# REEMPLAZAR por el nombre real de la interfaz (ej: eth0, ens3...)
interface=INTERFAZ_RED1

# === DHCP para Red1: 192.168.148.0/23 ===
dhcp-range=192.168.148.10,192.168.149.200,255.255.254.0,12h

# Puerta de enlace por defecto
dhcp-option=option:router,192.168.148.1

# Servidor DNS a usar (router4)
dhcp-option=option:dns-server,192.168.152.1

# Máscara de subred
dhcp-option=option:netmask,255.255.254.0

# IP fija para Workstation 1
# REEMPLAZAR XX:XX:XX:XX:XX:XX por la MAC real de ws1
dhcp-host=XX:XX:XX:XX:XX:XX,ws1,192.168.148.2,infinite
```

### Iniciar dnsmasq

```bash
systemctl restart dnsmasq
systemctl status dnsmasq
```

### Verificar en Workstation 1

```bash
# Solicitar IP por DHCP (reemplazar eth0 por el nombre de interfaz de ws1)
dhclient eth0

ip a show eth0        # debe mostrar 192.168.148.2/23
ip route              # debe mostrar default via 192.168.148.1
cat /etc/resolv.conf  # debe mostrar nameserver 192.168.152.1

ping -c2 router4.redes.cs.intra
ping -c2 google.com
```

---

## 16. Resumen de comandos rápidos

### Flujo completo en router4

```bash
# 1. Instalar
apt update && apt install -y bind9 bind9utils dnsutils

# 2. Directorio de zonas
mkdir -p /etc/bind/zones
chown -R bind:bind /etc/bind/zones

# 3. Backups
cp /etc/bind/named.conf.options /etc/bind/named.conf.options.bak
cp /etc/bind/named.conf.local   /etc/bind/named.conf.local.bak

# 4. Editar y pegar los archivos
nano /etc/bind/named.conf.options
nano /etc/bind/named.conf.local
nano /etc/bind/zones/redes.cs.intra.zone
nano /etc/bind/zones/148.168.192.zone
nano /etc/bind/zones/149.168.192.zone
nano /etc/bind/zones/150.168.192.zone
nano /etc/bind/zones/151.168.192.zone
nano /etc/bind/zones/152.168.192.zone
nano /etc/bind/zones/0.31.172.zone

# 5. Verificar sintaxis
named-checkconf
named-checkzone redes.cs.intra           /etc/bind/zones/redes.cs.intra.zone
named-checkzone 148.168.192.in-addr.arpa /etc/bind/zones/148.168.192.zone
named-checkzone 149.168.192.in-addr.arpa /etc/bind/zones/149.168.192.zone
named-checkzone 150.168.192.in-addr.arpa /etc/bind/zones/150.168.192.zone
named-checkzone 151.168.192.in-addr.arpa /etc/bind/zones/151.168.192.zone
named-checkzone 152.168.192.in-addr.arpa /etc/bind/zones/152.168.192.zone
named-checkzone 0.31.172.in-addr.arpa    /etc/bind/zones/0.31.172.zone

# 6. Iniciar
systemctl restart bind9
systemctl status bind9

# 7. Probar
dig @192.168.152.1 router1.redes.cs.intra
dig @192.168.152.1 router2-red7.redes.cs.intra
dig @192.168.152.1 -x 192.168.148.1
dig @192.168.152.1 -x 192.168.151.1
dig @192.168.152.1 -x 192.168.151.129
dig @192.168.152.1 -x 192.168.151.130
dig @192.168.152.1 -x 192.168.151.193
dig @192.168.152.1 -x 192.168.152.1
dig @192.168.152.1 -x 192.168.152.65
dig @192.168.152.1 -x 192.168.152.66
dig @192.168.152.1 -x 192.168.152.69
dig @192.168.152.1 -x 192.168.152.70
dig @192.168.152.1 -x 172.31.0.1
dig @192.168.152.1 -x 172.31.0.254
dig @192.168.152.1 google.com
```

### Cuando modificás un archivo de zona

```bash
# 1. Editar e incrementar el serial
nano /etc/bind/zones/redes.cs.intra.zone

# 2. Verificar
named-checkzone redes.cs.intra /etc/bind/zones/redes.cs.intra.zone

# 3. Recargar sin parar el servicio
rndc reload
```
