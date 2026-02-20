# Clasificador de Logs Inteligente con Google Gemini (ETL Pipeline)

## 🙍 Autor:
- Jonathan Eduardo Castilla Zamora

## 📋 Descripción del Proyecto
Este proyecto implementa un script en Python diseñado para automatizar el análisis de registros de servidor (logs). Utiliza Modelos de Lenguaje Grande (LLM), específicamente **Google Gemini 2.5 Flash** (o 1.5 Flash), para leer logs no estructurados y transformarlos en datos estructurados (JSON) con etiquetas temáticas.

El sistema funciona como un pipeline **ETL (Extract, Transform, Load)**:
1.  **Extract:** Lee logs crudos desde un archivo de texto.
2.  **Transform:** Utiliza inferencia de IA para interpretar el contexto y asignar etiquetas (ej: "Database", "Timeout", "Security").
3.  **Load:** Guarda los resultados en Google Drive y descarga una copia local en formato JSON.

## 🚀 Cómo ejecutar el script

### Requisitos Previos
* Una cuenta de Google y acceso a [Google Colab](https://colab.research.google.com/).
* Una **API Key** válida de Google AI Studio.

### Instrucciones paso a paso en Google Colab
1.  **Instalación:** Ejecutar la celda que contiene `!pip install -U google-genai` para actualizar el SDK.
2.  **Configuración:**
    * Pega tu API Key en la variable `API_KEY`.
    * El script montará automáticamente tu Google Drive en la ruta `/content/drive/MyDrive/Gemini_Logs_Project`.
3.  **Ejecución:** Corre la celda principal.
    * Si no existe un archivo `logs.txt`, el script creará uno de prueba automáticamente.
    * Si tienes tus propios logs, súbelos a la carpeta `Gemini_Logs_Project` en tu Drive.
4.  **Resultados:** Al finalizar, el archivo `output.json` se guardará en tu Drive y se descargará automáticamente a tu equipo.

## 🛠️ Decisiones Técnicas Relevantes

Durante el desarrollo se tomaron decisiones de diseño orientadas a la robustez y eficiencia en entornos de producción limitados (Capa Gratuita):

### 1. Arquitectura de Cliente Moderno (`google-genai`)
Se migró de la biblioteca en desuso `google.generativeai` a la nueva versión `google-genai` (v1.0+). Esto permite un manejo de sesiones más seguro y orientado a objetos mediante la clase `Client`.

### 2. Salida Estructurada (JSON Mode)
Se configuró el parámetro `response_mime_type="application/json"` en la inferencia.
* **¿Por qué?:** Evita tener que usar Expresiones Regulares (Regex) para limpiar la respuesta del chat. El modelo se ve forzado a devolver un JSON sintácticamente válido, eliminando errores de parsing.

### 3. Estrategia de "Backoff Exponencial" (Manejo de Error 429)
La API gratuita tiene límites estrictos (Rate Limits). Si el servidor responde con error `429 RESOURCE_EXHAUSTED`, el script no falla inmediatamente.
* **Solución propuesta:** Implementación de un algoritmo de espera exponencial (`wait_time = 2^{intento} + jitter`).
* **Efecto:** Si falla, espera 2s, luego 4s, luego 8s... permitiendo que la cuota se restablezca automáticamente.

### 4. Patrón "Circuit Breaker" (Interruptor)
Para evitar bucles infinitos o bloqueos prolongados, se implementó un contador de fallos consecutivos.
* **Lógica:** Si **3 logs consecutivos** fallan tras agotar sus reintentos, el script asume una caída sistémica o agotamiento total de cuota y ejecuta una **Detención de Emergencia**, guardando todo el progreso realizado hasta ese momento.

### 5. Persistencia en Google Drive
Los entornos de Colab son efímeros (se borran al cerrar).
* **Solución:** Se integró `google.colab.drive` para guardar los resultados directamente en la nube del usuario, garantizando que los datos procesados no se pierdan ante una desconexión.