# Bitácora de Requerimientos y Auditoría IA — Sesión 05
**Materia:** Lenguajes y Autómatas II (SCD-1016)  
**Tema:** Recorridos del AST y Notación Polaca  
**Docente Titular:** MCC. Iván Márquez Larios (`ijmarquezl`)  
**Estudiante:** Pablo Uriel Desiderio López  
**Matrícula:** 24531372  
**Track:** C (el harness evalúa C; la versión en Rust es opcional y no se califica con este harness)  

---

## 1. Fase 1: El Artesano (Dificultad Manual)
*Reto:* Impresión en Postorden / RPN  
*Describe qué problema o restricción de memoria/sintaxis identificaste al codificar a mano:*
> Intenté escribirlo a mano, pero me atoré en la estructura: no sabía cómo separar la recursión de la función `postorden`. Por eso pedí a la IA que me guiara paso a paso y yo fui escribiendo cada línea en la celda. Los errores que cometí en el camino fueron:
> - Puse la función `recorrer` dentro de `postorden`. En C no se pueden meter funciones dentro de otras, y además `recorrer` tiene que ir antes porque `postorden` la llama.
> - Llamé primero a `n->der` y luego a `n->izq`. Así me habría salido `3 8 -` en lugar de `8 3 -`, porque el postorden es izquierdo, derecho y al final la raíz.
> - Escribí `| |` con un espacio en medio en vez de `||`.
> - Puse dos espacios dentro de `strcat(buf, "  ")`. Como el harness compara el texto exacto, todos los casos habrían fallado.
> - El compilador me marcó `-Werror=misleading-indentation` porque el `if` del paso final quedó con sangría, como si fuera parte del `else`. El código estaba bien, pero con `-Werror` cualquier advertencia es error.
>
> La restricción más importante fue que la función no puede imprimir: tiene que escribir en un buffer de tamaño `cap` y siempre dejar un lugar para el `'\0'`. Por eso, antes de agregar cada token, reviso que quepa contando el espacio separador, y el espacio solo lo pongo si ya hay algo escrito, para que no quede uno al inicio ni al final.

---

## 2. Fase 2: El Científico (Ingeniería de Requerimientos C-R-E-O)

### 2.1 Matriz de Especificación Formal
| Elemento C-R-E-O | Definición Técnica |
| :--- | :--- |
| **Contexto (C)** | C99 / GCC o Rust 2021 sin bibliotecas no estándar |
| **Rol (R)** | Ingeniero de software de sistemas y compiladores |
| **Especificación (E)** | Invariantes de recorrido recursivo |
| **Output (O)** | Solo código modular y análisis de complejidad |

### 2.2 Prompt Estructurado
```text
CONTEXTO: Curso de compiladores. Lenguaje C11, compilado con
gcc -std=c11 -Wall -Wextra -Werror (una advertencia cuenta como error).
Solo biblioteca estándar. El archivo se compila junto con un harness que
ya tiene main(), así que tu código NO debe incluir main().
Esta es la cabecera que debes incluir y respetar sin cambios:

typedef enum { NODO_NUM, NODO_BINOP } TipoNodo;
typedef struct ASTNode {
    TipoNodo tipo; int valor; char op;
    struct ASTNode *izq; struct ASTNode *der;
} ASTNode;
void postorden(ASTNode *n, char *buf, int cap);

ROL: Actúa como ingeniero de software de sistemas y compiladores.

ESPECIFICACIÓN: Implementa postorden() para que escriba en buf el
recorrido postorden (izquierdo, derecho, raíz) del AST, es decir, su
notación polaca inversa (RPN).
- NODO_NUM se escribe con su valor entero; NODO_BINOP con su carácter op.
- Tokens separados por exactamente un espacio, sin espacio inicial ni final.
- No uses printf ni imprimas nada: solo escribe en buf.
- Nunca escribas más de cap-1 caracteres más el '\0'; buf siempre debe
  quedar terminado en '\0'.
- Un árbol NULL produce cadena vacía.
- Si buf es NULL o cap <= 0, no hagas nada.
- Ejemplos: 3+(5*2) -> "3 5 2 * +"; (1+2)*(3+4) -> "1 2 + 3 4 + *".

OUTPUT: Solo el archivo .c completo (con #include "api_sesion05.h"),
funciones auxiliares static, y un comentario breve con la complejidad
en tiempo y en espacio de pila.
```

---

## 3. Código Devuelto por el Modelo
```c
#include "api_sesion05.h"
#include <stdio.h>

/* Escribe el subarbol en postorden a partir de buf[pos].
 * Devuelve la nueva posicion del terminador. Invariante: pos <= cap-1
 * y buf[pos] == '\0' al entrar y al salir. Complejidad O(n) en tiempo,
 * O(h) en pila (h = altura del arbol). */
static int emitir(const ASTNode *n, char *buf, int cap, int pos) {
    int escrito;

    if (n == NULL) return pos;
    pos = emitir(n->izq, buf, cap, pos);
    pos = emitir(n->der, buf, cap, pos);
    if (pos >= cap - 1) return pos;                 /* buffer lleno */

    if (n->tipo == NODO_NUM)
        escrito = snprintf(buf + pos, (size_t)(cap - pos), "%s%d",
                           pos > 0 ? " " : "", n->valor);
    else
        escrito = snprintf(buf + pos, (size_t)(cap - pos), "%s%c",
                           pos > 0 ? " " : "", n->op);

    if (escrito < 0) return pos;
    if (escrito >= cap - pos) return cap - 1;       /* se trunco */
    return pos + escrito;
}

void postorden(ASTNode *n, char *buf, int cap) {
    if (buf == NULL || cap <= 0) return;
    buf[0] = '\0';
    emitir(n, buf, cap, 0);
}
```

---

## 4. Fase 3: El Auditor (Resultados del Test Harness)
*Pruebas ejecutadas:* Validación contra desbordamiento de pila

### Salida del harness

**Versión artesano** (`artesano_sesion05_c.c`):
```text
=== AUDITORIA SESION 05 - Recorrido postorden y RPN ===
Caso                               esperado:              obtenido:              resultado
[ 1] hoja 42                            esperado: 42           obtenido: 42           OK
[ 2] 3+(5*2)                            esperado: 3 5 2 * +    obtenido: 3 5 2 * +    OK
[ 3] arbol NULL                         esperado:              obtenido:              OK
[ 4] (1+2)*(3+4)
     esperado: 1 2 + 3 4 + *
     obtenido: 1 2 + 3 4 + *
     -> OK
[ 5] 8-3                                esperado: 8 3 -        obtenido: 8 3 -        OK

RESULTADO: 5/5 pruebas superadas
TODO EN VERDE. Documenta tu auditoria en auditorias/auditoria_sesion05.md
```

**Versión IA** (`ia_sesion05_c.c`):
```text
=== AUDITORIA SESION 05 - Recorrido postorden y RPN ===
Caso                               esperado:              obtenido:              resultado
[ 1] hoja 42                            esperado: 42           obtenido: 42           OK
[ 2] 3+(5*2)                            esperado: 3 5 2 * +    obtenido: 3 5 2 * +    OK
[ 3] arbol NULL                         esperado:              obtenido:              OK
[ 4] (1+2)*(3+4)
     esperado: 1 2 + 3 4 + *
     obtenido: 1 2 + 3 4 + *
     -> OK
[ 5] 8-3                                esperado: 8 3 -        obtenido: 8 3 -        OK

RESULTADO: 5/5 pruebas superadas
TODO EN VERDE. Documenta tu auditoria en auditorias/auditoria_sesion05.md
```

Las dos versiones obtienen 5/5 con `-Wall -Wextra -Werror`.

* [x] Caso base y límites superados sin *Crash*.
* [x] No existen accesos fuera de límites ni *Memory Leaks*.
* [x] La complejidad teórica coincide con la implementada.

**Evidencia adicional:** compilé ambas versiones con `-fsanitize=address,undefined` y una prueba extra (árbol `123 + (-45)` con `cap` = 0, 1, 2, 4, 5, 7, 8, 9, 10 y 64, más `buf = NULL`). AddressSanitizer no reportó escrituras fuera del buffer y no hay memoria dinámica (`malloc`), así que no hay fugas posibles. La complejidad es O(n) en tiempo (cada nodo se visita una vez) y O(h) en pila por la recursión, donde h es la altura del árbol.

### Reporte Crítico
*¿Qué caso borde o problema omitió la IA y cómo lo solucionaste?*
> El harness solo usa buffers de 16 a 64 bytes, donde todo cabe, así que las dos versiones pasan 5/5. Hice una prueba extra fuera del harness con el árbol `123 + (-45)` y buffers chicos, compilada con `-fsanitize=address,undefined`:
>
> | cap | IA | Mi versión (artesano) |
> | :-: | :--- | :--- |
> | 2 | `"1"` | `"+"` |
> | 5 | `"123 "` | `"123"` |
> | 7 | `"123 -4"` | `"123 +"` |
> | 9 | `"123 -45 "` | `"123 -45"` |
> | 10 | `"123 -45 +"` | `"123 -45 +"` |
>
> **Lo que omitió la IA:** cuando el buffer no alcanza, `snprintf` corta a mitad de token. Con `cap = 5` y `cap = 9` deja un **espacio final**, aunque el prompt pedía explícitamente "sin espacio final". Con `cap = 7` escribe `-4` en lugar de `-45`, un número equivocado. No hay desbordamiento (AddressSanitizer no reporta nada), pero la salida es incorrecta.
>
> **Lo que la prueba también encontró en mi versión:** no deja espacios ni tokens cortados, porque solo escribe un token si cabe completo. Pero si un token no cabe, lo salta y sigue con los siguientes, así que con `cap = 7` produce `"123 +"`, una RPN falsa a la que le falta el `-45`.
>
> **Solución:** en cuanto un token no quepa, hay que dejar de escribir por completo (por ejemplo, con una bandera `lleno` que se pasa por la recursión). Así el buffer solo contendría un prefijo correcto del recorrido: `"123"` con `cap = 7`.
>
> **Observación adicional:** con números negativos, el token `-45` se puede confundir con el operador `-`. El harness no prueba negativos.

--- | :--- |
> | 2 | `"1"` | `""` |
> | 5 | `"123 "` | `"123"` |
> | 7 | `"123 -4"` | `"123"` |
> | 9 | `"123 -45 "` | `"123 -45"` |
>
> Con `cap = 5` y `cap = 9` la IA deja un **espacio final**, lo que viola el criterio de aceptación "sin espacio final" que el propio prompt pedía. Con `cap = 7` deja el token `-4` en lugar de `-45`: un número equivocado que una máquina de pila evaluaría sin darse cuenta. No hay desbordamiento de memoria (el `snprintf` sí respeta `cap`), pero la salida es incorrecta.
>
> **Cómo lo detecté:** escribí una prueba extra fuera del harness que llama `postorden` con capacidades pequeñas y revisa `strlen` y el byte siguiente al buffer, compilada con AddressSanitizer.
>
> **Cómo lo solucioné:** en mi versión, antes de escribir calculo cuánto necesita el token *con* su espacio (`strlen(tok) + 1`). Si no cabe completo, no escribo nada y marco una bandera `lleno` que detiene todas las escrituras siguientes. Así el buffer solo contiene tokens completos, sin espacio colgando, y nunca se salta un token para escribir otro posterior (lo que daría una RPN falsa).
>
> **Observación adicional:** con números negativos (`-45`) el token se confunde visualmente con el operador `-`. El harness no tiene negativos, pero un evaluador RPN que lea esta salida tendría que distinguir "`-` solo" (operador) de "`-` seguido de dígitos" (literal).

---
**Firma digital:** pablouriellopez68-sketch
