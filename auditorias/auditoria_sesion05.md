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
> El recorrido en sí es sencillo (izquierdo → derecho → raíz), pero la dificultad está en que la función **no imprime**: tiene que escribir en un buffer de tamaño fijo `cap`. Eso obligó a resolver tres cosas a mano:
> 1. **Separadores:** el espacio va *antes* de cada token salvo el primero (`strlen(buf) > 0`); si lo pones después queda un espacio final y el harness falla por comparación exacta.
> 2. **Capacidad:** nunca escribir más de `cap-1` caracteres más el `'\0'`. Calculo `libre = cap - 1 - usado` y solo escribo si el token completo (con su espacio) cabe.
> 3. **Convertir el nodo a texto:** los números son `int` y los operadores `char`, así que uso `snprintf` a un buffer temporal `tok[16]` (un `int` de 32 bits ocupa como máximo 11 caracteres con signo).
>
> Además, `postorden` debe dejar `buf[0] = '\0'` antes de recorrer para que un árbol `NULL` produzca cadena vacía.

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

Ambas versiones (artesano e IA) obtienen 5/5 con `-Wall -Wextra -Werror`.

* [x] Caso base y límites superados sin *Crash*.
* [x] No existen accesos fuera de límites ni *Memory Leaks*.
* [x] La complejidad teórica coincide con la implementada.

**Evidencia adicional:** compilé ambas versiones con `-fsanitize=address,undefined` y una prueba extra (árbol `123 + (-45)` con `cap` = 0, 1, 2, 4, 5, 7, 8, 9, 10 y 64, más `buf = NULL`). AddressSanitizer no reportó escrituras fuera del buffer y no hay memoria dinámica (`malloc`), así que no hay fugas posibles. La complejidad es O(n) en tiempo (cada nodo se visita una vez) y O(h) en pila por la recursión, donde h es la altura del árbol.

### Reporte Crítico
*¿Qué caso borde o problema omitió la IA y cómo lo solucionaste?*
> El harness solo usa buffers de 16 a 64 bytes, donde todo cabe, así que el código de la IA pasa 5/5. El problema aparece cuando el buffer es **más chico que la salida**: la IA trunca con `snprintf` a mitad de token. Con el árbol `123 + (-45)`:
>
> | cap | IA | Artesano (corregido) |
> | :-: | :--- | :--- |
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
