# Bitácora de Requerimientos y Auditoría IA — Sesión 19
**Materia:** Lenguajes y Autómatas II (SCD-1016)  
**Tema:** Estructuras (struct) y Miembros  
**Docente Titular:** MCC. Iván Márquez Larios (`ijmarquezl`)  
**Estudiante:** Pablo Uriel Desiderio López  
**Matrícula:** 24531372  
**Track:** C  

---

## 1. Fase 1: El Artesano (Dificultad Manual)
*Reto:* Offsets de miembros en estructuras  
*Describe qué problema o restricción de memoria/sintaxis identificaste al codificar a mano:*
> Esta sesión se parecía a la 12, pero con varios campos seguidos. Para entenderla hice una tabla a mano con `struct { char a; int b; char c; }`: `a` va en 0, el cursor queda en 1, `b` se alinea a 4 y el cursor queda en 8, y `c` va en 8. Usé dos funciones auxiliares: `ancho`, con la tabla de tamaños, y `alinear`, con la misma fórmula de redondeo de la sesión 12.
>
> En `offset_campo` lo importante fue el orden dentro del ciclo: primero alinear el cursor al ancho del campo, luego revisar si es el que busco (y en ese caso regresar el cursor como su offset), y hasta después sumar lo que ocupa.
>
> En `tamano_struct` lo nuevo fue el redondeo final: el tamaño del struct completo tiene que ser múltiplo de su campo más grande. En el ejemplo el cursor termina en 9, pero como el campo más grande es `int` (4), el tamaño real es 12. Para eso guardo el ancho mayor en la variable `mayor`. Al principio escribí `return total;` para el struct vacío, que funciona porque `total` vale 0, pero lo cambié a `return 0;` porque se entiende mejor. También rechazo un `total` negativo, que no tiene sentido.

---

## 2. Fase 2: El Científico (Ingeniería de Requerimientos C-R-E-O)

### 2.1 Matriz de Especificación Formal
| Elemento C-R-E-O | Definición Técnica |
| :--- | :--- |
| **Contexto (C)** | C99 / GCC o Rust 2021 sin bibliotecas no estándar |
| **Rol (R)** | Ingeniero de software de sistemas y compiladores |
| **Especificación (E)** | Alineación de memoria por palabras |
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
Implementa exactamente estas declaraciones de api_sesion19.h, sin alterarlas:

typedef struct Campo {
    const char *nombre;
    const char *tipo;
} Campo;

int offset_campo(const Campo *campos, int total, const char *nombre);
int tamano_struct(const Campo *campos, int total);

Reglas del contrato:
- Cada campo se alinea a su propio ancho antes de asignarle offset.
- El tamaño total se redondea al múltiplo del campo más grande.
- Un campo inexistente, una lista NULL o un tipo desconocido devuelven -1.

Indica la complejidad en tiempo y memoria.

[OUTPUT]
Entrega únicamente el código C y el análisis de complejidad.
No incluyas explicación en prosa fuera de los comentarios del código.
```

---

## 3. Código Devuelto por el Modelo
```c
#include "api_sesion19.h"
#include <string.h>

static int ancho(const char *tipo) {
    if (tipo == NULL) return -1;
    if (strcmp(tipo, "char") == 0)    return 1;
    if (strcmp(tipo, "short") == 0)   return 2;
    if (strcmp(tipo, "int") == 0)     return 4;
    if (strcmp(tipo, "float") == 0)   return 4;
    if (strcmp(tipo, "double") == 0)  return 8;
    if (strcmp(tipo, "puntero") == 0) return 8;
    return -1;
}

static int alinear(int cursor, int a) {
    return (cursor + a - 1) / a * a;
}

int offset_campo(const Campo *campos, int total, const char *nombre) {
    if (campos == NULL || nombre == NULL) return -1;
    int cursor = 0;
    for (int i = 0; i < total; i++) {
        int w = ancho(campos[i].tipo);
        if (w < 0) return -1;
        cursor = alinear(cursor, w);
        if (strcmp(campos[i].nombre, nombre) == 0) return cursor;
        cursor += w;
    }
    return -1;
}

int tamano_struct(const Campo *campos, int total) {
    if (campos == NULL) return total == 0 ? 0 : -1;
    int cursor = 0, mayor = 1;
    for (int i = 0; i < total; i++) {
        int w = ancho(campos[i].tipo);
        if (w < 0) return -1;
        cursor = alinear(cursor, w);
        cursor += w;
        if (w > mayor) mayor = w;
    }
    return alinear(cursor, mayor);
}
```

**Modelo usado:** Claude (Anthropic).

**Complejidad declarada por el modelo:** O(n·L) en tiempo por llamada, con n los campos. En la versión corregida es O(n²) porque reviso nombres repetidos, que para structs de unos cuantos campos no importa. Memoria O(1).

---

## 4. Fase 3: El Auditor (Resultados del Test Harness)
*Pruebas ejecutadas:* Acceso a miembros inexistentes

### Salida del harness

Ejecutado en Google Colab.

Contra la implementación propia de la Fase 1 (`test_artesano`):

```text
=== AUDITORIA SESION 19 - Offsets de miembros y tamano de struct ===
Caso                               esperado:              obtenido:              resultado
[ 1] struct char e int: b               esperado: 4            obtenido: 4            OK
[ 2] struct de tres campos: c           esperado: 8            obtenido: 8            OK
[ 3] tamano total con padding           esperado: 12           obtenido: 12           OK
[ 4] double seguido de char             esperado: 16           obtenido: 16           OK
[ 5] miembro inexistente                esperado: -1           obtenido: -1           OK
[ 6] struct sin campos                  esperado: 0            obtenido: 0            OK

RESULTADO: 6/6 pruebas superadas
TODO EN VERDE. Documenta tu auditoria en auditorias/auditoria_sesion19.md
```

Contra el código devuelto por la IA (`test_ia`):

```text
=== AUDITORIA SESION 19 - Offsets de miembros y tamano de struct ===
Caso                               esperado:              obtenido:              resultado
[ 1] struct char e int: b               esperado: 4            obtenido: 4            OK
[ 2] struct de tres campos: c           esperado: 8            obtenido: 8            OK
[ 3] tamano total con padding           esperado: 12           obtenido: 12           OK
[ 4] double seguido de char             esperado: 16           obtenido: 16           OK
[ 5] miembro inexistente                esperado: -1           obtenido: -1           OK
[ 6] struct sin campos                  esperado: 0            obtenido: 0            OK

RESULTADO: 6/6 pruebas superadas
TODO EN VERDE. Documenta tu auditoria en auditorias/auditoria_sesion19.md
```

* [x] Caso base y límites superados sin *Crash*.
* [x] No existen accesos fuera de límites ni *Memory Leaks* (en la versión corregida; ver hallazgos).
* [x] La complejidad teórica coincide con la implementada.

**Entorno:** GCC 13.3.0 (Ubuntu 24.04), compilado con `-std=c11 -Wall -Wextra -Werror`. Para los casos extra usé además `-fsanitize=address,undefined`.

*Nota de transparencia:* para buscar casos borde fuera del harness me apoyé en Claude para escribir los programas de prueba extra; cada resultado de abajo lo corrí y lo revisé.

La IA alinea cada campo a su ancho y redondea el tamaño final al campo más grande; los seis casos pasan, incluido `double` seguido de `char` = 16. Lo que encontré es que las dos funciones no se ponen de acuerdo cuando el struct está mal formado.

### Hallazgo 1: `offset_campo` responde aunque el struct sea inválido

Si hay un tipo desconocido *después* del campo que buscas, `offset_campo` ya regresó antes de llegar a él, pero `tamano_struct` sí lo ve y devuelve -1. El mismo struct tiene offset y no tiene tamaño:

```text
offset de b con un tipo desconocido despues -> 4
tamano del mismo struct -> -1
```

### Hallazgo 2: Campos repetidos se aceptan

Un struct con dos campos `x` no es válido en C, y la IA devuelve el primero sin quejarse:

```text
campo repetido x: offset -> 0, tamano -> 16
```

### Hallazgo 3: `total` negativo cuenta como struct vacío

`tamano_struct(campos, -2)` devuelve 0 porque el `for` simplemente no entra:

```text
tamano con total negativo -> 0
```

### Resultados tras endurecer la implementación

Mismas pruebas extra contra la versión corregida:

**`offset_campo` responde aunque el struct sea inválido** — las dos funciones validan el struct completo antes de responder.

```text
offset de b con un tipo desconocido despues -> -1
tamano del mismo struct -> -1
```

**Campos repetidos se aceptan** — la validación rechaza nombres repetidos.

```text
campo repetido x: offset -> -1, tamano -> -1
```

**`total` negativo cuenta como struct vacío** — `total < 0` devuelve -1; solo `total == 0` es un struct vacío.

```text
tamano con total negativo -> -1
```

Y el harness sigue en verde con la versión corregida:

```text
RESULTADO: 6/6 pruebas superadas
TODO EN VERDE. Documenta tu auditoria en auditorias/auditoria_sesion19.md
```

| Caso borde | IA | Corregido | ¿Lo prueba el harness? |
| :--- | :--- | :--- | :--- |
| Los casos del harness | Pasa | Pasa | Sí |
| **`offset_campo` responde aunque el struct sea inválido** | **FALLA** | Pasa | No |
| **Campos repetidos se aceptan** | **FALLA** | Pasa | No |
| **`total` negativo cuenta como struct vacío** | **FALLA** | Pasa | No |

**Complejidad verificada:** O(n·L) en tiempo por llamada, con n los campos. En la versión corregida es O(n²) porque reviso nombres repetidos, que para structs de unos cuantos campos no importa. Memoria O(1). Los chequeos que agregué son comparaciones constantes y no cambian el orden, salvo donde lo indico.

### Reporte Crítico
*¿Qué caso borde o problema omitió la IA y cómo lo solucionaste?*
> **1. `offset_campo` responde aunque el struct sea inválido.** *Lo que hice:* las dos funciones validan el struct completo antes de responder.
>
> **2. Campos repetidos se aceptan.** *Lo que hice:* la validación rechaza nombres repetidos.
>
> **3. `total` negativo cuenta como struct vacío.** *Lo que hice:* `total < 0` devuelve -1; solo `total == 0` es un struct vacío.
>
> **Lo que me llevo.** Las dos funciones de la IA recorren los campos cada una a su manera, y por eso pueden contradecirse. La lección es que cuando dos funciones describen el mismo objeto deben validar lo mismo; si no, el resto del compilador puede usar un offset de un struct que según otra función ni siquiera existe.

---
**Firma digital:** pablouriellopez68-sketch