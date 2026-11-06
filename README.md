# 🛡️ Segmentación de Red y Hardening de Capa 3 con Cisco IOS

## 📌 Resumen del Proyecto

Este proyecto documenta la implementación de una arquitectura de red segmentada para mitigar el riesgo de movimiento lateral de amenazas en un entorno corporativo. Se dividió un bloque de red global utilizando **Subnetting (VLSM /25)** para aislar el departamento de **Ventas** del departamento de **TI**, y se aplicó una **Lista de Control de Acceso (ACL) extendida** en un router Cisco para implementar una política de acceso restrictiva (*Zero Trust*).

---

## 🎯 Objetivo de Ciberseguridad

Evitar que la red crítica de TI sea accesible desde la red de Ventas sin supervisión. En caso de que un dispositivo de Ventas sea comprometido por malware (ej. Ransomware o un gusano de red), la segmentación y el filtrado en el router evitan el pivotaje o la propagación hacia los activos críticos.

---

## 📐 Topología y Diagrama de Red

```
[ PC Ventas ] (192.168.50.10)
      │
[ Switch 0 ]
      │
      ├── (Gig0/0/0 - Gateway: 192.168.50.1/25) ── [ Router Cisco ISR4331 ]
      │
[ PC TI ] ────────────────── [ Switch 1 ] ──────────────┘ (Gig0/0/1 - Gateway: 192.168.50.129/25)
(192.168.50.140)
```

---

## 🔢 Tabla de Direccionamiento IP

| Departamento | Red / Prefijo       | Rango Utilizable         | Gateway (Interfaz Router)   | Máscara de Subred |
| ------------ | ------------------- | ------------------------ | --------------------------- | ------------------ |
| **Ventas**   | `192.168.50.0/25`   | `192.168.50.1` - `126`   | `192.168.50.1` (Gig0/0/0)   | `255.255.255.128`  |
| **TI**       | `192.168.50.128/25` | `192.168.50.129` - `254` | `192.168.50.129` (Gig0/0/1) | `255.255.255.128`  |

---

## 🛠️ Configuración Aplicada en Router Cisco (CLI)

### 1. Creación de la Lista de Control de Acceso (ACL Extendida)

Se definió una regla estricta que bloquea el tráfico del bloque de Ventas hacia TI y, por defecto, deniega todo lo demás. A diferencia de una configuración tradicional, aquí no hay un permit any any al final; aplicamos el principio de menor privilegio, donde solo el tráfico explícitamente permitido fluye:

```
! Definición de la ACL 100
enable
configure terminal
! Bloqueo explícito del tráfico de Ventas hacia TI
access-list 100 deny ip 192.168.50.0 0.0.0.127 192.168.50.128 0.0.0.127
! Denegar todo lo demás por defecto (Zero Trust)
access-list 100 deny ip any any
```

Se aplicó la lista en la interfaz GigabitEthernet0/0/0 (puerta de entrada de Ventas) para interceptar el tráfico en el punto de origen:

```
interface GigabitEthernet0/0/0
 ip access-group 100 in
 exit
```

### Prueba 1: Verificación de Enrutamiento (Pre-ACL)

Antes de aplicar la política de seguridad, el router permitía la comunicación bidireccional entre ambas subredes a través de trazas ICMP (Ping exitoso con 0% de pérdida tras la resolución ARP inicial).

### Prueba 2: Validación del Firewall Interno (Post-ACL)

Se ejecutó un escaneo ICMP desde la PC de Ventas (192.168.50.10) intentando alcanzar a la PC de TI (192.168.50.140):

```
C:\>ping 192.168.50.140

Pinging 192.168.50.140 with 32 bytes of data:
Reply from 192.168.50.1: Destination host unreachable.
Reply from 192.168.50.1: Destination host unreachable.
Reply from 192.168.50.1: Destination host unreachable.
Reply from 192.168.50.1: Destination host unreachable.

Ping statistics for 192.168.50.140:
    Packets: Sent = 4, Received = 0, Lost = 4 (100% loss)
```

**Análisis del Resultado:** La respuesta `Destination host unreachable` proveniente del Gateway (`192.168.50.1`) confirma que el router descartó los paquetes en la interfaz de entrada, impidiendo que la solicitud llegase a la subred de TI.

---

## 📝 Troubleshooting y Lecciones Aprendidas

- **Mapeo Físico de Interfaces en Hardware Específico:** Durante el despliegue en la serie ISR4331, se identificó que la nomenclatura de interfaz utiliza tres niveles (`GigabitEthernet0/0/0` en lugar de `GigabitEthernet0/0`), lo que fue corregido identificando el direccionamiento físico en el mapa de puertos.
- **Uso de Máscaras Wildcard:** Se aplicó el cálculo inverso de la máscara /25 (`255.255.255.128` ➔ `0.0.0.127`) para la correcta lectura de bloques de subred en la sintaxis de Cisco IOS.
- **Puntos de Inspección Correctos:** Se validó que las pruebas de intrusión/conectividad deben realizarse desde los segmentos de origen específicos para activar la regla *inbound* en la interfaz del router.

---

*Laboratorio realizado con fines educativos en entorno controlado (Cisco Packet Tracer).*
