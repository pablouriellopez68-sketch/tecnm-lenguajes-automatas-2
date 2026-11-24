# Bitácora de Requerimientos y Auditoría IA — Sesión 17
**Materia:** Lenguajes y Autómatas II (SCD-1016)  
**Tema:** Comprobación Semántica de Retornos  
**Docente Titular:** MCC. Iván Márquez Larios (`ijmarquezl`)  
**Estudiante:** Pablo Uriel Desiderio López  
**Matrícula:** 24531372  
**Track:** C (el harness evalúa C; la versión en Rust es opcional y no se califica con este harness)  

---

## 1. Fase 1: El Artesano (Dificultad Manual)
*Reto:* Validación de tipo de sentencia return  
*Describe qué problema o restricción de memoria/sintaxis identificaste al codificar a mano:*
> ✏️ **PENDIENTE (Pablo):** esta fase es sin IA. Escribe aquí, con tus palabras, en qué te atoraste al programar `artesano_sesion17_c.c` a mano (errores de compilación, punteros, casos que no se te ocurrieron, etc.).

---

## 2. Fase 2: El Científico (Ingeniería de Requerimientos C-R-E-O)

### 2.1 Matriz de Especificación Formal
| Elemento C-R-E-O | Definición Técnica |
| :--- | :--- |
| **Contexto (C)** | C99 / GCC o Rust 2021 sin bibliotecas no estándar |
| **Rol (R)** | Ingeniero de software de sistemas y compiladores |
| **Especificación (E)** | Rutas de ejecución con retorno garantizado |
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
Implementa exactamente estas declaraciones de api_sesion17.h, sin alterarlas:

int diagnosticar_retorno(const char *tipo_funcion, const char *tipo_expresion, int rutas_totales, int rutas_con_return);

Reglas del contrato:
- La conversión solo se acepta cuando el tipo destino es igual o más ancho que el origen.
- Devolver un valor en una función void es siempre un error de tipo.
- Una firma NULL o un conteo de rutas incoherente (cero rutas o más retornos que rutas) devuelve -1.

Indica la complejidad en tiempo y memoria.

[OUTPUT]
Entrega únicamente el código C y el análisis de complejidad.
No incluyas explicación en prosa fuera de los comentarios del código.
```

---

## 3. Código Devuelto por el Modelo
```c
#include "api_sesion17.h"
#include <string.h>

static int rango(const char *t) {
    if (strcmp(t, "char") == 0)   return 1;
    if (strcmp(t, "short") == 0)  return 2;
    if (strcmp(t, "int") == 0)    return 3;
    if (strcmp(t, "long") == 0)   return 4;
    if (strcmp(t, "float") == 0)  return 5;
    if (strcmp(t, "double") == 0) return 6;
    return 0;
}

int diagnosticar_retorno(const char *tipo_funcion, const char *tipo_expresion, int rutas_totales, int rutas_con_return) {
    if (tipo_funcion == NULL || tipo_expresion == NULL) return -1;
    if (rutas_totales <= 0 || rutas_con_return > rutas_totales) return -1;

    if (strcmp(tipo_funcion, "void") == 0) {
        if (strcmp(tipo_expresion, "void") != 0) return 1;
    } else if (rango(tipo_funcion) < rango(tipo_expresion)) {
        return 1;
    }

    if (rutas_con_return < rutas_totales) return 2;
    return 0;
}
```

**Modelo usado:** Claude (Anthropic).

**Complejidad declarada por el modelo:** O(1) en tiempo y memoria: hace un número fijo de `strcmp` contra una lista de seis tipos.

---

## 4. Fase 3: El Auditor (Resultados del Test Harness)
*Pruebas ejecutadas:* Funciones sin retorno o retornos mixtos

### Salida del harness

Contra la implementación propia de la Fase 1:

```text
✏️ PENDIENTE (Pablo): pega aquí la salida de la celda
!gcc ... harness_sesion17_c.c artesano_sesion17_c.c -o test_artesano && ./test_artesano
```

Contra el código devuelto por la IA:

```text
=== AUDITORIA SESION 17 - Validacion de retornos y rutas ===
Caso                               esperado:              obtenido:              resultado
[ 1] int con int y dos rutas            esperado: 0            obtenido: 0            OK
[ 2] void devuelve valor                esperado: 1            obtenido: 1            OK
[ 3] ruta sin retorno                   esperado: 2            obtenido: 2            OK
[ 4] double acepta int                  esperado: 0            obtenido: 0            OK
[ 5] int no acepta double               esperado: 1            obtenido: 1            OK
[ 6] cero rutas de ejecucion            esperado: -1           obtenido: -1           OK

RESULTADO: 6/6 pruebas superadas
TODO EN VERDE. Documenta tu auditoria en auditorias/auditoria_sesion17.md
```

* [x] Caso base y límites superados sin *Crash*.
* [x] No existen accesos fuera de límites ni *Memory Leaks* (en la versión corregida; ver hallazgos).
* [x] La complejidad teórica coincide con la implementada.

**Entorno:** GCC 13.3.0 (Ubuntu 24.04), compilado con `-std=c11 -Wall -Wextra -Werror`. Para los casos extra usé además `-fsanitize=address,undefined`.

*Nota de transparencia:* para buscar casos borde fuera del harness me apoyé en Claude para escribir los programas de prueba extra; cada resultado de abajo lo corrí y lo revisé.

La IA revisa primero el tipo y después las rutas, como pide el contrato, y los seis casos pasan. El hueco está en `rango()`, que devuelve 0 para todo lo que no reconoce, incluido `void`.

### Hallazgo 1: Una función `int` acepta devolver `void`

Como `rango("void") = 0` y `rango("int") = 3`, la comparación `3 < 0` es falsa y la IA lo da por bueno. Es decir, `int f() { return g(); }` con `g` de tipo `void` pasa la revisión. El contrato dice que void con valor siempre es error, y aplica en los dos sentidos.

```text
int      <- void     rutas 1/1 -> 0
```

### Hallazgo 2: Tipos desconocidos se aceptan

Por la misma razón, cualquier tipo que no esté en la lista (`struct`, o `Int` con mayúscula) vale 0 y siempre "cabe":

```text
int      <- struct   rutas 1/1 -> 0
int      <- Int      rutas 1/1 -> 0
```

### Hallazgo 3: Conteo de rutas negativo

La IA revisa `rutas_con_return > rutas_totales` pero no que sea negativo, así que `-1` de 2 rutas se reporta como "ruta sin retorno" en vez de como datos incoherentes:

```text
int      <- int      rutas -1/2 -> 2
```

### Resultados tras endurecer la implementación

Mismas pruebas extra contra la versión corregida:

**Una función `int` acepta devolver `void`** — comparo `void` contra no-`void` en los dos sentidos antes de mirar rangos.

```text
int      <- void     rutas 1/1 -> 1
```

**Tipos desconocidos se aceptan** — un tipo desconocido devuelve -1, el mismo código que una firma inválida.

```text
int      <- struct   rutas 1/1 -> -1
int      <- Int      rutas 1/1 -> -1
```

**Conteo de rutas negativo** — `rutas_con_return < 0` también devuelve -1.

```text
int      <- int      rutas -1/2 -> -1
```

Y el harness sigue en verde con la versión corregida:

```text
RESULTADO: 6/6 pruebas superadas
TODO EN VERDE. Documenta tu auditoria en auditorias/auditoria_sesion17.md
```

| Caso borde | IA | Corregido | ¿Lo prueba el harness? |
| :--- | :--- | :--- | :--- |
| Los casos del harness | Pasa | Pasa | Sí |
| **Una función `int` acepta devolver `void`** | **FALLA** | Pasa | No |
| **Tipos desconocidos se aceptan** | **FALLA** | Pasa | No |
| **Conteo de rutas negativo** | **FALLA** | Pasa | No |

**Complejidad verificada:** O(1) en tiempo y memoria: hace un número fijo de `strcmp` contra una lista de seis tipos. Los chequeos que agregué son comparaciones constantes y no cambian el orden, salvo donde lo indico.

### Reporte Crítico
*¿Qué caso borde o problema omitió la IA y cómo lo solucionaste?*
> **1. Una función `int` acepta devolver `void`.** *Lo que hice:* comparo `void` contra no-`void` en los dos sentidos antes de mirar rangos.
>
> **2. Tipos desconocidos se aceptan.** *Lo que hice:* un tipo desconocido devuelve -1, el mismo código que una firma inválida.
>
> **3. Conteo de rutas negativo.** *Lo que hice:* `rutas_con_return < 0` también devuelve -1.
>
> **Lo que me llevo.** Los tres problemas vienen de lo mismo: usar 0 como valor de "no sé" dentro de una escala donde comparar con 0 tiene significado. La IA puso `void` en el mismo saco que los tipos desconocidos y los dos terminaron siendo "el tipo más chico", que siempre cabe. Una escala numérica necesita un valor que no se pueda comparar, o revisar antes si el dato es válido.

---
**Firma digital:** pablouriellopez68-sketch