# Rastreo de Dispositivos Móviles Mediante el Uso de Paquetes de Control 802.11 con Dispositivos de Baja Potencia

**Tesis para la Licenciatura en Sistemas**

**Autor:** Agustin C. Alexander  
**Supervisor:** Alexis Tcach Lufrano  
**Universidad Nacional de General Sarmiento**  
**Instituto de Industria (IDEI)**

---

## Introducción y Motivación

### El Contexto: Ciudades Inteligentes (Smart Cities)

### El Problema
- El monitoreo urbano (tráfico, flujo de personas) es crucial para la planificación
- Las soluciones actuales (cámaras, sensores en el asfalto) son **costosas** y **complejas** de desplegar

### La Pregunta de Investigación
¿Es posible monitorear el movimiento de dispositivos móviles de forma **pasiva**, **económica** y **escalable**, utilizando la infraestructura Wi-Fi que ya emiten?

### La Propuesta
Desarrollar un sistema de **bajo costo** que rastree dispositivos sin que el usuario necesite conectarse a una red o instalar una aplicación.

---

## Objetivo Principal y Alcance

### Objetivo Principal
Investigar, diseñar e implementar un sistema para estimar la ubicación y el recorrido de dispositivos móviles utilizando el RSSI de paquetes 802.11.

### Objetivos Específicos
1. **Evaluar la viabilidad** de usar microcontroladores de baja potencia (ESP8266) como sensores
2. **Desarrollar un modelo** para convertir la señal RSSI en una distancia estimada
3. **Implementar un algoritmo** de multilateración para calcular la posición 2D
4. **Validar el sistema** mediante simulaciones y experimentos de campo

### Alcance
- El trabajo se enfoca en un **prototipo funcional** para demostrar el concepto
- Se evita el problema de la aleatorización de MAC utilizando un dispositivo objetivo con una MAC fija

---

## Conceptos Fundamentales

### ¿Cómo funciona? Los Pilares Técnicos

#### 1. Captura de Paquetes 802.11
- Los dispositivos móviles emiten paquetes **Probe Request** buscando redes Wi-Fi
- Nuestros sensores ("Sniffers") en modo promiscuo capturan estos paquetes

#### 2. RSSI (Received Signal Strength Indicator)
- Es la "potencia" con la que se recibe la señal
- A mayor distancia, menor RSSI
- Es una métrica **ruidosa** y muy sensible al entorno

#### 3. Modelo de Path Loss
```
RSSI = -A - N * log(d)
```
- Fórmula matemática para convertir el RSSI en una distancia (d)
- A y N son constantes que se deben calibrar para cada entorno (perfilado)

#### 4. Multilateración
- Principio geométrico: si conocemos la distancia desde un punto a 3 o más sensores, podemos triangular su posición

---

## Arquitectura del Sistema

### Flujo de Datos del Sistema de Rastreo

```
1. Captura
   Sensores ESP8266 capturan paquetes y extraen (MAC, Seq Num, RSSI)
   ↓
2. Filtrado
   Se aplica un Filtro de Kalman para suavizar la señal RSSI y reducir el ruido
   ↓
3. Estimación de Distancia
   Se usa la fórmula de Path Loss (A, N) para convertir el RSSI filtrado en metros
   ↓
4. Multilateración
   Con las distancias de ≥4 sensores, se calcula la posición (x, y) del dispositivo
```

---

## Fase 1 - Experimento Simulado (NS3)

### Validación en un Entorno Ideal: Simulación

**Herramienta:** NS3 (Network Simulator 3)

**Propósito:**
- Validar la lógica del algoritmo de multilateración sin interferencias del mundo real
- Determinar la cantidad mínima de nodos necesarios

**Configuración:**
- Un nodo objetivo se mueve en una trayectoria conocida
- Ocho nodos "Sniffer" estáticos registran su señal

---

## Resultados de la Simulación

### La Predicción Funciona en Condiciones Ideales

**Gráfico:** Recorrido real (azul) vs. el predicho (verde) en la simulación NS3

**Hallazgos Clave:**
- El algoritmo reconstruye la trayectoria con **alta precisión** en la simulación
- Se confirmó que se necesitan **mínimo 4 nodos** para una predicción 2D estable
- La precisión disminuye drásticamente cuando el objetivo está fuera del alcance de suficientes nodos

---

## Fase 2 - Experimentos de Campo

### Pruebas en el Mundo Real: Infraestructura

**Hardware:**
- **Sniffers:** 8 nodos basados en ESP8266
- **Alimentación:** Baterías Li-ion
- **Central:** Una Raspberry Pi como servidor UDP y punto de acceso

**Entorno:**
- Campo de deportes y campus de la UNGS
- Se desplegó una grilla de **10x10 metros**

**Objetivo:**
- Un smartphone en modo Access Point para generar paquetes constantes (Beacon Frames)

---

## Resultados del Experimento de Campo

### Reconstrucción de la Trayectoria Real

**Gráfico:** Trayectoria real (azul) vs. la trayectoria estimada (verde) en el campus de la UNGS

**Observaciones:**
- El sistema logra capturar la **forma general** del movimiento
- Existen deformaciones y desplazamientos significativos en la trayectoria predicha
- La precisión es mayor en el **centro de la red** de sensores

---

## Análisis de Errores y Desafíos

### ¿Por qué la predicción no es perfecta?

#### 1. Interferencia Ambiental y Variabilidad de la Señal
- **La principal fuente de error**
- Reflexiones de señal (multipath), obstáculos (personas, edificios) y otras redes Wi-Fi alteran el RSSI

#### 2. Limitaciones del Hardware
- Antena del ESP8266 no es perfectamente omnidireccional
- Inestabilidad en la alimentación eléctrica de los nodos causó pérdida de datos

#### 3. El Gran Desafío Futuro: Aleatorización de MAC
- Los dispositivos modernos cambian su dirección MAC para evitar este tipo de rastreo
- Este trabajo lo eludió, pero es un obstáculo clave para un despliegue real

---

## Contribuciones del Trabajo

### Aportes Principales

#### 1. Prueba de Concepto Funcional
- Se demostró la viabilidad técnica de un sistema de rastreo **pasivo** y de **bajo costo**

#### 2. Caracterización del Rendimiento
- Se cuantificó el rango efectivo (**~10 metros**) para los ESP8266 en este tipo de aplicación

#### 3. Desarrollo de Herramientas de Software (Open Source)
- **Easy-Trilateration:** Biblioteca Python para multilateración
- **RSSI-Filter-Profiling:** Herramientas para el perfilado y filtrado de señales
- **NS3-RSSI-Trilateration:** Entorno de simulación completo

---

## Trabajo Futuro

### Próximos Pasos y Líneas de Investigación

#### Mejorar el Hardware
- Utilizar dispositivos con mejores antenas o múltiples radios para capturar y transmitir simultáneamente
- Añadir un GPS a los nodos para una calibración más precisa

#### Avanzar en el Software
- Desarrollar algoritmos de **Machine Learning** para:
  - Compensar la interferencia del entorno
  - Abordar el problema de la aleatorización de MAC

#### Expandir las Aplicaciones
- Realizar pruebas con vehículos para el caso de uso de **Smart Parking**
- Explorar la detección de patrones de movimiento (peatones, ciclistas)

---

## Conclusión

### Conclusiones

- Este trabajo **validó con éxito** que es posible rastrear pasivamente dispositivos móviles usando una red de sensores Wi-Fi de bajo costo
- Se logró un **prototipo funcional** que, a pesar de las interferencias del mundo real, captura la esencia del movimiento
- Los principales desafíos son la **variabilidad de la señal RSSI** y la **aleatorización de MAC**
- La investigación sienta una **base sólida** y proporciona herramientas de código abierto para futuros trabajos en el área de la sensórica urbana de bajo costo

---

## Fin

### Gracias por su atención

**Preguntas** 