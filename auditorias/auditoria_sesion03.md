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
- `ejecutar` lanza el comando y devuelve su código de salida real.
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
#include <sys/wait.h>

int escribir_texto(const char *ruta, const char *texto) {
    if (ruta == NULL || texto == NULL) {
        return -1;
    }

    FILE *archivo = fopen(ruta, "w");
    if (archivo == NULL) {
        return -1;
    }

    if (fputs(texto, archivo) == EOF) {
        fclose(archivo);
        return -1;
    }

    fclose(archivo);
    return 0;
}

int contar_lineas(const char *ruta) {
    if (ruta == NULL) {
        return -1;
    }

    FILE *archivo = fopen(ruta, "r");
    if (archivo == NULL) {
        return -1;
    }

    int lineas = 0;
    int ch;
    int ultimo_ch = EOF;

    while ((ch = fgetc(archivo)) != EOF) {
        if (ch == '\n') {
            lineas++;
        }
        ultimo_ch = ch;
    }

    // Caso borde: el archivo tiene contenido pero no termina en salto de línea
    if (ultimo_ch != EOF && ultimo_ch != '\n') {
        lineas++;
    }

    fclose(archivo);
    return lineas;
}

int ejecutar(const char *comando) {
    if (comando == NULL) {
        return -1;
    }

    int estado = system(comando);
    if (estado == -1) {
        return -1;
    }

    // Retornamos el código de salida real usando WEXITSTATUS si terminó normalmente
    if (WIFEXITED(estado)) {
        return WEXITSTATUS(estado);
    }

    return -1;
}
```

**Complejidad declarada:** `escribir_texto` es O(n) sobre la longitud del texto; `contar_lineas` es O(n) sobre el tamaño del archivo con O(1) de memoria auxiliar, al leer carácter por carácter sin cargarlo completo; `ejecutar` delega el costo al proceso hijo.

---

## 4. Fase 3: El Auditor (Resultados del Test Harness)
*Pruebas ejecutadas:* Detección de errores de compilación GCC

* [x] Caso base y límites superados sin *Crash*.
* [x] No existen accesos fuera de límites ni *Memory Leaks*.
* [x] La complejidad teórica coincide con la implementada.

**Entorno:** GCC 13.3.0 en Google Colab, compilado con `-std=c11 -Wall -Wextra -Werror`.

### Ejecución del harness oficial

El código del modelo compila sin una sola advertencia y supera los seis casos a la primera:

```
=== AUDITORIA SESION 03 - Archivos y codigos de salida ===
[ 1] escribir_texto(...)==0             esperado: 0            obtenido: 0            OK
[ 2] contar_lineas(3 lineas)            esperado: 3            obtenido: 3            OK
[ 3] contar_lineas(sin salto final)     esperado: 2            obtenido: 2            OK
[ 4] contar_lineas(inexistente)         esperado: -1           obtenido: -1           OK
[ 5] ejecutar("true")                   esperado: 0            obtenido: 0            OK
[ 6] ejecutar("false")                  esperado: 1            obtenido: 1            OK

RESULTADO: 6/6 pruebas superadas
TODO EN VERDE. Documenta tu auditoria en auditorias/auditoria_sesion03.md
```

El modelo manejó correctamente las dos trampas previsibles: contó la última línea aunque no termine en `'\n'`, y extrajo el código de salida con `WEXITSTATUS` en lugar de devolver el valor crudo de `system()`. También comprobó los punteros nulos, verificó `fputs` contra `EOF` y cerró el archivo antes de salir por error.

Un 6/6 con el código limpio deja la auditoría en su caso más difícil: hay que buscar lo que las pruebas no miden. Construí tres casos que el harness no contempla.

### Hallazgo 1: `escribir_texto` reporta éxito sin haber escrito nada

`fputs` solo deposita los bytes en el búfer de la biblioteca; el envío real al sistema de archivos ocurre al vaciar ese búfer, normalmente dentro de `fclose`. El modelo comprueba `fputs` pero ignora el valor que devuelve `fclose`, de modo que un error de escritura que aparezca en ese momento pasa desapercibido.

Lo comprobé escribiendo a `/dev/full`, un dispositivo de Linux que acepta la apertura y falla toda escritura, simulando un disco lleno:

```
A) escribir en /dev/full (disco lleno simulado)
   devuelve: 0   (esperado: -1, el texto NO se escribio)
```

La función informa éxito habiendo perdido el contenido completo. En un compilador que emite su salida a disco, esto significa reportar una compilación correcta dejando un binario truncado.

*Solución:* comprobar también `fclose`, porque devuelve distinto de cero cuando el vaciado del búfer falla.

### Hallazgo 2: `contar_lineas` confunde "fin de archivo" con "error de lectura"

`fgetc` devuelve `EOF` en dos situaciones distintas: cuando se acabó el archivo y cuando ocurrió un error de E/S. El bucle del modelo las trata igual, así que un fallo de lectura se reporta como un conteo normal.

Lo comprobé pasando la ruta de un directorio. En Linux `fopen` abre directorios sin error —la apertura es legal— y es `fgetc` quien falla:

```
B) contar_lineas sobre un directorio
   devuelve: 0   (esperado: -1, no es un archivo legible)
```

Devolver 0 es peor que devolver -1: el llamador concluye que el archivo existe y está vacío, cuando en realidad nunca se pudo leer.

*Solución:* tras el bucle, consultar `ferror()` sobre el flujo y devolver -1 si está activa. Es la única forma de distinguir las dos causas de `EOF`.

### Hallazgo 3: la rama de señales es inalcanzable en la práctica

El modelo devuelve -1 cuando `WIFEXITED` es falso, es decir, cuando el proceso murió por una señal. La intención es correcta, pero esa rama casi nunca se ejecuta: `system()` no lanza el comando directamente, sino que invoca `/bin/sh -c`. Si el comando muere por una señal, el que termina es el hijo del shell; el shell mismo sale de forma normal con el código 128+N.

```
C) ejecutar un comando que muere por senal (SIGSEGV)
   devuelve: 139   (shell reporta 139)
```

139 es 128+11, el convenio para SIGSEGV. El resultado es indistinguible de un programa que terminara normalmente devolviendo 139. No es un defecto del código —es una limitación de `system()`— pero conviene documentarlo: quien necesite distinguir una terminación por señal debe usar `fork` y `execvp` con `waitpid`, no `system`.

### Resultados tras endurecer la implementación

```
A) escribir en /dev/full (disco lleno simulado)
   devuelve: -1   (esperado: -1, el texto NO se escribio)

B) contar_lineas sobre un directorio
   devuelve: -1   (esperado: -1, no es un archivo legible)
```

Y el harness oficial sigue aprobando los seis casos:

```
RESULTADO: 6/6 pruebas superadas
```

| Caso borde | IA | Endurecido | ¿Lo prueba el harness? |
| :--- | :--- | :--- | :--- |
| Archivo con salto final | Pasa | Pasa | Sí |
| Archivo sin salto final | Pasa | Pasa | Sí |
| Archivo inexistente | Pasa | Pasa | Sí |
| Rutas NULL | Pasa | Pasa | No |
| `ejecutar` con código 0 y 1 | Pasa | Pasa | Sí |
| **Error al vaciar el búfer** | **FALLA** | Pasa | No |
| **Error de lectura (directorio)** | **FALLA** | Pasa | No |
| Terminación por señal | Limitado | Limitado | No |

**Complejidad verificada:** las correcciones no alteran el orden de ninguna función. `ferror` es una consulta en tiempo constante y comprobar el valor de `fclose` no añade recorridos. `contar_lineas` sigue en O(n) tiempo y O(1) memoria.

### Reporte Crítico
*¿Qué caso borde o problema omitió la IA y cómo lo solucionaste?*
> El modelo pasó los seis casos del harness al primer intento, y no por suerte: manejó explícitamente las dos trampas que esta sesión plantea —la última línea sin salto final y el desempaquetado del código de salida con `WEXITSTATUS`— además de comprobar punteros nulos y verificar `fputs`. Eso obligó a auditar en el escenario más incómodo: código que aprueba todas las pruebas.
>
> **1. Confiar en que escribir es escribir.** `escribir_texto` comprueba `fputs` pero no `fclose`. Como la escritura está almacenada en búfer, un error de E/S no se manifiesta al escribir sino al cerrar, y ahí el valor devuelto se descarta. Lo comprobé escribiendo a `/dev/full`: la función devolvió 0 sin haber guardado un solo byte. *Solución:* comprobar también el valor de `fclose`. Esta es, de los tres hallazgos, la que más me interesó, porque el error no está en lo que el código hace sino en lo que asume sobre cuándo ocurren las cosas.
>
> **2. Dos significados para el mismo valor.** `fgetc` devuelve `EOF` tanto al terminar el archivo como al fallar la lectura, y el bucle no los distingue. Pasándole un directorio —que `fopen` abre sin protestar en Linux— la función devolvió 0 líneas en lugar de -1, afirmando que el archivo está vacío cuando en realidad jamás se pudo leer. *Solución:* consultar `ferror()` después del bucle. El patrón general es que en C el valor de retorno rara vez alcanza para saber qué pasó; hay que preguntar por el estado de error por separado.
>
> **3. Una rama que casi nunca se ejecuta.** El `return -1` para procesos terminados por señal es correcto en intención pero inalcanzable con `system()`, porque entre el comando y nosotros hay un shell que sí termina normalmente y traduce la señal a 128+N. Un comando abortado por SIGSEGV devuelve 139, idéntico a uno que salga voluntariamente con 139. No es un error del modelo sino un límite de la herramienta, y lo documento como tal.
>
> **4. El contrato se contradice a sí mismo.** La matriz C-R-E-O exige "sin bibliotecas no estándar", pero `WEXITSTATUS` pertenece a `<sys/wait.h>`, que es POSIX y no forma parte de C99 ni de C11. Cumplir el requisito funcional obliga a romper la restricción declarada, porque el estándar de C no define el concepto de proceso. El modelo eligió funcionar en lugar de ser portable, que es la decisión correcta, pero habría sido mejor que la señalara.
>
> **Conclusión metodológica.** En las sesiones 01 y 02 el harness aprobaba código defectuoso y el trabajo consistía en encontrar el fallo. Aquí el código era bueno, y el trabajo fue distinto: inventar entradas que ningún caso contemplaba. Las tres que funcionaron —un dispositivo que falla al escribir, un directorio en lugar de un archivo, un proceso muerto por señal— tienen algo en común: ninguna es una entrada "equivocada" en el sentido obvio. Son situaciones del sistema operativo, no del programa. Eso sugiere que auditar código de sistemas exige conocer el entorno tanto como el lenguaje, y que las pruebas más útiles no salen de leer la especificación sino de preguntarse qué puede fallar por debajo de ella.

---
**Firma digital:** pablouriellopez68-sketch
