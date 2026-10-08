
# 1 ICMP y primer contacto con Wireshark.









### Configuración de red (equipo con Windows, adaptador Wi-Fi)

Obtenida con `ipconfig /all`:

| Dato | Valor |
|---|---|
| Adaptador | Intel(R) Wireless-AC 9560 (Wi-Fi) |
| Dirección IPv4 | 192.168.1.20 |
| Máscara de subred | 255.255.255.0 (/24) |
| Gateway por defecto | 192.168.1.1 |
| Dirección MAC | 84-5C-F3-5A-1D-79 |
| Servidor DHCP | 192.168.1.1 |



<img width="663" height="568" alt="image" src="https://github.com/user-attachments/assets/21f905b6-d0e7-4221-9054-b9855809ebf8" />





<img width="1568" height="638" alt="image" src="https://github.com/user-attachments/assets/bd270725-7994-4994-957a-708d776dfdb7" />



### Tabla de capas (Echo Request a 8.8.8.8, frame 14509)


<img width="1908" height="987" alt="image" src="https://github.com/user-attachments/assets/cdd5838c-6079-413f-a178-98079e1c3202" />



| Capa (Wireshark) | Origen | Destino | Campo que indica el protocolo de adentro |
|---|---|---|---|
| Ethernet II | 84:5c:f3:5a:1d:79 (Intel_5a:1d:79, mi placa Wi-Fi) | b8:9f:cc:c1:4c:50 (HuaweiTechno_c1:4c:50, mi router/gateway) | **Type: IPv4 (0x0800)** |
| Internet Protocol Version 4 | 192.168.1.20 | 8.8.8.8 | **Protocol: ICMP (1)** |
| Internet Control Message Protocol | No tiene direcciones propias: viaja entre las IP del encabezado IPv4. Se identifica con Type 8 (Echo request), Code 0, Identifier 1 y Sequence 945 | No tiene (ídem) | No tiene campo de protocolo: después del encabezado ICMP (8 bytes) vienen directamente los datos |
| Datos / payload (Data) | No tiene: son bytes sin encabezado propio, dentro del mensaje ICMP | No tiene | No hay protocolo adentro: son 32 bytes de relleno (`abcdefghijklmnopqrstuvwabcdefghi`) que el Reply devuelve idénticos |

### a) MAC destino

La MAC destino del Echo Request a 8.8.8.8 es `b8:9f:cc:c1:4c:50`, que **no** es la de 8.8.8.8 sino la de mi router (HuaweiTechno, gateway 192.168.1.1). Es idéntica a la MAC destino del ping al gateway (frame 14379). 8.8.8.8 está fuera de mi LAN, así que la PC entrega la trama al router y este la reenvía. Conclusión: la dirección MAC solo tiene alcance local (un salto) y cambia en cada salto; la dirección IP identifica a los extremos y se mantiene de punta a punta.

### b) Echo Request (frame 14509) vs. Echo Reply (frame 14510)


<img width="1635" height="978" alt="image" src="https://github.com/user-attachments/assets/53d454c2-39ae-4bc8-bb34-38fe505ed454" />





### b) Echo Request (frame 14509) vs. Echo Reply (frame 14510)

Wireshark los vincula con `[Response frame: 14510]` (en el Request) y `[Request frame: 14509]` (en el Reply).

| Capa | Campos que cambian | Campos que se mantienen |
|---|---|---|
| Ethernet | MAC origen y destino (se invierten); Type (0x0800 → 0x8100, por el tag 802.1Q del Reply) | Las dos MAC (solo cambian de rol) |
| IPv4 | IP origen y destino (se invierten); TTL (128 → 118); Identification (0xa889 → 0x0000); Header Checksum (0x0000 → 0x72f5) | Versión, Header Length, DSCP, Total Length (60), Flags, Fragment Offset, Protocol (ICMP) |
| ICMP | Type (8 → 0); Checksum (0x49aa → 0x51aa) | Code (0), Identifier (1), Sequence Number (945), Data (32 bytes, idénticos) |

**¿Por qué tiene sentido cada cambio?**

- **MAC e IP origen/destino:** se invierten porque el Reply viaja en sentido contrario.
- **Ethernet Type:** el router agregó un tag 802.1Q (VLAN ID 0, solo prioridad) al Reply; por eso el Type es 0x8100 y el frame mide 78 bytes en vez de 74.
- **TTL:** cada emisor define el suyo (mi PC arranca con 128) y cada router intermedio le resta 1.
- **Identification:** es un contador propio de cada emisor, no tiene por qué coincidir.
- **Header Checksum IP:** en el Request vale 0x0000 porque la placa lo calcula al enviar (checksum offloading); en el Reply aparece el valor calculado por quien lo envió.
- **ICMP Type:** distingue Request (8) de Reply (0).
- **ICMP Checksum:** cambia porque cambió el Type (0x49aa + 0x0800 = 0x51aa).

**¿Por qué el Identifier y el Sequence Number se mantienen?**

Porque `ping` los usa para asociar cada respuesta con su pedido: el Identifier identifica al proceso `ping` y el Sequence Number a cada pedido dentro de esa ejecución. Por eso el Reply los devuelve sin modificar, igual que el payload (eco).
### c) Payload

El payload está dentro del mensaje ICMP, después de Identifier y Sequence Number (campo "Data" en Wireshark). Tiene 32 bytes y contiene los caracteres `abcdefghijklmnopqrstuvwabcdefghi` (`6162636465666768696a6b6c6d6e6f7071727374757677616263646566676869`). En el Reply es exactamente igual: el Reply devuelve los mismos datos que recibió (eco).

Comparación con Linux: no se probó en otra computadora. Linux suele enviar 56 bytes de datos (incluye un timestamp y bytes incrementales), lo que sugiere que el contenido del payload lo define cada implementación del SO, ya que el estándar solo exige que el Reply lo devuelva igual.

### d) TTL

- Echo Request enviado: TTL = **128** (valor inicial de Windows).
- Echo Reply recibido de 8.8.8.8: TTL = **118**.

No son iguales porque cada emisor arranca con su propio TTL inicial y cada router que atraviesa el paquete le resta 1; si llega a 0, el router lo descarta (esto evita que los paquetes circulen indefinidamente por bucles de ruteo). Suponiendo que 8.8.8.8 respondió con TTL inicial 128, la diferencia de 10 indica unos 10 saltos en el camino de vuelta. Para comparar: el Reply del gateway llegó con TTL 64, porque está a un solo salto y no atravesó routers.

### e) Encapsulación (Echo Request, frame 14509)

```
┌─ Frame: 74 bytes ───────────────────────────────────────┐
│ ┌─ Ethernet II: 14 bytes ──────────────────────────────┐│
│ │ Dst b8:9f:cc:c1:4c:50 | Src 84:5c:f3:5a:1d:79 | 0800  ││
│ └──────────────────────────────────────────────────────┘│
│ ┌─ IPv4: 60 bytes (Total Length) ──────────────────────┐│
│ │ Cabecera IP: 20 bytes                                ││
│ │ 192.168.1.20 → 8.8.8.8 | TTL 128 | Protocol 1        ││
│ │ ┌─ ICMP: 40 bytes ─────────────────────────────────┐ ││
│ │ │ Cabecera ICMP: 8 bytes                           │ ││
│ │ │ Type 8 | Code 0 | Checksum | ID 1 | Seq 945      │ ││
│ │ │ ┌─ Data: 32 bytes ─────────────────────────────┐ │ ││
│ │ │ │ abcdefghijklmnopqrstuvwabcdefghi             │ │ ││
│ │ │ └──────────────────────────────────────────────┘ │ ││
│ │ └──────────────────────────────────────────────────┘ ││
│ └──────────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────┘
```

Verificación: 14 + 20 + 8 + 32 = 74 bytes.
