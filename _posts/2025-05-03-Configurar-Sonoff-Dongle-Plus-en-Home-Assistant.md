---
title: Configurar Sonoff Dongle Plus en Home Assistant
date: 2024-05-03
categories:
  - tutorial
  - devops
  - raspberry
---
# Configurar Sonoff Dongle Plus en Home Assistant

Para configurar el Sonoff Dongle Plus-P en nuestro home assistant, lo primero que debemos hacer es comprobar el puerto serie en el que está conectado. Para ello, ejecutaremos el siguiente comando:

```bash
ls -l /dev/serial/by-id/
```

Debería aparecer algo como esto:

```bash
lrwxrwxrwx 1 root root 13 Mar 27 10:00 usb-ITead_Sonoff_Zigbee_3.0_USB_Dongle_Plus_1234567890-if00-port0 -> ../../ttyUSB0
```

El nombre exacto puede variar, pero debería haber algo que contenga "Sonoff" o "ITead". La ruta que aparece después del `->` es el dispositivo actual (en este ejemplo, `/dev/ttyUSB0`).

Posteriormente, deberemos ir a Home Assistant, a la parte de *Configuración > Integraciones*. Buscamos la integración ZHA (Zigbee Home Assistant) y pulsamos sobre configurar.

Cuando nos solicite 
