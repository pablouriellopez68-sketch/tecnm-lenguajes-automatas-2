# Bitácora de Requerimientos y Auditoría IA — Sesión 01
**Materia:** Lenguajes y Autómatas II (SCD-1016)
**Tema:** Encuadre, Diagnóstico de Punteros y Metodología IA
**Docente Titular:** MCC. Iván Márquez Larios (`ijmarquezl`)
**Estudiante:** Pablo Uriel Desiderio López
**Matrícula:** 24531372
**Track:** C

---

## 1. Fase 1: El Artesano (Dificultad Manual)
*Reto:* Inversión de buffer in-place (char*)
*Describe qué problema o restricción de memoria/sintaxis identificaste al codificar a mano:*
> El algoritmo es simple —dos punteros que se acercan hasta cruzarse— pero la restricción de O(1) memoria prohíbe la salida fácil de reservar un buffer auxiliar y copiar hacia atrás. Hay que modificar el original en su sitio, y ahí aparecen los detalles que solo se ven al ejecutar.
>
> El primero es que el terminador no forma parte del dato. Si uno se posiciona en `s + strlen(s)` y empieza a intercambiar desde ahí, mete el `'\0'` al inicio de la cadena y la trunca a longitud cero. Hay que arrancar en el último carácter real, uno antes.
>
> El segundo es que `strlen` devuelve `size_t`, que no tiene signo. Con la cadena vacía, `strlen(s) - 1` no da `-1` sino el valor máximo del tipo, y el acceso resultante se sale del buffer. Por eso la guarda del caso vacío tiene que ir antes de cualquier resta. Preferí recorrer con puntero (`while (*fin) fin++`) para no depender de esa aritmética.
>
> El tercero es la condición del bucle: debe ser `ini < fin`, no `<=`. Con `<=`, una longitud impar intercambia el carácter central consigo mismo —inofensivo— pero los punteros se cruzan y la siguiente iteración escribe fuera del rango ya procesado.
>
> Y la restricción de memoria propiamente dicha: no todo `char *` es escribible. `char *s = "hola";` apunta a la sección de solo lectura del ejecutable, e invertirlo in-place termina en `Segmentation fault`. La función no puede distinguirlo de un arreglo, así que es una precondición que el llamador debe respetar y que conviene documentar explícitamente.

---

## 2. Fase 2: El Científico (Ingeniería de Requerimientos C-R-E-O)

### 2.1 Matriz de Especificación Formal

| Elemento C-R-E-O       | Definición Técnica                                |
|:-----------------------|:---------------------------------------------------|
| **Contexto (C)**       | C99 / GCC o Rust 2021 sin bibliotecas no estándar |
| **Rol (R)**            | Ingeniero de software de sistemas y compiladores  |
| **Especificación (E)** | Punteros, O(n) tiempo, O(1) memoria               |
| **Output (O)**         | Solo código modular y análisis de complejidad     |

### 2.2 Prompt Estructurado

```
[CONTEXTO]
Estándar C11 compilado con GCC usando -std=c11 -Wall -Wextra -Werror.
Únicamente biblioteca estándar de C; sin dependencias externas.

[ROL]
Actúa como ingeniero de software de sistemas y compiladores.

[ESPECIFICACIÓN]
Implementa exactamente esta firma, sin alterarla:

    void invertir(char *s);

- Invierte in-place la cadena terminada en '\0' apuntada por s.
- Complejidad obligatoria: O(n) en tiempo, O(1) en memoria auxiliar.
  Queda prohibido reservar un buffer temporal o llamar a malloc.
- Si s es NULL, no debe desreferenciarlo.
- Debe manejar correctamente la cadena vacía, longitud 1, y
  longitudes pares e impares.
- Indica la complejidad en tiempo y memoria.

[OUTPUT]
Entrega únicamente el código C y el análisis de complejidad.
No incluyas explicación en prosa fuera de los comentarios del código.
```

---

## 3. Código Devuelto por el Modelo

```c
#include "api_sesion01.h"
#include <string.h>

void invertir(char *s)
{
    if (s == NULL) {
        return;
    }
    size_t len = strlen(s);
    if (len == 0) {
        return;
    }
    size_t i = 0;
    size_t j = len - 1;
    while (i < j) {
        char tmp = s[i];
        s[i] = s[j];
        s[j] = tmp;
        i++;
        j--;
    }
}
```

**Complejidad declarada por el modelo:** O(n) en tiempo —una pasada de `strlen` más `n/2` intercambios— y O(1) en memoria auxiliar, con solo dos índices y un `char` temporal en la pila.

---

## 4. Fase 3: El Auditor (Resultados del Test Harness)
*Pruebas ejecutadas:* Valores NULL, buffers vacíos y longitudes impares

* [x] Caso base y límites superados sin *Crash*.
* [x] No existen accesos fuera de límites ni *Memory Leaks*.
* [x] La complejidad teórica coincide con la implementada.

**Entorno:** GCC 13.3.0 en Google Colab, compilado con `-std=c11 -Wall -Wextra -Werror` para el harness oficial, y con `-fsanitize=address,undefined -g` para la auditoría adicional.

### Ejecución del harness oficial

Contra la implementación propia de la Fase 1:

```
=== AUDITORIA SESION 01 - Inversion de buffer in-place ===
[ 1] invertir("HOLA")          esperado: ALOH        obtenido: ALOH        OK
[ 2] invertir("abc")           esperado: cba         obtenido: cba         OK
[ 3] invertir("")              esperado:             obtenido:             OK
[ 4] invertir("AB")            esperado: BA          obtenido: BA          OK
[ 5] invertir("a")             esperado: a           obtenido: a           OK
[ 6] invertir(NULL)            esperado: sin crash   obtenido: sin crash   OK

RESULTADO: 6/6 pruebas superadas
TODO EN VERDE. Documenta tu auditoria en auditorias/auditoria_sesion01.md
```

Contra el código devuelto por la IA:

```
=== AUDITORIA SESION 01 - Inversion de buffer in-place ===
[ 1] invertir("HOLA")          esperado: ALOH        obtenido: ALOH        OK
[ 2] invertir("abc")           esperado: cba         obtenido: cba         OK
[ 3] invertir("")              esperado:             obtenido:             OK
[ 4] invertir("AB")            esperado: BA          obtenido: BA          OK
[ 5] invertir("a")             esperado: a           obtenido: a           OK
[ 6] invertir(NULL)            esperado: sin crash   obtenido: sin crash   OK

RESULTADO: 6/6 pruebas superadas
TODO EN VERDE. Documenta tu auditoria en auditorias/auditoria_sesion01.md
```

Las dos implementaciones aprueban por igual. Los seis casos del harness usan únicamente texto ASCII, de modo que el defecto descrito abajo nunca llega a dispararse.

### Hallazgo: corrupción de UTF-8

La función invierte *bytes*, no *caracteres*. Con cualquier texto fuera de ASCII —es decir, con español— rompe las secuencias multibyte. Comprobado en ejecución:

```
pedido   : "Autómatas" (10 bytes, 9 caracteres)
obtenido : "satam??tuA"
esperado : "satamótuA"
```

La `ó` ocupa dos bytes (`0xC3 0xB3`) y la inversión global los deja en orden invertido (`0xB3 0xC3`), que no es una secuencia UTF-8 válida. Nótese que la cadena mide 10 bytes aunque tenga 9 caracteres: esa diferencia es justamente la que el algoritmo ignora.

### Hallazgo secundario: precondición de escritura no verificable

La función no puede distinguir un arreglo de un literal de cadena: ambos llegan como `char *`. Al pasarle `char *s = "hola";` el programa termina en `Segmentation fault` (código de salida 139), porque los literales viven en una sección de solo lectura del ejecutable. No es corregible dentro de la firma pedida —C no ofrece forma de interrogar en tiempo de ejecución si un puntero es escribible—, así que lo traté como un asunto de contrato y lo documenté como precondición.

### Resultados tras la corrección

```
[OK  ] NULL                     esperado="sin crash"  obtenido="(sin crash)"
[OK  ] cadena vacia             esperado=""  obtenido=""
[OK  ] un caracter              esperado="a"  obtenido="a"
[OK  ] longitud par             esperado="dcba"  obtenido="dcba"
[OK  ] longitud impar           esperado="edcba"  obtenido="edcba"
[OK  ] doble inversion          esperado="x"  obtenido="x"
[OK  ] UTF-8 acentuado          esperado="satamótuA"  obtenido="satamótuA"
[OK  ] UTF-8 extremos           esperado="údnañ"  obtenido="údnañ"
[OK  ] UTF-8 mixto              esperado="6202 oña"  obtenido="6202 oña"

TODAS LAS PRUEBAS PASARON (0 fallos)
```

| Caso borde                 | Resultado | Observación                                      |
|:---------------------------|:----------|:--------------------------------------------------|
| `NULL`                     | Pasa      | No se desreferencia                               |
| Cadena vacía `""`          | Pasa      | La guarda previene el underflow de `size_t`       |
| Longitud 1                 | Pasa      | La condición del bucle es falsa de entrada        |
| Longitud par / impar       | Pasa      | El carácter central permanece fijo                |
| Doble inversión            | Pasa      | Involución: `f(f(x)) = x`                         |
| **UTF-8 multibyte**        | **FALLA** | Rompe la secuencia; ver reporte crítico           |
| **Literal de solo lectura**| **FALLA** | `SIGSEGV`; precondición no verificable            |

AddressSanitizer y UndefinedBehaviorSanitizer no reportaron accesos fuera de límites ni lecturas no inicializadas en ninguna de las pruebas.

### Reporte Crítico
*¿Qué caso borde o problema omitió la IA y cómo lo solucionaste?*
> El modelo cubrió todos los casos que el prompt enumeró explícitamente —NULL, cadena vacía, longitud 1, pares e impares— y el harness lo aprueba con 6/6. Precisamente ahí está la lección de ingeniería de requerimientos: **la IA no auditó el espacio de entradas que yo no especifiqué.**
>
> **El fallo real es la corrupción de UTF-8.** La función invierte bytes, no caracteres, así que rompe cualquier texto en español. Comprobé que `"Autómatas"` —10 bytes para 9 caracteres— se convierte en una cadena con una secuencia inválida, porque los dos bytes de la `ó` quedan en orden invertido. El modelo nunca declaró esta limitación pese a que el contexto pedía código de sistemas, y el harness oficial no la detecta porque sus seis casos son puro ASCII.
>
> *Solución:* una versión que conserva las garantías de complejidad. Invierte globalmente todos los bytes y después recorre el resultado re-invirtiendo en sitio cada secuencia multibyte. Tras la inversión global, los bytes de continuación (patrón `10xxxxxx`) quedan *antes* de su byte líder, de modo que detectarlos con la máscara `0xC0` basta para delimitar y restaurar cada secuencia. Cada byte se toca como máximo dos veces, así que sigue siendo **O(n) en tiempo y O(1) en memoria**.
>
> **El segundo problema no es corregible dentro de la firma pedida.** La función recibe un `char *` y no puede saber si apunta a memoria escribible; con un literal de cadena el programa muere con `SIGSEGV`. Lo traté como contrato: quedó documentado como precondición explícita y las pruebas usan siempre arreglos. Si se admitiera cambiar la firma, `void invertir_n(char *s, size_t len)` resolvería además el caso de buffers binarios sin terminador.
>
> **Conclusión metodológica.** El código generado era correcto *respecto a la especificación que le di*, y aun así inservible para texto en español. La auditoría no consiste en revisar si la IA respetó el prompt —eso casi siempre lo hace— sino en atacar el código con las entradas que el prompt olvidó nombrar. En la siguiente iteración incluiría la codificación esperada y el modelo de propiedad de la memoria dentro del elemento **Especificación (E)** de la matriz C-R-E-O.

---

**Firma digital:** pablouriellopez68-sketch
