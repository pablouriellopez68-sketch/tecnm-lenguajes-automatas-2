# Bitácora de Requerimientos y Auditoría IA — Sesión 02
**Materia:** Lenguajes y Autómatas II (SCD-1016)
**Tema:** Gestión en GitHub Web y Nodos Dinámicos
**Docente Titular:** MCC. Iván Márquez Larios (`ijmarquezl`)
**Estudiante:** Pablo Desiderio López
**Matrícula:** 24531272
**Track:** C

---

## 1. Fase 1: El Artesano (Dificultad Manual)
*Reto:* Lista enlazada de tokens en Heap
*Describe qué problema o restricción de memoria/sintaxis identificaste al codificar a mano:*
>

---

## 2. Fase 2: El Científico (Ingeniería de Requerimientos C-R-E-O)

### 2.1 Matriz de Especificación Formal

| Elemento C-R-E-O       | Definición Técnica                                 |
|:-----------------------|:---------------------------------------------------|
| **Contexto (C)**       | C99 / GCC o Rust 2021 sin bibliotecas no estándar  |
| **Rol (R)**            | Ingeniero de software de sistemas y compiladores   |
| **Especificación (E)** | Gestión de memoria dinámica en C99 / Rust          |
| **Output (O)**         | Solo código modular y análisis de complejidad      |

### 2.2 Prompt Estructurado

```
[CONTEXTO]
Estándar C99 compilado con GCC usando -std=c99 -Wall -Wextra -pedantic.
Únicamente biblioteca estándar; sin dependencias externas. Destino:
front-end de un compilador, tabla de tokens construida en memoria dinámica.

[ROL]
Actúa como ingeniero de software de sistemas y compiladores.

[ESPECIFICACIÓN]
Implementa una lista enlazada simple de tokens alojada en el heap:

    typedef struct Token { char *lexema; struct Token *sig; } Token;
    typedef struct { Token *cabeza; size_t n; } Lista;

    void lista_init(Lista *l);
    int  lista_agregar(Lista *l, const char *lexema);
    void lista_liberar(Lista *l);

- `lista_agregar` inserta un token nuevo con el lexema indicado y
  devuelve 0 en éxito.
- `lista_liberar` libera toda la memoria reservada por la lista.
- Debe manejar entradas nulas sin desreferenciar punteros inválidos.
- Complejidad obligatoria: inserción en O(1), liberación en O(n) con
  O(1) de memoria auxiliar.

[OUTPUT]
Entrega únicamente:
1. El código C modular (cabecera + implementación).
2. Un test harness con main() que ejercite la lista.
3. El análisis de complejidad en tiempo y memoria, justificado.
No incluyas explicación en prosa fuera de los comentarios del código.
```

---

## 3. Código Devuelto por el Modelo

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

typedef struct Token {
    char *lexema;
    struct Token *sig;
} Token;

typedef struct {
    Token *cabeza;
} Lista;

void lista_init(Lista *l)
{
    l->cabeza = NULL;
}

int lista_agregar(Lista *l, const char *lexema)
{
    Token *n = malloc(sizeof(Token));
    n->lexema = strdup(lexema);
    n->sig = l->cabeza;
    l->cabeza = n;
    return 0;
}

void lista_liberar(Lista *l)
{
    Token *p = l->cabeza;
    while (p) {
        Token *sig = p->sig;
        free(p);
        p = sig;
    }
}
```

---

## 4. Fase 3: El Auditor (Resultados del Test Harness)
*Pruebas ejecutadas:* Prevención de Use-After-Free y cadenas nulas

* [x] Caso base y límites superados sin *Crash*.
* [x] No existen accesos fuera de límites ni *Memory Leaks*.
* [x] La complejidad teórica coincide con la implementada.

**Entorno:** GCC 13.3.0, compilado con `-std=c99 -Wall -Wextra -pedantic -fsanitize=address,undefined -g`.

### Auditoría del código devuelto por la IA

**A) Bajo el contrato C99 declarado, el programa ni siquiera corre:**

```
AddressSanitizer:DEADLYSIGNAL
==289==ERROR: AddressSanitizer: SEGV on unknown address 0x000000000070
==289==The signal is caused by a READ memory access.
```

Advertencia del compilador que lo explica:

```
warning: implicit declaration of function 'strdup' [-Wimplicit-function-declaration]
warning: assignment to 'char *' from 'int' makes pointer from integer without a cast
```

**B) Compilando con `-std=gnu99`, donde `strdup` sí existe, aparece la fuga:**

```
Direct leak of 6 byte(s) in 1 object(s) allocated from:
Direct leak of 2 byte(s) in 1 object(s) allocated from:
SUMMARY: AddressSanitizer: 8 byte(s) leaked in 2 allocation(s).
```

### Resultados tras la corrección

```
[OK  ] init deja lista vacia
[OK  ] agregar devuelve 0
[OK  ] agregar 2do token
[OK  ] conteo correcto
[OK  ] lexema copiado
[OK  ] copia independiente
[OK  ] lexema NULL rechazado
[OK  ] lista NULL no truena
[OK  ] cadena vacia valida
[OK  ] liberar anula cabeza
[OK  ] doble liberar inocuo
[OK  ] liberar NULL inocuo

TODAS LAS PRUEBAS PASARON (0 fallos)
```

| Caso borde                       | Resultado | Observación                                        |
|:---------------------------------|:----------|:---------------------------------------------------|
| Lista recién inicializada        | Pasa      | `cabeza == NULL`, `n == 0`                          |
| `lexema` NULL                    | Pasa      | Devuelve -1 sin desreferenciar                      |
| Puntero a lista NULL             | Pasa      | Devuelve -1 sin desreferenciar                      |
| Cadena vacía `""`                | Pasa      | Lexema válido, no error                             |
| Independencia de la copia        | Pasa      | Modificar el origen no altera la lista              |
| Doble `lista_liberar`            | Pasa      | `cabeza = NULL` hace inocua la segunda llamada      |
| `lista_liberar(NULL)`            | Pasa      | Guarda de entrada                                   |
| **Fuga del lexema (código IA)**  | **FALLA** | 8 bytes en 2 reservas; ver reporte crítico          |
| **`strdup` bajo C99 (código IA)**| **FALLA** | Puntero truncado a `int` → SEGV                     |

**Complejidad verificada:** `lista_agregar` inserta por la cabeza, sin recorrer: O(1). `lista_liberar` visita cada nodo exactamente una vez con dos punteros de trabajo: O(n) en tiempo, O(1) en memoria auxiliar, sin recursión.


### Reporte Crítico
*¿Qué caso borde o problema omitió la IA y cómo lo solucionaste?*
> El código devuelto compila sin errores aparentes y la lógica de encadenado es correcta, pero falla en los dos puntos donde C exige disciplina de memoria. Ninguno de los dos es visible leyendo el código: aparecen al compilar con el estándar declarado y al ejecutar bajo AddressSanitizer.
>
> **1. `strdup` no pertenece a C99.** Es una función POSIX. El contexto del prompt exigía C99 con `-pedantic`, y bajo ese estándar `strdup` no está declarada: GCC la asume como función que devuelve `int`, y en una plataforma de 64 bits ese `int` **trunca el puntero** devuelto por la reserva. El resultado no es una advertencia cosmética sino un `SEGV on unknown address 0x70` en el primer acceso al lexema. La IA eligió la función más idiomática sin verificar que perteneciera al estándar que yo mismo había fijado en el Contexto.
>
> *Solución:* sustituí `strdup` por una función estática propia que reserva `strlen(s) + 1` bytes y copia con `memcpy`, usando solo `<string.h>` y `<stdlib.h>` de C99. El código vuelve a compilar con `-pedantic` sin una sola advertencia.
>
> **2. Fuga del lexema en la liberación.** `lista_liberar` recorría la lista liberando cada nodo con `free(p)`, pero el nodo contiene un `char *` que apunta a *otra* reserva independiente. Liberar la estructura no libera la cadena. ASan lo confirma: `8 byte(s) leaked in 2 allocation(s)` con solo dos tokens; en una tabla de símbolos real serían miles de fugas proporcionales al tamaño del programa analizado.
>
> *Solución:* liberar primero `p->lexema` y después `p`, guardando `p->sig` en una variable temporal **antes** de cualquier `free` —leerlo después sería un use-after-free—, y dejar `l->cabeza = NULL` al terminar, de modo que una segunda llamada sea inocua en lugar de provocar un doble free.
>
> **Lo que la IA tampoco cubrió, aunque yo tampoco lo pedí:** no comprobaba el valor devuelto por `malloc`, de modo que un fallo de reserva se convertía en desreferencia de NULL; y no distinguía entre lexema NULL (error) y cadena vacía (lexema válido). Ambos casos los añadí al harness y los resolví en la implementación corregida.
>
> **Conclusión metodológica.** El patrón se repite respecto a la Sesión 01: la IA cumple lo que el prompt enumera y no audita lo que el prompt calla. Aquí el detalle revelador es que el propio Contexto decía "C99 sin bibliotecas no estándar" y aun así el modelo usó POSIX — es decir, ni siquiera basta con declarar la restricción, hay que **verificarla compilando**. La lección para la siguiente iteración es que el elemento Especificación debe incluir las invariantes de propiedad de la memoria (quién reserva, quién libera, qué queda tras liberar), porque son justamente las que ningún compilador comprueba por ti.

---

**Firma digital:** pablouriellopez68-sketch
