# Bitácora de Requerimientos y Auditoría IA — Sesión 15
**Materia:** Lenguajes y Autómatas II (SCD-1016)  
**Tema:** Bloques Anidados y Ciclo de Vida  
**Docente Titular:** MCC. Iván Márquez Larios (`ijmarquezl`)  
**Estudiante:** Pablo Uriel Desiderio López  
**Matrícula:** 24531372  
**Track:** C (el harness evalúa C; la versión en Rust es opcional y no se califica con este harness)  

---

## 1. Fase 1: El Artesano (Dificultad Manual)
*Reto:* Operaciones Push y Pop de ámbito  
*Describe qué problema o restricción de memoria/sintaxis identificaste al codificar a mano:*
> ✏️ **PENDIENTE (Pablo):** esta fase es sin IA. Escribe aquí, con tus palabras, en qué te atoraste al programar `artesano_sesion15_c.c` a mano (errores de compilación, punteros, casos que no se te ocurrieron, etc.).

---

## 2. Fase 2: El Científico (Ingeniería de Requerimientos C-R-E-O)

### 2.1 Matriz de Especificación Formal
| Elemento C-R-E-O | Definición Técnica |
| :--- | :--- |
| **Contexto (C)** | C99 / GCC o Rust 2021 sin bibliotecas no estándar |
| **Rol (R)** | Ingeniero de software de sistemas y compiladores |
| **Especificación (E)** | Liberación de memoria al cerrar bloque |
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
Implementa exactamente estas declaraciones de api_sesion15.h, sin alterarlas:

typedef struct Ambito {
    const char *nombre;
    struct Ambito *padre;
} Ambito;

Ambito *push_ambito(Ambito *actual, const char *nombre);
Ambito *pop_ambito(Ambito *actual);

Reglas del contrato:
- push_ambito() devuelve NULL si el ámbito actual o el nombre son NULL.
- pop_ambito() devuelve NULL sobre la raíz y no libera memoria que no le pertenece.
- Cada push se empareja con un pop: no se dejan ámbitos colgando en la cadena.

Indica la complejidad en tiempo y memoria.

[OUTPUT]
Entrega únicamente el código C y el análisis de complejidad.
No incluyas explicación en prosa fuera de los comentarios del código.
```

---

## 3. Código Devuelto por el Modelo
```c
#include "api_sesion15.h"
#include <stdlib.h>

Ambito *push_ambito(Ambito *actual, const char *nombre) {
    if (actual == NULL || nombre == NULL) return NULL;
    Ambito *nuevo = malloc(sizeof *nuevo);
    if (nuevo == NULL) return NULL;
    nuevo->nombre = nombre;
    nuevo->padre = actual;
    return nuevo;
}

Ambito *pop_ambito(Ambito *actual) {
    if (actual == NULL || actual->padre == NULL) return NULL;
    Ambito *padre = actual->padre;
    free(actual);
    return padre;
}
```

**Modelo usado:** Claude (Anthropic).

**Complejidad declarada por el modelo:** O(1) en tiempo para `push` y `pop`. Memoria: un `malloc` por cada `push`, que se libera en su `pop`.

---

## 4. Fase 3: El Auditor (Resultados del Test Harness)
*Pruebas ejecutadas:* Aislamiento de variables fuera de bloque

### Salida del harness

Contra la implementación propia de la Fase 1:

```text
✏️ PENDIENTE (Pablo): pega aquí la salida de la celda
!gcc ... harness_sesion15_c.c artesano_sesion15_c.c -o test_artesano && ./test_artesano
```

Contra el código devuelto por la IA:

```text
=== AUDITORIA SESION 15 - Push y pop de ambitos ===
Caso                               esperado:              obtenido:              resultado
[ 1] nombre del ambito nuevo            esperado: if           obtenido: if           OK
[ 2] profundidad con dos push           esperado: 3            obtenido: 3            OK
[ 3] pop devuelve el padre              esperado: 1            obtenido: 1            OK
[ 4] pop sobre la raiz                  esperado: 1            obtenido: 1            OK
[ 5] push con padre NULL                esperado: 1            obtenido: 1            OK
[ 6] profundidad tras pop               esperado: 2            obtenido: 2            OK

RESULTADO: 6/6 pruebas superadas
TODO EN VERDE. Documenta tu auditoria en auditorias/auditoria_sesion15.md
```

* [x] Caso base y límites superados sin *Crash*.
* [x] No existen accesos fuera de límites ni *Memory Leaks* (en la versión corregida; ver hallazgos).
* [x] La complejidad teórica coincide con la implementada.

**Entorno:** GCC 13.3.0 (Ubuntu 24.04), compilado con `-std=c11 -Wall -Wextra -Werror`. Para los casos extra usé además `-fsanitize=address,undefined`.

*Nota de transparencia:* para buscar casos borde fuera del harness me apoyé en Claude para escribir los programas de prueba extra; cada resultado de abajo lo corrí y lo revisé.

La IA hace `malloc` en `push` y `free` en `pop`, y para la raíz (padre NULL) devuelve NULL sin liberar. Los seis casos pasan. Lo que no cubre es cuando le pasan a `pop` algo que no salió de `push`.

### Hallazgo 1: `pop` libera memoria que no es suya

El contrato dice que `pop_ambito` "no libera memoria que no le pertenece". La IA solo protege la raíz. Si le pasas un ámbito local que sí tiene padre pero que no se creó con `push`, le hace `free` a la pila:

```text
==929==ERROR: AddressSanitizer: attempting free on address which was not malloc()-ed
```

### Hallazgo 2: Doble `pop` del mismo ámbito

Después del primer `pop` el ámbito ya está liberado, y el segundo lo lee para sacar el padre:

```text
==933==ERROR: AddressSanitizer: heap-use-after-free on address 0x502000000018
```

### Resultados tras endurecer la implementación

Mismas pruebas extra contra la versión corregida:

**`pop` libera memoria que no es suya** — llevo un registro de los ámbitos que creó `push`; `pop` solo libera los que están en ese registro.

```text
pop_ambito sobre un ambito que NO salio de push...
rc=0
```

**Doble `pop` del mismo ámbito** — el mismo registro resuelve esto: tras el primer `pop` ya no está registrado y el segundo devuelve NULL.

```text
doble pop del mismo ambito...
rc=0
```

Y el harness sigue en verde con la versión corregida:

```text
RESULTADO: 6/6 pruebas superadas
TODO EN VERDE. Documenta tu auditoria en auditorias/auditoria_sesion15.md
```

| Caso borde | IA | Corregido | ¿Lo prueba el harness? |
| :--- | :--- | :--- | :--- |
| Los casos del harness | Pasa | Pasa | Sí |
| **`pop` libera memoria que no es suya** | **CRASH** | Pasa | No |
| **Doble `pop` del mismo ámbito** | **CRASH** | Pasa | No |

**Complejidad verificada:** O(1) en tiempo para `push` y `pop`. Memoria: un `malloc` por cada `push`, que se libera en su `pop`. Los chequeos que agregué son comparaciones constantes y no cambian el orden, salvo donde lo indico.

### Reporte Crítico
*¿Qué caso borde o problema omitió la IA y cómo lo solucionaste?*
> **1. `pop` libera memoria que no es suya.** *Lo que hice:* llevo un registro de los ámbitos que creó `push`; `pop` solo libera los que están en ese registro.
>
> **2. Doble `pop` del mismo ámbito.** *Lo que hice:* el mismo registro resuelve esto: tras el primer `pop` ya no está registrado y el segundo devuelve NULL.
>
> **Lo que me llevo.** Con la firma que da el contrato (`Ambito *`) la función no tiene forma de saber de dónde vino la memoria, y la IA asumió que siempre sería de `push`. Mi corrección usa un registro estático con tope de 256, que funciona pero tiene un costo: no es seguro con hilos y limita cuántos ámbitos puede haber abiertos. Lo dejo anotado porque es un parche; la solución limpia sería un campo en la estructura que diga si el ámbito es dinámico, pero eso cambia la cabecera.

---
**Firma digital:** pablouriellopez68-sketch