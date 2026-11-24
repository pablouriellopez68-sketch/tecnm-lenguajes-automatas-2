# Bitácora de Requerimientos y Auditoría IA — Sesión 14
**Materia:** Lenguajes y Autómatas II (SCD-1016)  
**Tema:** Declaración vs Uso de Variables  
**Docente Titular:** MCC. Iván Márquez Larios (`ijmarquezl`)  
**Estudiante:** Pablo Uriel Desiderio López  
**Matrícula:** 24531372  
**Track:** C (el harness evalúa C; la versión en Rust es opcional y no se califica con este harness)  

---

## 1. Fase 1: El Artesano (Dificultad Manual)
*Reto:* Detección de variables no declaradas  
*Describe qué problema o restricción de memoria/sintaxis identificaste al codificar a mano:*
> ✏️ **PENDIENTE (Pablo):** esta fase es sin IA. Escribe aquí, con tus palabras, en qué te atoraste al programar `artesano_sesion14_c.c` a mano (errores de compilación, punteros, casos que no se te ocurrieron, etc.).

---

## 2. Fase 2: El Científico (Ingeniería de Requerimientos C-R-E-O)

### 2.1 Matriz de Especificación Formal
| Elemento C-R-E-O | Definición Técnica |
| :--- | :--- |
| **Contexto (C)** | C99 / GCC o Rust 2021 sin bibliotecas no estándar |
| **Rol (R)** | Ingeniero de software de sistemas y compiladores |
| **Especificación (E)** | Invariante de definición previa |
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
Implementa exactamente estas declaraciones de api_sesion14.h, sin alterarlas:

#define TABLA_MAX 16
typedef struct Tabla {
    const char *nombres[TABLA_MAX];
    int total;
} Tabla;

int declarar(Tabla *tabla, const char *nombre);
int usar(const Tabla *tabla, const char *nombre);

Reglas del contrato:
- declarar() devuelve 1 si insertó, 0 si el nombre ya estaba y -1 si recibe NULL o un nombre vacío.
- usar() devuelve 1 solo si el nombre ya fue declarado; en cualquier otro caso devuelve 0.
- La comparación es exacta: "x" y "xy" son nombres distintos.

Indica la complejidad en tiempo y memoria.

[OUTPUT]
Entrega únicamente el código C y el análisis de complejidad.
No incluyas explicación en prosa fuera de los comentarios del código.
```

---

## 3. Código Devuelto por el Modelo
```c
#include "api_sesion14.h"
#include <string.h>

int declarar(Tabla *tabla, const char *nombre) {
    if (tabla == NULL || nombre == NULL || nombre[0] == '\0') return -1;
    for (int i = 0; i < tabla->total; i++)
        if (strcmp(tabla->nombres[i], nombre) == 0) return 0;
    tabla->nombres[tabla->total++] = nombre;
    return 1;
}

int usar(const Tabla *tabla, const char *nombre) {
    if (tabla == NULL || nombre == NULL) return 0;
    for (int i = 0; i < tabla->total; i++)
        if (strcmp(tabla->nombres[i], nombre) == 0) return 1;
    return 0;
}
```

**Modelo usado:** Claude (Anthropic).

**Complejidad declarada por el modelo:** O(n) en tiempo para `declarar` y `usar`, con n los nombres en la tabla (máximo 16), y O(1) en memoria.

---

## 4. Fase 3: El Auditor (Resultados del Test Harness)
*Pruebas ejecutadas:* Uso antes de declarar y redeclaración

### Salida del harness

Contra la implementación propia de la Fase 1:

```text
✏️ PENDIENTE (Pablo): pega aquí la salida de la celda
!gcc ... harness_sesion14_c.c artesano_sesion14_c.c -o test_artesano && ./test_artesano
```

Contra el código devuelto por la IA:

```text
=== AUDITORIA SESION 14 - Declaracion previa y redeclaracion ===
Caso                               esperado:              obtenido:              resultado
[ 1] usar sin declarar                  esperado: 0            obtenido: 0            OK
[ 2] usar despues de declarar           esperado: 1            obtenido: 1            OK
[ 3] redeclarar el mismo nombre         esperado: 0            obtenido: 0            OK
[ 4] total tras redeclarar              esperado: 1            obtenido: 1            OK
[ 5] prefijo no es el mismo nombre      esperado: 0            obtenido: 0            OK
[ 6] declarar nombre vacio              esperado: -1           obtenido: -1           OK

RESULTADO: 6/6 pruebas superadas
TODO EN VERDE. Documenta tu auditoria en auditorias/auditoria_sesion14.md
```

* [x] Caso base y límites superados sin *Crash*.
* [x] No existen accesos fuera de límites ni *Memory Leaks* (en la versión corregida; ver hallazgos).
* [x] La complejidad teórica coincide con la implementada.

**Entorno:** GCC 13.3.0 (Ubuntu 24.04), compilado con `-std=c11 -Wall -Wextra -Werror`. Para los casos extra usé además `-fsanitize=address,undefined`.

*Nota de transparencia:* para buscar casos borde fuera del harness me apoyé en Claude para escribir los programas de prueba extra; cada resultado de abajo lo corrí y lo revisé.

La IA usa `strcmp`, así que `"x"` y `"xy"` son distintos (el caso del prefijo pasa), y rechaza el duplicado sin cambiar el total. Los seis casos en verde. Pero hay un límite que se le olvidó por completo.

### Hallazgo 1: No revisa `TABLA_MAX`: la declaración 17 escribe fuera del arreglo

`declarar` hace `nombres[total++] = nombre` sin preguntar si todavía cabe. Con 16 nombres la tabla está llena y la siguiente declaración escribe en `nombres[16]`, que ya es la memoria de lo que esté después:

```text
ia_sesion14_c.c:13:19: runtime error: index 16 out of bounds for type 'char *[16]'
```

### Resultados tras endurecer la implementación

Mismas pruebas extra contra la versión corregida:

**No revisa `TABLA_MAX`: la declaración 17 escribe fuera del arreglo** — agregué `if (tabla->total >= TABLA_MAX) return -1;` antes de insertar, y en `usar` recorto el ciclo a `TABLA_MAX` por si el total viniera corrupto.

```text
declarar #16 -> 1
declarar #17 -> -1
total=16
```

Y el harness sigue en verde con la versión corregida:

```text
RESULTADO: 6/6 pruebas superadas
TODO EN VERDE. Documenta tu auditoria en auditorias/auditoria_sesion14.md
```

| Caso borde | IA | Corregido | ¿Lo prueba el harness? |
| :--- | :--- | :--- | :--- |
| Los casos del harness | Pasa | Pasa | Sí |
| **No revisa `TABLA_MAX`: la declaración 17 escribe fuera del arreglo** | **CRASH** | Pasa | No |

**Complejidad verificada:** O(n) en tiempo para `declarar` y `usar`, con n los nombres en la tabla (máximo 16), y O(1) en memoria. Los chequeos que agregué son comparaciones constantes y no cambian el orden, salvo donde lo indico.

### Reporte Crítico
*¿Qué caso borde o problema omitió la IA y cómo lo solucionaste?*
> **1. No revisa `TABLA_MAX`: la declaración 17 escribe fuera del arreglo.** *Lo que hice:* agregué `if (tabla->total >= TABLA_MAX) return -1;` antes de insertar, y en `usar` recorto el ciclo a `TABLA_MAX` por si el total viniera corrupto.
>
> **Lo que me llevo.** Es un solo hallazgo pero es el más grave de todo el bloque: un desbordamiento de arreglo clásico. La IA cuidó el nombre vacío, el duplicado y el prefijo porque el enunciado los menciona, y no cuidó el tamaño porque el enunciado no lo menciona, aunque esté en la cabecera (`#define TABLA_MAX 16`). Me llevo que hay que leer la cabecera también como parte del contrato, no solo las viñetas.

---
**Firma digital:** pablouriellopez68-sketch