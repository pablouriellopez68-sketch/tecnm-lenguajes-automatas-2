# Bitácora de Requerimientos y Auditoría IA — Sesión 23
**Materia:** Lenguajes y Autómatas II (SCD-1016)  
**Tema:** Etiquetas y Saltos Incondicionales  
**Docente Titular:** MCC. Iván Márquez Larios (`ijmarquezl`)  
**Estudiante:** Pablo Uriel Desiderio López  
**Matrícula:** 24531372  
**Track:** C  

---

## 1. Fase 1: El Artesano (Dificultad Manual)
*Reto:* Generador de etiquetas e instrucciones goto  
*Describe qué problema o restricción de memoria/sintaxis identificaste al codificar a mano:*
> Esta sesión me salió sin errores porque `nueva_etiqueta` es igual a `nuevo_temporal` de la sesión 21: solo cambia `_t` por `L`. Escribo la etiqueta con el valor actual del contador (`*contador`) y después lo incremento con `(*contador)++`, así la primera es `L0`.
>
> En `emitir_flujo` lo que tuve que entender es la diferencia entre un salto y una marca: `GOTO L2` significa "brinca a L2", y `L2:` significa "aquí está el lugar L2". La misma etiqueta se usa de las dos formas, y `es_salto` decide cuál se escribe.
>
> Para el caso NULL usé `dst[0] = '\0'`: con poner el fin de cadena en la primera posición, la cadena queda vacía. Ya lo había visto en el esqueleto de la sesión 9, que hacía exactamente eso.

---

## 2. Fase 2: El Científico (Ingeniería de Requerimientos C-R-E-O)

### 2.1 Matriz de Especificación Formal
| Elemento C-R-E-O | Definición Técnica |
| :--- | :--- |
| **Contexto (C)** | C99 / GCC o Rust 2021 sin bibliotecas no estándar |
| **Rol (R)** | Ingeniero de software de sistemas y compiladores |
| **Especificación (E)** | Trazabilidad del flujo de control |
| **Output (O)** | Solo código modular y análisis de complejidad |

### 2.2 Prompt Estructurado
```text
[CONTEXTO]
Estándar C11 compilado con GCC usando -std=c11 -Wall -Wextra -Werror.
Cualquier advertencia cuenta como error. Solo biblioteca estándar.
El archivo se compila junto con un harness que ya trae main(), así que
no debe incluir main().

[ROL]
Actúa como ingeniero de software de sistemas y compiladores.

[ESPECIFICACIÓN]
Implementa exactamente estas declaraciones de api_sesion23.h, sin alterarlas:

void nueva_etiqueta(char *dst, int cap, int *contador);
void emitir_flujo(char *dst, int cap, const char *etiqueta, int es_salto);

Reglas del contrato:
- nueva_etiqueta() usa el valor actual y después lo incrementa: la primera etiqueta es L0.
- emitir_flujo() con es_salto distinto de cero escribe GOTO; con cero escribe la marca con dos puntos.
- Una etiqueta NULL produce cadena vacía; nunca escribas fuera de cap.

Indica la complejidad en tiempo y memoria.

[OUTPUT]
Entrega únicamente el código C y el análisis de complejidad.
No incluyas explicación en prosa fuera de los comentarios del código.
```

---

## 3. Código Devuelto por el Modelo
```c
#include "api_sesion23.h"
#include <stdio.h>

void nueva_etiqueta(char *dst, int cap, int *contador) {
    if (dst == NULL || cap <= 0 || contador == NULL) return;
    snprintf(dst, (size_t)cap, "L%d", *contador);
    (*contador)++;
}

void emitir_flujo(char *dst, int cap, const char *etiqueta, int es_salto) {
    if (dst == NULL || cap <= 0) return;
    if (etiqueta == NULL) { dst[0] = '\0'; return; }
    if (es_salto) snprintf(dst, (size_t)cap, "GOTO %s", etiqueta);
    else          snprintf(dst, (size_t)cap, "%s:", etiqueta);
}
```

**Modelo usado:** Claude (Anthropic).

**Complejidad declarada por el modelo:** O(L) en tiempo por `snprintf`, con L la longitud de la etiqueta, y O(1) en memoria.

---

## 4. Fase 3: El Auditor (Resultados del Test Harness)
*Pruebas ejecutadas:* Saltos adelante y atrás estructurados

### Salida del harness

Ejecutado en Google Colab.

Contra la implementación propia de la Fase 1 (`test_artesano`):

```text
=== AUDITORIA SESION 23 - Etiquetas y saltos incondicionales ===
Caso                               esperado:              obtenido:              resultado
[ 1] nueva_etiqueta(L0)                 esperado: L0           obtenido: L0           OK
[ 2] nueva_etiqueta(L5)                 esperado: L5           obtenido: L5           OK
[ 3] emitir_flujo(GOTO L2)              esperado: GOTO L2      obtenido: GOTO L2      OK
[ 4] emitir_flujo(marca L2)             esperado: L2:          obtenido: L2:          OK
[ 5] emitir_flujo(NULL)                 esperado:              obtenido:              OK
[ 6] nueva_etiqueta(L10)                esperado: L10          obtenido: L10          OK

RESULTADO: 6/6 pruebas superadas
TODO EN VERDE. Documenta tu auditoria en auditorias/auditoria_sesion23.md
```

Contra el código devuelto por la IA (`test_ia`):

```text
=== AUDITORIA SESION 23 - Etiquetas y saltos incondicionales ===
Caso                               esperado:              obtenido:              resultado
[ 1] nueva_etiqueta(L0)                 esperado: L0           obtenido: L0           OK
[ 2] nueva_etiqueta(L5)                 esperado: L5           obtenido: L5           OK
[ 3] emitir_flujo(GOTO L2)              esperado: GOTO L2      obtenido: GOTO L2      OK
[ 4] emitir_flujo(marca L2)             esperado: L2:          obtenido: L2:          OK
[ 5] emitir_flujo(NULL)                 esperado:              obtenido:              OK
[ 6] nueva_etiqueta(L10)                esperado: L10          obtenido: L10          OK

RESULTADO: 6/6 pruebas superadas
TODO EN VERDE. Documenta tu auditoria en auditorias/auditoria_sesion23.md
```

* [x] Caso base y límites superados sin *Crash*.
* [x] No existen accesos fuera de límites ni *Memory Leaks* (en la versión corregida; ver hallazgos).
* [x] La complejidad teórica coincide con la implementada.

**Entorno:** GCC 13.3.0 (Ubuntu 24.04), compilado con `-std=c11 -Wall -Wextra -Werror`. Para los casos extra usé además `-fsanitize=address,undefined`.

*Nota de transparencia:* para buscar casos borde fuera del harness me apoyé en Claude para escribir los programas de prueba extra; cada resultado de abajo lo corrí y lo revisé.

La IA usa el contador y después lo incrementa (la primera etiqueta es `L0`), escribe `GOTO Lk` o `Lk:` según `es_salto` y deja la cadena vacía con una etiqueta NULL. Los seis casos pasan. Lo que probé son los mismos casos raros que en la sesión 21, porque las funciones son casi iguales.

### Hallazgo 1: Contador negativo o en INT_MAX

Con un contador negativo sale `L-1`, que no es una etiqueta válida. Y en `INT_MAX` el incremento desborda:

```text
ia_sesion23_c.c:12:16: runtime error: signed integer overflow: 2147483647 + 1 cannot be represented in type 'int'
contador -1 -> "L-1"
contador INT_MAX -> "L2147483647", contador queda en -2147483648
```

### Hallazgo 2: Un GOTO que no cabe se queda sin destino

Con `cap = 6`, `GOTO L123` se corta a `GOTO `. Es un salto que no dice a dónde, y el resto del compilador lo trataría como instrucción válida:

```text
GOTO L123 con cap=6 -> "GOTO "
```

### Resultados tras endurecer la implementación

Mismas pruebas extra contra la versión corregida:

**Contador negativo o en INT_MAX** — un contador negativo o en `INT_MAX` deja la cadena vacía y no se incrementa.

```text
contador -1 -> ""
contador INT_MAX -> "", contador queda en 2147483647
```

**Un GOTO que no cabe se queda sin destino** — si `snprintf` dice que no cupo, dejo la cadena vacía; nunca un `GOTO` sin destino.

```text
GOTO L123 con cap=6 -> ""
```

Y el harness sigue en verde con la versión corregida:

```text
RESULTADO: 6/6 pruebas superadas
TODO EN VERDE. Documenta tu auditoria en auditorias/auditoria_sesion23.md
```

| Caso borde | IA | Corregido | ¿Lo prueba el harness? |
| :--- | :--- | :--- | :--- |
| Los casos del harness | Pasa | Pasa | Sí |
| **Contador negativo o en INT_MAX** | **CRASH** | Pasa | No |
| **Un GOTO que no cabe se queda sin destino** | **FALLA** | Pasa | No |

**Complejidad verificada:** O(L) en tiempo por `snprintf`, con L la longitud de la etiqueta, y O(1) en memoria. Los chequeos que agregué son comparaciones constantes y no cambian el orden, salvo donde lo indico.

### Reporte Crítico
*¿Qué caso borde o problema omitió la IA y cómo lo solucionaste?*
> **1. Contador negativo o en INT_MAX.** *Lo que hice:* un contador negativo o en `INT_MAX` deja la cadena vacía y no se incrementa.
>
> **2. Un GOTO que no cabe se queda sin destino.** *Lo que hice:* si `snprintf` dice que no cupo, dejo la cadena vacía; nunca un `GOTO` sin destino.
>
> **Lo que me llevo.** Igual que en la 21, la IA confió en que el buffer siempre alcanza. Lo que me llevo es que en código intermedio una instrucción cortada no truena: se ve como una instrucción más y el error aparece mucho después, cuando el programa salta a donde no debe.

---
**Firma digital:** pablouriellopez68-sketch