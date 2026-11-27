# Bitácora de Requerimientos y Auditoría IA — Sesión 24
**Materia:** Lenguajes y Autómatas II (SCD-1016)  
**Tema:** Control de Flujo: Sentencia if-else  
**Docente Titular:** MCC. Iván Márquez Larios (`ijmarquezl`)  
**Estudiante:** Pablo Uriel Desiderio López  
**Matrícula:** 24531372  
**Track:** C  

---

## 1. Fase 1: El Artesano (Dificultad Manual)
*Reto:* Saltos condicionales y evaluación booleana  
*Describe qué problema o restricción de memoria/sintaxis identificaste al codificar a mano:*
> Esta sesión me salió sin errores, pero tuve que entender el cortocircuito antes de escribir el código. En `a && b`, si `a` es falso ya no importa `b`, porque todo es falso. Por eso se usa `IFFALSE`: si el operando es falso, brinca directo a la salida. En `a || b` es al revés: si `a` es verdadero ya no importa `b`, y por eso se usa `IFTRUE`.
>
> Para elegir la palabra usé el operador `? :` que aprendí en la sesión 21: `(op == '&') ? "IFFALSE" : "IFTRUE"`. Así el resto de la función es igual para los dos operadores.
>
> También usé `lineas[n++]`, que escribe en la posición `n` y después la aumenta, así la segunda línea cae en `lineas[1]` y al final `n` ya dice cuántas líneas escribí. Cuando `b` es NULL la condición es simple y solo se emite un salto, sin inventar una segunda comparación.

---

## 2. Fase 2: El Científico (Ingeniería de Requerimientos C-R-E-O)

### 2.1 Matriz de Especificación Formal
| Elemento C-R-E-O | Definición Técnica |
| :--- | :--- |
| **Contexto (C)** | C99 / GCC o Rust 2021 sin bibliotecas no estándar |
| **Rol (R)** | Ingeniero de software de sistemas y compiladores |
| **Especificación (E)** | Cortocircuito lógico en expresiones |
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
Implementa exactamente estas declaraciones de api_sesion24.h, sin alterarlas:

int emitir_condicion(char lineas[][64], int max, const char *a, const char *b, char op, const char *etiqueta);

Reglas del contrato:
- Para & usa IFFALSE: si el operando es falso, se salta a la etiqueta de salida.
- Para | usa IFTRUE: si el operando es verdadero, se salta a la etiqueta de salida.
- Si el segundo operando es NULL, emite un solo salto; no inventes una comparación.

Indica la complejidad en tiempo y memoria.

[OUTPUT]
Entrega únicamente el código C y el análisis de complejidad.
No incluyas explicación en prosa fuera de los comentarios del código.
```

---

## 3. Código Devuelto por el Modelo
```c
#include "api_sesion24.h"
#include <stdio.h>

int emitir_condicion(char lineas[][64], int max, const char *a, const char *b, char op, const char *etiqueta) {
    if (lineas == NULL || a == NULL || etiqueta == NULL || max <= 0) return 0;
    const char *salto = (op == '&') ? "IFFALSE" : "IFTRUE";
    int n = 0;
    snprintf(lineas[n++], 64, "%s %s GOTO %s", salto, a, etiqueta);
    if (b != NULL && n < max)
        snprintf(lineas[n++], 64, "%s %s GOTO %s", salto, b, etiqueta);
    return n;
}
```

**Modelo usado:** Claude (Anthropic).

**Complejidad declarada por el modelo:** O(L) en tiempo por los `snprintf`, con L la longitud de los operandos, y O(1) en memoria. Emite como máximo dos líneas.

---

## 4. Fase 3: El Auditor (Resultados del Test Harness)
*Pruebas ejecutadas:* Condiciones compuestas con AND y OR

### Salida del harness

Ejecutado en Google Colab.

Contra la implementación propia de la Fase 1 (`test_artesano`):

```text
=== AUDITORIA SESION 24 - Saltos condicionales con cortocircuito ===
Caso                               esperado:              obtenido:              resultado
[ 1] a && b (primer salto)
     esperado: IFFALSE a GOTO L9
     obtenido: IFFALSE a GOTO L9
     -> OK
[ 2] a && b (segundo salto)
     esperado: IFFALSE b GOTO L9
     obtenido: IFFALSE b GOTO L9
     -> OK
[ 3] a && b (dos saltos)                esperado: 2            obtenido: 2            OK
[ 4] a || b (primer salto)
     esperado: IFTRUE a GOTO L9
     obtenido: IFTRUE a GOTO L9
     -> OK
[ 5] a || b (segundo salto)
     esperado: IFTRUE b GOTO L9
     obtenido: IFTRUE b GOTO L9
     -> OK
[ 6] condicion simple (b NULL)
     esperado: IFFALSE a GOTO L0
     obtenido: IFFALSE a GOTO L0
     -> OK

RESULTADO: 6/6 pruebas superadas
TODO EN VERDE. Documenta tu auditoria en auditorias/auditoria_sesion24.md
```

Contra el código devuelto por la IA (`test_ia`):

```text
=== AUDITORIA SESION 24 - Saltos condicionales con cortocircuito ===
Caso                               esperado:              obtenido:              resultado
[ 1] a && b (primer salto)
     esperado: IFFALSE a GOTO L9
     obtenido: IFFALSE a GOTO L9
     -> OK
[ 2] a && b (segundo salto)
     esperado: IFFALSE b GOTO L9
     obtenido: IFFALSE b GOTO L9
     -> OK
[ 3] a && b (dos saltos)                esperado: 2            obtenido: 2            OK
[ 4] a || b (primer salto)
     esperado: IFTRUE a GOTO L9
     obtenido: IFTRUE a GOTO L9
     -> OK
[ 5] a || b (segundo salto)
     esperado: IFTRUE b GOTO L9
     obtenido: IFTRUE b GOTO L9
     -> OK
[ 6] condicion simple (b NULL)
     esperado: IFFALSE a GOTO L0
     obtenido: IFFALSE a GOTO L0
     -> OK

RESULTADO: 6/6 pruebas superadas
TODO EN VERDE. Documenta tu auditoria en auditorias/auditoria_sesion24.md
```

* [x] Caso base y límites superados sin *Crash*.
* [x] No existen accesos fuera de límites ni *Memory Leaks* (en la versión corregida; ver hallazgos).
* [x] La complejidad teórica coincide con la implementada.

**Entorno:** GCC 13.3.0 (Ubuntu 24.04), compilado con `-std=c11 -Wall -Wextra -Werror`. Para los casos extra usé además `-fsanitize=address,undefined`.

*Nota de transparencia:* para buscar casos borde fuera del harness me apoyé en Claude para escribir los programas de prueba extra; cada resultado de abajo lo corrí y lo revisé.

La IA eligió `IFFALSE` para `&` y `IFTRUE` para `|`, emite un salto por operando y solo uno cuando `b` es NULL, que es el cortocircuito que pide el contrato. Pasa los seis casos. Lo que revisé es qué hace con un operador que no es `&` ni `|` y con poco espacio.

### Hallazgo 1: Cualquier operador que no es `&` se trata como `|`

La IA decide con `op == '&' ? "IFFALSE" : "IFTRUE"`. Entonces un operador inválido, como `'^'`, se traduce como si fuera OR, y el programa evalúa otra condición sin ningún aviso:

```text
op '^' (no es & ni |) -> 2 lineas: "IFTRUE a GOTO L1" / "IFTRUE b GOTO L1"
```

### Hallazgo 2: Con espacio para una sola línea, se pierde el segundo operando

Con `max = 1`, `a && b` emite solo el salto de `a` y devuelve 1. El código queda como si la condición fuera solo `a`, o sea, cambia el significado del programa:

```text
a && b con max=1 -> 1 linea: "IFFALSE a GOTO L1"
```

### Resultados tras endurecer la implementación

Mismas pruebas extra contra la versión corregida:

**Cualquier operador que no es `&` se trata como `|`** — si el operador no es `&` ni `|`, no emito nada y devuelvo 0.

```text
op '^' (no es & ni |) -> 0 lineas
```

**Con espacio para una sola línea, se pierde el segundo operando** — calculo antes cuántas líneas necesito; si no caben todas, no emito ninguna.

```text
a && b con max=1 -> 0 lineas
```

Y el harness sigue en verde con la versión corregida:

```text
RESULTADO: 6/6 pruebas superadas
TODO EN VERDE. Documenta tu auditoria en auditorias/auditoria_sesion24.md
```

| Caso borde | IA | Corregido | ¿Lo prueba el harness? |
| :--- | :--- | :--- | :--- |
| Los casos del harness | Pasa | Pasa | Sí |
| **Cualquier operador que no es `&` se trata como `|`** | **FALLA** | Pasa | No |
| **Con espacio para una sola línea, se pierde el segundo operando** | **FALLA** | Pasa | No |

**Complejidad verificada:** O(L) en tiempo por los `snprintf`, con L la longitud de los operandos, y O(1) en memoria. Emite como máximo dos líneas. Los chequeos que agregué son comparaciones constantes y no cambian el orden, salvo donde lo indico.

### Reporte Crítico
*¿Qué caso borde o problema omitió la IA y cómo lo solucionaste?*
> **1. Cualquier operador que no es `&` se trata como `|`.** *Lo que hice:* si el operador no es `&` ni `|`, no emito nada y devuelvo 0.
>
> **2. Con espacio para una sola línea, se pierde el segundo operando.** *Lo que hice:* calculo antes cuántas líneas necesito; si no caben todas, no emito ninguna.
>
> **Lo que me llevo.** El cortocircuito de la IA está bien, que es lo central de la sesión. Lo que falló es cómo maneja lo inesperado: un operador desconocido se convirtió en OR y una condición incompleta se emitió como si estuviera completa. Me quedo con que, cuando una traducción no se puede hacer completa, es mejor no emitir nada que emitir algo que significa otra cosa.

---
**Firma digital:** pablouriellopez68-sketch