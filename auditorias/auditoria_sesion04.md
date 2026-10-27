# Bitácora de Requerimientos y Auditoría IA — Sesión 04
**Materia:** Lenguajes y Autómatas II (SCD-1016)
**Tema:** Anatomía de un Árbol Sintáctico (AST)
**Docente Titular:** MCC. Iván Márquez Larios (`ijmarquezl`)
**Estudiante:** Pablo Uriel Desiderio López
**Matrícula:** 24531372
**Track:** C

---

## 1. Fase 1: El Artesano (Dificultad Manual)
*Reto:* Estructura de nodo binario (ASTNode)
*Describe qué problema o restricción de memoria/sintaxis identificaste al codificar a mano:*
> Las dos primeras funciones salieron rápido porque se parecen mucho a la lista enlazada de la Sesión 02. Pides memoria con `malloc` y llenas los campos, nada más que aquí son cinco en vez de dos. La que me costó fue `evaluar`, que es recursiva.
>
> Lo primero que me pasó fue que ni compilaba. GCC me tiró `implicit declaration of function 'malloc'` y ahí mismo me decía que incluyera `<stdlib.h>`. La celda traía `stdio.h` y `string.h` pero ese no. Con `-Werror` cualquier advertencia es error, así que no avanzaba nada hasta agregarlo.
>
> Lo que más se me fue fue la flecha. Puse `n>op`, `n>izq` y `n>der` sin el guion, las tres seguidas. C lo entiende como "mayor que" y truena. Es la segunda vez que me pasa, en la Sesión 02 hice lo mismo con `p>siguiente`. No se me hace automático todavía.
>
> Tuve otras tonterías: escribí `return return -1;` y también `int = a = evaluar(...)` con un igual de sobra. Cosas de dedo, pero igual te frenan.
>
> El que de verdad me trabó fue el `if`. Escribí `if (n->tipo)NODO_NUM ==`, cerrando el paréntesis antes de poner la comparación. Me quedé viéndolo sin entender qué estaba mal. Lo que no había captado es que todo lo que comparas va adentro del paréntesis: `if`, abres, la comparación completa con su `==`, cierras, y ya después la llave.
>
> La recursión pensé que iba a estar peor. Al final se entiende: para sacar `3 + 5*2` necesitas lo de la izquierda y lo de la derecha, y como la derecha es otro árbol, haces lo mismo otra vez con esa parte. Lo que evita que se cicle son los dos `if` del principio, porque si el nodo es `NULL` o es un número la función contesta directo y ya no se vuelve a llamar.
>
> Lo de la memoria lo noté viendo la cabecera: hay tres funciones que reservan nodos y ninguna que los libere. Todo lo que construyes se queda ahí hasta que acaba el programa.

---

## 2. Fase 2: El Científico (Ingeniería de Requerimientos C-R-E-O)

### 2.1 Matriz de Especificación Formal
| Elemento C-R-E-O | Definición Técnica |
| :--- | :--- |
| **Contexto (C)** | C99 / GCC o Rust 2021 sin bibliotecas no estándar |
| **Rol (R)** | Ingeniero de software de sistemas y compiladores |
| **Especificación (E)** | Gramática E -> E + T \| T con tipos de nodo |
| **Output (O)** | Solo código modular y análisis de complejidad |

### 2.2 Prompt Estructurado
```text
[CONTEXTO]
Estándar C11 compilado con GCC usando -std=c11 -Wall -Wextra -Werror.
Cualquier advertencia cuenta como error. Solo biblioteca estándar.

[ROL]
Actúa como ingeniero de software de sistemas y compiladores.

[ESPECIFICACIÓN]
Implementa exactamente este contrato, sin alterar ninguna firma:

    typedef enum { NODO_NUM, NODO_BINOP } TipoNodo;

    typedef struct ASTNode {
        TipoNodo tipo;
        int valor;
        char op;
        struct ASTNode *izq;
        struct ASTNode *der;
    } ASTNode;

    ASTNode *crear_numero(int v);
    ASTNode *crear_binario(char op, ASTNode *izq, ASTNode *der);
    int evaluar(ASTNode *n);

- `crear_numero` reserva una hoja con el valor indicado.
- `crear_binario` reserva un nodo con el operador y enlaza los hijos.
- `evaluar` recorre el árbol y devuelve el resultado de la expresión.
  Debe soportar los operadores +, - y *.
- `evaluar(NULL)` devuelve -1 sin desreferenciar.
- Indica la complejidad en tiempo y memoria.

[OUTPUT]
Entrega únicamente el código C y el análisis de complejidad.
No incluyas explicación en prosa fuera de los comentarios del código.
```

---

## 3. Código Devuelto por el Modelo
```c
#include "api_sesion04.h"
#include <stdlib.h>

ASTNode *crear_numero(int v) {
    ASTNode *n = malloc(sizeof(ASTNode));
    if (n == NULL) return NULL;
    n->tipo = NODO_NUM;
    n->valor = v;
    n->op = 0;
    n->izq = NULL;
    n->der = NULL;
    return n;
}

ASTNode *crear_binario(char op, ASTNode *izq, ASTNode *der) {
    ASTNode *n = malloc(sizeof(ASTNode));
    if (n == NULL) return NULL;
    n->tipo = NODO_BINOP;
    n->valor = 0;
    n->op = op;
    n->izq = izq;
    n->der = der;
    return n;
}

int evaluar(ASTNode *n) {
    if (n == NULL) return -1;
    if (n->tipo == NODO_NUM) return n->valor;
    int a = evaluar(n->izq);
    int b = evaluar(n->der);
    switch (n->op) {
        case '+': return a + b;
        case '-': return a - b;
        case '*': return a * b;
    }
    return -1;
}
```

**Complejidad declarada por el modelo:** construir cada nodo es O(1); `evaluar` visita cada nodo exactamente una vez, por lo que es O(n) en tiempo sobre el número de nodos. La memoria auxiliar es O(h), donde h es la altura del árbol, por el uso de la pila de llamadas recursivas.

---

## 4. Fase 3: El Auditor (Resultados del Test Harness)
*Pruebas ejecutadas:* Árbol vacío y evaluación de nodo raíz

* [x] Caso base y límites superados sin *Crash*.
* [x] No existen accesos fuera de límites ni *Memory Leaks*.
* [x] La complejidad teórica coincide con la implementada.

**Entorno:** GCC 13.3.0 en Google Colab, compilado con `-std=c11 -Wall -Wextra -Werror`, y con `-fsanitize=address,undefined` para la auditoría adicional.

### Ejecución del harness oficial

Contra la implementación propia de la Fase 1:

```
=== AUDITORIA SESION 04 - Nodo binario de AST ===
[ 1] evaluar(nodo 42)               esperado: 42       obtenido: 42       OK
[ 2] evaluar(3+5*2)                 esperado: 13       obtenido: 13       OK
[ 3] hijo izquierdo de (* 5 2)      esperado: 5        obtenido: 5        OK
[ 4] operador guardado              esperado: *        obtenido: *        OK
[ 5] evaluar(NULL)                  esperado: -1       obtenido: -1       OK
[ 6] evaluar(10-4)                  esperado: 6        obtenido: 6        OK

RESULTADO: 6/6 pruebas superadas
```

Contra el código devuelto por la IA:

```
=== AUDITORIA SESION 04 - Nodo binario de AST ===
[ 1] evaluar(nodo 42)               esperado: 42       obtenido: 42       OK
[ 2] evaluar(3+5*2)                 esperado: 13       obtenido: 13       OK
[ 3] hijo izquierdo de (* 5 2)      esperado: 5        obtenido: 5        OK
[ 4] operador guardado              esperado: *        obtenido: *        OK
[ 5] evaluar(NULL)                  esperado: -1       obtenido: -1       OK
[ 6] evaluar(10-4)                  esperado: 6        obtenido: 6        OK

RESULTADO: 6/6 pruebas superadas
```

Las dos pasan igual. Pero viendo los seis casos, todos usan árboles bien armados, con los tres operadores de siempre y números chicos. Ninguno se acerca a los límites. Así que me puse a construir casos que sí los tocaran.

### Hallazgo 1: `-1` significa dos cosas distintas

El contrato dice que `evaluar(NULL)` devuelva `-1`. El problema es que `-1` también es un resultado normal: es lo que vale `0 - 1`. Y no hay forma de distinguirlos:

```
A) -1 como valor legitimo vs codigo de error
   evaluar(0-1)   = -1
   evaluar(NULL)  = -1   <- el mismo valor
```

Quien llama a la función recibe `-1` y no sabe si la cuenta dio menos uno o si el árbol venía vacío. Esto no es culpa del modelo: está en el contrato que yo le pasé, que usa como señal de error un número que también puede ser una respuesta válida.

### Hallazgo 2: un árbol roto devuelve un número cualquiera

Si un nodo binario tiene un hijo en `NULL` —cosa que puede pasar si la expresión del archivo fuente está incompleta— la recursión evalúa ese hijo, le sale `-1`, y lo usa como si fuera un número más:

```
B) arbol malformado: binario con hijo NULL
   evaluar(NULL + 5) = 4   (esperado: senalar error)
```

Ni truena ni avisa: contesta 4, que es `-1 + 5`. Un error de sintaxis se convierte en un número que parece correcto, y esa es la peor manera de fallar porque nadie se da cuenta.

### Hallazgo 3: operador desconocido, otra vez `-1`

El `switch` maneja `+`, `-` y `*`. Con cualquier otro carácter cae al `return -1` del final:

```
C) operador desconocido
   evaluar(10 / 2)   = -1   (no esta en el switch)
```

La intención está bien, pero ese `-1` se confunde con los dos anteriores. Tres cosas distintas —árbol vacío, árbol roto y operador que no existe— acaban dando la misma respuesta.

### Hallazgo 4: el árbol nunca se libera

En la cabecera hay tres funciones que piden memoria y ninguna que la devuelva. Cada árbol que construyes se queda en el heap. ASan sobre la corrida del harness:

```
SUMMARY: AddressSanitizer: 352 byte(s) leaked in 11 allocation(s).
```

Once nodos, uno por cada `crear_numero` o `crear_binario` que hace el harness. En un compilador de verdad, que arma un árbol por cada expresión del archivo, eso se multiplica.

### Hallazgo 5: desbordamiento de entero

La multiplicación no revisa si el resultado cabe en un `int`. UBSan lo marca:

```
runtime error: signed integer overflow: 2000000000 * 3 cannot be represented in type 'int'
   evaluar(2000000000 * 3) = 1705032704
```

Y eso en C es comportamiento indefinido, no solo un número equivocado: el compilador puede asumir que nunca pasa y optimizar según esa suposición, así que el resultado puede cambiar dependiendo de cómo compiles.

### Resultados tras endurecer la implementación

Agregué `evaluar_seguro`, que devuelve el valor por un lado y si fue válido por otro, y `liberar_ast`, que recorre el árbol liberando de abajo hacia arriba:

```
A) evaluar_seguro(0-1)   valor=-1  ok=1  <- -1 valido
   evaluar_seguro(NULL)  valor=0   ok=0  <- error distinguible

B) arbol malformado      valor=0   ok=0  (antes daba 4)
C) operador desconocido  valor=0   ok=0
F) desbordamiento        valor=0   ok=0  (antes daba 1705032704)
```

Con `liberar_ast` ASan ya no reporta ninguna fuga. Y el harness sigue dando 6/6, porque dejé `evaluar` con su firma de siempre, nada más que por dentro llama a la versión segura:

```
RESULTADO: 6/6 pruebas superadas
```
| Caso borde | IA | Endurecido | ¿Lo prueba el harness? |
| :--- | :--- | :--- | :--- |
| Hoja numérica | Pasa | Pasa | Sí |
| Expresión anidada (3+5*2) | Pasa | Pasa | Sí |
| `evaluar(NULL)` | Pasa | Pasa | Sí |
| Los tres operadores | Pasa | Pasa | Sí |
| **`-1` como resultado legítimo** | **FALLA** | Pasa | No |
| **Nodo binario con hijo NULL** | **FALLA** | Pasa | No |
| **Operador no soportado** | Ambiguo | Pasa | No |
| **Desbordamiento de entero** | **FALLA** | Pasa | No |
| **Liberación del árbol** | **FALLA** | Pasa | No |

**Complejidad verificada:** `crear_numero` y `crear_binario` son O(1). `evaluar` visita cada nodo una vez: O(n) en tiempo, y O(h) en memoria auxiliar por la pila de recursión, donde h es la altura. Para un árbol degenerado —una cadena de operaciones sin anidamiento equilibrado— h se acerca a n, de modo que el consumo de pila crece linealmente con el tamaño de la expresión. `liberar_ast` tiene el mismo perfil. Las comprobaciones añadidas son todas de tiempo constante y no alteran el orden.

### Reporte Crítico
*¿Qué caso borde o problema omitió la IA y cómo lo solucionaste?*
> El modelo pasó los seis casos, igual que mi versión. El código está bien para árboles normales y números chicos, que es justo lo que prueba el harness. Los cinco problemas los encontré armando entradas que las pruebas no contemplan.
>
> **1. Lo más raro no fue un error del modelo sino del contrato.** Pedir que `evaluar(NULL)` devuelva `-1` significa usar como señal de error un número que también es una respuesta válida. Lo comprobé: `evaluar(0-1)` y `evaluar(NULL)` dan lo mismo. El modelo hizo exactamente lo que le pedí, el problema venía desde el prompt. *Lo arreglé* separando las dos cosas: `evaluar_seguro(n, &ok)` devuelve el valor por el retorno y avisa si fue válido por un parámetro aparte. Dejé `evaluar` con su firma original para no romper el harness.
>
> **2. Un árbol a medias da un número que parece bueno.** Si un nodo binario tiene un hijo en `NULL`, la función evalúa ese hijo, le sale `-1`, y lo suma: `evaluar(NULL + 5)` contesta 4. Y esto no es un caso inventado, porque una expresión incompleta en el código fuente genera justo ese árbol. *Lo arreglé* revisando que un `NODO_BINOP` tenga sus dos hijos antes de seguir, y pasando el error hacia arriba.
>
> **3. Nada libera el árbol.** La cabecera tiene tres funciones que reservan y ninguna que libere. ASan marca 352 bytes en 11 reservas en una sola corrida. No es que el modelo lo haya olvidado —no podía inventar una función que la cabecera no declara— pero sí pudo haberlo mencionado. *Lo arreglé* con `liberar_ast`, que libera primero los hijos y después el padre. Ese orden importa: si liberas el nodo primero ya no puedes llegar a los hijos, y leer `n->izq` después del `free` sería un use-after-free, el mismo error de la Sesión 02.
>
> **4. El desbordamiento pasa sin que nadie lo note.** `2000000000 * 3` da 1705032704. UBSan dice que es desbordamiento de entero con signo, o sea comportamiento indefinido. *Lo arreglé* usando las comprobaciones de desbordamiento del compilador y marcando el resultado como inválido en vez de devolver un número mal.
>
> **5. Algo que noté de la recursión.** La memoria que usa es O(h), la altura del árbol, por las llamadas apiladas. Para expresiones normales no importa, pero una cadena larga de sumas sin paréntesis hace un árbol muy desbalanceado donde la altura casi iguala al número de nodos. Con una expresión suficientemente grande se acabaría la pila. El modelo declaró bien la complejidad pero no dijo nada de eso.
>
> **Lo que me llevo de esta sesión.** Pasó algo parecido a la 03: código que aprueba todo y falla fuera de las pruebas. Pero aquí lo principal estaba en el contrato, no en el código. El modelo cumplió al pie de la letra lo que le pedí, incluido el error de usar `-1` como señal. Entonces auditar no es nada más revisar si el código hace lo que dice la especificación, sino revisar también la especificación: una función que no puede distinguir un error de un resultado bueno está mal diseñada aunque esté perfectamente programada.
---
**Firma digital:** pablouriellopez68-sketch
