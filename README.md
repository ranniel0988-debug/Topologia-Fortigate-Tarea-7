# Documentación Técnica: Implementación de Seguridad Perimetral y WAF en FortiGate

**Autor:** Ranniel Sanchez Espinal
**Matrícula:** 2025-0889
**Institución:** Instituto Tecnológico de Las Américas (ITLA)
**Fecha de ejecución:** Julio 2026

---

## Tabla de Contenido

1. [Objetivo de la Red](#1-objetivo-de-la-red)
2. [Topología de Red](#2-topología-de-red)
3. [Configuración de Interfaces y Enrutamiento Base](#3-configuración-de-interfaces-y-enrutamiento-base)
4. [Políticas de Firewall y NAT](#4-políticas-de-firewall-y-nat)
5. [Perfiles de Seguridad (Security Profiles)](#5-perfiles-de-seguridad-security-profiles)
6. [Capturas de Pantalla y Evidencias de Configuración](#6-capturas-de-pantalla-y-evidencias-de-configuración)
7. [Pruebas de Validación](#7-pruebas-de-validación)
8. [Conclusiones](#8-conclusiones)

---

## 1. Objetivo de la Red

El propósito de este proyecto es diseñar, configurar y validar una infraestructura de red segura utilizando un appliance **FortiGate**. La topología exige la segmentación de la red en una **LAN de Usuarios** y una **LAN de Servidores (DMZ)**, garantizando el acceso a Internet mediante NAT, y aplicando políticas restrictivas de seguridad de Capa 7.

Todo el despliegue se realizó exclusivamente a través de la **Interfaz Gráfica de Usuario (GUI)** de FortiOS, cumpliendo con los requerimientos de:

- Filtrado web y de dominios (DNS Filter / Web Filter)
- Control de aplicaciones (Application Control)
- Prevención de intrusiones (IPS)
- Firewall de aplicaciones web (WAF)

---

## 2. Topología de Red

### 2.1 Descripción general

La red se segmenta a nivel de **interfaces físicas dedicadas** del FortiGate (no se implementaron VLANs 802.1Q en este despliegue; cada zona vive en un puerto físico independiente). El bloque base `10.8.89.0/24` se dividió con **VLSM** para separar usuarios y servidores en dominios de broadcast distintos.

```
                        ┌────────────────────┐
                        │      Internet       │
                        └──────────┬──────────┘
                                   │
                             Port 1 (WAN)
                                   │
                         ┌─────────┴─────────┐
                         │      FortiGate      │
                         │  (Firewall / WAF)   │
                         └───┬─────────────┬───┘
                    Port 2   │             │  Port 3
                 (LAN-USUARIOS)      (LAN-SERVIDORES)
                       │                     │
              ┌────────┴────────┐   ┌────────┴────────┐
              │   Kali Linux     │   │   Parrot OS      │
              │  10.8.89.2       │   │  + Apache        │
              │  (DHCP)          │   │  10.8.89.130      │
              │  Usuario/Atacante│   │  (IP estática)    │
              └──────────────────┘   └──────────────────┘
```

### 2.2 Interfaces y direccionamiento IP

| Interfaz | Zona | Red / Máscara | Rango IP Útil | Gateway (FortiGate) | Modo |
|---|---|---|---|---|---|
| `Port 1` | WAN | Asignada por ISP | — | — | DHCP Client |
| `Port 2` | LAN-USUARIOS | `10.8.89.0/25` (`255.255.255.128`) | `.2` – `.126` | `10.8.89.1` | DHCP Server habilitado |
| `Port 3` | LAN-SERVIDORES | `10.8.89.128/28` (`255.255.255.240`) | `.130` – `.142` | `10.8.89.129` | Estático (DHCP deshabilitado) |

> **Nota sobre VLANs:** este diseño no requirió segmentación por VLAN (802.1Q) porque cada zona tiene un puerto físico dedicado en el FortiGate. Si se necesitara escalar a múltiples redes sobre un mismo enlace troncal, el mismo esquema de direccionamiento podría reasignarse a sub-interfaces VLAN bajo `Network > Interfaces > Create New > VLAN`.

### 2.3 Nodos finales desplegados

| Rol | Sistema Operativo | Dirección IP | Asignación |
|---|---|---|---|
| Máquina Atacante / Usuario | Kali Linux | `10.8.89.2` | DHCP (Port 2) |
| Servidor Web DMZ | Parrot OS + Apache | `10.8.89.130` | Estática (Port 3) |

---

## 3. Configuración de Interfaces y Enrutamiento Base

### 3.1 Interfaces y DHCP

Ruta en la GUI: **`Network > Interfaces`**

| Interfaz | Configuración aplicada |
|---|---|
| `Port 1` (WAN) | Modo **DHCP Client**, para recibir la IP pública del ISP y habilitar la salida a Internet. |
| `Port 2` (LAN-USUARIOS) | IP estática `10.8.89.1/25`. **DHCP Server** habilitado, rango `10.8.89.2` – `10.8.89.126`. |
| `Port 3` (LAN-SERVIDORES) | IP estática `10.8.89.129/28`. DHCP deshabilitado (entorno de servidores). |

### 3.2 Ruta por defecto (Default Route)

Ruta en la GUI: **`Network > Static Routes`**

| Campo | Valor |
|---|---|
| Destination | `0.0.0.0/0` |
| Gateway Address | IP del gateway del ISP |
| Interface | `Port 1` |

---

## 4. Políticas de Firewall y NAT

Ruta en la GUI: **`Policy & Objects > Firewall Policy`**, bajo un esquema de **"Zero Trust"** (denegación por defecto).

### 4.1 Política 1 — Acceso a Internet (Usuarios)

Permite la navegación de los usuarios hacia el exterior aplicando traducción de direcciones (NAT).

| Campo | Valor |
|---|---|
| Incoming Interface | `Port 2` (LAN-USUARIOS) |
| Outgoing Interface | `Port 1` (WAN) |
| Source / Destination / Service | `all` / `all` / `ALL` |
| Action | `ACCEPT` |
| NAT | Habilitado (*Use Outgoing Interface Address*) |

### 4.2 Política 2 — Acceso a Servidores (Tráfico interno HTTP)

Permite estrictamente el tráfico web hacia la DMZ, bloqueando cualquier otro protocolo (incluyendo ICMP/Ping).

| Campo | Valor |
|---|---|
| Incoming Interface | `Port 2` (LAN-USUARIOS) |
| Outgoing Interface | `Port 3` (LAN-SERVIDORES) |
| Source | `all` |
| Destination | Objeto *Address* para IP `10.8.89.130` (Servidor Web) |
| Service | `HTTP` (bloquea implícitamente todo lo demás) |
| Action | `ACCEPT` |
| NAT | Deshabilitado (comunicación interna enrutada) |

---

## 5. Perfiles de Seguridad (Security Profiles)

Todos los perfiles se aplicaron sobre la política `Regla_Bloqueos_Estrictos` (tráfico de salida) y `Acceso_Servidor_Web` (tráfico interno).

### 5.1 Bloqueo de redes sociales y dominios específicos

Ruta: **`Security Profiles > DNS Filter`**

Se configuró un *Static Domain Filter* tipo **Wildcard** con acción **Redirect to Block Portal** para:

- `*facebook.com*`
- `*instagram.com*`
- `*tiktok.com*`
- `*itla.edu.do*` (bloqueando dominio principal y todos sus subdominios)

### 5.2 Bloqueo de llamadas de WhatsApp (Application Control)

Ruta: **`Security Profiles > Application Control`**

Se localizó la firma **`WhatsApp_Call`** dentro de la categoría *Video/Audio* y se configuró con acción **Block**, permitiendo la mensajería de texto pero cortando la transmisión de voz/video sobre IP.

### 5.3 Detección y bloqueo de escáneres de red (IPS)

Ruta: **`Security Profiles > Intrusion Prevention`**

Se incluyeron las firmas de las categorías *Information Gathering* y *Network Scanners* (herramientas como Nmap y Nessus), cambiando la acción por defecto de *Monitor* a **Block**.

### 5.4 Web Application Firewall (WAF)

Ruta: **`Security Profiles > Web Application Firewall`**

Protección de Capa 7 para el servidor Apache (`10.8.89.130`). En la sección *Signatures* se habilitaron en modo **Enable / Block**:

- SQL Injection
- Cross Site Scripting (XSS)
- Known Exploits (Shellshock / CVEs publicados)
- Generic Attacks (Path Traversal / LFI)

---

## 6. Capturas de Pantalla y Evidencias de Configuración

> Inserta cada captura dentro de la carpeta `screenshots/` del repositorio y respeta los nombres de archivo usados abajo (o actualiza las rutas si usas otros nombres).

### 6.1 Interfaces de red


*Explicación:* vista de `Network > Interfaces` mostrando `Port 1` en modo DHCP Client (WAN), `Port 2` con IP `10.8.89.1/25` y DHCP Server habilitado, y `Port 3` con IP estática `10.8.89.129/28`.

### 6.2 Ruta estática por defecto


*Explicación:* regla en `Network > Static Routes` con destino `0.0.0.0/0` saliendo por `Port 1` hacia el gateway del ISP.

### 6.3 Políticas de firewall

![Políticas de firewall](screenshots/03-firewall-policies.png)

*Explicación:* listado de `Policy & Objects > Firewall Policy` mostrando las dos reglas activas: acceso a Internet para usuarios (con NAT) y acceso restringido a HTTP hacia el servidor DMZ (sin NAT).

### 6.4 DNS Filter — bloqueo de dominios

![DNS Filter](screenshots/04-dns-filter.png)

*Explicación:* perfil de `Security Profiles > DNS Filter` con las entradas *wildcard* para Facebook, Instagram, TikTok e ITLA configuradas con acción *Redirect to Block Portal*.

### 6.5 Application Control — bloqueo de llamadas WhatsApp


*Explicación:* firma `WhatsApp_Call` localizada en la categoría *Video/Audio* con acción `Block` aplicada.

### 6.6 IPS — sensor de escáneres de red

*Explicación:* sensor de `Security Profiles > Intrusion Prevention` con las firmas de *Information Gathering* y *Network Scanners* en acción `Block`.

### 6.7 WAF — firmas habilitadas


*Explicación:* panel de `Security Profiles > Web Application Firewall` con las categorías SQL Injection, XSS, Known Exploits y Generic Attacks en `Enable / Block`.

---

## 7. Pruebas de Validación

### Evidencia 1 — Bloqueo de tráfico NO-HTTP hacia los servidores

**Prueba:**
```bash
ping 10.8.89.130
```
**Resultado esperado:** 100% de pérdida de paquetes (*Packet loss*).

**Explicación:** la política `Acceso_Servidor_Web` restringe el servicio exclusivamente a HTTP (puerto 80); el protocolo ICMP es descartado por el FortiGate.

---

### Evidencia 2 — Filtrado de redes sociales y dominios

**Prueba:** acceso a `https://www.facebook.com` y `https://www.tiktok.com` desde el navegador del cliente.

**Resultado esperado:** el navegador muestra *"Did Not Connect: Potential Security Issue"* o *"Connection Timed Out"*.

**Explicación:** el DNS Filter intercepta la consulta DNS; al intentar redirigir el tráfico hacia el portal de bloqueo en una conexión HTTPS con HSTS activo, el navegador corta la conexión, logrando el bloqueo efectivo.

<img width="769" height="381" alt="image" src="https://github.com/user-attachments/assets/63b50fd1-6af8-4b11-afac-be594b78176b" />


---

### Evidencia 3 — Prevención de ataques web (WAF en acción)

**Prueba (SQL Injection simulada):**
```bash
curl -v "http://10.8.89.130/?user=admin'%20OR%20'1'='1"
```
**Resultado esperado:**
```
Recv failure: Conexión reinicializada por la máquina remota
```
**Explicación:** el motor WAF analiza el payload HTTP, reconoce la firma de Inyección SQL y envía un paquete TCP RST, cortando la comunicación antes de que alcance al servidor Apache.
<img width="685" height="328" alt="image" src="https://github.com/user-attachments/assets/80c560e5-b6a2-4330-8759-e436bd9cc2fc" />


---

### Evidencia 4 — Registros de auditoría (logs de seguridad)

**Prueba:** revisión del panel `Log & Report > Security Events` tras lanzar los ataques.

**Resultado esperado:** registros en color rojo en la categoría *WAF / Forward Traffic*.

**Explicación:** el sistema registra la IP de origen (`10.8.89.2`), la IP de destino (`10.8.89.130`) y la acción ejecutada (`blocked`), proporcionando trazabilidad forense del intento de ataque.

<img width="1013" height="403" alt="image" src="https://github.com/user-attachments/assets/aaa83a7f-c83b-41f6-9c9b-de1a8e5064cb" />

---

## 8. Conclusiones

La implementación validó de forma exitosa un esquema de **defensa en profundidad** sobre FortiGate, combinando:

- Segmentación de red mediante VLSM
- Políticas de firewall bajo el principio de mínimo privilegio
- Perfiles de seguridad de Capa 7 (DNS Filter, Application Control, IPS y WAF)

Las pruebas de validación confirmaron que el tráfico no autorizado —ya sea por protocolo, dominio, aplicación o firma de ataque— es identificado y bloqueado de forma consistente, y que cada evento queda registrado para su posterior auditoría forense.

---

## Estructura sugerida del repositorio

```
.
├── README.md
└── screenshots/
    ├── 01-interfaces.png
    ├── 02-static-route.png
    ├── 03-firewall-policies.png
    ├── 04-dns-filter.png
    ├── 05-application-control.png
    ├── 06-ips-sensor.png
    ├── 07-waf-signatures.png
    ├── 08-ping-bloqueado.png
    ├── 09-bloqueo-redes-sociales.png
    ├── 10-waf-sqli.png
    └── 11-logs-seguridad.png
```
