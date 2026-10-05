---
title: "¿Tu Wi-Fi resiste un deauth? WPA2 vs WPA3 en la práctica"
excerpt: "Monté el ataque clásico de deauth + captura de handshake contra mi propia red y comparé por qué WPA3 (PMF + SAE) rompe la cadena que WPA2 permite."
date: 2026-10-04
categories:
  - wifi
tags:
  - wifi
  - wpa2
  - wpa3
  - deauth
  - aircrack-ng
  - pmf
  - sae
  - purple-team
toc: true
toc_sticky: true
---

> ⚠️ **Lab autorizado.** Todo esto se hizo sobre **redes propias**. Deauth-ear o capturar
> handshakes de redes ajenas es ilegal. Los identificadores (BSSID/MAC) son placeholders.

## El objetivo

Quería responder algo concreto sobre mi propia casa: **¿mi Wi-Fi es vulnerable a un ataque
de desautenticación y captura de clave?** De paso, comparar mis dos bandas:

- `MI_RED_5G` → **WPA3-Personal (SAE/H2E)**, 5 GHz.
- `MI_RED_24` → **WPA2-PSK (CCMP)**, 2.4 GHz.

## Por qué el ataque depende de dos piezas

El ataque "clásico" encadena dos cosas, y **el AP decide** si cada una funciona:

1. **Deauth**: enviar tramas de desautenticación falsificadas para echar al cliente.
2. **Captura + crackeo**: grabar el **4-way handshake** cuando el cliente reconecta, y
   romperlo offline con diccionario.

Las defensas que lo rompen:

- **PMF / 802.11w** (Protected Management Frames) → firma las tramas de management, así que
  el deauth falsificado se ignora.
- **WPA3-SAE** (Dragonfly) → sustituye el handshake PSK por un *PAKE*; no hay nada
  crackeable offline.

## Montaje

Adaptador en modo monitor (un **Atheros AR9271**, solo 2.4 GHz — para 5 GHz hace falta uno
dual-band tipo MT7612U):

```bash
sudo airmon-ng check kill
sudo airmon-ng start wlan0        # -> wlan0mon
```

Reconocimiento y captura dirigida:

```bash
sudo airodump-ng --band bg wlan0mon                               # ubicar BSSID + canal
sudo airodump-ng -c <CH> --bssid AA:BB:CC:DD:EE:FF -w cap wlan0mon
```

Y el deauth dirigido al cliente (en otra terminal):

```bash
sudo aireplay-ng --deauth 10 -a AA:BB:CC:DD:EE:FF -c 11:22:33:44:55:66 --ignore-negative-one wlan0mon
```

## Resultados

| Red | Deauth | Handshake | Lectura |
|---|---|---|---|
| **WPA2-PSK** (2.4) | Cliente cae y reconecta | 4-way capturable → crackeable offline | 🔴 Vulnerable |
| **WPA3-SAE** (5) | Cliente **no** cae (PMF) | No hay 4-way crackeable | 🟢 Resistente |

El hallazgo que no esperaba: mi banda de **2.4 GHz seguía en WPA2-PSK** mientras la de 5 GHz
ya estaba en WPA3. La banda "vieja" era el eslabón débil de mi propia casa.

### Detalle que cuesta tiempo: la señal

Con el AP a **-76 dBm** la inyección del deauth es poco fiable. Rango práctico:

| RSSI | Para este ataque |
|---|---|
| -30 a -65 dBm | ✅ Fiable |
| -66 a -70 dBm | ⚠️ Límite |
| < -75 dBm | ❌ Inyección intermitente |

Mejor que fiarse del dBm, medir la inyección real:

```bash
sudo aireplay-ng --test -a AA:BB:CC:DD:EE:FF wlan0mon   # % de ACKs del AP
```

Y ojo: un **hotspot sin clientes conectados no produce handshake** — necesitas que alguien
se (re)asocie.

## Verificar la captura

```bash
aircrack-ng cap-01.cap            # "(1 handshake)" junto al BSSID = listo
# En Wireshark:  filtro  eapol   -> buscar "Message 2 of 4" / "3 of 4"
```

## Purple team: cómo lo detecto y lo mitigo

**Detección (blue):**
- El deauth genera **ruido** aunque no tumbe la red: ráfagas de deauth/disassoc.
- MITRE ATT&CK **T1498** (Network DoS). Alertable en el SIEM con logs del AP/controladora
  o un WIDS (tasa anómala de deauth por BSSID).

**Mitigación (CIS/NIST):**
- **PMF = Required** (no "Optional") en todos los SSID.
- **WPA3-only**; evitar el modo transición WPA2/WPA3 (permite downgrade).
- Si algo debe quedar en WPA2: PSK larga y aleatoria (≥20 chars) para encarecer el crackeo.
- Desactivar WPS. Segmentar IoT en SSID/VLAN aparte.

## Conclusión

El ataque "de siempre" **ya no funciona contra WPA3 bien configurado** (PMF + SAE + H2E,
sin modo transición). El riesgo real hoy está en: **bandas/SSID heredados en WPA2**, modo
transición, y evil-twin/phishing. Primer fix que apliqué en casa: **subir la 2.4 GHz a WPA3**.
