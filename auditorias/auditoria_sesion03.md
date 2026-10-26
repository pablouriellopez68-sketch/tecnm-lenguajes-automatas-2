# Bitácora de Requerimientos y Auditoría IA — Sesión 03
**Materia:** Lenguajes y Autómatas II (SCD-1016)
**Tema:** Google Colab, Mágicos y Flujo de Entrega
**Docente Titular:** MCC. Iván Márquez Larios (`ijmarquezl`)
**Estudiante:** Pablo Uriel Desiderio López
**Matrícula:** 24531372
**Track:** C

---

## 1. Fase 1: El Artesano (Dificultad Manual)
*Reto:* Manipulación de archivos y llamadas OS
*Describe qué problema o restricción de memoria/sintaxis identificaste al codificar a mano:*
>

---

## 2. Fase 2: El Científico (Ingeniería de Requerimientos C-R-E-O)

### 2.1 Matriz de Especificación Formal
| Elemento C-R-E-O | Definición Técnica |
| :--- | :--- |
| **Contexto (C)** | C99 / GCC o Rust 2021 sin bibliotecas no estándar |
| **Rol (R)** | Ingeniero de software de sistemas y compiladores |
| **Especificación (E)** | Salida estándar y códigos de retorno ABI |
| **Output (O)** | Solo código modular y análisis de complejidad |

### 2.2 Prompt Estructurado
```text
[CONTEXTO]
Estándar C11 compilado con GCC usando -std=c11 -Wall -Wextra -Werror.
Cualquier advertencia cuenta como error. Entorno Linux.

[ROL]
Actúa como ingeniero de software de sistemas y compiladores.

[ESPECIFICACIÓN]
Implementa exactamente este contrato, sin alterar ninguna firma:

    int escribir_texto(const char *ruta, const char *texto);
    int contar_lineas(const char *ruta);
    int ejecutar(const char *comando);

- `escribir_texto` crea o sobrescribe el archivo y escribe el texto.
  Devuelve 0 en éxito.
- `contar_lineas` devuelve el número de líneas del archivo.
- `ejecutar` lanza el comando y devuelve su código de salida.
- Un archivo inexistente se reporta devolviendo -1, sin abortar el
  programa.
- Los archivos abiertos deben cerrarse siempre.
- El archivo no debe contener main().

[OUTPUT]
Entrega únicamente el código C y el análisis de complejidad.
No incluyas explicación en prosa fuera de los comentarios del código.
```

---

## 3. Código Devuelto por el Modelo
```c
#include "api_sesion03.h"
#include <stdio.h>
#include <stdlib.h>

int escribir_texto(const char *ruta, const char *texto)
{
    FILE *f = fopen(ruta, "w");
    if (f == NULL) {
        return -1;
    }
    fputs(texto, f);
    fclose(f);
    return 0;
}

int contar_lineas(const char *ruta)
{
    FILE *f = fopen(ruta, "r");
    if (f == NULL) {
        return -1;
    }
    int lineas = 0;
    int c;
    while ((c = fgetc(f)) != EOF) {
        if (c == '\n') {
            lineas++;
        }
    }
    fclose(f);
    return lineas;
}

int ejecutar(const char *comando)
{
    return system(comando);
}
```

**Complejidad declarada por el modelo:** `escribir_texto` O(n) sobre la longitud del texto; `contar_lineas` O(n) sobre el tamaño del archivo, con O(1) de memoria auxiliar al leer carácter por carácter; `ejecutar` depende del comando invocado.

---

## 4. Fase 3: El Auditor (Resultados del Test Harness)
*Pruebas ejecutadas:* Detección de errores de compilación GCC

* [x] Caso base y límites superados sin *Crash*.
* [x] No existen accesos fuera de límites ni *Memory Leaks*.
* [x] La complejidad teórica coincide con la implementada.

**Entorno:** GCC 13.3.0 en Google Colab, compilado con `-std=c11 -Wall -Wextra -Werror`.

### Ejecución del harness sobre el código de la IA

El código del modelo compila sin una sola advertencia, incluso con `-Werror`. Aun así, el harness lo rechaza:

```
=== AUDITORIA SESION 03 - Archivos y llamadas OS ===
[ 1] escribir_texto(datos.txt)          esperado: 0            obtenido: 0            OK
[ 2] contar_lineas(datos.txt)           esperado: 3            obtenido: 3            OK
[ 3] contar_lineas sin salto final      esperado: 2            obtenido: 1            FALLA
[ 4] contar_lineas(inexistente)         esperado: -1           obtenido: -1           OK
[ 5] ejecutar("true")                   esperado: 0            obtenido: 0            OK
[ 6] ejecutar("false")                  esperado: 1            obtenido: 256          FALLA

RESULTADO: 4/6 pruebas superadas
```

Dos fallos, y ninguno de los dos es un error de sintaxis: son suposiciones equivocadas sobre cómo funcionan el formato de archivo de texto y la ABI de procesos.

### Hallazgo 1: contar saltos de línea no es contar líneas

`contar_lineas` incrementa el contador en cada `'\n'`. Con un archivo que termina en salto de línea el resultado coincide por casualidad, pero con `"a\nb"` —dos líneas, un solo salto— devuelve 1.

El convenio de POSIX es que toda línea termina en `'\n'`, pero los archivos reales no siempre lo respetan: un editor mal configurado, una descarga truncada o una cadena generada en memoria producen exactamente este caso. Un contador de líneas que se equivoca en el último renglón es inservible para un analizador léxico, que es a dónde va este código.

### Hallazgo 2: `system()` no devuelve el código de salida

`ejecutar("false")` devolvió **256**, no 1. El valor que entrega `system()` no es el código de salida del proceso, sino un entero empaquetado donde el byte alto guarda ese código y el bajo guarda información sobre señales. Para `false`, que sale con 1, el empaquetado da `1 << 8 = 256`.

Es un error silencioso y peligroso: cualquier comparación del tipo `if (ejecutar(cmd) != 0)` parece funcionar —256 también es distinto de cero— y el programa se comporta bien hasta que alguien necesita el código exacto. El desempaquetado correcto requiere las macros `WIFEXITED` y `WEXITSTATUS`.

### Resultados tras la corrección

```
=== AUDITORIA SESION 03 - Archivos y llamadas OS ===
[ 1] escribir_texto(datos.txt)          esperado: 0            obtenido: 0            OK
[ 2] contar_lineas(datos.txt)           esperado: 3            obtenido: 3            OK
[ 3] contar_lineas sin salto final      esperado: 2            obtenido: 2            OK
[ 4] contar_lineas(inexistente)         esperado: -1           obtenido: -1           OK
[ 5] ejecutar("true")                   esperado: 0            obtenido: 0            OK
[ 6] ejecutar("false")                  esperado: 1            obtenido: 1            OK

RESULTADO: 6/6 pruebas superadas
```

| Caso borde | IA | Corregido | Observación |
| :--- | :--- | :--- | :--- |
| Archivo con salto final | Pasa | Pasa | Coincidencia: un `'\n'` por línea |
| **Archivo sin salto final** | **FALLA** | Pasa | Falta sumar la última línea incompleta |
| Archivo inexistente | Pasa | Pasa | `fopen` devuelve NULL y se reporta -1 |
| Archivo vacío | Pasa | Pasa | Cero líneas, sin sumar de más |
| `ejecutar` con éxito | Pasa | Pasa | El código 0 empaquetado sigue siendo 0 |
| **`ejecutar` con fallo** | **FALLA** | Pasa | 256 en vez de 1: falta `WEXITSTATUS` |
| Rutas NULL | **FALLA** | Pasa | El modelo las pasa directo a `fopen` |
| Error al cerrar el archivo | **FALLA** | Pasa | El modelo ignora el valor devuelto por `fclose` |

**Complejidad verificada:** `escribir_texto` es O(n) sobre la longitud del texto; `contar_lineas` es O(n) sobre el tamaño del archivo con O(1) de memoria auxiliar, al leer carácter por carácter sin cargar el archivo completo; `ejecutar` delega el costo al proceso hijo. La declaración del modelo es correcta.

### Reporte Crítico
*¿Qué caso borde o problema omitió la IA y cómo lo solucionaste?*
> A diferencia de las sesiones anteriores, aquí el harness sí atrapó a la IA: 4/6. El código compila limpio con `-Werror`, así que los dos fallos no son de sintaxis sino de suposiciones equivocadas sobre el entorno.
>
> **1. Confundir saltos de línea con líneas.** El modelo contó caracteres `'\n'`. Eso funciona solo si el archivo termina en salto, que es el convenio de POSIX pero no una garantía. Con `"a\nb"` —dos líneas, un salto— devolvió 1. *Solución:* recordar el último carácter leído y, si el archivo no está vacío y no termina en `'\n'`, sumar una línea más. Es una variable extra, así que la complejidad sigue siendo O(n) en tiempo y O(1) en memoria.
>
> **2. Devolver el valor crudo de `system()`.** `ejecutar("false")` entregó 256 en lugar de 1. El entero que devuelve `system()` no es el código de salida: es un estado empaquetado donde ese código ocupa el byte alto. *Solución:* comprobar con `WIFEXITED` que el proceso terminó normalmente y extraer el código con `WEXITSTATUS`; si terminó por una señal, devolver -1 porque no hay código de salida que reportar. Añadí también la guarda de `system()` devolviendo -1, que ocurre cuando ni siquiera se pudo crear el proceso.
>
> **3. Dos huecos que el harness tampoco prueba.** El modelo pasa `ruta` directo a `fopen` sin comprobar NULL, e ignora el valor devuelto por `fclose`. Esto último importa más de lo que parece: con escritura almacenada en búfer, un disco lleno o un error de E/S no se manifiesta en `fputs` sino al cerrar, de modo que `escribir_texto` podría devolver 0 habiendo dejado el archivo incompleto. Ninguno de los seis casos del harness lo detecta.
>
> **4. El contrato se contradice a sí mismo.** La matriz C-R-E-O exige "sin bibliotecas no estándar", pero `WEXITSTATUS` pertenece a `<sys/wait.h>`, que es POSIX y no forma parte de C99 ni de C11. Cumplir el requisito funcional obliga a romper la restricción declarada. Lo resolví del lado de la funcionalidad y lo documento aquí como deuda del contrato: en C puro no hay forma portable de obtener el código de salida de un proceso hijo, porque el estándar no define procesos.
>
> **Conclusión metodológica.** La lección de esta sesión invierte la de las dos anteriores. En la 01 y la 02 el harness aprobaba código defectuoso, y hubo que salir a buscar el fallo con casos propios y sanitizers. Aquí el harness hizo su trabajo: los dos casos que la IA falló estaban escritos de antemano, precisamente porque quien lo diseñó ya conocía estas dos trampas. Eso define qué es una buena prueba — no la que confirma que el código funciona, sino la que codifica el conocimiento de por dónde suele romperse. Y aun así se le escaparon la ruta NULL y el `fclose` sin comprobar, lo que confirma que ningún harness sustituye a la lectura del código.

---
**Firma digital:** pablouriellopez68-sketch
