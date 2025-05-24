# ANC Portátil con ESP32 (FxLMS)

**Autor:** Mohamed Lamarti Bentria

## Descripción Corta
Este proyecto implementa un sistema de Control Activo de Ruido (ANC) portátil utilizando un microcontrolador ESP32. El objetivo es reducir el ruido no deseado en un entorno específico mediante la generación de una señal de "anti-ruido". El sistema utiliza el algoritmo FxLMS (Filtered-X Least Mean Squares) para una adaptación y cancelación efectivas, e incluye calibración en el dispositivo de la ruta secundaria acústica y una interfaz serie para el ajuste de parámetros en tiempo real.

*(Puedes añadir aquí una o dos frases sobre el objetivo inicial, como la reducción de ruido del ventilador de un portátil, y su potencial de adaptación a otros entornos.)*

## Características Principales
* **Algoritmo Avanzado:** Implementación del algoritmo FxLMS para compensar la latencia acústica.
* **NLMS Optimizado:** Uso de Normalized Least Mean Squares (NLMS) con cálculo de potencia de la señal de referencia filtrada optimizado (O(1) incremental).
* **Calibración de Ruta Secundaria (S(z)) en el Dispositivo:** Rutina integrada para estimar los coeficientes del filtro de la ruta secundaria (`s_hat_coeffs`) usando LMS, esencial para FxLMS.
* **Procesamiento en Tiempo Real:** Rutina de servicio de interrupción (ISR) de alta frecuencia para la captura de audio, procesamiento y generación de la señal de anti-ruido.
* **Interfaz Serie Interactiva:** Permite el ajuste en tiempo real de parámetros clave como la tasa de aprendizaje (`mu`), la ganancia de salida (`output_gain`) y la activación/desactivación de NLMS.
* **Persistencia de Parámetros:** Guarda y carga automáticamente los parámetros de ajuste y los coeficientes de la ruta secundaria calibrada usando la biblioteca `Preferences` del ESP32.
* **Calibración Automática de Offset DC:** Los offsets de los micrófonos se calibran al inicio.

## Componentes Hardware
* **Microcontrolador:** ESP32 (cualquier placa de desarrollo con los pines necesarios).
* **Micrófono de Referencia:** 1x Módulo de micrófono KY-037 (o similar con salida analógica).
* **Micrófono de Error:** 1x Módulo de micrófono KY-037 (o similar con salida analógica).
* **Amplificador de Audio:** 1x Módulo amplificador PAM8403 (o similar).
* **Altavoz:** 1x Altavoz pequeño (ej. 8 Ohm, 0.5W - 2W).
* Cables de conexión, protoboard (opcional).

## Conexiones
A continuación se describen las conexiones principales. Se recomienda consultar los diagramas esquemáticos (`diagrama_esquematico.png`, `diagrama_fritzing.png` - *debes añadir estos archivos a tu repositorio*).

* **ESP32:**
    * `GPIO34` (ADC1_CH6) <- Salida analógica del Micrófono de Referencia.
    * `GPIO35` (ADC1_CH7) <- Salida analógica del Micrófono de Error.
    * `GPIO25` (DAC_CH1) -> Entrada de audio del amplificador PAM8403 (ej. INL o INR).
    * `3V3` -> Alimentación VCC de los micrófonos KY-037 y (opcionalmente) VCC del PAM8403 (ver notas).
    * `GND` -> Común para todos los componentes.
* **Micrófonos KY-037:**
    * `OUT` (o `AO`) -> Al pin ADC correspondiente del ESP32.
    * `GND` -> Al GND común del ESP32.
    * `VCC` -> Al pin 3V3 del ESP32.
* **Amplificador PAM8403:**
    * `VCC` -> A 3V3 o 5V. Si usas 5V (recomendado para más potencia), asegúrate de tener una fuente de 5V disponible (ej. pin VIN del ESP32 si se alimenta por USB). *El código y los esquemas iniciales se basaban en 3V3, ajusta según tus necesidades de potencia*.
    * `GND` -> Al GND común del ESP32.
    * `INL` o `INR` (una entrada) -> Al pin GPIO25 del ESP32.
    * `ROUT+` y `ROUT-` (o `LOUT+` y `LOUT-`) -> Al Altavoz.

*(Nota sobre PAM8403: Aunque puede funcionar a 3.3V, el PAM8403 generalmente ofrece mejor rendimiento y más potencia a 5V. Si alimentas el ESP32 vía USB, el pin VIN suele proporcionar 5V que podrías usar para el PAM8403. Asegúrate de que tu ESP32 y la fuente de alimentación puedan manejar la corriente total.)*

## Estructura del Software y Algoritmo
El firmware está escrito en C/C++ para el framework Arduino ESP32.

1.  **`setup()`**:
    * Inicializa la comunicación serie y la biblioteca `Preferences` (namespace "ancConfig").
    * Carga los parámetros guardados (`mu`, `output_gain`, `leakage_factor`, `USE_NLMS`) y los coeficientes `s_hat_coeffs`.
    * Configura los pines ADC para los micrófonos y el pin DAC para la salida de audio.
    * Realiza la calibración del offset DC para ambos micrófonos.
    * Configura e inicia un timer hardware para disparar la ISR `processANC_ISR` a la `SAMPLE_RATE_HZ` definida.
2.  **`loop()`**:
    * Llama a `handleSerialCommands()` para procesar comandos entrantes para ajuste de parámetros y calibración.
3.  **`processANC_ISR()`** (Rutina de Servicio de Interrupción - Núcleo del ANC):
    * Lee la muestra del micrófono de referencia (`x_n`).
    * Actualiza el buffer circular de la señal de referencia (`x_buffer`).
    * Filtra `x_buffer` con los coeficientes de la ruta secundaria estimada (`s_hat_coeffs`) para obtener la señal de referencia filtrada (`x_ref_filtered_buffer`) y actualiza su potencia (`x_ref_filtered_power`) de forma incremental (FxLMS).
    * Calcula la señal de anti-ruido (`y_n`) usando el filtro adaptativo principal (`weights`) y `x_buffer`.
    * Envía `y_n` al DAC (y luego al amplificador/altavoz).
    * Lee la muestra del micrófono de error (`e_n`).
    * Actualiza los pesos del filtro adaptativo (`weights`) usando `e_n` y `x_ref_filtered_buffer` según el algoritmo NLMS/LMS (FxLMS).
4.  **`calibrateSecondaryPath()`**:
    * Rutina para la estimación en línea de los coeficientes `s_hat_coeffs` de la ruta secundaria.
    * Genera una señal de excitación (ruido blanco limitado).
    * La envía directamente al DAC (con amplitud fija, independiente de `output_gain`).
    * Lee la respuesta del sistema a través del micrófono de error.
    * Utiliza un algoritmo LMS para adaptar los `s_est` y obtener `s_hat_coeffs`.
    * Guarda los coeficientes calibrados en `Preferences`.
5.  **`handleSerialCommands()`**:
    * Interpreta comandos recibidos por el puerto serie para:
        * Ajustar `mu` (tasa de aprendizaje).
        * Ajustar `output_gain` (ganancia de salida).
        * Activar/Desactivar NLMS (`nlms on`/`nlms off`).
        * Iniciar la calibración de `S_hat(z)` (`calibrate s_hat`).
        * Mostrar ayuda (`help`).
    * Guarda los parámetros modificados en `Preferences`.
6.  **Funciones Auxiliares:**
    * `calibrateDCOffset()`: Calcula el offset DC de los micrófonos.
    * `readMicrophone()`: Lee y procesa la señal del micrófono.
    * `updateReferenceNoiseBuffer()`: Gestiona el buffer de la señal de referencia.
    * `filterReferenceSignal_FxLMS()`: Realiza la convolución S(z)\*x(n) y actualiza la potencia.
    * `calculateFilterOutput_AntiNoise()`: Calcula la salida del filtro W(z).
    * `outputToDAC()`: Envía la señal procesada al DAC, aplicando la ganancia de salida.
    * `updateWeights_FxLMS()`: Implementa la actualización de pesos del filtro adaptativo.

## Instalación y Configuración
1.  **Clonar el Repositorio:**
    ```bash
    git clone https://tu_url_de_github/ANC-Portatil-ESP32.git
    cd ANC-Portatil-ESP32
    ```
2.  **Entorno de Desarrollo:**
    * Asegúrate de tener el entorno de desarrollo para ESP32 configurado, ya sea mediante el [Arduino IDE con el core de ESP32](https://github.com/espressif/arduino-esp32) o [PlatformIO](https://platformio.org/).
3.  **Bibliotecas:**
    * El código utiliza `Preferences.h`, que es parte del core de ESP32 para Arduino. No se requieren bibliotecas externas adicionales.
4.  **Conexión del Hardware:**
    * Conecta los componentes según la sección "Conexiones" y los diagramas que proporciones.
5.  **Subir el Código:**
    * Abre el archivo `.ino` principal en el Arduino IDE (o importa el proyecto en PlatformIO).
    * Selecciona tu placa ESP32 y el puerto COM correcto.
    * Sube el código a tu ESP32.
6.  **Monitor Serie:**
    * Abre el Monitor Serie a una velocidad de **115200 baudios** para ver los mensajes de depuración e interactuar con el sistema.

## Calibración y Uso

1.  **Offset DC:** La calibración del offset DC de los micrófonos se realiza automáticamente al iniciar el ESP32.
2.  **Calibración de la Ruta Secundaria S(z) (`s_hat_coeffs`):**
    * La primera vez que uses el sistema, o si cambias la configuración física del altavoz/micrófonos, necesitarás calibrar la ruta secundaria.
    * Envía el comando `calibrate s_hat` por el monitor serie.
    * El proceso tomará unos segundos. Los coeficientes resultantes se guardarán automáticamente.
    * Si no hay coeficientes guardados o si son cero, el sistema te advertirá al inicio.
3.  **Ajuste de Parámetros (Tuning):**
    * Una vez calibrada S(z), puedes ajustar los siguientes parámetros en tiempo real mediante comandos serie para optimizar la cancelación de ruido:
        * `mu <valor>`: Ajusta la tasa de aprendizaje. Ejemplo: `mu 0.00005`
        * `gain <valor>`: Ajusta la ganancia de salida. Ejemplo: `gain 0.7`
        * `nlms on` o `nlms off`: Activa o desactiva la normalización LMS.
    * Envía `help` para ver la lista de comandos disponibles.
    * Los cambios en `mu`, `gain` y `USE_NLMS` se guardan automáticamente en la memoria no volátil.

## Licencia
Este proyecto se distribuye bajo la licencia **Creative Commons Attribution-NonCommercial-ShareAlike 4.0 International (CC BY-NC-SA 4.0)**.
Ver detalles: [https://creativecommons.org/licenses/by-nc-sa/4.0/](https://creativecommons.org/licenses/by-nc-sa/4.0/)

---