# Implementación y Análisis de Tecnologías VPN: IPSec, DMVPN y L2TP

**Autor:** Robinson De Los Santos Santos

**Matricula:** 2024-1252

**Materia:** Seguridad de Redes  

**Plataforma de Simulación:** GNS3  

---

## 🗺️ Topología Global y Direccionamiento Base

Todo el laboratorio está construido sobre el bloque de red principal **`24.12.52.0`**, subneteado para cubrir los enlaces públicos (WAN) simulados y las redes locales (LAN). 

**Nota importante sobre la topología:** Los routers **R1** y **R2** se utilizan para todas las configuraciones Site-to-Site y L2TP. El router **R3** (Sucursal 2) se incorpora *exclusivamente* en las configuraciones de DMVPN para demostrar la conectividad multipunto.

### 🌐 Direccionamiento WAN (Red Pública / Simulación ISP)
* **ISP Gateway (Gi0/0):** `24.12.52.1 /29`
* **R1 WAN (Gi0/0):** `24.12.52.2 /29`
* **R2 WAN (Gi0/0):** `24.12.52.3 /29`
* **R3 WAN (Gi0/0):** `24.12.52.4 /29` *(Solo para DMVPN)*

### 🖥️ Direccionamiento LAN (Redes Privadas /27 - 255.255.255.224)
* **Sede Central / Hub (R1) - Red `24.12.52.32/27`**
  * Gateway (Gi0/1): `24.12.52.33`
  * **PC1:** `ip 24.12.52.34 255.255.255.224 24.12.52.33`
* **Sucursal 1 / Spoke 1 (R2) - Red `24.12.52.64/27`**
  * Gateway (Gi0/1): `24.12.52.65`
  * **PC2:** `ip 24.12.52.66 255.255.255.224 24.12.52.65`
* **Sucursal 2 / Spoke 2 (R3) - Red `24.12.52.96/27`** *(Solo para DMVPN)*
  * Gateway (Gi0/1): `24.12.52.97`
  * **PC3:** `ip 24.12.52.98 255.255.255.224 24.12.52.97`
##  Topologia para los punto a punto
<img width="581" height="469" alt="image" src="https://github.com/user-attachments/assets/41d1701a-6113-4383-bb6f-353b3be49807" />

## Topologia para los Multipunto

<img width="707" height="484" alt="image" src="https://github.com/user-attachments/assets/42490a1f-d810-4898-8d81-1cb20cef28d1" />


---

## 🏢 PARTE 1: IPSec IKEv1 (Site-to-Site)

### VPN Site-to-Site Basado en Políticas (IKEv1) - Escenario 1
**Objetivo:** Interconectar R1 y R2 cifrando el tráfico estipulado mediante una Lista de Control de Acceso (ACL) y un Crypto Map, utilizando el estándar clásico IKEv1.
* **Parámetros:** Encriptación AES-256, Hash SHA-256, Grupo DH 14, PSK `CISCO123`. Tráfico de interés: de `24.12.52.32/27` hacia `24.12.52.64/27`.
**[ `show crypto ipsec sa`]**
> *Explicación: Se evidencian los contadores `pkts encaps` y `pkts decaps` aumentando, confirmando que el tráfico entre PC1 y PC2 está siendo cifrado por el Crypto Map de IKEv1. También se observan las "local ident" y "remote ident" con las subredes LAN.*

### VPN Site-to-Site Basado en Enrutamiento (IKEv1) - Escenario 2
**Objetivo:** Eliminar el uso de ACLs y Crypto Maps complejos creando una interfaz lógica de túnel (VTI - Virtual Tunnel Interface) con IKEv1. Todo el tráfico enrutado a esta interfaz se encripta de forma transparente.
* **Parámetros:** Interfaz `Tunnel0` en R1 y R2, `tunnel mode ipsec ipv4`, IPsec Profile vinculado a la política IKEv1, rutas estáticas apuntando a la IP del túnel.
**[ `show ip interface brief`]**
> *Explicación: Se observa la interfaz `Tunnel0` en estado `up / up`, demostrando que la interfaz VTI está activa y el enrutamiento lógico maneja la seguridad IKEv1 sin necesidad de Crypto Maps.*

### VPN Site-to-Site con Túnel GRE (IKEv1) - Escenario 4 *(Nota: Numerado como 3 para mantener el orden secuencial de tu lista)*
**Objetivo:** Permitir el paso de protocolos de enrutamiento dinámico (Multicast) encapsulando el tráfico en GRE y cifrando el paquete con IKEv1/IPsec.
* **Parámetros:** Interfaz Tunnel modo GRE (`tunnel mode gre ip`), Transform-Set en modo Transporte (`mode transport`) para evitar doble encabezado, Enrutamiento OSPF/EIGRP activado sobre el túnel.
**[`show ip ospf neighbor` o `show ip eigrp neighbors`]**
> *Explicación: Se muestra que R1 y R2 han formado adyacencia de enrutamiento a través de la interfaz Tunnel0, confirmando que el túnel GRE soporta Multicast protegido por IKEv1.*

---

## 🚀 PARTE 2: IPSec IKEv2 (Site-to-Site)

### VPN Site-to-Site Basado en Políticas (IKEv2) - Escenario 4
**Objetivo:** Modernizar la VPN del Escenario 1 migrando la negociación a IKEv2 (más seguro, rápido y resistente a ataques DoS), manteniendo la selección de tráfico por ACL.
* **Parámetros:** IKEv2 Proposal (AES-CBC-256, SHA256), IKEv2 Profile, Keyring `CISCO123`, Transform-Set acoplado al Crypto Map.
**[`show crypto ikev2 sa`]**
> *Explicación: La captura muestra la Asociación de Seguridad (SA) de IKEv2 en estado `READY`, confirmando el establecimiento exitoso del canal seguro IKEv2 entre las IPs públicas de R1 y R2.*

### VPN Site-to-Site Basado en Enrutamiento (IKEv2) - Escenario 5
**Objetivo:** Aplicar los beneficios de enrutamiento simplificado de VTI utilizando la robustez de la negociación de IKEv2 en lugar de IKEv1.
* **Parámetros:** IKEv2 Profile aplicado directamente a la interfaz `Tunnel0` mediante IPsec Profile en modo Túnel.
**[`ping 24.12.52.66`]**
> *Explicación: El ping exitoso entre PC1 y PC2 demuestra la conectividad de extremo a extremo enrutada a través del túnel VTI y protegida con la robustez de IKEv2.*

### VPN Site-to-Site con Túnel GRE (IKEv2) - Escenario 6
**Objetivo:** Mantener el soporte Multicast de GRE, elevando la seguridad perimetral implementando perfiles IKEv2 en modo transporte.
* **Parámetros:** Interfaz Tunnel GRE protegida con IPsec Profile enlazado a IKEv2.
**[`show crypto ipsec sa`]**
> *Explicación: En el apartado "local ident" y "remote ident", el protocolo mostrado es el **47 (GRE)**, demostrando que IKEv2/IPsec está cifrando la carga útil GRE entre los routers.*

---

## 🕸️ PARTE 3: DMVPN (1 Hub y 2 Spokes)

*Nota: Aquí se integra el Router 3 (Spoke 2) y la PC3.*

### VPN Hub and Spoke Punto a Multipunto DMVPN Fase 2 con IKEv1 con Enrutamiento Dinámico - Escenario 7
**Objetivo:** Crear una red superpuesta donde los Spokes (R2 y R3) pueden establecer túneles directos entre ellos sin que el tráfico pase por el Hub (R1).
* **Parámetros:** Túnel `gre multipoint`, NHRP. **Truco Fase 2:** Comandos `no ip next-hop-self eigrp 1` y `no ip split-horizon eigrp 1` en el Hub para alterar la propagación de rutas. Seguridad IKEv1.
**[ `trace 24.12.52.98` (Hacia PC3)]**
> *Explicación: El traceroute demuestra que los paquetes de PC2 a PC3 viajan con un solo salto interno de túnel (Spoke a Spoke), evadiendo pasar por el Hub gracias a la Fase 2.*

### VPN Hub and Spoke Punto a Multipunto DMVPN Fase 3 con IKEv2 con Enrutamiento Dinámico - Escenario 8
**Objetivo:** Mejorar el escalamiento utilizando reescritura de rutas CEF (NHRP). IKEv2 levanta las sesiones dinámicas entre Spokes de forma más eficiente.
* **Parámetros:** Hub con `ip nhrp redirect`. Spokes con `ip nhrp shortcut`. Perfiles IKEv2.
**[`show ip route`]**
> *Explicación: Se resalta la ruta hacia la red de R3 (`24.12.52.96/27`) marcada con un símbolo de **Atajo NHRP (`H` o `%`)**, evidenciando que la Fase 3 sobreescribió la ruta para usar el túnel directo.*
**[ `show ip nhrp`]**
> *Explicación: Muestra el mapeo NHRP dinámico creado directamente hacia la IP pública de R3 (`24.12.52.4`) tras recibir el mensaje Redirect del Hub.*

---

## 💻 PARTE 4: L2TP (Client to Site)

### VPN Client to Site Punto a Multipunto IPSec IKEv1 con L2TP - Escenario 9
**Objetivo:** Proveer acceso corporativo seguro a usuarios remotos. R1 valida al cliente, establece un túnel L2TP y le asigna una IP de la red interna de forma cifrada mediante IKEv1/IPsec.
* **Parámetros:** VPDN activado, `Virtual-Template1`, Autenticación MS-CHAP-v2, Pool DHCP `24.12.52.200`. Transform-set en Modo Transporte.
**[`24.12.52.200`]**
> *Explicación: Confirma que la interfaz PPP (ppp0) subió exitosamente y el cliente remoto recibió la IP del pool local asignado en R1.*
**[ `show vpdn session` y `show crypto session`]**
> *Explicación: Demuestra la validación del usuario remoto (L2TP/PPP) y el estado `UP-ACTIVE` de la criptografía IPsec protegiendo la sesión del cliente.*
