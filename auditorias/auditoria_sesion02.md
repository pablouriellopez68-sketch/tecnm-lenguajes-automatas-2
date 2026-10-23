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
> Completé las cuatro funciones a mano, partiendo de 2/6 con los esqueletos vacíos. Los tropiezos fueron de sintaxis de punteros, no de lógica: la idea del recorrido la tenía clara desde el principio, pero traducirla a C me costó varios intentos.
>
> El primero fue confundir el encabezado de la función con el cuerpo. Escribí `NodoToken *crear_nodo(...) = ...` tratando de usar el nombre de la función como si fuera una variable. Me faltaba entender que el encabezado solo declara qué recibe la función, y que los parámetros —como `lexema`— se nombran a secas una vez dentro de las llaves. Una versión de ese mismo error me costó el mensaje `'lexema' undeclared here (not in a function)`: había escrito las líneas fuera del cuerpo de `agregar_token`.
>
> El segundo, y el más repetido, fue olvidar el operador flecha. Escribí `strcpy(lexema, ...)` cuando el destino era `n->lexema`, y `p = siguiente;` cuando debía ser `p = p->siguiente;`. Un campo no existe por sí solo: siempre hay que decir a qué nodo pertenece. También escribí una vez `p>siguiente`, sin el guion, y el compilador lo interpretó como una comparación "mayor que" en lugar de un acceso a campo.
>
> El tercero fue escribir `p->siguiente;` como instrucción suelta dentro del `while`, sin la asignación `p = `. Compila, pero `p` nunca cambia y el bucle no termina nunca. Me hizo ver que en C una expresión válida no es necesariamente una instrucción útil.
>
> También intenté reutilizar `n` como puntero caminante en `agregar_token`. Entender por qué no se puede fue el momento más útil: `n` guarda la dirección del nodo nuevo, y si la sobrescribo para recorrer la lista pierdo la única referencia que tengo a ese nodo, que queda reservado en el heap sin que nadie pueda liberarlo.
>
> La restricción de memoria propiamente dicha apareció en `liberar_lista`. `malloc` entrega memoria sin inicializar, así que `crear_nodo` tiene que dejar los dos campos en estado válido antes de devolver el nodo —copiar el lexema y poner `siguiente` en NULL— porque un `siguiente` con basura haría que el recorrido saltara a una dirección inválida en vez de detenerse. Y al liberar, el orden importa: `free(p)` devuelve la memoria del nodo, de modo que leer `p->siguiente` después de liberarlo sería acceder a memoria ajena. Por eso hay que guardar el enlace en una variable temporal antes del `free`. Ese es el use-after-free que da nombre a la Fase 3 de esta sesión.

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
Estándar C11 compilado con GCC usando -std=c11 -Wall -Wextra -Werror.
Únicamente biblioteca estándar de C; sin dependencias externas.
Destino: tabla de tokens de un analizador léxico, construida en heap.

[ROL]
Actúa como ingeniero de software de sistemas y compiladores.

[ESPECIFICACIÓN]
Implementa exactamente este contrato, sin alterar ninguna firma:

    typedef struct NodoToken {
        char lexema[16];
        struct NodoToken *siguiente;
    } NodoToken;

    NodoToken *crear_nodo(const char *lexema);
    void agregar_token(NodoToken **cabeza, const char *lexema);
    int  contar_tokens(NodoToken *cabeza);
    void liberar_lista(NodoToken *cabeza);

- `crear_nodo` reserva un nodo en heap y copia el lexema recibido.
- `agregar_token` enlaza el nodo nuevo AL FINAL, conservando el orden
  de llegada de los tokens.
- `contar_tokens` recorre la lista y devuelve el número de nodos.
- `liberar_lista` libera todos los nodos sin perder el enlace al
  siguiente.
- Las funciones deben tolerar punteros nulos sin desreferenciarlos.
- Indica la complejidad en tiempo y memoria de cada operación.

[OUTPUT]
Entrega únicamente el código C y el análisis de complejidad.
No incluyas explicación en prosa fuera de los comentarios del código.
```

---

## 3. Código Devuelto por el Modelo

```c
#include "api_sesion02.h"
#include <stdlib.h>
#include <string.h>

NodoToken *crear_nodo(const char *lexema)
{
    NodoToken *n = malloc(sizeof(NodoToken));
    strcpy(n->lexema, lexema);
    n->siguiente = NULL;
    return n;
}

void agregar_token(NodoToken **cabeza, const char *lexema)
{
    NodoToken *n = crear_nodo(lexema);
    if (*cabeza == NULL) {
        *cabeza = n;
        return;
    }
    NodoToken *p = *cabeza;
    while (p->siguiente != NULL) {
        p = p->siguiente;
    }
    p->siguiente = n;
}

int contar_tokens(NodoToken *cabeza)
{
    int c = 0;
    NodoToken *p = cabeza;
    while (p != NULL) {
        c++;
        p = p->siguiente;
    }
    return c;
}

void liberar_lista(NodoToken *cabeza)
{
    NodoToken *p = cabeza;
    while (p != NULL) {
        NodoToken *sig = p->siguiente;
        free(p);
        p = sig;
    }
}
```

**Complejidad declarada por el modelo:** `crear_nodo` O(1); `agregar_token` O(n) por el recorrido hasta el final; `contar_tokens` O(n); `liberar_lista` O(n) con O(1) de memoria auxiliar.

---

## 4. Fase 3: El Auditor (Resultados del Test Harness)
*Pruebas ejecutadas:* Prevención de Use-After-Free y cadenas nulas

* [x] Caso base y límites superados sin *Crash*.
* [x] No existen accesos fuera de límites ni *Memory Leaks*.
* [x] La complejidad teórica coincide con la implementada.

**Entorno:** GCC 13.3.0, compilado con `-std=c11 -Wall -Wextra -Werror -fsanitize=address,undefined -g`.

### Ejecución del harness oficial

### Ejecución del harness oficial

Compilado en Google Colab con GCC 13.3.0 y los flags que indica el cuaderno (`-std=c11 -Wall -Wextra -Werror`).

Contra la implementación propia de la Fase 1:

```
=== AUDITORIA SESION 02 - Lista enlazada de tokens en heap ===
[ 1] crear_nodo("if")->lexema      esperado: if           obtenido: if           OK
[ 2] contar_tokens(3 tokens)       esperado: 3            obtenido: 3            OK
[ 3] orden: segundo token          esperado: dos          obtenido: dos          OK
[ 4] ultimo->siguiente == NULL     esperado: 1            obtenido: 1            OK
[ 5] contar_tokens(NULL)           esperado: 0            obtenido: 0            OK
[ 6] liberar_lista(NULL)           esperado: sin crash    obtenido: sin crash    OK

RESULTADO: 6/6 pruebas superadas
TODO EN VERDE. Documenta tu auditoria en auditorias/auditoria_sesion02.md
```

Contra el código devuelto por la IA:

```
=== AUDITORIA SESION 02 - Lista enlazada de tokens en heap ===
[ 1] crear_nodo("if")->lexema      esperado: if           obtenido: if           OK
[ 2] contar_tokens(3 tokens)       esperado: 3            obtenido: 3            OK
[ 3] orden: segundo token          esperado: dos          obtenido: dos          OK
[ 4] ultimo->siguiente == NULL     esperado: 1            obtenido: 1            OK
[ 5] contar_tokens(NULL)           esperado: 0            obtenido: 0            OK
[ 6] liberar_lista(NULL)           esperado: sin crash    obtenido: sin crash    OK

RESULTADO: 6/6 pruebas superadas
TODO EN VERDE. Documenta tu auditoria en auditorias/auditoria_sesion02.md
```

Ambas implementaciones obtienen el mismo 6/6, pese a que la del modelo contiene un desbordamiento de heap. Los seis casos del harness usan los lexemas `"if"`, `"x"`, `";"`, `"uno"`, `"dos"`, `"a"` y `"b"`: ninguno pasa de tres caracteres, así que el límite de 16 del campo `lexema` nunca llega a tocarse. Ese contraste es el punto de partida de la auditoría que sigue.

Ambas implementaciones obtienen el mismo 6/6, pese a que la del modelo contiene un desbordamiento de heap. Los seis casos del harness usan los lexemas `"if"`, `"x"`, `";"`, `"uno"`, `"dos"`, `"a"` y `"b"`: ninguno pasa de tres caracteres, así que el límite de 16 del campo `lexema` nunca llega a tocarse. Ese contraste es el punto de partida de la auditoría que sigue.

### Hallazgo 1: desbordamiento de heap en `crear_nodo`

El contrato declara `char lexema[16]`, de modo que el nodo completo mide 24 bytes (16 del arreglo más 8 del puntero `siguiente`). Al pasar un identificador de 29 caracteres, AddressSanitizer reporta:

```
==311==ERROR: AddressSanitizer: heap-buffer-overflow on address 0x503000000058
WRITE of size 30 at 0x503000000058 thread T0
0x503000000058 is located 0 bytes after 24-byte region [0x503000000040,0x503000000058)
SUMMARY: AddressSanitizer: heap-buffer-overflow in strcpy
```

### Hallazgo 2: corrupción silenciosa entre 16 y 23 caracteres

Más peligroso que el crash: con un lexema de 19 caracteres la escritura rebasa el arreglo pero cabe dentro de los 24 bytes del nodo, así que ASan no la detecta y el programa no falla. El lexema queda truncado sin aviso y su terminador acaba fuera del arreglo:

```
pedido  : 19 caracteres
guardado: "identificador_la" (16 caracteres)
```

Con solo 15 caracteres útiles, dos identificadores distintos que compartan ese prefijo se convierten en el mismo token y la tabla de símbolos los confunde sin emitir ningún error.

### Resultados tras la corrección

```
[OK  ] crear_nodo copia el lexema
[OK  ] nodo nuevo sin siguiente
[OK  ] contar_tokens = 3
[OK  ] orden preservado: 2o es 'dos'
[OK  ] ultimo->siguiente == NULL
[OK  ] 15 caracteres cabe entero
[OK  ] lexema largo truncado a 15
[OK  ] terminador dentro del arreglo
[OK  ] lexema de 29 no desborda
[OK  ] crear_nodo(NULL) devuelve NULL
[OK  ] agregar_token(NULL,..) no truena
[OK  ] contar_tokens(NULL) = 0
[OK  ] liberar_lista(NULL) no truena

TODAS LAS PRUEBAS PASARON (0 fallos)
```

| Caso borde                       | Resultado | Observación                                        |
|:---------------------------------|:----------|:----------------------------------------------------|
| Lexema de 1 a 15 caracteres      | Pasa      | Copia exacta con su terminador                      |
| **Lexema de 16 a 23 caracteres** | **FALLA** | Truncamiento silencioso; terminador fuera del arreglo |
| **Lexema de 24+ caracteres**     | **FALLA** | `strcpy` desborda el nodo: `WRITE of size 30` en 24 bytes |
| `crear_nodo(NULL)`               | **FALLA** | El modelo desreferencia en `strcpy`                 |
| `agregar_token(NULL, ...)`       | **FALLA** | El modelo desreferencia `*cabeza`                   |
| `contar_tokens(NULL)`            | Pasa      | El bucle no entra                                   |
| `liberar_lista(NULL)`            | Pasa      | El bucle no entra                                   |
| Orden de inserción               | Pasa      | El enlace al final preserva la secuencia léxica     |
| Fugas de memoria                 | Pasa      | Una sola reserva por nodo: el lexema vive dentro    |

**Complejidad verificada:** al ser `lexema` un arreglo embebido y no un puntero, cada token requiere una única reserva y no existen cadenas huérfanas que fugar. `agregar_token` recorre hasta el final en cada llamada: O(n) por inserción y O(n²) para construir una tabla de n tokens.

### Reporte Crítico
*¿Qué caso borde o problema omitió la IA y cómo lo solucionaste?*
> El modelo respetó el contrato al pie de la letra: no alteró ninguna firma, enlazó al final para conservar el orden de los tokens y liberó la lista guardando el enlace al siguiente antes de cada `free`. El harness del cuaderno lo aprueba con 6/6. Y aun así el código es inseguro.
>
> **1. Desbordamiento de heap.** El contrato declara `char lexema[16]`, pero el modelo copió con `strcpy`, que no conoce ese límite. Con un identificador de 29 caracteres AddressSanitizer reporta `WRITE of size 30` sobre una región de 24 bytes: la escritura sale del nodo y pisa memoria ajena. Es una vulnerabilidad de clase CWE-787, y en un analizador léxico basta con alimentarle un nombre de variable largo.
>
> **2. Corrupción silenciosa, que considero el hallazgo más grave.** Entre 16 y 23 caracteres la escritura rebasa el arreglo pero sigue dentro de los 24 bytes del nodo, así que ni ASan la detecta ni el programa falla. Comprobé que `"identificador_largo"` (19 caracteres) se guarda como `"identificador_la"` y su terminador queda fuera del arreglo. Con 15 caracteres útiles, dos identificadores que compartan prefijo colapsan en el mismo token y la tabla de símbolos los confunde sin emitir ningún diagnóstico. Un fallo que no truena es más caro que uno que sí, porque sobrevive a las pruebas y llega a producción.
>
> *Solución a ambos:* medir con `strlen`, truncar a `sizeof(lexema) - 1` cuando el lexema excede el campo, copiar con `memcpy` y escribir el `'\0'` de forma explícita dentro del arreglo. Deduje el tamaño con `sizeof(((NodoToken *)0)->lexema)` en lugar de escribir 16, para que la corrección siga siendo válida si el contrato cambia el tamaño del campo.
>
> **3. Entradas nulas sin proteger.** `crear_nodo(NULL)` pasa el puntero directo a `strcpy` y revienta; `agregar_token(NULL, "x")` desreferencia `*cabeza` sin comprobarlo. El modelo solo manejó los nulos donde el bucle los absorbe por casualidad (`contar_tokens` y `liberar_lista` no entran al `while`), que no es protección sino coincidencia. Añadí guardas explícitas en ambas funciones.
>
> **4. `malloc` sin comprobar.** Si la reserva falla, `crear_nodo` desreferencia NULL de inmediato. La corrección devuelve NULL y `agregar_token` descarta el nodo en lugar de enlazar basura. El contrato declara `void` para `agregar_token`, así que el fallo no puede propagarse al llamador: es una limitación de la API, no del modelo, y la documento como tal.
>
> **Conclusión metodológica.** El hallazgo más incómodo no es ninguno de los cuatro anteriores, sino que **el harness oficial aprueba por igual mi implementación y la del modelo**, aunque una de las dos desborde el heap. Sus seis casos usan lexemas de una a tres letras, así que el límite de 16 nunca se toca. Un 6/6 en verde certifica que el código hace lo que las pruebas miden, no que sea correcto: la prueba que falta es exactamente la que nadie escribió. En la Sesión 01 la lección fue que la IA no audita lo que el prompt calla; aquí la lección es que el harness tampoco audita lo que sus casos callan. Por eso la auditoría real exige herramientas que observen la ejecución y casos construidos a propósito para rebasar los límites declarados en el contrato — y aun así, el caso de 19 caracteres demuestra que ni siquiera ASan encuentra todo: ese lo detecté comparando la cadena guardada contra la pedida.

---

**Firma digital:** pablouriellopez68-sketch
