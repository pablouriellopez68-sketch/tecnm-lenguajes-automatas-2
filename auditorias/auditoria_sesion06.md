# Bitácora de Requerimientos y Auditoría IA — Sesión 06
**Materia:** Lenguajes y Autómatas II (SCD-1016)
**Tema:** Sistema de Tipos Primitivos
**Docente Titular:** MCC. Iván Márquez Larios (`ijmarquezl`)
**Estudiante:** Pablo Uriel Desiderio López
**Matrícula:** 24531372
**Track:** C

---

## 1. Fase 1: El Artesano (Dificultad Manual)
*Reto:* Chequeo de compatibilidad de tipos
*Describe qué problema o restricción de memoria/sintaxis identificaste al codificar a mano:*
> Esta sesión fue la más corta de todas: dos funciones, sin punteros, sin `malloc` y sin recursión. Aun así me tropecé un par de veces.
>
> `nombre_tipo` salió a la primera. Es un `switch` igual al que hice ayer en `evaluar`, nada más que comparando valores del `enum` en vez de caracteres y devolviendo cadenas. Lo que sí me costó fue pegarlo: dejé abajo los restos del stub —el `(void)t;`, otro `return "desconocido";` y una llave de más— y esas líneas quedaron fuera de la función, así que ni compilaba. Tuve que borrarlas a mano.
>
> En `compatible` escribí `<==` con dos iguales en lugar de `<=`. Son operadores distintos y el compilador no lo deja pasar.
>
> Lo que más vueltas me dio fue dónde poner el `return origen <= destino;`. Lo metí dentro del `if`, después del `return 0;`, y ahí nunca se iba a ejecutar porque el `return 0;` ya había salido de la función. Me costó ver que la llave que cierra el `if` tenía que subir un renglón para que el último `return` quedara afuera. La idea es: si alguno de los dos tipos es `TIPO_ERROR`, sales devolviendo 0; si llegaste más abajo es que ninguno lo era, y entonces comparas.
>
> Sobre la restricción del ejercicio: la comparación `origen <= destino` funciona porque los valores del `enum` están numerados en el mismo orden que la jerarquía de conversión —`char` es 1, `int` es 2 y `float` es 3— así que un origen "más chico" cabe en el destino. Me quedé pensando en que eso amarra el significado del lenguaje al orden en que están escritos los valores en la cabecera, que es algo que nadie declaró en ningún lado.

---

## 2. Fase 2: El Científico (Ingeniería de Requerimientos C-R-E-O)

### 2.1 Matriz de Especificación Formal
| Elemento C-R-E-O | Definición Técnica |
| :--- | :--- |
| **Contexto (C)** | C99 / GCC o Rust 2021 sin bibliotecas no estándar |
| **Rol (R)** | Ingeniero de software de sistemas y compiladores |
| **Especificación (E)** | Matriz de compatibilidad estática |
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

    typedef enum {
        TIPO_ERROR = 0,
        TIPO_CHAR  = 1,
        TIPO_INT   = 2,
        TIPO_FLOAT = 3
    } Tipo;

    int compatible(Tipo destino, Tipo origen);
    const char *nombre_tipo(Tipo t);

- `compatible` devuelve 1 si un valor de tipo `origen` puede asignarse
  a una variable de tipo `destino`, y 0 si no.
- Se permite ensanchar: char -> int -> float.
- No se permite estrechar: float no puede asignarse a int.
- `TIPO_ERROR` no es compatible con nada, en ninguna dirección.
- `nombre_tipo` devuelve el nombre del tipo como cadena: "error",
  "char", "int" o "float".
- Indica la complejidad en tiempo y memoria.

[OUTPUT]
Entrega únicamente el código C y el análisis de complejidad.
No incluyas explicación en prosa fuera de los comentarios del código.
```

---

## 3. Código Devuelto por el Modelo
```c
#include "api_sesion06.h"

int compatible(Tipo destino, Tipo origen) {
    if (destino == TIPO_ERROR || origen == TIPO_ERROR) {
        return 0;
    }
    return origen <= destino;
}

const char *nombre_tipo(Tipo t) {
    switch (t) {
        case TIPO_ERROR: return "error";
        case TIPO_CHAR:  return "char";
        case TIPO_INT:   return "int";
        case TIPO_FLOAT: return "float";
    }
    return "desconocido";
}
```

**Complejidad declarada por el modelo:** las dos funciones son O(1) en tiempo y memoria. No hay bucles ni reservas: `compatible` hace dos comparaciones y `nombre_tipo` se resuelve con un salto de tabla.

---

## 4. Fase 3: El Auditor (Resultados del Test Harness)
*Pruebas ejecutadas:* Rechazo de asignaciones incompatibles

### Salida del harness

Contra la implementación propia de la Fase 1:

```text
=== AUDITORIA SESION 06 - Tabla de compatibilidad de tipos ===
Caso                            esperado:        obtenido:        resultado
[ 1] compatible(INT,INT)        esperado: 1      obtenido: 1      OK
[ 2] compatible(FLOAT,INT)      esperado: 1      obtenido: 1      OK
[ 3] compatible(INT,FLOAT)      esperado: 0      obtenido: 0      OK
[ 4] compatible(INT,ERROR)      esperado: 0      obtenido: 0      OK
[ 5] nombre_tipo(INT)           esperado: int    obtenido: int    OK
[ 6] nombre_tipo(ERROR)         esperado: error  obtenido: error  OK

RESULTADO: 6/6 pruebas superadas
TODO EN VERDE. Documenta tu auditoria en auditorias/auditoria_sesion06.md
```

Contra el código devuelto por la IA:

```text
=== AUDITORIA SESION 06 - Tabla de compatibilidad de tipos ===
Caso                            esperado:        obtenido:        resultado
[ 1] compatible(INT,INT)        esperado: 1      obtenido: 1      OK
[ 2] compatible(FLOAT,INT)      esperado: 1      obtenido: 1      OK
[ 3] compatible(INT,FLOAT)      esperado: 0      obtenido: 0      OK
[ 4] compatible(INT,ERROR)      esperado: 0      obtenido: 0      OK
[ 5] nombre_tipo(INT)           esperado: int    obtenido: int    OK
[ 6] nombre_tipo(ERROR)         esperado: error  obtenido: error  OK

RESULTADO: 6/6 pruebas superadas
TODO EN VERDE. Documenta tu auditoria en auditorias/auditoria_sesion06.md
```

* [x] Caso base y límites superados sin *Crash*.
* [x] No existen accesos fuera de límites ni *Memory Leaks*.
* [x] La complejidad teórica coincide con la implementada.

**Entorno:** GCC 13.3.0 en Google Colab, compilado con `-std=c11 -Wall -Wextra -Werror`.

El código del modelo es corto, compila sin ninguna advertencia y resuelve bien los seis casos. La regla de ensanchamiento está bien puesta y `TIPO_ERROR` queda fuera en las dos direcciones. Pero viendo las pruebas, todas usan valores normales del `enum`, así que me puse a probar con valores que no lo son.

### Hallazgo 1: un valor que no está en el `enum` pasa como tipo válido

Resulta que en C una variable de tipo `enum` no está obligada a contener solo los valores declarados: puede guardar cualquier número de su tipo entero. Eso pasa si viene de una conversión desde `int`, de una tabla de símbolos dañada o de una variable sin inicializar. El modelo no lo revisa:

```text
A) valor fuera del enum (un int cualquiera convertido a Tipo)
   compatible((Tipo)99, TIPO_INT) = 1   (esperado: 0, no es un tipo)
   compatible((Tipo)-5, TIPO_INT) = 1
```

Como `origen <= destino` se cumple con cualquier número grande, el verificador aprueba una asignación hacia un tipo que ni siquiera existe.

### Hallazgo 2: la validación es asimétrica

El mismo valor inválido se trata distinto dependiendo de en qué parámetro vaya:

```text
  compatible((Tipo)99, TIPO_INT) = 1   <- basura como DESTINO: aceptada
  compatible(TIPO_INT, (Tipo)99) = 0   <- basura como ORIGEN: rechazada
```

Y no es que el código distinga los dos casos: es pura aritmética. Como destino, un número grande deja pasar todo; como origen, no deja pasar nada. O sea que cuando acierta, acierta de casualidad.

### Hallazgo 3: las dos funciones no se ponen de acuerdo

`nombre_tipo` sí detecta el valor raro, porque su `switch` solo tiene los cuatro casos:

```text
  nombre_tipo((Tipo)99) = "desconocido"
```

Entonces una función dice que `(Tipo)99` puede recibir un `int` y la otra dice que no sabe ni cómo se llama. Si armas un mensaje de error con las dos te sale algo como *"no se puede asignar int a desconocido"*, después de haber aprobado la asignación.

### Hallazgo 4: la semántica depende de cómo esté numerado el `enum`

Este me pareció el peor, y es el que menos se ve. El `origen <= destino` funciona nada más porque los valores del `enum` están numerados en el mismo orden que la jerarquía. Reordené la cabecera sin tocar una línea del código:

```text
   Con el enum reordenado (FLOAT=1, INT=2, CHAR=3), el MISMO codigo dice:
   compatible(TIPO_FLOAT, TIPO_INT) = 0   (antes era 1)
   compatible(TIPO_CHAR,  TIPO_INT) = 1   (antes era 0)
```

Las reglas se invirtieron solas. Si alguien reordena el `enum` para que se vea más claro, o mete un `TIPO_DOUBLE` en medio, cambia el comportamiento del compilador sin enterarse y sin que falle ninguna prueba.

### Resultados tras endurecer la implementación

Agregué dos funciones auxiliares: `tipo_valido`, que revisa con un `switch` si el valor está entre los cuatro declarados, y `rango`, que define la jerarquía aparte de los números del `enum`.

```text
Asimetria con valores invalidos:
  compatible((Tipo)99, TIPO_INT) = 0
  compatible(TIPO_INT, (Tipo)99) = 0
```

Y reordenando el `enum`, ahora sí aguanta:

```text
--- version de la IA con el enum reordenado ---
  compatible(FLOAT, INT) = 0   (debe seguir siendo 1)
  compatible(CHAR,  INT) = 1   (debe seguir siendo 0)
--- version endurecida con el enum reordenado ---
  compatible(FLOAT, INT) = 1   (debe seguir siendo 1)
  compatible(CHAR,  INT) = 0   (debe seguir siendo 0)
```

El harness sigue dando 6/6:

```text
RESULTADO: 6/6 pruebas superadas
```

| Caso borde | IA | Endurecido | ¿Lo prueba el harness? |
| :--- | :--- | :--- | :--- |
| Mismo tipo (int ← int) | Pasa | Pasa | Sí |
| Ensanchar (float ← int) | Pasa | Pasa | Sí |
| Estrechar (int ← float) | Pasa | Pasa | Sí |
| `TIPO_ERROR` como origen | Pasa | Pasa | Sí |
| Nombres de tipo | Pasa | Pasa | Sí |
| **Valor fuera del `enum` como destino** | **FALLA** | Pasa | No |
| **Coherencia entre las dos funciones** | **FALLA** | Pasa | No |
| **Independencia del orden del `enum`** | **FALLA** | Pasa | No |
| `char ← float` (estrechar fuerte) | Pasa | Pasa | No |

**Complejidad verificada:** las dos siguen en O(1) de tiempo y memoria. Los `switch` que agregué son de cuatro casos fijos, así que el compilador los resuelve con un salto de tabla o una cadena corta de comparaciones; no hay bucles ni nada que dependa del tamaño de la entrada. Lo que declaró el modelo era correcto y la corrección no lo cambia.

### Reporte Crítico
*¿Qué caso borde o problema omitió la IA y cómo lo solucionaste?*
> El modelo entregó doce líneas que pasan los seis casos, compilan con `-Werror` sin advertencias y capturan bien la regla de ensanchamiento. Es el código más corto de todas las sesiones y aun así el que tiene el error más difícil de ver.
>
> **1. Resulta que un `enum` no es un conjunto cerrado.** Esto fue lo que más me sorprendió. Yo pensaba que una variable `Tipo` solo podía valer uno de los cuatro nombres declarados, pero C permite que guarde cualquier número de su tipo entero. Basta una conversión desde `int` o una variable sin inicializar. Probé `compatible((Tipo)99, TIPO_INT)` y devuelve 1, o sea que aprueba una asignación hacia un tipo que no existe. *Lo arreglé* con una función `tipo_valido` que revisa con un `switch` si el valor es uno de los cuatro, antes de comparar nada.
>
> **2. La validación era asimétrica, y cuando acertaba era por casualidad.** El mismo valor inválido se acepta como destino y se rechaza como origen. No porque el código distinga los dos papeles, sino porque `99 <= 99` es cierto y `99 <= 2` no. Que rechace el segundo caso parece que está validando, pero no está validando nada, y eso me parece peor que un error obvio.
>
> **3. Las dos funciones se contradicen entre ellas.** `nombre_tipo` sí cacha el valor raro y contesta `"desconocido"`, mientras `compatible` lo deja pasar. Un compilador hecho con estas dos podría aprobar una asignación y luego, al querer explicar algo, no saber cómo se llama el tipo que acaba de aprobar. Ninguna prueba lo detecta porque ninguna llama a las dos con el mismo valor fuera de rango.
>
> **4. El que más me llamó la atención: el significado dependía de la numeración.** El `origen <= destino` funciona nada más porque los valores del `enum` están numerados en el mismo orden que la jerarquía de conversión. Reordené la cabecera sin tocar el código y las reglas del lenguaje se dieron la vuelta: `float ← int` pasó de permitido a prohibido. No falló nada, no avisó nada. *Lo arreglé* separando las dos cosas con una función `rango` que define la jerarquía por su cuenta, para que reordenar o ampliar el `enum` ya no cambie el comportamiento.
>
> **Lo que me llevo.** En las sesiones pasadas los errores estaban en lo que el código hacía con entradas raras. Aquí estaban en dos cosas que el código daba por hechas sin decirlo: que los valores del `enum` son los únicos posibles, y que el orden en que están numerados significa algo. Las dos son falsas y ninguna está escrita en ningún lado. Entonces auditar también es buscar esos supuestos que nadie declaró. Y lo de amarrar el significado del programa al orden de una lista me quedó como advertencia: no falla hoy, falla el día que a alguien se le ocurra ordenar la lista.

---
**Firma digital:** pablouriellopez68-sketch
