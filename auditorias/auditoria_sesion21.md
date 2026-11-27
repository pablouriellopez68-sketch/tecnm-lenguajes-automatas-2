# Bitácora de Requerimientos y Auditoría IA — Sesión 21
**Materia:** Lenguajes y Autómatas II (SCD-1016)  
**Tema:** Fundamentos de Código de 3 Direcciones  
**Docente Titular:** MCC. Iván Márquez Larios (`ijmarquezl`)  
**Estudiante:** Pablo Uriel Desiderio López  
**Matrícula:** 24531372  
**Track:** C  

---

## 1. Fase 1: El Artesano (Dificultad Manual)
*Reto:* Representación en Cuádruplos (op,a,b,r)  
*Describe qué problema o restricción de memoria/sintaxis identificaste al codificar a mano:*
> Esta sesión me salió sin errores porque ya había usado `snprintf` en la sesión 9. Lo nuevo fue el contador como puntero: para leer el número hay que escribir `*contador`, y para aumentarlo `(*contador)++`. Los paréntesis son necesarios, porque `*contador++` movería el puntero en lugar de sumarle uno al número. El orden también importa: primero escribo `_t` con el valor actual y después incremento, por eso la primera llamada da `_t0`.
>
> En `emitir_cuadruplo` aprendí el operador `? :`: `a1 ? a1 : "_"` significa "si `a1` existe úsalo, y si es NULL pon un guion bajo". Es como un `if` / `else` dentro de una sola expresión, y me sirvió para no escribir cuatro `if` distintos.
>
> También entendí para qué sirven los cuádruplos: cada operación se vuelve una línea con el operador, sus dos argumentos y dónde se guarda el resultado. Cuando falta un argumento, como en una asignación `x = a` o en un `goto`, el guion bajo deja claro que ese lugar está vacío a propósito.

---

## 2. Fase 2: El Científico (Ingeniería de Requerimientos C-R-E-O)

### 2.1 Matriz de Especificación Formal
| Elemento C-R-E-O | Definición Técnica |
| :--- | :--- |
| **Contexto (C)** | C99 / GCC o Rust 2021 sin bibliotecas no estándar |
| **Rol (R)** | Ingeniero de software de sistemas y compiladores |
| **Especificación (E)** | Generación de temporales únicos (_t0..) |
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
Implementa exactamente estas declaraciones de api_sesion21.h, sin alterarlas:

void nuevo_temporal(char *dst, int cap, int *contador);
void emitir_cuadruplo(char *dst, int cap, const char *op, const char *a1, const char *a2, const char *res);

Reglas del contrato:
- El contador se usa y después se incrementa: la primera llamada siempre produce _t0.
- Un argumento NULL se escribe como guion bajo en el cuádruplo; no se omite ni se inventa.
- Escribe siempre en el buffer recibido: no reserves memoria ni uses variables globales.

Indica la complejidad en tiempo y memoria.

[OUTPUT]
Entrega únicamente el código C y el análisis de complejidad.
No incluyas explicación en prosa fuera de los comentarios del código.
```

---

## 3. Código Devuelto por el Modelo
```c
#include "api_sesion21.h"
#include <stdio.h>

void nuevo_temporal(char *dst, int cap, int *contador) {
    if (dst == NULL || cap <= 0 || contador == NULL) return;
    snprintf(dst, (size_t)cap, "_t%d", *contador);
    (*contador)++;
}

void emitir_cuadruplo(char *dst, int cap, const char *op, const char *a1, const char *a2, const char *res) {
    if (dst == NULL || cap <= 0) return;
    snprintf(dst, (size_t)cap, "(%s, %s, %s, %s)",
             op ? op : "_", a1 ? a1 : "_", a2 ? a2 : "_", res ? res : "_");
}
```

**Modelo usado:** Claude (Anthropic).

**Complejidad declarada por el modelo:** O(L) en tiempo, donde L es la longitud del texto que arma `snprintf`, y O(1) en memoria: escribe directo en el buffer del llamador, sin `malloc` ni variables globales.

---

## 4. Fase 3: El Auditor (Resultados del Test Harness)
*Pruebas ejecutadas:* Expresiones aritméticas compuestas

### Salida del harness

Ejecutado en Google Colab.

Contra la implementación propia de la Fase 1 (`test_artesano`):

```text
=== AUDITORIA SESION 21 - Cuadruplos y temporales unicos ===
Caso                               esperado:              obtenido:              resultado
[ 1] nuevo_temporal(_t0)                esperado: _t0          obtenido: _t0          OK
[ 2] nuevo_temporal(_t7)                esperado: _t7          obtenido: _t7          OK
[ 3] cuadruplo de a + b
     esperado: (+, a, b, _t0)
     obtenido: (+, a, b, _t0)
     -> OK
[ 4] cuadruplo de _t0 * c
     esperado: (*, _t0, c, _t1)
     obtenido: (*, _t0, c, _t1)
     -> OK
[ 5] asignacion simple                  esperado: (=, a, _, x) obtenido: (=, a, _, x) OK
[ 6] goto sin operandos
     esperado: (goto, L1, _, _)
     obtenido: (goto, L1, _, _)
     -> OK

RESULTADO: 6/6 pruebas superadas
TODO EN VERDE. Documenta tu auditoria en auditorias/auditoria_sesion21.md
```

Contra el código devuelto por la IA (`test_ia`):

```text
=== AUDITORIA SESION 21 - Cuadruplos y temporales unicos ===
Caso                               esperado:              obtenido:              resultado
[ 1] nuevo_temporal(_t0)                esperado: _t0          obtenido: _t0          OK
[ 2] nuevo_temporal(_t7)                esperado: _t7          obtenido: _t7          OK
[ 3] cuadruplo de a + b
     esperado: (+, a, b, _t0)
     obtenido: (+, a, b, _t0)
     -> OK
[ 4] cuadruplo de _t0 * c
     esperado: (*, _t0, c, _t1)
     obtenido: (*, _t0, c, _t1)
     -> OK
[ 5] asignacion simple                  esperado: (=, a, _, x) obtenido: (=, a, _, x) OK
[ 6] goto sin operandos
     esperado: (goto, L1, _, _)
     obtenido: (goto, L1, _, _)
     -> OK

RESULTADO: 6/6 pruebas superadas
TODO EN VERDE. Documenta tu auditoria en auditorias/auditoria_sesion21.md
```

* [x] Caso base y límites superados sin *Crash*.
* [x] No existen accesos fuera de límites ni *Memory Leaks* (en la versión corregida; ver hallazgos).
* [x] La complejidad teórica coincide con la implementada.

**Entorno:** GCC 13.3.0 (Ubuntu 24.04), compilado con `-std=c11 -Wall -Wextra -Werror`. Para los casos extra usé además `-fsanitize=address,undefined`.

*Nota de transparencia:* para buscar casos borde fuera del harness me apoyé en Claude para escribir los programas de prueba extra; cada resultado de abajo lo corrí y lo revisé.

La IA resolvió las dos funciones con `snprintf`, usa el contador y después lo incrementa (la primera llamada da `_t0`) y cambia cada argumento NULL por `_`. Pasa los seis casos. Lo que revisé fue qué pasa cuando el buffer es chico o el contador llega a un valor raro.

### Hallazgo 1: Con un buffer chico, dos temporales distintos salen con el mismo nombre

Con `cap = 3` solo caben dos caracteres, así que `_t12` y `_t13` se cortan a `_t`. La función igual avanza el contador, y el que llama recibe dos temporales que se llaman igual. En código intermedio eso significa que un resultado pisa al otro.

```text
cap=3, contador 12 y 13 -> "_t" y "_t" (contador queda en 14)
```

### Hallazgo 2: El contador se desborda en INT_MAX

`(*contador)++` sobre `INT_MAX` es comportamiento indefinido. UBSan lo marca y el contador queda negativo, así que el siguiente temporal sería `_t-2147483648`:

```text
ia_sesion21_c.c:12:16: runtime error: signed integer overflow: 2147483647 + 1 cannot be represented in type 'int'
contador INT_MAX -> "_t2147483647", contador queda en -2147483648
contador -5 -> "_t-5"
```

### Hallazgo 3: Un cuádruplo que no cabe se entrega cortado

Con `cap = 12` el cuádruplo queda a la mitad y sin el paréntesis de cierre. Quien lo lea después no puede saber cuál era el resultado:

```text
cuadruplo con cap=12 -> "(+, alfa, b"
```

### Resultados tras endurecer la implementación

Mismas pruebas extra contra la versión corregida:

**Con un buffer chico, dos temporales distintos salen con el mismo nombre** — reviso lo que devuelve `snprintf`: si el nombre no cupo completo, dejo la cadena vacía y no avanzo el contador.

```text
cap=3, contador 12 y 13 -> "" y "" (contador queda en 12)
```

**El contador se desborda en INT_MAX** — rechazo un contador negativo o en `INT_MAX` sin escribir nombre ni incrementar.

```text
contador INT_MAX -> "", contador queda en 2147483647
contador -5 -> ""
```

**Un cuádruplo que no cabe se entrega cortado** — si `snprintf` dice que no cupo, dejo la cadena vacía; un cuádruplo a medias es peor que ninguno.

```text
cuadruplo con cap=12 -> ""
```

Y el harness sigue en verde con la versión corregida:

```text
RESULTADO: 6/6 pruebas superadas
TODO EN VERDE. Documenta tu auditoria en auditorias/auditoria_sesion21.md
```

| Caso borde | IA | Corregido | ¿Lo prueba el harness? |
| :--- | :--- | :--- | :--- |
| Los casos del harness | Pasa | Pasa | Sí |
| **Con un buffer chico, dos temporales distintos salen con el mismo nombre** | **FALLA** | Pasa | No |
| **El contador se desborda en INT_MAX** | **CRASH** | Pasa | No |
| **Un cuádruplo que no cabe se entrega cortado** | **FALLA** | Pasa | No |

**Complejidad verificada:** O(L) en tiempo, donde L es la longitud del texto que arma `snprintf`, y O(1) en memoria: escribe directo en el buffer del llamador, sin `malloc` ni variables globales. Los chequeos que agregué son comparaciones constantes y no cambian el orden, salvo donde lo indico.

### Reporte Crítico
*¿Qué caso borde o problema omitió la IA y cómo lo solucionaste?*
> **1. Con un buffer chico, dos temporales distintos salen con el mismo nombre.** *Lo que hice:* reviso lo que devuelve `snprintf`: si el nombre no cupo completo, dejo la cadena vacía y no avanzo el contador.
>
> **2. El contador se desborda en INT_MAX.** *Lo que hice:* rechazo un contador negativo o en `INT_MAX` sin escribir nombre ni incrementar.
>
> **3. Un cuádruplo que no cabe se entrega cortado.** *Lo que hice:* si `snprintf` dice que no cupo, dejo la cadena vacía; un cuádruplo a medias es peor que ninguno.
>
> **Lo que me llevo.** El código de la IA es correcto con buffers grandes, que es lo que usa el harness. El problema es que `snprintf` evita escribir fuera del arreglo pero no evita entregar un dato cortado, y en código intermedio un nombre cortado no es un error que se note: es un temporal que choca con otro. Aprendí que el valor que devuelve `snprintf` también hay que revisarlo.

---
**Firma digital:** pablouriellopez68-sketch