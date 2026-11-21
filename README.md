# Laboratorio 1: Tokenización y Guardrails para LLMs

CC3103 - Procesamiento de Lenguaje Natural

Pipeline básico que normaliza, revisa, protege, tokeniza y analiza texto antes de enviarlo a un modelo de lenguaje.

```text
Texto de entrada
  -> normalización básica
  -> detección de datos sensibles
  -> acción de guardrail (ALLOW / WARN / REDACT / BLOCK)
  -> tokenización
  -> estadísticas
  -> salida final
```

## Archivos

| Archivo | Contenido |
|---|---|
| `laboratorio_1_cesar_lopez.py` | Implementación completa del laboratorio. |
| `salida_ejecucion.txt` | Evidencia de ejecución (salida completa del programa). |
| `README.md` | Descripción, decisiones de diseño y reflexión. |

## Ejecución

Requiere Python 3.8+ y solo usa la biblioteca estándar (`re`, `collections`).

```bash
python laboratorio_1_cesar_lopez.py
```

Para regenerar la evidencia:

```bash
python laboratorio_1_cesar_lopez.py > salida_ejecucion.txt
```

## Contenido del programa

1. **Textos de prueba (Parte A):** 14 textos simulados que cubren correo, teléfono, URL, palabra sensible, DPI, número largo, una instrucción sospechosa, un correo ofuscado y textos sin datos sensibles.
2. **Normalización (Parte B):** `normalizar_espacios` elimina espacios repetidos y saltos de línea. No se convierte a minúsculas en el pipeline para no perder información.
3. **Tokenización (Parte C):** por espacios, Regex básica y Regex mixta (conserva correos y URLs). `demo_comparar_tokenizadores` las compara sobre varios textos y muestra una tabla de tokens y tokens únicos por tokenizador.
4. **Estadísticas (Parte D):** caracteres, total de tokens, tokens únicos y top 10 de frecuencias.
5. **Detección (Parte E):** patrones `EMAIL`, `URL`, `PHONE`, `SECRET_WORD`, `DPI` y `LONG_NUMBER`. Cada hallazgo incluye tipo, valor y posición.
6. **Guardrail (Parte F):**

   | Condición | Acción |
   |---|---|
   | Contiene `SECRET_WORD` | `BLOCK` |
   | Contiene `EMAIL`, `PHONE`, `DPI` o `LONG_NUMBER` | `REDACT` |
   | Contiene solo `URL` | `WARN` |
   | Sin hallazgos | `ALLOW` |

7. **Redacción (Parte G):** reemplaza por `[EMAIL_REDACTED]`, `[PHONE_REDACTED]`, `[DPI_REDACTED]` y `[NUMBER_REDACTED]`. Los textos bloqueados no se tokenizan ni se analizan.
8. **Ejercicios:** funciones que verifican la cobertura de los textos de prueba, prueban el patrón DPI con casos válidos e inválidos y validan la política de guardrail con casos controlados.

## Decisiones de diseño

- **Detección sin traslapes:** varios patrones se solapan (un DPI también cumple `PHONE`, y un número largo también puede cumplirlo). Los patrones se evalúan por prioridad (`SECRET_WORD`, `URL`, `EMAIL`, `DPI`, `LONG_NUMBER`, `PHONE`) y se descarta cualquier coincidencia que se traslape con una de mayor prioridad. Así cada fragmento se reporta una sola vez y con el tipo más específico.
- **Redacción por posiciones:** la redacción usa las posiciones de los hallazgos, por lo que siempre coincide con lo detectado y no hay reemplazos dobles.
- **URLs no redactadas:** según la política, una URL solo genera `WARN`. Si un texto se redacta por otro dato (por ejemplo un teléfono), la URL se conserva.

## Resultados

| # | Texto (resumen) | Hallazgos | Acción |
|---:|---|---|---|
| 1 | Ayuda con cuenta + correo | EMAIL | REDACT |
| 2 | Respuesta de GPT-4.1 | — | ALLOW |
| 3 | Teléfono + sitio web | PHONE, URL | REDACT |
| 4 | `API_KEY=...` | SECRET_WORD | BLOCK |
| 5 | Palabras sueltas + `www.` | URL | WARN |
| 6 | Código del curso y hora | — | ALLOW |
| 7 | Correo + teléfono con saltos de línea | EMAIL, PHONE | REDACT |
| 8 | DPI simulado | DPI | REDACT |
| 9 | Password temporal | SECRET_WORD | BLOCK |
| 10 | Documentación en URL | URL | WARN |
| 11 | Instrucción sospechosa pidiendo un token | SECRET_WORD | BLOCK |
| 12 | Expediente + fecha | LONG_NUMBER, PHONE (falso positivo) | REDACT |
| 13 | Correo ofuscado con `[arroba]` | — (falso negativo) | ALLOW |
| 14 | Texto sin datos sensibles | — | ALLOW |

La salida completa está en `salida_ejecucion.txt`.

## Reflexión

**Falsos positivos:** el patrón `PHONE` es muy general y marca cualquier secuencia de dígitos con espacios o guiones, por eso la fecha `20-11-2026` se redactó como teléfono. `SECRET_WORD` también bloquearía frases inocentes como "olvidé mi clave" o "un token es la unidad mínima de texto", porque detecta la palabra y no un secreto real.

**Falsos negativos:** "secreta" no coincide con `\bsecret\b`, el correo ofuscado "soporte [arroba] example [punto] com" pasó como `ALLOW`, y tampoco se detectarían una contraseña escrita sin la palabra clave delante, números escritos en letras, nombres de personas o direcciones. Además, un teléfono de 8 dígitos sin separadores se clasifica como `LONG_NUMBER`.

**Información perdida al redactar:** se pierde contexto útil para el modelo: el dominio del correo, el código de país del teléfono o si dos menciones se refieren al mismo dato. Esto puede empeorar la respuesta, por ejemplo si el usuario pide ayuda justamente con su correo.

**¿Es suficiente Regex?** No para una empresa real: no entiende contexto, es fácil de evadir con variaciones de formato y no cubre datos sin estructura fija.

**Capas adicionales:** agregaría un detector de PII basado en NER (por ejemplo spaCy o Presidio), detección de secretos por entropía, validación de formatos con checksum, seudonimización reversible guardando el mapeo de forma segura, revisión de la salida del modelo y controles de acceso con auditoría de lo que se envía.

> Todos los datos utilizados en este laboratorio son simulados.
