# Bitácora de Requerimientos y Auditoría IA — Sesión 01
**Materia:** Lenguajes y Autómatas II (SCD-1016)  
**Tema:** Encuadre, Diagnóstico de Punteros y Metodología IA  
**Docente Titular:** MCC. Iván Márquez Larios (`ijmarquezl`)  
**Estudiante:** Pablo Uriel Desiderio López [Nombre Completo]  
**Matrícula:** 24531372 [No. de Control]  
**Track:** [ C | Rust ]  

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
[Pega aquí el prompt formal que redactaste para la IA]
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
// Código devuelto por el LLM
```

---

## 4. Fase 3: El Auditor (Resultados del Test Harness)
*Pruebas ejecutadas:* Valores NULL, buffers vacios y longitudes impares

* [ ] Caso base y límites superados sin *Crash*.
* [ ] No existen accesos fuera de límites ni *Memory Leaks*.
* [ ] La complejidad teórica coincide con la implementada.

### Reporte Crítico
*¿Qué caso borde o problema omitió la IA y cómo lo solucionaste?*
> 

---
**Firma digital:** [Usuario de GitHub]
