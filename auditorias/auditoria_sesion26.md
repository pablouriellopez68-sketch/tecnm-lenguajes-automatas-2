# Bitácora de Requerimientos y Auditoría IA — Sesión 26
**Materia:** Lenguajes y Autómatas II (SCD-1016)  
**Tema:** TAC para Llamadas a Funciones  
**Docente Titular:** MCC. Iván Márquez Larios (`ijmarquezl`)  
**Estudiante:** Pablo Uriel Desiderio López  
**Matrícula:** 24531372  
**Track:** C  

---

## 1. Fase 1: El Artesano (Dificultad Manual)
*Reto:* Instrucciones param, call y return  
*Describe qué problema o restricción de memoria/sintaxis identificaste al codificar a mano:*
> En esta sesión mi error se repitió dos veces y fue el mismo: confundí el formato de `snprintf` con el valor que lo llena. En los `PARAM` escribí `"PARAM %s", %d`, poniendo `%d` afuera de las comillas como si fuera una variable, cuando ahí iba el argumento (`a1` o `a2`). Luego en `emitir_return` repetí el formato y quedó `"RETURN %s", "RETURN %s", valor`, que hubiera escrito `RETURN RETURN %s` y además el compilador lo marca por tener un argumento de más. Al final entendí la regla: el formato va una sola vez entre comillas, y después de la coma van los valores que reemplazan cada `%s` o `%d`, en el mismo orden.
>
> Lo que sí entendí bien es cómo se arma una llamada en TAC: primero un `PARAM` por cada argumento que existe y después el `CALL` con la cantidad real de parámetros. Para eso usé la variable `params`, que solo aumenta cuando de verdad escribo un `PARAM`, así `p()` sin argumentos queda como `CALL p, 0`.
>
> También usé `inicio` para guardar dónde estaba `*n` al empezar, y al final regreso `*n - inicio`, que es cuántas líneas agregué. Mi versión escribe los dos `PARAM` uno por uno, mientras que la de la IA mete los argumentos en un arreglo y los recorre con un `for`: hacen lo mismo, pero la mía se lee más directo.

---

## 2. Fase 2: El Científico (Ingeniería de Requerimientos C-R-E-O)

### 2.1 Matriz de Especificación Formal
| Elemento C-R-E-O | Definición Técnica |
| :--- | :--- |
| **Contexto (C)** | C99 / GCC o Rust 2021 sin bibliotecas no estándar |
| **Rol (R)** | Ingeniero de software de sistemas y compiladores |
| **Especificación (E)** | Convención de paso de argumentos TAC |
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
Implementa exactamente estas declaraciones de api_sesion26.h, sin alterarlas:

int emitir_llamada(char lineas[][64], int max, int *n, const char *funcion, const char *a1, const char *a2, const char *resultado);
void emitir_return(char *dst, int cap, const char *valor);

Reglas del contrato:
- Cada argumento no NULL genera un PARAM, en el orden a1, a2.
- El número que acompaña a CALL es la cantidad real de PARAM emitidos.
- Si resultado es NULL, el CALL no asigna; si el valor de retorno es NULL, escribe solo RETURN.

Indica la complejidad en tiempo y memoria.

[OUTPUT]
Entrega únicamente el código C y el análisis de complejidad.
No incluyas explicación en prosa fuera de los comentarios del código.
```

---

## 3. Código Devuelto por el Modelo
```c
#include "api_sesion26.h"
#include <stdio.h>

int emitir_llamada(char lineas[][64], int max, int *n, const char *funcion, const char *a1, const char *a2, const char *resultado) {
    if (lineas == NULL || n == NULL || funcion == NULL) return 0;
    int inicio = *n, params = 0;
    const char *args[2] = { a1, a2 };
    for (int i = 0; i < 2; i++) {
        if (args[i] != NULL && *n < max) {
            snprintf(lineas[(*n)++], 64, "PARAM %s", args[i]);
            params++;
        }
    }
    if (*n < max) {
        if (resultado != NULL) snprintf(lineas[(*n)++], 64, "%s = CALL %s, %d", resultado, funcion, params);
        else                   snprintf(lineas[(*n)++], 64, "CALL %s, %d", funcion, params);
    }
    return *n - inicio;
}

void emitir_return(char *dst, int cap, const char *valor) {
    if (dst == NULL || cap <= 0) return;
    if (valor == NULL) snprintf(dst, (size_t)cap, "RETURN");
    else               snprintf(dst, (size_t)cap, "RETURN %s", valor);
}
```

**Modelo usado:** Claude (Anthropic).

**Complejidad declarada por el modelo:** O(L) en tiempo por los `snprintf`, con L la longitud de los nombres, y O(1) en memoria. Una llamada emite como máximo tres líneas (dos PARAM y el CALL).

---

## 4. Fase 3: El Auditor (Resultados del Test Harness)
*Pruebas ejecutadas:* Llamadas anidadas f(g(x))

### Salida del harness

Ejecutado en Google Colab.

Contra la implementación propia de la Fase 1 (`test_artesano`):

```text
=== AUDITORIA SESION 26 - Llamadas, parametros y retorno ===
Caso                               esperado:              obtenido:              resultado
[ 1] PARAM x                            esperado: PARAM x      obtenido: PARAM x      OK
[ 2] PARAM y                            esperado: PARAM y      obtenido: PARAM y      OK
[ 3] CALL con dos argumentos
     esperado: _t0 = CALL f, 2
     obtenido: _t0 = CALL f, 2
     -> OK
[ 4] llamada anidada f(g(x))
     esperado: _t1 = CALL f, 1
     obtenido: _t1 = CALL f, 1
     -> OK
[ 5] CALL sin argumentos
     esperado: _t0 = CALL p, 0
     obtenido: _t0 = CALL p, 0
     -> OK
[ 6] RETURN con valor                   esperado: RETURN _t0   obtenido: RETURN _t0   OK

RESULTADO: 6/6 pruebas superadas
TODO EN VERDE. Documenta tu auditoria en auditorias/auditoria_sesion26.md
```

Contra el código devuelto por la IA (`test_ia`):

```text
=== AUDITORIA SESION 26 - Llamadas, parametros y retorno ===
Caso                               esperado:              obtenido:              resultado
[ 1] PARAM x                            esperado: PARAM x      obtenido: PARAM x      OK
[ 2] PARAM y                            esperado: PARAM y      obtenido: PARAM y      OK
[ 3] CALL con dos argumentos
     esperado: _t0 = CALL f, 2
     obtenido: _t0 = CALL f, 2
     -> OK
[ 4] llamada anidada f(g(x))
     esperado: _t1 = CALL f, 1
     obtenido: _t1 = CALL f, 1
     -> OK
[ 5] CALL sin argumentos
     esperado: _t0 = CALL p, 0
     obtenido: _t0 = CALL p, 0
     -> OK
[ 6] RETURN con valor                   esperado: RETURN _t0   obtenido: RETURN _t0   OK

RESULTADO: 6/6 pruebas superadas
TODO EN VERDE. Documenta tu auditoria en auditorias/auditoria_sesion26.md
```

* [x] Caso base y límites superados sin *Crash*.
* [x] No existen accesos fuera de límites ni *Memory Leaks* (en la versión corregida; ver hallazgos).
* [x] La complejidad teórica coincide con la implementada.

**Entorno:** GCC 13.3.0 (Ubuntu 24.04), compilado con `-std=c11 -Wall -Wextra -Werror`. Para los casos extra usé además `-fsanitize=address,undefined`.

*Nota de transparencia:* para buscar casos borde fuera del harness me apoyé en Claude para escribir los programas de prueba extra; cada resultado de abajo lo corrí y lo revisé.

La IA emite un `PARAM` por cada argumento no NULL, en orden, y el `CALL` lleva el número real de parámetros emitidos. Sin resultado escribe `CALL f, n` y `RETURN` sin valor cuando es NULL. Los seis casos pasan, incluida la llamada anidada `f(g(x))`.

### Hallazgo 1: Con poco espacio se emiten los PARAM pero se pierde el CALL

Con `max = 1`, `f(x, y)` deja escrito `PARAM x` y nada más. La función devuelve 1 como si todo estuviera bien, y queda un parámetro apilado que ninguna llamada va a consumir:

```text
f(x,y) con max=1 -> devuelve 1, n=1, linea: "PARAM x"
```

### Hallazgo 2: `f(NULL, y)` cambia la posición del argumento

Si el primer argumento es NULL y el segundo no, se emite `PARAM y` y `CALL f, 1`. Eso cumple el contrato (cada argumento no NULL genera un PARAM), pero la función llamada recibe `y` como su primer parámetro. No lo cambié porque el contrato lo pide así, pero lo dejo anotado:

```text
f(NULL, y) -> "PARAM y" / "_t0 = CALL f, 1"
```

### Resultados tras endurecer la implementación

Mismas pruebas extra contra la versión corregida:

**Con poco espacio se emiten los PARAM pero se pierde el CALL** — calculo antes cuántas líneas hacen falta (los PARAM más el CALL); si no caben todas, no emito ninguna.

```text
f(x,y) con max=1 -> devuelve 0, n=0
```

**`f(NULL, y)` cambia la posición del argumento** — no lo cambié porque el contrato pide exactamente ese comportamiento; lo documento como limitación.

```text
f(NULL, y) -> "PARAM y" / "_t0 = CALL f, 1"
```

Y el harness sigue en verde con la versión corregida:

```text
RESULTADO: 6/6 pruebas superadas
TODO EN VERDE. Documenta tu auditoria en auditorias/auditoria_sesion26.md
```

| Caso borde | IA | Corregido | ¿Lo prueba el harness? |
| :--- | :--- | :--- | :--- |
| Los casos del harness | Pasa | Pasa | Sí |
| **Con poco espacio se emiten los PARAM pero se pierde el CALL** | **FALLA** | Pasa | No |
| `f(NULL, y)` cambia la posición del argumento | Limitación | Sin cambio (lo pide el contrato) | No |

**Complejidad verificada:** O(L) en tiempo por los `snprintf`, con L la longitud de los nombres, y O(1) en memoria. Una llamada emite como máximo tres líneas (dos PARAM y el CALL). Los chequeos que agregué son comparaciones constantes y no cambian el orden, salvo donde lo indico.

### Reporte Crítico
*¿Qué caso borde o problema omitió la IA y cómo lo solucionaste?*
> **1. Con poco espacio se emiten los PARAM pero se pierde el CALL.** *Lo que hice:* calculo antes cuántas líneas hacen falta (los PARAM más el CALL); si no caben todas, no emito ninguna.
>
> **2. `f(NULL, y)` cambia la posición del argumento.** *Lo que hice:* no lo cambié porque el contrato pide exactamente ese comportamiento; lo documento como limitación.
>
> **Lo que me llevo.** La secuencia PARAM/CALL de la IA es correcta, que es lo que importa para el Hito 3. El problema más serio es el mismo que vi en las sesiones 22, 24 y 25: emitir instrucciones a medias cuando no hay espacio. En el TAC, un PARAM sin su CALL desacomoda la pila de parámetros de la siguiente llamada. La regla que me llevo para todo el generador de código es: todo o nada.

---
**Firma digital:** pablouriellopez68-sketch