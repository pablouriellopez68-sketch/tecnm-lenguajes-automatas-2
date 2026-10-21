# Bitácora de Requerimientos y Auditoría IA — Sesión 01
**Materia:** Lenguajes y Autómatas II (SCD-1016)  
**Tema:** Encuadre, Diagnóstico de Punteros y Metodología IA  
**Docente Titular:** MCC. Iván Márquez Larios (`ijmarquezl`)  
**Estudiante:**  Pablo Uriel Desiderio López .
**Matrícula:** 24531372 .
**Track:** C

---

## 1. Fase 1: El Artesano (Dificultad Manual)
*Reto:* Inversión de buffer in-place (char*)  
*Describe qué problema o restricción de memoria/sintaxis identificaste al codificar a mano:*
>1.- El terminador no es parte del dato. La primera versión que escribí hacía end = buf + strlen(buf) y empezaba a intercambiar ahí: eso mete el '\0' al inicio de la cadena y la trunca a longitud cero. Hay que posicionarse en end - 1.

>2.- strlen devuelve size_t, que es sin signo. Con la cadena vacía, strlen(buf) - 1 no da -1: da SIZE_MAX, y el acceso resultante se sale del buffer. Por eso la guarda del caso vacío tiene que ir antes de cualquier resta, y preferí recorrer con puntero (while (*end) end++) en vez de aritmética sobre size_t.

>3.- La condición del bucle debe ser start < end, no <=. Con <=, una longitud impar intercambia el carácter central consigo mismo —inofensivo— pero los punteros se cruzan y la siguiente iteración escribe fuera del rango ya procesado.

>4.- No todo char* es escribible. char *s = "hola"; apunta a .rodata; invertirlo in-place es comportamiento indefinido y en mi equipo termina en SIGSEGV. La restricción O(1) de memoria obliga a modificar el original, así que el contrato de la función tiene que exigir explícitamente un arreglo (char s[] = "hola";) o memoria de malloc.

>La restricción de O(1) memoria es la que vuelve interesante el ejercicio: prohíbe la salida fácil de reservar un buffer auxiliar y copiar hacia atrás.


## 2. Fase 2: El Científico (Ingeniería de Requerimientos C-R-E-O)

### 2.1 Matriz de Especificación Formal
| Elemento C-R-E-O | Definición Técnica |
| :--- | :--- |
| **Contexto (C)** | C99 / GCC o Rust 2021 sin bibliotecas no estándar |
| **Rol (R)** | Ingeniero de software de sistemas y compiladores |
| **Especificación (E)** | Punteros, O(n) tiempo, O(1) memoria |
| **Output (O)** | Solo código modular y análisis de complejidad |

### 2.2 Prompt Estructurado
```text

[ROL]
Actúa como ingeniero de software de sistemas y compiladores. Tu criterio
de calidad es la seguridad de memoria y el cumplimiento estricto del
estándar, no la brevedad.

[ESPECIFICACIÓN]
Implementa:  char *reverse_buffer(char *buf);
- Invierte in-place la cadena terminada en '\0' apuntada por buf.
- Recorrido exclusivamente por aritmética de punteros, sin indexación [].
- Complejidad obligatoria: O(n) en tiempo, O(1) en memoria auxiliar.
  Queda prohibido reservar un buffer temporal o llamar a malloc.
- Devuelve el mismo puntero recibido; si buf es NULL, devuelve NULL
  sin desreferenciar.
- Debe manejar correctamente: puntero NULL, cadena vacía, longitud 1,
  longitudes pares e impares, y buffers cuya capacidad excede la
  longitud de la cadena (no debe tocar los bytes posteriores al '\0').
- Declara explícitamente cualquier precondición que el llamador deba
  garantizar.

[OUTPUT]
Entrega únicamente:
1. El código C, modular, con la función de utilidad separada si aplica.
2. Un test harness con main() que ejerza los casos borde listados.
3. El análisis de complejidad en tiempo y memoria, justificado.
No incluyas explicación en prosa fuera de los comentarios del código.
```

---

## 3. Código Devuelto por el Modelo
```c
// #include <stddef.h>

/* Intercambio de un rango cerrado [a, b]. */
static void reverse_range(char *a, char *b)
{
    while (a < b) {
        char tmp = *a;
        *a++ = *b;
        *b-- = tmp;
    }
}

/* Invierte in-place la cadena terminada en '\0' apuntada por buf.
 * Devuelve el mismo puntero (o NULL si la entrada es NULL).
 * PRECONDICIÓN: buf debe apuntar a memoria escribible, nunca a un
 * literal de cadena (char *s = "x" vive en .rodata -> SIGSEGV).
 */
char *reverse_buffer(char *buf)
{
    if (buf == NULL) {
        return NULL;                 /* contrato defensivo: NULL in, NULL out */
    }

    char *end = buf;
    while (*end != '\0') {           /* medición sin size_t: evita el underflow */
        end++;
    }
    if (end == buf) {
        return buf;                  /* cadena vacía: nada que invertir */
    }

    reverse_range(buf, end - 1);     /* end-1 = último carácter real */
    return buf;
}
```

---

## 4. Fase 3: El Auditor (Resultados del Test Harness)
*Pruebas ejecutadas:* Valores NULL, buffers vacios y longitudes impares

* [x] Caso base y límites superados sin *Crash*.
* [x] No existen accesos fuera de límites ni *Memory Leaks*.
* [x] La complejidad teórica coincide con la implementada.

**Entorno:** GCC 13.3.0, compilado con `-std=c99 -Wall -Wextra -pedantic -fsanitize=address,undefined -g`. Cero advertencias.

**Salida del harness:**

```
== V1: reverse_buffer (byte a byte) ==
[OK  ] NULL                   esperado="(null)"  obtenido="(null)"
[OK  ] cadena vacia           esperado=""  obtenido=""
[OK  ] un caracter            esperado="a"  obtenido="a"
[OK  ] longitud par           esperado="dcba"  obtenido="dcba"
[OK  ] longitud impar         esperado="edcba"  obtenido="edcba"
[OK  ] palabra                esperado="sejaugneL"  obtenido="sejaugneL"
[OK  ] doble inversion        esperado="automatas"  obtenido="automatas"
[OK  ] buffer holgado         esperado="aloh"  obtenido="aloh"
[OK  ] bytes de control       esperado="9	x"  obtenido="9	x"
[NOTA] V1 sobre UTF-8     "Autómatas" -> "satam??tuA" (mojibake esperado)

== V2: reverse_utf8 (consciente de multibyte) ==
[OK  ] UTF-8 acentuado        esperado="satamótuA"  obtenido="satamótuA"
[OK  ] UTF-8 extremos         esperado="údnañ"  obtenido="údnañ"
[OK  ] UTF-8 mixto            esperado="6202 oña"  obtenido="6202 oña"
[OK  ] ASCII via V2           esperado="edcba"  obtenido="edcba"
[OK  ] vacia via V2           esperado=""  obtenido=""
[OK  ] NULL via V2            esperado="(null)"  obtenido="(null)"

TODAS LAS PRUEBAS PASARON (0 fallos)

```

| Caso borde                 | Resultado | Observación                                     |
|:---------------------------|:----------|:------------------------------------------------|
| `NULL`                     | Pasa      | No se desreferencia; devuelve `NULL`             |
| Cadena vacía `""`          | Pasa      | La guarda `end == buf` previene el underflow     |
| Longitud 1                 | Pasa      | `start < end` es falso de entrada; no hay swaps  |
| Longitud par / impar       | Pasa      | El carácter central permanece fijo               |
| Buffer mayor que la cadena | Pasa      | Los bytes tras el `'\0'` quedan intactos         |
| Bytes de control           | Pasa      | Opera sobre bytes, no sobre texto imprimible     |
| Doble inversión            | Pasa      | Involución: `f(f(x)) = x`                        |
| **UTF-8 multibyte**        | **FALLA** | Ver reporte crítico                              |

AddressSanitizer y UndefinedBehaviorSanitizer no reportaron accesos fuera de límites, lecturas no inicializadas ni fugas de memoria.

### Reporte Crítico
*¿Qué caso borde o problema omitió la IA y cómo lo solucionaste?*

> La IA cubrió todos los casos que el prompt enumeró explícitamente. Ahí está la lección: **no auditó el espacio de entradas que yo no especifiqué.** Se le escaparon dos problemas, ambos invisibles al leer el código y evidentes al ejecutarlo.
>
> **1. Corrupción de UTF-8.** La función invierte *bytes*, no *caracteres*, así que rompe las secuencias multibyte de cualquier texto en español. Comprobado: `"Autómatas"` (10 bytes, no 9) se convierte en `satam??tuA`, porque la `ó` (`0xC3 0xB3`) queda como `0xB3 0xC3`, una secuencia inválida. El modelo nunca declaró esta limitación pese a que el contexto pedía código portable.
>
> *Solución:* implementé `reverse_utf8`, que invierte globalmente todos los bytes y luego re-invierte en sitio cada secuencia multibyte. Tras la inversión global, los bytes de continuación (`10xxxxxx`) quedan antes de su byte líder, así que detectarlos con la máscara `0xC0` basta para delimitarlos y restaurarlos. Cada byte se toca dos veces como máximo: sigue siendo **O(n) tiempo y O(1) memoria**. Verificado con `"Autómatas"` → `"satamótuA"`, `"ñandú"` → `"údnañ"` y `"año 2026"` → `"6202 oña"`.
>
> **2. Precondición de escritura no verificable.** La función no puede distinguir un arreglo de un literal: ambos son `char*`. Al pasarle `char *s = "hola";` el programa muere con `Segmentation fault` (salida 139), porque los literales viven en `.rodata`. No es corregible dentro de la firma pedida —C no permite interrogar la escribibilidad de un puntero en tiempo de ejecución—, así que lo traté como contrato: quedó documentado como precondición explícita y el harness usa siempre `char x[] = "..."`. Si se admitiera cambiar la firma, `reverse_buffer_n(char *buf, size_t len)` resolvería además los buffers binarios sin terminador.
>
> **Conclusión metodológica.** El código era correcto *respecto a la especificación que le di*, y aun así inservible para texto en español. Auditar no es verificar que la IA obedeció el prompt —eso casi siempre lo hace—, sino atacar el código con las entradas que el prompt olvidó nombrar. En la siguiente iteración incluiría la codificación esperada y el modelo de propiedad de la memoria dentro del elemento **Especificación (E)** de la matriz C-R-E-O.

---
**Firma digital:**  pablouriellopez68-sketch 
