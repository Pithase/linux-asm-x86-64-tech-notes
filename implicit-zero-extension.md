# Limpieza Automática de Registros Parciales en x86-64

Una particularidad crítica en la arquitectura x86-64 —definida originalmente por AMD e implementada también por Intel— es la **limpieza automática de los 32 bits superiores de los registros generales de 64 bits** cuando se escribe en su porción inferior de 32 bits.

> A lo largo de este capítulo se utilizará el término registro parcial para referirse a cualquier variante de menor tamaño de un registro general de 64 bits. Por ejemplo, `EAX`, `AX`, `AH` y `AL` son todos registros parciales del registro completo `RAX`.


Este comportamiento, conocido como **_implicit zero-extension_**, puede resultar confuso para quienes migran desde arquitecturas anteriores (x86 de 32 bits o inferiores), en las cuales dicha limpieza no ocurre automáticamente. **En este artículo** se analizará en profundidad su funcionamiento, justificación y consecuencias prácticas.

La arquitectura x86-64 define **16 registros de propósito general**:
`RAX`, `RBX`, `RCX`, `RDX`, `RSI`, `RDI`, `RBP`, `RSP`, `R8`-`R15`.

Todos ellos pueden ser accedidos en sus variantes de:
- **64 bits:** `RAX`, `RBX`, ..., `R15`  
- **32 bits:** `EAX`, `EBX`, ..., `R15D`  
- **16 bits:** `AX`, `BX`, ..., `R15W`

Solo los registros `AX`, `BX`, `CX` y `DX` disponen de acceso explícito a su byte alto mediante los registros `AH`, `BH`, `CH` y `DH`.

Ningún otro registro general —incluyendo los heredados `SI`, `DI`, `BP`, `SP`, o los extendidos `R8` a `R15`— posee un equivalente de 8 bits altos accesible.

En estos casos, el acceso de 8 bits está limitado al byte bajo (`SIL`, `DIL`, `R8B`, etc.).

En la siguiente tabla se muestra, para cada registro general de 64 bits:
- Su correspondiente registro de 32 bits que **dispara la limpieza automática**
- Sus nombres alternativos de 16 bits
- Accesos válidos a los bytes bajos y altos

| 64 bits | 32 bits | 16 bits | 8 bits<br>altos | 8 bits<br>bajos |    
|:-------:|:-------:|:-------:|:------:|:------:|  
|   `RAX`   |   `EAX`   |   `AX`    |   `AH`   |  `AL`    |  
|   `RBX`   |   `EBX`   |   `BX`    |   `BH`   |  `BL`    |  
|   `RCX`   |   `ECX`   |   `CX`    |   `CH`   |  `CL`    |  
|   `RDX`   |   `EDX`   |   `DX`    |   `DH`   |  `DL`    |  
|   `RSI`   |   `ESI`   |   `SI`    |   —    |  `SIL`   |  
|   `RDI`   |   `EDI`   |   `DI`    |   —    |  `DIL`   |  
|   `RBP`   |   `EBP`   |   `BP`    |   —    |  `BPL`   |  
|   `RSP`   |   `ESP`   |   `SP`    |   —    |  `SPL`   |  
|   `R8`    |   `R8D`   |   `R8W`   |   —    |  `R8B`   |  
|   `R9`    |   `R9D`   |   `R9W`   |   —    |  `R9B`   |  
|   `R10`   |   `R10D`  |   `R10W`  |   —    |  `R10B`  | 
|   `R11`   |   `R11D`  |   `R11W`  |   —    |  `R11B`  |  
|   `R12`   |   `R12D`  |   `R12W`  |   —    |  `R12B`  |  
|   `R13`   |   `R13D`  |   `R13W`  |   —    |  `R13B`  |  
|   `R14`   |   `R14D`  |   `R14W`  |   —    |  `R14B`  | 
|   `R15`   |   `R15D`  |   `R15W`  |   —    |  `R15B`  |  

## Contexto Histórico y Técnico  

Como se ha visto, la arquitectura original **x86** define un conjunto de registros generales, entre ellos los conocidos `AX`, `BX`, `CX` y `DX`, cada uno de 16 bits. Con el avance hacia procesadores de 32 bits (Intel 80386 y posteriores), estos registros se extendieron a 32 bits (`EAX`, `EBX`, etc.). La transición mantuvo retrocompatibilidad al conservar la capacidad de acceder parcialmente a partes más pequeñas de los registros, como `AX` (los 16 bits bajos), o incluso a los fragmentos de 8 bits como `AL` (bits [7:0]) y `AH` (bits [15:8]) dentro de `AX`.

Con la definición de la arquitectura **x86-64** (también denominada **AMD64** o **Intel 64**), estos registros fueron extendidos nuevamente, ahora a 64 bits (`RAX`, `RBX`, etc.). Aquí surgió una necesidad técnica y operativa muy específica:

- ¿Qué sucede cuando un  de 32 bits se carga en un registro de 64 bits?  
- ¿Deben conservarse o limpiarse los bits superiores no escritos explícitamente?  

La decisión tomada fue fundamental para evitar errores y simplificar el trabajo tanto del compilador como del programador: **cuando se escribe explícitamente en la porción inferior de 32 bits de un registro de 64 bits, los bits superiores se limpian automáticamente (se ponen a cero)**.

**Ejemplo concreto**  
Si se ejecuta la instrucción:

```nasm
mov eax, 0x12345678
```

El procesador automáticamente establece a cero los bits superiores del registro `RAX` (específicamente, los bits [63:32]). Por tanto, el contenido de `RAX` resulta ser:

```
RAX = 0x0000000012345678
```

Este mecanismo —como se explicó anteriormente— se denomina **implicit zero-extension** (_extensión con ceros implícita_).

## Comportamiento según el tamaño del registro

El comportamiento exacto varía según el tamaño del registro parcial modificado. Comprender estas reglas es esencial para evitar efectos colaterales en operaciones de bajo nivel:
| Registro<BR>destino | Tamaño<BR>escrito  | ¿Se limpian bits superiores del registro completo? |
|:--------:|:-------:|----------------------------------------------------|
|   `RAX`    | 64 bits | No aplica (se escribe el registro completo)        |
|   `EAX`    | 32 bits | Sí. Los bits [63:32] se ponen a cero               |
|   `AX`     | 16 bits | No. Los bits [63:16] permanecen intactos           |
|   `AH`     |  8 bits | No. Los bits [63:16] permanecen intactos           |
|   `AL`     |  8 bits | No. Los bits [63:8]  permanecen intactos           |

**Importante**
- Al escribir en registros parciales como `AL`, `AH` o `AX`, solo se modifican porciones específicas de los 16 bits inferiores de `RAX`, dejando los bits superiores intactos:  
     - `AL`  modifica los bits [7:0].  
     - `AH` modifica los bits [15:8].  
     - `AX` modifica los bits [15:0].
- Los bits superiores del registro (`RAX`) permanecen sin cambios, lo cual puede generar comportamientos inesperados si contienen datos residuales de operaciones anteriores.  
- **Recomendación**: si se necesita garantizar que `RAX` esté completamente limpio antes de realizar operaciones parciales, se debe realizar una limpieza explícita mediante una instrucción como:  

```nasm
xor eax, eax
```

Esto asegura que:

```
RAX = 0x0000000000000000 
```

antes de modificar partes parciales del registro.

Esta instrucción limpia tanto los 32 bits bajos como los 32 superiores de `RAX`, dejándolo completamente en cero.

## Motivaciones Técnicas del Implicit Zero-Extension

La razón principal detrás de esta decisión arquitectónica es la coherencia y seguridad en el tratamiento de direcciones, offsets y cálculos aritméticos entre código de 32 y 64 bits coexistiendo en memoria o migrado automáticamente por compiladores:

- **Compatibilidad hacia atrás**:
El código originalmente escrito para 32 bits que utilizaba registros como `EAX` ahora funciona naturalmente al ser reutilizado o recompilado en entornos de 64 bits, sin generar comportamientos inesperados debido a basura residual en los bits superiores del registro.

- **Prevención de errores sutiles**:
Imaginemos el siguiente escenario donde la limpieza automática no existiera:

    ```nasm
    mov rax, 0xdeadbeefcafebabe
    mov eax, 0x1234abcd
    ```

    Si el procesador no limpiara automáticamente los bits superiores, el resultado sería:

    ```
    RAX = 0xdeadbeef1234abcd 
    ```

    Este comportamiento sería extremadamente peligroso e impredecible, especialmente si se utiliza posteriormente el  en operaciones de punteros, direcciones de memoria o aritmética sensible.

- **Optimización de rendimiento**:
Sin el zero-extension implícito, los compiladores o programadores deberían insertar instrucciones adicionales (como `and rax, 0xFFFFFFFF`) para limpiar explícitamente los bits superiores. Esto implica costos en rendimiento y espacio binario.

## Mecanismo de implementación en el procesador

Internamente, el procesador implementa este comportamiento mediante microcódigo y lógica interna, que asegura que siempre que la instrucción afecte exclusivamente a los 32 bits inferiores del registro (como `EAX`, `EBX`, etc.), se realice una extensión automática con ceros hacia los bits superiores.

A nivel microarquitectónico, este comportamiento no genera overhead significativo, ya que está optimizado al máximo en los núcleos modernos de CPU.

## Verificación Práctica mediante Debugging

Es posible y recomendable demostrar y verificar claramente este comportamiento usando herramientas de depuración, como GDB en Linux:

`Programa: zero-extension.asm`

```nasm
section .text
    global _start

_start:
    mov rax, 0xFFFFFFFFFFFFFFFF
    mov eax, 0x12345678

    ; resultado esperado: rax = 0x0000000012345678

    mov rax, 60
    xor rdi, rdi
    syscall
```

`Compilar: nasm -f elf64 zero-extension.asm -o zero-extension.o`  
`Linkear : ld zero-extension.o -o zero-extension`  

## Verificación con GDB

`$ gdb –q ./zero-extension`

```
(gdb) break _start
(gdb) run
(gdb) nexti
(gdb) nexti
(gdb) print /x $rax
```

Con GDB se confirma claramente la limpieza automática de bits superiores.

```
$1 = 0x12345678
```

## Conclusiones finales y consideraciones prácticas

Este mecanismo implica una buena práctica recomendada a nivel de programación en ensamblador en sistemas x86-64:

- Usar consistentemente instrucciones completas sobre registros de 32 bits (`EAX`) si deseamos asegurar limpieza de bits superiores del registro `RAX`.

- Usar instrucciones parciales (`AX`, `AH`, `AL`) sólo cuando se desea explícitamente preservar los bits superiores o cuando se tiene certeza de no necesitarlos en operaciones futuras que involucren el registro completo.

La comprensión profunda de esta particularidad arquitectónica es imprescindible para cualquier profesional involucrado en desarrollo.

Dominar este tema garantiza claridad técnica, seguridad operativa y desempeño óptimo en código de bajo nivel desarrollado para sistemas x86-64.

Este comportamiento puede resultar contraintuitivo inicialmente, pero tiene razones sólidas basadas en la consistencia operativa y la eficiencia de ejecución.

## Evitando errores mediante la extensión explícita de registros parciales

Cuando se trabaja directamente con registros parciales como `AL`, `AH` o `AX`, surge el problema potencial de conservar es no deseados en los bits superiores del registro completo, debido a que la arquitectura x86-64 no realiza la limpieza automática en estos casos. Esto puede derivar en comportamientos no definidos, errores sutiles en cálculos, punteros corruptos o incluso fallos de seguridad críticos en aplicaciones sensibles.

Existen técnicas específicas recomendadas por Intel y AMD para garantizar la correcta "limpieza" o extensión segura de los bits superiores al cargar es en registros parciales:

1. **Extensión con cero utilizando XOR (Forma clásica)**
La técnica más sencilla y tradicional es limpiar primero el registro completo y luego cargar el  parcial deseado:

```nasm
xor eax, eax             ; limpia el registro `EAX` (y por ende, `RAX`) 
mov al, [byte_data]      ; carga el byte_data de 8 bits en `AL`
```

Esto garantiza que `RAX` contendrá exactamente el valor cargado en `AL`, extendido de forma segura con ceros.

Estado final del registro (ejemplo con el valor de la variable `byte_data` igual a 0x15):

```
RAX = 0x0000000000000015
```

2. **Uso explícito de MOVZX (Move with Zero Extension)**
Una alternativa altamente recomendada es utilizar la instrucción especializada movzx, que fue diseñada precisamente con este propósito:

```nasm
movzx eax, byte [byte_data]      ; carga byte y extiende automáticamente con ceros
```

Esta instrucción realiza la extensión segura en un solo paso, reduciendo código y mejorando claridad. Es la técnica más limpia desde una perspectiva didáctica, aunque ocupa ligeramente más bytes en términos binarios.

Estado final del registro (mismo ejemplo con el valor de la variable `byte_data` igual a 0x15):

```
RAX = 0x0000000000000015
```

3. **Uso explícito de MOVSX (Move with Sign Extension)**
En contextos aritméticos donde el valor a cargar es un número con signo (por ejemplo, int8_t en C), puede ser necesario preservar el bit de signo. Para estos casos se utiliza la instrucción movsx, que extiende el byte cargado replicando el bit 7 (el más significativo) hacia los bits superiores del registro:

```nasm
movsx eax, byte [byte_data]  ; suponiendo que [byte_data] = 0x83  0x83 = 10000011 en binario (−125 en int8_t)
```

Esta instrucción **no realiza extensión con ceros**, sino que **rellena los bits superiores con el bit de signo**.

Estado final del registro:

```
RAX = 0xFFFFFFFFFFFFFF83
```

- Solo los **8 bits inferiores** se conservan (0x83).
- Los **56 bits superiores** se rellenan con 1, porque el bit 7 es 1 (**1**0000011).

## Comparativa entre MOVZX y MOVSX (con byte en memoria)

| Byte en<BR>memoria | Valor interpretado<BR>(como int8_t) | Instrucción utilizada | Resultado en `RAX`   |
|:-------:|:------------------:|:---------------------:|:------------------:|
|   0x15  |        +21         | movzx eax, byte [byte_data]<BR>movsx eax, byte [byte_data] | `0x0000000000000015`<BR>`0x0000000000000015` |
|   0x80  |       −128         | movzx eax, byte [byte_data]<BR>movsx eax, byte [byte_data] | `0x0000000000000080`<BR>`0xFFFFFFFFFFFFFF80` |
|   0xF0  |        −16         | movzx eax, byte [byte_data]<BR>movsx eax, byte [byte_data] | `0x00000000000000F0`<BR>`0xFFFFFFFFFFFFFFF0` |
|   0xFF  |         −1         | movzx eax, byte [byte_data]<BR>movsx eax, byte [byte_data] | `0x00000000000000FF`<BR>`0xFFFFFFFFFFFFFFFF` |

Aunque las instrucciones **movzx** y **movsx** son seguras y claras para realizar extensiones, existen otras formas más manuales que los programadores emplean frecuentemente, como escribir directamente en registros parciales (**`AL`**, **`AH`**, etc.). Estas técnicas requieren cuidado adicional para evitar efectos colaterales en los bits superiores del registro.

**Importante:**
No existe una instrucción mov eax, byte [...] en ensamblador Intel

Si se usa mov al, [...] sin limpiar previamente `RAX`, los bits superiores del registro pueden conservar **contenido residual**, lo que puede producir resultados incorrectos o inseguros.

Por esta razón, se recomienda utilizar movzx o la técnica xor eax, eax + mov al, [...] para garantizar una extensión limpia con ceros.

## Extensión desde operandos de 32 bits

Las instrucciones **movzx** y **movsx** funcionan con operandos de 8 (byte) y 16 (word) bits, pero **no aceptan extensiones desde valores de 32 (dword) bits**. En x86-64, las operaciones que escriben en registros de 32 bits (`EAX`, `EBX`, etc.) **provocan automáticamente una extensión con ceros hacia el registro de 64 bits completo**.

Para realizar una extensión con signo desde 32 bits, se utiliza una instrucción distinta: **movsxd**.

Por lo tanto, la sintaxis es inválida para una instrucción como:

```nasm
movzx rax, dword [byte_data]  ; inválido
```

En este caso simplemente se utiliza:

```nasm
mov eax, [byte_data] ; correcto
```

Este comportamiento implica que **escribir en `EAX` provoca una zero-extension implícita**: los bits [63:32] de `RAX` se limpian automáticamente, dejando el registro en un estado definido y seguro.

En otras palabras, **mov eax, [...]** es funcionalmente equivalente a **movzx rax, eax**.

**Nota:** A pesar de su nombre similar, **movsxd** no es una variante de **movsx**, sino una **instrucción independiente** introducida en la arquitectura x86-64.

Está diseñada específicamente para extender un valor de 32 bits con signo (dword) a 64 bits (qword), preservando el bit de signo (bit 31).

**Nota**: Aunque movsxd y movsx comparten parte del nombre, son instrucciones totalmente independientes. La instrucción movsxd fue introducida específicamente en x86-64 para extender desde valores de 32 bits (dword) hacia registros de 64 bits (qword), preservando el bit de signo.
Cuando se desea realizar una **sign-extension** desde 32 bits (por ejemplo, para preservar el bit de signo de un int32_t), se puede utilizar:

```nasm
movsxd rax, eax               ; extiende con el bit 31
movsxd rax, [byte_data]       ; también válido desde memoria
```

Este tipo de extensión explícita es útil cuando se trabaja con valores con signo y se requiere una representación correcta en 64 bits, como puede ocurrir al interactuar con estructuras de C o al realizar operaciones aritméticas en registros extendidos.

## Recomendaciones prácticas finales

Para desarrollar código robusto, claro y profesional en ensamblador x86-64, especialmente en contextos sensibles, la recomendación más segura y clara es utilizar siempre una técnica explícita de extensión:

- Si se trabaja con valores sin signo, usar preferentemente la técnica con XOR o MOVZX.
- Si se trabaja con valores con signo, utilizar MOVSX.

Finalmente, es importante recalcar que depender exclusivamente de operaciones parciales (`AL`, `AH`, `AX`) sin una extensión explícita puede conducir a errores inesperados en situaciones críticas, razón por la cual este concepto debe dominarse completamente al escribir código de bajo nivel.

Dominar y aplicar consistentemente estas técnicas protege el código contra vulnerabilidades sutiles y garantiza el correcto funcionamiento en escenarios avanzados de desarrollo ofensivo.

Finalmente, para cerrar este capítulo, se presentan preguntas frecuentes y aclaraciones técnicas adicionales que suelen surgir durante el desarrollo real en ensamblador x86-64.

## Consideraciones prácticas en el desarrollo en Ensamblador x86-64

**¿Por qué se utiliza xor eax, eax y no xor rax, rax?**  
Ambas instrucciones tienen el **mismo efecto funcional**: limpian completamente el registro `RAX`. 
Desde el punto de vista del código máquina:
- xor eax, eax ocupa **2 bytes**.
- xor rax, rax ocupa **3 bytes**.

**¿Por qué no usar siempre movzx si es más claro?**  
**movzx** es limpia, segura y realiza la extensión con ceros en una única instrucción, lo cual mejora la claridad y reduce el riesgo de errores.
Desde el punto de vista del código máquina:
- movzx eax, byte [byte_data] ocupa **3 bytes**.
- xor eax, eax + mov al, [...] ocupa **4 bytes** (2 + 2).

Entonces, **movzx es incluso más compacto**, además de más claro.

Sin embargo, algunos programadores prefieren la secuencia **xor** + **mov** por motivos como:
- Tener mayor control sobre registros parciales (**mov al**)
- Reutilizar código sin depender de instrucciones específicas del set extendido
  
Ambas técnicas son válidas. La elección depende del contexto, el objetivo (claridad, tamaño, compatibilidad), y las restricciones de entorno.
