# Bitácora de Requerimientos y Auditoría IA — Sesión 16
**Materia:** Lenguajes y Autómatas II (SCD-1016)  
**Tema:** Firmas de Función y Parámetros  
**Docente Titular:** MCC. Iván Márquez Larios (`ijmarquezl`)  
**Estudiante:** Pablo Uriel Desiderio López  
**Matrícula:** 24531372  
**Track:** C (el harness evalúa C; la versión en Rust es opcional y no se califica con este harness)  

---

## 1. Fase 1: El Artesano (Dificultad Manual)
*Reto:* Registro de prototipos y aridad  
*Describe qué problema o restricción de memoria/sintaxis identificaste al codificar a mano:*
> ✏️ **PENDIENTE (Pablo):** esta fase es sin IA. Escribe aquí, con tus palabras, en qué te atoraste al programar `artesano_sesion16_c.c` a mano (errores de compilación, punteros, casos que no se te ocurrieron, etc.).

---

## 2. Fase 2: El Científico (Ingeniería de Requerimientos C-R-E-O)

### 2.1 Matriz de Especificación Formal
| Elemento C-R-E-O | Definición Técnica |
| :--- | :--- |
| **Contexto (C)** | C99 / GCC o Rust 2021 sin bibliotecas no estándar |
| **Rol (R)** | Ingeniero de software de sistemas y compiladores |
| **Especificación (E)** | Coincidencia estricta de parámetros |
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
Implementa exactamente estas declaraciones de api_sesion16.h, sin alterarlas:

typedef struct Funcion {
    const char *nombre;
    int aridad;
} Funcion;

int verificar_llamada(const Funcion *catalogo, int total, const char *nombre, int nargs);

Reglas del contrato:
- La búsqueda del nombre es exacta y recorre el catálogo completo.
- Una función registrada con cero parámetros solo acepta cero argumentos.
- Un catálogo NULL, un nombre NULL o una aridad negativa devuelven -1.

Indica la complejidad en tiempo y memoria.

[OUTPUT]
Entrega únicamente el código C y el análisis de complejidad.
No incluyas explicación en prosa fuera de los comentarios del código.
```

---

## 3. Código Devuelto por el Modelo
```c
#include "api_sesion16.h"
#include <string.h>

int verificar_llamada(const Funcion *catalogo, int total, const char *nombre, int nargs) {
    if (catalogo == NULL || nombre == NULL || nargs < 0) return -1;
    for (int i = 0; i < total; i++) {
        if (strcmp(catalogo[i].nombre, nombre) == 0)
            return catalogo[i].aridad == nargs ? 1 : 0;
    }
    return -1;
}
```

**Modelo usado:** Claude (Anthropic).

**Complejidad declarada por el modelo:** O(t·L) en tiempo, con t las entradas del catálogo y L la longitud del nombre, y O(1) en memoria.

---

## 4. Fase 3: El Auditor (Resultados del Test Harness)
*Pruebas ejecutadas:* Llamadas con número incorrecto de args

### Salida del harness

Contra la implementación propia de la Fase 1:

```text
✏️ PENDIENTE (Pablo): pega aquí la salida de la celda
!gcc ... harness_sesion16_c.c artesano_sesion16_c.c -o test_artesano && ./test_artesano
```

Contra el código devuelto por la IA:

```text
=== AUDITORIA SESION 16 - Registro de prototipos y aridad ===
Caso                               esperado:              obtenido:              resultado
[ 1] suma con 3 argumentos              esperado: 1            obtenido: 1            OK
[ 2] suma con 2 argumentos              esperado: 0            obtenido: 0            OK
[ 3] suma con 4 argumentos              esperado: 0            obtenido: 0            OK
[ 4] funcion no registrada              esperado: -1           obtenido: -1           OK
[ 5] main sin argumentos                esperado: 1            obtenido: 1            OK
[ 6] nombre NULL                        esperado: -1           obtenido: -1           OK

RESULTADO: 6/6 pruebas superadas
TODO EN VERDE. Documenta tu auditoria en auditorias/auditoria_sesion16.md
```

* [x] Caso base y límites superados sin *Crash*.
* [x] No existen accesos fuera de límites ni *Memory Leaks* (en la versión corregida; ver hallazgos).
* [x] La complejidad teórica coincide con la implementada.

**Entorno:** GCC 13.3.0 (Ubuntu 24.04), compilado con `-std=c11 -Wall -Wextra -Werror`. Para los casos extra usé además `-fsanitize=address,undefined`.

*Nota de transparencia:* para buscar casos borde fuera del harness me apoyé en Claude para escribir los programas de prueba extra; cada resultado de abajo lo corrí y lo revisé.

La IA busca el nombre y compara la aridad. Los seis casos pasan. Pero el contrato dice algo que la IA no cumplió: "la búsqueda recorre el catálogo completo".

### Hallazgo 1: Se detiene en la primera coincidencia y no ve las demás firmas

La IA hace `return` en cuanto encuentra el nombre. Si el catálogo tiene dos firmas de la misma función (por ejemplo `imprimir` con 1 y con 2 parámetros), una llamada con 2 argumentos se rechaza aunque exista la firma correcta:

```text
imprimir con 2 args (hay firma de 2) -> 0
```

### Hallazgo 2: Una entrada con nombre NULL tumba la búsqueda

`strcmp(catalogo[i].nombre, nombre)` con un nombre NULL en el catálogo es comportamiento indefinido:

```text
ia_sesion16_c.c:12:13: runtime error: null pointer passed as argument 1, which is declared to never be null
```

### Resultados tras endurecer la implementación

Mismas pruebas extra contra la versión corregida:

**Se detiene en la primera coincidencia y no ve las demás firmas** — recorro todo el catálogo; devuelvo 1 si alguna firma coincide, 0 si el nombre existe pero ninguna aridad coincide, y -1 si no existe.

```text
imprimir con 2 args (hay firma de 2) -> 1
```

**Una entrada con nombre NULL tumba la búsqueda** — salto las entradas con nombre NULL y rechazo `total` negativo.

```text
entrada con nombre NULL...
1
```

Y el harness sigue en verde con la versión corregida:

```text
RESULTADO: 6/6 pruebas superadas
TODO EN VERDE. Documenta tu auditoria en auditorias/auditoria_sesion16.md
```

| Caso borde | IA | Corregido | ¿Lo prueba el harness? |
| :--- | :--- | :--- | :--- |
| Los casos del harness | Pasa | Pasa | Sí |
| **Se detiene en la primera coincidencia y no ve las demás firmas** | **FALLA** | Pasa | No |
| **Una entrada con nombre NULL tumba la búsqueda** | **CRASH** | Pasa | No |

**Complejidad verificada:** O(t·L) en tiempo, con t las entradas del catálogo y L la longitud del nombre, y O(1) en memoria. Los chequeos que agregué son comparaciones constantes y no cambian el orden, salvo donde lo indico.

### Reporte Crítico
*¿Qué caso borde o problema omitió la IA y cómo lo solucionaste?*
> **1. Se detiene en la primera coincidencia y no ve las demás firmas.** *Lo que hice:* recorro todo el catálogo; devuelvo 1 si alguna firma coincide, 0 si el nombre existe pero ninguna aridad coincide, y -1 si no existe.
>
> **2. Una entrada con nombre NULL tumba la búsqueda.** *Lo que hice:* salto las entradas con nombre NULL y rechazo `total` negativo.
>
> **Lo que me llevo.** El primer hallazgo es un caso donde la IA pasó todas las pruebas y aun así no cumplió el contrato escrito. El harness no tiene un catálogo con firmas repetidas, así que el `return` temprano nunca se nota. Me sirvió para ver que las pruebas en verde no prueban que se cumplió el enunciado: hay que leer cada viñeta y preguntarse si alguna prueba la cubre.

---
**Firma digital:** pablouriellopez68-sketch