# Bitácora de Requerimientos y Auditoría IA — Sesión 22
**Materia:** Lenguajes y Autómatas II (SCD-1016)  
**Tema:** Generación de TAC para Expresiones  
**Docente Titular:** MCC. Iván Márquez Larios (`ijmarquezl`)  
**Estudiante:** Pablo Uriel Desiderio López  
**Matrícula:** 24531372  
**Track:** C  

---

## 1. Fase 1: El Artesano (Dificultad Manual)
*Reto:* Traductor de nodos AST a TAC  
*Describe qué problema o restricción de memoria/sintaxis identificaste al codificar a mano:*
> Esta sesión me salió sin errores, pero fue la que más me costó entender, porque usa recursión. La idea es que la función `traducir` se llama a sí misma con cada hijo. Si el nodo es una hoja, solo regresa su nombre (`n->valor`). Si es un operador, primero traduce el hijo izquierdo y el derecho, después pide un temporal nuevo, escribe la línea `_tX = izq op der` y regresa el nombre del temporal.
>
> Lo que me ayudó fue seguir a mano el ejemplo `a + b * c`: el `+` traduce primero la `a` (que regresa `"a"`) y luego el `*`, que escribe `_t0 = b * c` y regresa `"_t0"`. Hasta el final el `+` escribe `_t1 = a + _t0`. Por eso la multiplicación sale primero aunque esté a la derecha: el árbol ya trae la precedencia y el postorden la respeta.
>
> También aprendí que `%c` en `snprintf` imprime un solo carácter, que es lo que necesitaba para el operador, y que el temporal que regresa cada llamada es lo que conecta una línea con la siguiente.

---

## 2. Fase 2: El Científico (Ingeniería de Requerimientos C-R-E-O)

### 2.1 Matriz de Especificación Formal
| Elemento C-R-E-O | Definición Técnica |
| :--- | :--- |
| **Contexto (C)** | C99 / GCC o Rust 2021 sin bibliotecas no estándar |
| **Rol (R)** | Ingeniero de software de sistemas y compiladores |
| **Especificación (E)** | Conservación de precedencia de operadores |
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
Implementa exactamente estas declaraciones de api_sesion22.h, sin alterarlas:

typedef struct Nodo {
    char tipo;
    char op;
    char valor[16];
    struct Nodo *izq;
    struct Nodo *der;
} Nodo;

int generar_tac(const Nodo *raiz, char lineas[][64], int max);

Reglas del contrato:
- Primero se calculan los hijos y después se emite el operador: el recorrido es postorden.
- Cada operador usa un temporal nuevo: llama a nuevo_temporal, no inventes el nombre.
- Nunca escribas más de max líneas ni más de 63 caracteres por línea.

Indica la complejidad en tiempo y memoria.

[OUTPUT]
Entrega únicamente el código C y el análisis de complejidad.
No incluyas explicación en prosa fuera de los comentarios del código.
```

---

## 3. Código Devuelto por el Modelo
```c
#include "api_sesion22.h"
#include <stdio.h>
#include <string.h>

/* Los temporales ya estan resueltos: usalos tal cual. */
typedef struct {
    char nombre[64][16];
    int usados;
} Pool;

const char *nuevo_temporal(Pool *p, int *contador) {
    if (p->usados >= 64) return "_t?";
    snprintf(p->nombre[p->usados], sizeof p->nombre[0], "_t%d", (*contador)++);
    return p->nombre[p->usados++];
}

static const char *traducir(const Nodo *n, char lineas[][64], int max, int *total, Pool *p, int *contador) {
    if (n->tipo == 'H') return n->valor;
    const char *izq = traducir(n->izq, lineas, max, total, p, contador);
    const char *der = traducir(n->der, lineas, max, total, p, contador);
    const char *t = nuevo_temporal(p, contador);
    if (*total < max) {
        snprintf(lineas[*total], 64, "%s = %s %c %s", t, izq, n->op, der);
        (*total)++;
    }
    return t;
}

int generar_tac(const Nodo *raiz, char lineas[][64], int max) {
    if (raiz == NULL || lineas == NULL || max <= 0) return 0;
    Pool p;
    p.usados = 0;
    int contador = 0, total = 0;
    traducir(raiz, lineas, max, &total, &p, &contador);
    return total;
}
```

**Modelo usado:** Claude (Anthropic).

**Complejidad declarada por el modelo:** O(n) en tiempo, con n los nodos del árbol, porque cada nodo se visita una vez en postorden. En memoria es O(h) de pila por la recursión, con h la altura del árbol, más el `Pool` de 64 temporales (1 KB) en la pila.

---

## 4. Fase 3: El Auditor (Resultados del Test Harness)
*Pruebas ejecutadas:* Expresiones anidadas con paréntesis

### Salida del harness

Ejecutado en Google Colab.

Contra la implementación propia de la Fase 1 (`test_artesano`):

```text
=== AUDITORIA SESION 22 - Traductor de AST a TAC con precedencia ===
Caso                               esperado:              obtenido:              resultado
[ 1] a + b * c                          esperado: _t0 = b * c  obtenido: _t0 = b * c  OK
[ 2] a + b * c (2a linea)
     esperado: _t1 = a + _t0
     obtenido: _t1 = a + _t0
     -> OK
[ 3] (a + b) * c
     esperado: _t1 = _t0 * c
     obtenido: _t1 = _t0 * c
     -> OK
[ 4] x * (y - z) + w
     esperado: _t2 = _t1 + w
     obtenido: _t2 = _t1 + w
     -> OK
[ 5] hoja sin operaciones               esperado: 0            obtenido: 0            OK
[ 6] (a + b) * (c - d)
     esperado: _t2 = _t0 * _t1
     obtenido: _t2 = _t0 * _t1
     -> OK

RESULTADO: 6/6 pruebas superadas
TODO EN VERDE. Documenta tu auditoria en auditorias/auditoria_sesion22.md
```

Contra el código devuelto por la IA (`test_ia`):

```text
=== AUDITORIA SESION 22 - Traductor de AST a TAC con precedencia ===
Caso                               esperado:              obtenido:              resultado
[ 1] a + b * c                          esperado: _t0 = b * c  obtenido: _t0 = b * c  OK
[ 2] a + b * c (2a linea)
     esperado: _t1 = a + _t0
     obtenido: _t1 = a + _t0
     -> OK
[ 3] (a + b) * c
     esperado: _t1 = _t0 * c
     obtenido: _t1 = _t0 * c
     -> OK
[ 4] x * (y - z) + w
     esperado: _t2 = _t1 + w
     obtenido: _t2 = _t1 + w
     -> OK
[ 5] hoja sin operaciones               esperado: 0            obtenido: 0            OK
[ 6] (a + b) * (c - d)
     esperado: _t2 = _t0 * _t1
     obtenido: _t2 = _t0 * _t1
     -> OK

RESULTADO: 6/6 pruebas superadas
TODO EN VERDE. Documenta tu auditoria en auditorias/auditoria_sesion22.md
```

* [x] Caso base y límites superados sin *Crash*.
* [x] No existen accesos fuera de límites ni *Memory Leaks* (en la versión corregida; ver hallazgos).
* [x] La complejidad teórica coincide con la implementada.

**Entorno:** GCC 13.3.0 (Ubuntu 24.04), compilado con `-std=c11 -Wall -Wextra -Werror`. Para los casos extra usé además `-fsanitize=address,undefined`.

*Nota de transparencia:* para buscar casos borde fuera del harness me apoyé en Claude para escribir los programas de prueba extra; cada resultado de abajo lo corrí y lo revisé.

La IA hizo una función recursiva `traducir` que primero resuelve el hijo izquierdo, luego el derecho y al final emite el operador, que es justo el postorden que pide el contrato. Usa `nuevo_temporal` del esqueleto y respeta `max`. Pasa los seis casos, incluido `(a + b) * (c - d)`. Probé árboles más grandes y árboles mal formados.

### Hallazgo 1: Después de 64 operadores, todos los temporales se llaman `_t?`

El `Pool` del esqueleto solo guarda 64 nombres. Cuando se acaba, `nuevo_temporal` devuelve `_t?`, y la IA lo usa sin revisar. Con 70 operadores, a partir de la línea 64 todos los resultados se guardan en el mismo nombre:

```text
arbol con 70 operadores -> 70 lineas
  linea 63: _t63 = _t62 + a
  linea 64: _t? = _t63 + a
  linea 65: _t? = _t? + a
```

### Hallazgo 2: Si no caben todas las líneas, se cortan sin avisar

Con 10 operadores y `max = 4`, la IA escribe 4 líneas y devuelve 4. El que llama no tiene forma de saber que faltan 6 y que el resultado final nunca se calculó:

```text
arbol con 10 operadores y max=4 -> devuelve 4
```

### Hallazgo 3: Un operador sin hijo truena

Si un nodo `'B'` tiene `izq` en NULL, la recursión lee la dirección cero:

```text
ia_sesion22_c.c:23:10: runtime error: member access within null pointer of type 'const struct Nodo'
```

### Resultados tras endurecer la implementación

Mismas pruebas extra contra la versión corregida:

**Después de 64 operadores, todos los temporales se llaman `_t?`** — antes de emitir cuento los operadores; si son más de 64 o más que `max`, devuelvo -1 y no escribo nada.

```text
arbol con 70 operadores -> -1
```

**Si no caben todas las líneas, se cortan sin avisar** — el mismo conteo previo: si no cabe completo, devuelvo -1.

```text
arbol con 10 operadores y max=4 -> devuelve -1
```

**Un operador sin hijo truena** — el conteo previo valida que cada operador tenga sus dos hijos y que el tipo sea `'H'` o `'B'`; si no, devuelvo -1.

```text
operador sin hijo izquierdo...
-1
```

Y el harness sigue en verde con la versión corregida:

```text
RESULTADO: 6/6 pruebas superadas
TODO EN VERDE. Documenta tu auditoria en auditorias/auditoria_sesion22.md
```

| Caso borde | IA | Corregido | ¿Lo prueba el harness? |
| :--- | :--- | :--- | :--- |
| Los casos del harness | Pasa | Pasa | Sí |
| **Después de 64 operadores, todos los temporales se llaman `_t?`** | **FALLA** | Pasa | No |
| **Si no caben todas las líneas, se cortan sin avisar** | **FALLA** | Pasa | No |
| **Un operador sin hijo truena** | **CRASH** | Pasa | No |

**Complejidad verificada:** O(n) en tiempo, con n los nodos del árbol, porque cada nodo se visita una vez en postorden. En memoria es O(h) de pila por la recursión, con h la altura del árbol, más el `Pool` de 64 temporales (1 KB) en la pila. Los chequeos que agregué son comparaciones constantes y no cambian el orden, salvo donde lo indico.

### Reporte Crítico
*¿Qué caso borde o problema omitió la IA y cómo lo solucionaste?*
> **1. Después de 64 operadores, todos los temporales se llaman `_t?`.** *Lo que hice:* antes de emitir cuento los operadores; si son más de 64 o más que `max`, devuelvo -1 y no escribo nada.
>
> **2. Si no caben todas las líneas, se cortan sin avisar.** *Lo que hice:* el mismo conteo previo: si no cabe completo, devuelvo -1.
>
> **3. Un operador sin hijo truena.** *Lo que hice:* el conteo previo valida que cada operador tenga sus dos hijos y que el tipo sea `'H'` o `'B'`; si no, devuelvo -1.
>
> **Lo que me llevo.** El recorrido de la IA es correcto y la precedencia sale bien porque el árbol ya la trae. Lo que no hizo fue revisar los límites que el esqueleto ya traía (64 temporales) ni avisar cuando el resultado queda incompleto. En un compilador, un TAC al que le faltan líneas o con nombres repetidos genera un programa que hace otra cosa sin marcar ningún error, y eso es peor que fallar.

---
**Firma digital:** pablouriellopez68-sketch