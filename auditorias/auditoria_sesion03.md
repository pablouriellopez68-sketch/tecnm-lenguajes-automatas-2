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
```

---

## 4. Fase 3: El Auditor (Resultados del Test Harness)
*Pruebas ejecutadas:* Detección de errores de compilación GCC

* [ ] Caso base y límites superados sin *Crash*.
* [ ] No existen accesos fuera de límites ni *Memory Leaks*.
* [ ] La complejidad teórica coincide con la implementada.

### Reporte Crítico
*¿Qué caso borde o problema omitió la IA y cómo lo solucionaste?*
> 

---
**Firma digital:** [Usuario de GitHub]
