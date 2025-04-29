Guia detallada con código, lista de Hardware, pasos para montaje/Prueba. Bajo licencia CC BY-NC-SA. Asi que construye, modifica, comparte.. pero si comercializas, acuerdate de mi.
# Guía ANC Portátil - Silenciar sin Apagar

Proyecto abierto de cancelación activa de ruido para portátiles ruidosos.  
Hecho con piezas accesibles y mucho corazón.

---

## Descripción

Este proyecto permite construir un sistema de cancelación activa de ruido (ANC) utilizando un ESP32, micrófonos KY-037 y un altavoz. Ideal para reducir el ruido de los ventiladores de portátiles sin apagar el equipo.

---

## Hardware necesario

- 1x ESP32
- 2x Micrófonos KY-037
- 1x Amplificador PAM8403
- 1x Altavoz pequeño (4-8Ω)
- Cables, protoboard o soldaduras

---

## Código y funcionamiento

Usa un algoritmo NLMS adaptativo con buffer circular y compensación automática de offset DC.  
Optimizado para trabajar en tiempo real.

Consulta el archivo ForGithub Guía construcción ANC.pdf para detalles de montaje y calibración.

---

## Cómo usar

1. Carga el código en el ESP32 usando el IDE de Arduino.
2. Coloca los micrófonos correctamente: uno captando el ruido y otro como referencia.
3. Alimenta el ESP32 y escucha el milagro.

---

## Licencia

CC BY-NC-SA 4.0  
Puedes compartir, adaptar y construir sobre este proyecto siempre que:

- Des crédito al autor (Mohamed Lamarti).
- No lo uses con fines comerciales.
- Distribuyas bajo la misma licencia.

> Así que constrúyelo, modifícalo, compártelo...  
> pero si lo vendes, acuérdate de mí.

---

## Créditos

Autor: Mohamed Lamarti  
Inspirado por: la necesidad de silencio y la voluntad de compartir.

---

¡Dale a la humanidad algo que merezca oírse en paz!
