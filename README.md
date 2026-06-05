# Ataque MitM mediante ARP Spoofing
**Nombre:** Henry Vicente Quezada | **Matrícula:** 2025-1332 | **Fecha:** Junio 2026

---

## 🎬 Video Demostrativo
https://youtu.be/iapwx4i7zfk?list=PLhmycmsx2nBtM_kPjpLdj-vl3sWWjShMS

---

## 1. Objetivo del Laboratorio
Demostrar cómo el protocolo ARP puede ser explotado para realizar un ataque Man in the Middle (MitM) mediante ARP Spoofing, interceptando el tráfico entre VPC10 y el gateway R1, y aplicar DAI (Dynamic ARP Inspection) para mitigarlo.

---

## 2. Objetivo del Script
Enviar respuestas ARP falsas tanto a la víctima como al gateway, asociando la MAC del atacante con las IPs de ambos, logrando que todo el tráfico entre ellos pase por la máquina Kali.

### 2.1 Parámetros Usados
| Parámetro | Descripción | Ejemplo |
|---|---|---|
| `victima` | IP de la víctima | `10.13.32.11` |
| `gateway` | IP del gateway | `10.13.32.1` |
| `interfaz` | Interfaz de red | `eth0` |

### 2.2 Requisitos
- Sistema operativo: **Kali Linux**
- Python 3.x
- Librería Scapy: `pip install scapy`
- Permisos de root: `sudo`
- IP forwarding habilitado (el script lo activa automáticamente)

---

## 3. Funcionamiento del Script
1. Obtiene la MAC real de la víctima y del gateway mediante ARP
2. Envía ARP Reply falso a la víctima: MAC del gateway → MAC de Kali
3. Envía ARP Reply falso al gateway: MAC de la víctima → MAC de Kali
4. Habilita IP forwarding para no cortar la conexión
5. Repite el proceso cada 2 segundos para mantener el envenenamiento
6. Al detener con Ctrl+C restaura las tablas ARP originales

```bash
sudo python3 arp_mitm.py <victima> <gateway> <interfaz>
sudo python3 arp_mitm.py 10.13.32.11 10.13.32.1 eth0
```

---

## 4. Documentación de la Red

### Topología
 <img width="682" height="584" alt="image" src="https://github.com/user-attachments/assets/f458562e-8e80-4a6f-b3bc-e59b984e8984" />

### Tabla de Direccionamiento
| Dispositivo | Interfaz | VLAN | IP | Máscara | Rol |
|---|---|---|---|---|---|
| R1 | e0/0.10 | 10 | 10.13.32.1 | /24 | Gateway VLAN10 TI |
| R1 | e0/0.20 | 20 | 10.13.33.1 | /24 | Gateway VLAN20 Gerencia |
| R1 | e0/0.99 | 99 | 10.13.99.1 | /24 | Gateway Management |
| SW1 | vlan 99 | 99 | 10.13.99.2 | /24 | Gestión Switch |
| VPC10 | eth0 | 10 | 10.13.32.11 | /24 | **Víctima** |
| VPC20 | eth0 | 20 | 10.13.33.11 | /24 | Cliente VLAN20 |
| Kali | eth0 | 10 | 10.13.32.5 | /24 | **Atacante** |

### VLANs
| VLAN | Nombre | Descripción |
|---|---|---|
| 10 | TI | Red de usuarios TI |
| 20 | Gerencia | Red de Gerencia |
| 99 | Management | Red de gestión |

### Interfaces SW1
| Puerto | Modo | VLAN | Conectado a |
|---|---|---|---|
| e0/0 | Trunk 802.1q | 10,20,99 | R1 |
| e0/1 | Access | 10 | VPC10 |
| e0/2 | Access | 20 | VPC20 |
| e0/3 | Access | 10 | **Kali Linux** |

---

## 5. Capturas de Pantalla

### Antes del ataque
<img width="633" height="463" alt="image" src="https://github.com/user-attachments/assets/d6fa853a-1695-49f0-8b0f-6fa36ff5ef5a" />

> 📷 `VPC10# show arp` — MAC real de R1: `aa:bb:cc:00:01:00`

### Script en ejecución
<img width="640" height="530" alt="image" src="https://github.com/user-attachments/assets/475e1a6a-ed6f-4e3e-8a1d-38d0f1b66abe" />

> 📷 Kali ejecutando `arp_mitm.py` mostrando paquetes ARP enviados

### Durante el ataque
<img width="643" height="474" alt="image" src="https://github.com/user-attachments/assets/43d4dd31-0723-4147-a682-81c0ccac4323" />

> 📷 `VPC10# show arp` — MAC de Kali como gateway: `00:0c:29:xx:xx:xx`

### Contramedida aplicada
<img width="621" height="454" alt="image" src="https://github.com/user-attachments/assets/4d756895-73c3-4fec-929d-46333c8f1705" />

> 📷 `SW1# show ip arp inspection statistics` — paquetes ARP bloqueados

---

## 6. Contramedidas

```
SW1(config)# ip dhcp snooping
SW1(config)# ip dhcp snooping vlan 10
SW1(config)# ip dhcp snooping vlan 20
SW1(config)# no ip dhcp snooping information option
SW1(config)# interface e0/0
SW1(config-if)# ip dhcp snooping trust
SW1(config-if)# ip arp inspection trust
SW1(config)# ip arp inspection vlan 10
SW1(config)# ip arp inspection vlan 20

! Verificación
SW1# show ip arp inspection statistics
SW1# show ip arp inspection
```

DAI valida cada paquete ARP contra la tabla de DHCP Snooping. Si la MAC/IP no coincide con un binding conocido, el paquete es descartado. El puerto hacia R1 se marca como trust para permitir su tráfico ARP legítimo.
