# Lenguaje Ensamblador x86-64 en Linux

### Optimización de código

> ### code: hex_to_int

Partiremos de un fragmento de código tomado del programa [`dns-payload-loader-full-dynamic-checksum.asm`](https://github.com/Pithase/asm-payloads-loaders/blob/main/dns-payload-loader-full-dynamic-checksum.asm), que implementa una rutina para convertir un carácter hexadecimal (en formato ASCII) en su valor decimal equivalente (de 0 a 15).

El fragmento de código original es el siguiente:

```nasm
    ;=========================================================================================
    ; 17. Convierte un carácter ASCII (almacenado en DL) a su valor numérico (0-15)
    ;     Entrada: DL = carácter ('0'-'9', 'a'-'f')
    ;     Salida : DL = valor numérico (0-15)
    ;     Se asume que las letras siempre están en minúsculas
    ;=========================================================================================
hex_to_int:
    cmp dl, '9'                          ; si es menor o igual a '9', es un dígito decimal
    jbe .digit_ok
    sub dl, 'a'                          ; 'a' (ASCII 97) -> 10 decimal
    add dl, 10
    ret

.digit_ok:
    sub dl, '0'                          ; convertir '0'-'9' en 0-9
    ret
```

Luego de realizar las adaptaciones necesarias para que funcione como programa independiente, el código queda de la siguiente forma:


```nasm
;======================================================================================
; Archivo      : hex_to_int.asm
; Creado       : 08/04/2025
; Modificado   : 08/04/2025
; Autor        : Gastón M. González
; Plataforma   : Linux
; Arquitectura : x86-64
;======================================================================================
; Descripción  : Convierte un carácter hexadecimal en su valor numérico (0–15)
;                Se asume que el carácter está en minúsculas y es válido
;======================================================================================
; Compilar     : nasm -f elf64 -O0 hex_to_int.asm -o hex_to_int.o
; Linkear      : ld hex_to_int.o -o hex_to_int
; Ejecutar     : ./hex_to_int ; echo $?
;======================================================================================

global _start

section .data
    hex db 'c'         ; letra 'c' -> 0xC = 12

section .text
_start:
    mov al, [hex]      ; carga el valor ASCII desde la variable 'hex' en AL
    call hex_to_int

    mov rdi, 0         ; inicializa RDI en 0
    mov dil, al        ; copia el resultado a DIL -> se reflejará en RDI
    mov rax, 60        ; inicializa RAX en 60 -> syscall: exit
    syscall            ; finaliza el proceso con el código en RDI

;--------------------------------------------------------------------------------------
; Función      : hex_to_int
; Descripción  : Convierte un carácter ASCII hexadecimal ('0'-'9', 'a'-'f') en su
;                valor numérico correspondiente (0–15)
;
; Entrada      : AL = carácter hexadecimal válido (en minúsculas)
; Salida       : AL = valor numérico entre 0 y 15
; Restricciones: No realiza validación de caracteres fuera del rango permitido
;--------------------------------------------------------------------------------------
hex_to_int:
    cmp al, '9'        ; si es menor o igual a '9', es un dígito decimal
    jbe .digit_ok
    sub al, 'a'        ; 'a' (ASCII 97) -> 10 decimal
    add al, 10
    ret

.digit_ok:
    sub al, '0'        ; convertir '0'-'9' en 0-9
    ret
```

Para facilitar la validación del funcionamiento del programa —sin "ensuciarlo" con código adicional ni recurrir a un debugger—, el resultado se envía como **código de salida** (`exit`). Esto permite visualizarlo fácilmente desde la terminal.

#### Compilar, enlazar y ejecutar
```
$ nasm -f elf64 -O0 hex_to_int.asm -o hex_to_int.o
$ ld hex_to_int.o -o hex_to_int
$ ./hex_to_int ; echo $?
$ 12
```

Usamos la opción `-O0` (optimización deshabilitada) para indicarle a NASM que no realice transformaciones automáticas en las instrucciones. Esto es útil para preservar el código **exactamente como lo escribimos**, lo que resulta ideal para fines didácticos, debugging, o análisis de **opcodes** sin modificaciones inesperadas.

Un **opcode** es el código de una instrucción en lenguaje máquina —puede abarcar de 1 a varios bytes— que indica al procesador la operación a realizar. Conservar los opcodes originales facilita un análisis detallado del comportamiento del programa.

**Nota:** En la práctica —especialmente en contextos de exploit development, red teaming o análisis de shellcode— el término **opcodes** se emplea para referirse a **todos los bytes** que conforman una instrucción completa, incluyendo tanto el **opcode** como los **operandos**. En este artículo usaremos esta acepción.

En futuras etapas puede ser útil aplicar optimizaciones, pero durante el aprendizaje progresivo es preferible trabajar con la optimización deshabilitada.

Una vez ejecutado, se observa que la rutina funciona correctamente: transforma el carácter `'c'` (que representa al hexadecimal `0xc`) en su valor numérico correspondiente, `12` decimal.

A continuación se muestra el desensamblado del binario —es decir, del programa ya enlazado— utilizando **objdump** con **sintaxis Intel**. Esto, además, nos permite observar los **opcodes generados**:

```
$ objdump -d -M intel hex_to_int

hex_to_int:     file format elf64-x86-64


Disassembly of section .text:

0000000000401000 <_start>:
  401000:       8a 04 25 00 20 40 00    mov    al,BYTE PTR ds:0x402000
  401007:       e8 19 00 00 00          call   401025 <hex_to_int>
  40100c:       48 bf 00 00 00 00 00    movabs rdi,0x0
  401013:       00 00 00
  401016:       40 88 c7                mov    dil,al
  401019:       48 b8 3c 00 00 00 00    movabs rax,0x3c
  401020:       00 00 00
  401023:       0f 05                   syscall

0000000000401025 <hex_to_int>:
  401025:       3c 39                   cmp    al,0x39
  401027:       76 05                   jbe    40102e <hex_to_int.digit_ok>
  401029:       2c 61                   sub    al,0x61
  40102b:       04 0a                   add    al,0xa
  40102d:       c3                      ret

000000000040102e <hex_to_int.digit_ok>:
  40102e:       2c 30                   sub    al,0x30
  401030:       c3                      ret

```

Como se puede observar, algunas instrucciones no se muestran exactamente como fueron escritas en el código fuente, lo que puede generar confusión al principio. Por ejemplo:

- La instrucción `mov rdi, 0` aparece como `movabs rdi, 0x0`
- La instrucción `mov rax, 60` se representa como `movabs rax, 0x3c`

Esto se debe a que **objdump no interpreta el código fuente**, sino que desensambla directamente a partir de los **bytes** del binario. Para mostrar las instrucciones, recurre a sus propios criterios y prioriza su **representación estándar**, que puede diferir de la forma en que se escribió originalmente en el ensamblador.

Para evitar este inconveniente, utilizaremos la herramienta **objcopy** para generar un archivo binario que contenga únicamente la sección de código correspondiente a `.text` del programa objeto. 

También se podría generar el binario directamente con:

```
nasm -f bin hex_to_int.asm -o hex_to_int.bin
```

Sin embargo, en ese caso se incluiría también la sección `.data`, que para este análisis no se tendrá en cuenta.

#### objcopy
```
$ objcopy -O binary -j .text hex_to_int hex_to_int.bin
```

Este comando extrae la sección `.text` del archivo objeto y la guarda como un binario plano (`.bin`), eliminando cualquier otra sección como `.data`, `.bss` o cabeceras ELF. De esta manera, obtenemos un archivo que contiene **exclusivamente las instrucciones de código**, listo para análisis.

Finalmente, desensamblamos el archivo resultante utilizando la herramienta **ndisasm**:

#### ndisasm
```
$ ndisasm -b 64 hex_to_int.bin

00000000  8A042500204000    mov al,[0x402000]
00000007  E819000000        call 0x25
0000000C  48BF000000000000  mov rdi,0x0
         -0000
00000016  4088C7            mov dil,al
00000019  48B83C0000000000  mov rax,0x3c
         -0000
00000023  0F05              syscall
00000025  3C39              cmp al,0x39
00000027  7605              jna 0x2e
00000029  2C61              sub al,0x61
0000002B  040A              add al,0xa
0000002D  C3                ret
0000002E  2C30              sub al,0x30
00000030  C3                ret
```

Como puede observarse, `ndisasm` muestra cada instrucción junto a su offset y representación hexadecimal (opcode). Esta herramienta no interpreta símbolos ni secciones: opera directamente sobre los bytes del binario, lo que la hace ideal para obtener una visión fiel del código generado, especialmente en análisis de bajo nivel.

Ahora obtenemos los **opcodes** y la cantidad de **bytes** que ocupa el código del programa.

Para ello utilizamos el siguiente **one-liner**, escrito en lenguaje **Bash**:

#### opcodes y bytes
```
$ ndisasm -b 64 hex_to_int.bin | awk 'BEGIN{printf "Opcodes: ";count=0;partial=""} /^[[:space:]]*[0-9A-Fa-f]+[[:space:]]+[0-9A-Fa-f]+/ {if(partial!=""){process(partial);partial=""} split($0,arr,/[[:space:]]+/);partial=arr[2];next} /^[[:space:]]*-[0-9A-Fa-f]+/ {sub(/^[[:space:]]*-/,"",$0);partial=partial $0;next} END{if(partial!=""){process(partial)} printf "\nBytes..: %d\n",count} function process(str){gsub(/[^0-9A-Fa-f]/,"",str);for(i=1;i<=length(str);i+=2){byte=substr(str,i,2);if(byte~ /^[0-9A-Fa-f]{2}$/){printf toupper(byte)" ";count++}}}'

Opcodes: 8A 04 25 00 20 40 00 E8 19 00 00 00 48 BF 00 00 00 00 00 00 00 00 40 88 C7 48 B8 3C 00 00 00 00 00 00 00 0F 05 3C 39 76 05 2C 61 04 0A C3 2C 30 C3
Bytes..: 49
```

Otro dato interesante a tener en cuenta al comparar la evolución del código es la cantidad de **instrucciones efectivamente ejecutadas** en tiempo de ejecución (no debe confundirse con la cantidad de instrucciones escritas en el programa).

Para obtener esa información con `qemu-x86_64`, usamos el siguiente **one-liner**:

### qemu
```
$ qemu-x86_64 -d in_asm ./hex_to_int 2>&1 | grep -Eo '^0x[0-9a-f]+:' | wc -l | xargs echo "Instrucciones ejecutadas:"

Instrucciones ejecutadas: 11
```

> Esto captura la traza dinámica de ejecución y cuenta cuántas instrucciones fueron realmente ejecutadas por la CPU durante todo el ciclo de vida del binario.

**Nota:** la métrica que falta en este análisis es la cantidad de **ciclos de CPU consumidos**. Este dato puede obtenerse con herramientas como `perf`, siempre que el hardware y el entorno lo permitan. En entornos virtualizados (como VirtualBox, VMware o EC2 de AWS), esta información suele estar restringida o no disponible.

Volvamos a [ndisasm](#ndisasm) para analizar la información que generó y notar que existen diferencias respecto al código fuente original:

> **Notas técnicas:**

> `jbe .digit_ok` vs `jna 0x2e ≡ jna .digit_ok`
>
> `jbe` y `jna` son exactamente la **misma instrucción**. Ambas generan el **mismo opcode**, pero ofrecen nombres alternativos que el programador puede elegir según su perspectiva, con el objetivo de expresar su idea de forma más clara.
> - **jbe** (*Jump if Below or Equal*): se utiliza habitualmente en comparaciones sin signo.
> - **jna** (*Jump if Not Above*): es su equivalente lógico, solo que con una formulación inversa.
>

> `mov al, [hex]` vs `mov al, [0x402000]`
>
> Cuando escribimos en ensamblador:
>
> ```nasm
> mov al, [hex]
> ```
>
> esperamos que el ensamblador resuelva `hex` como una **etiqueta relativa a RIP** (registro de instrucción), algo habitual en código de 64 bits. Sin embargo, si no se utiliza explícitamente el modificador `rel`, NASM puede generar una instrucción que accede **de forma absoluta** a la dirección donde se encuentra la variable `hex` en memoria. Por ejemplo:
>
> ```nasm
> mov al, BYTE PTR ds:0x402000
> ```
>
> Esto representa una **dirección absoluta codificada en el binario**, no relativa. Desde el punto de vista del código ensamblado, es equivalente a escribir:
>
> ```nasm
> mov al, [0x402000]
> ```
>
> Este tipo de acceso tiene implicancias importantes en términos de portabilidad y uso en payload/shellcode, ya que:
>
> - **No es independiente de la ubicación del código en memoria**
> - **Puede incluir bytes nulos (`0x00`)** en la dirección codificada
> - **Es menos flexible** para posicionamiento dinámico

## Comenzando con la optimización

En esta primera etapa respetaremos la **estructura lógica** del programa original, pero comenzaremos a aplicar mejoras centradas en dos objetivos clave:

1. **Aportar claridad** al código, para que sea más legible y explícito.  
2. **Reducir el tamaño** en bytes de las instrucciones, optimizando su representación sin alterar la lógica.

> ### Paso 1

#### Mejorar la claridad

Una buena práctica al escribir código en ensamblador es ser explícito con las intenciones. En lugar de:

`mov al, [hex]` escribimos: `mov al, byte [hex]`

En este caso no se modifica el comportamiento, pero **hace más claro** que estamos accediendo a un solo byte —algo relevante cuando trabajamos con direcciones o registros de mayor tamaño—.

---

#### Reducir el tamaño del código

Existen instrucciones que realizan lo mismo pero ocupan menos espacio binario. Los casos detectados son los siguientes:

`mov rdi, 0` → lo reemplazamos por `mov edi, 0`

`mov rax, 60` → lo reemplazamos por `mov eax, 60`

Estas versiones más cortas aprovechan la [implicit zero-extension](https://github.com/Pithase/linux-asm-x86-64-tech-notes/blob/main/implicit-zero-extension.md) del modo 64 bits: al escribir en un registro de 32 bits como `EAX` o `EDI`, los bits superiores del registro de 64 bits (`RAX`, `RDI`) se ponen automáticamente en cero.

Este tipo de sustituciones **no afecta la lógica** del programa, pero sí su tamaño final, algo clave en la optimización de payloads y shellcodes.

---

<details>
<summary><strong>Código corregido</strong> (expandir)</summary>

```nasm
;======================================================================================
; Archivo      : hex_to_int.asm
; Creado       : 08/04/2025
; Modificado   : 08/04/2025
; Autor        : Gastón M. González
; Plataforma   : Linux
; Arquitectura : x86-64
;======================================================================================
; Descripción  : Convierte un carácter hexadecimal en su valor numérico (0–15)
;                Se asume que el carácter está en minúsculas y es válido
;======================================================================================
; Compilar     : nasm -f elf64 -O0 hex_to_int.asm -o hex_to_int.o
; Linkear      : ld hex_to_int.o -o hex_to_int
; Ejecutar     : ./hex_to_int ; echo $?
;======================================================================================

global _start

section .data
    hex db 'c'         ; letra 'c' -> 0xC = 12

section .text
_start:
    mov al, byte[hex]  ; carga el valor ASCII desde la variable 'hex' en AL
    call hex_to_int

    mov edi, 0         ; inicializa EDI en 0, por consiguiente RDI en 0
    mov dil, al        ; copia el resultado a DIL -> se reflejará en RDI
    mov eax, 60        ; inicializa EAX en 60, por consiguiente RAX en 60 -> syscall: exit
    syscall            ; finaliza el proceso con el código en RDI

;-------------------------------------------------------------------------------------------
; Función      : hex_to_int
; Descripción  : Convierte un carácter ASCII hexadecimal ('0'-'9', 'a'-'f') en su
;                valor numérico correspondiente (0–15)
;
; Entrada      : AL = carácter hexadecimal válido (en minúsculas)
; Salida       : AL = valor numérico entre 0 y 15
; Restricciones: No realiza validación de caracteres fuera del rango permitido
;------------------------------------------------------------------------------------------
hex_to_int:
    cmp al, '9'        ; si es menor o igual a '9', es un dígito decimal
    jbe .digit_ok
    sub al, 'a'        ; 'a' (ASCII 97) -> 10 decimal
    add al, 10
    ret

.digit_ok:
    sub al, '0'        ; convertir '0'-'9' en 0-9
    ret
```
</details>

Ahora [compilamos, enlazamos y ejecutamos](#compilar-enlazar-y-ejecutar). Verificamos que el resultado siga siendo el correcto (12 decimal) y generamos el archivo binario con [objcopy](#objcopy).

#### Métricas

Ejecutamos el [one-liner](#opcodes-y-bytes) para obtener los opcodes y la cantidad de bytes que ocupa el programa:

```
Opcodes: 8A 04 25 00 20 40 00 E8 0F 00 00 00 BF 00 00 00 00 40 88 C7 B8 3C 00 00 00 0F 05 3C 39 76 05 2C 61 04 0A C3 2C 30 C3
Bytes..: 39
```

Luego, ejecutamos [qemu](#qemu) para medir la cantidad de instrucciones efectivamente ejecutadas en tiempo de ejecución:

```
Instrucciones ejecutadas: 11
```

#### Comparativa

| Versión  | Tamaño (bytes) | Instrucciones ejecutadas |
|----------|:--------------:|:------------------------:|
| Anterior |             49 |                       11 |
| Actual   |             39 |                       11 |

#### Diferencias

Para aportar mayor rigurosidad al análisis, se puede ejecutar [ndisasm](#ndisasm) y comparar las diferencias con la salida de la versión anterior. Esto permite observar con precisión cómo se modificaron las instrucciones a nivel binario.

```nasm
; Versión anterior
48BF0000000000000000  mov rdi,0x0
48B83C00000000000000  mov rax,0x3c

; Versión actual
BF00000000            mov edi,0x0     ; 5 bytes menos
B83C000000            mov eax,0x3c    ; 5 bytes menos
```

> ### Paso 2

#### Reducir el tamaño del código

`mov edi, 0` → lo reemplazamos por `xor edi, edi`

Al ejecutar `xor edi, edi`, cada bit de `EDI` se combina mediante XOR con su propio valor, lo que produce 0 en cada bit y deja el registro EDI en cero. Debido a la [implicit zero-extension](https://github.com/Pithase/linux-asm-x86-64-tech-notes/blob/main/implicit-zero-extension.md) en modo 64 bits, esto también limpia los bits superiores de RDI

---

<details>
<summary><strong>Código corregido</strong> (expandir)</summary>

```nasm
;======================================================================================
; Archivo      : hex_to_int.asm
; Creado       : 08/04/2025
; Modificado   : 08/04/2025
; Autor        : Gastón M. González
; Plataforma   : Linux
; Arquitectura : x86-64
;======================================================================================
; Descripción  : Convierte un carácter hexadecimal en su valor numérico (0–15)
;                Se asume que el carácter está en minúsculas y es válido
;======================================================================================
; Compilar     : nasm -f elf64 -O0 hex_to_int.asm -o hex_to_int.o
; Linkear      : ld hex_to_int.o -o hex_to_int
; Ejecutar     : ./hex_to_int ; echo $?
;======================================================================================

global _start

section .data
    hex db 'c'         ; letra 'c' -> 0xC = 12

section .text
_start:
    mov al, byte[hex]  ; carga el valor ASCII desde la variable 'hex' en AL
    call hex_to_int

    xor edi, edi       ; inicializa EDI en 0, por consiguiente RDI en 0
    mov dil, al        ; copia el resultado a DIL -> se reflejará en RDI
    mov eax, 60        ; inicializa EAX en 60, por consiguiente RAX en 60 -> syscall: exit
    syscall            ; finaliza el proceso con el código en RDI

;-------------------------------------------------------------------------------------------
; Función      : hex_to_int
; Descripción  : Convierte un carácter ASCII hexadecimal ('0'-'9', 'a'-'f') en su
;                valor numérico correspondiente (0–15)
;
; Entrada      : AL = carácter hexadecimal válido (en minúsculas)
; Salida       : AL = valor numérico entre 0 y 15
; Restricciones: No realiza validación de caracteres fuera del rango permitido
;------------------------------------------------------------------------------------------
hex_to_int:
    cmp al, '9'        ; si es menor o igual a '9', es un dígito decimal
    jbe .digit_ok
    sub al, 'a'        ; 'a' (ASCII 97) -> 10 decimal
    add al, 10
    ret

.digit_ok:
    sub al, '0'        ; convertir '0'-'9' en 0-9
    ret
```
</details>

#### Métricas

```
Opcodes: 8A 04 25 00 20 40 00 E8 0C 00 00 00 31 FF 40 88 C7 B8 3C 00 00 00 0F 05 3C 39 76 05 2C 61 04 0A C3 2C 30 C3
Bytes..: 36
```

```
Instrucciones ejecutadas: 11
```

#### Comparativa

| Versión  | Tamaño (bytes) | Instrucciones ejecutadas |
|----------|:--------------:|:------------------------:|
| Anterior |             39 |                       11 |
| Actual   |             36 |                       11 |

#### Diferencias

```nasm
; Versión anterior
BF00000000            mov edi,0x0

; Versión actual
31FF                  xor edi,edi     ; 3 bytes menos
```

> ### Paso 3

#### Reducir el tamaño del código

`xor edi, edi` y `mov dil, al` → lo reemplazamos por `movzx rdi, al`

Esta instrucción realiza una extensión explícita: toma el valor de `AL` (8 bits sin signo) y lo coloca en `RDI`, llenando con ceros los bits restantes. De esta forma, `AL` pasa a ocupar la parte baja de `RDI`, mientras que los bits [63:8] quedan automáticamente en cero.

---

<details>
<summary><strong>Código corregido</strong> (expandir)</summary>

```nasm
;======================================================================================
; Archivo      : hex_to_int.asm
; Creado       : 08/04/2025
; Modificado   : 08/04/2025
; Autor        : Gastón M. González
; Plataforma   : Linux
; Arquitectura : x86-64
;======================================================================================
; Descripción  : Convierte un carácter hexadecimal en su valor numérico (0–15)
;                Se asume que el carácter está en minúsculas y es válido
;======================================================================================
; Compilar     : nasm -f elf64 -O0 hex_to_int.asm -o hex_to_int.o
; Linkear      : ld hex_to_int.o -o hex_to_int
; Ejecutar     : ./hex_to_int ; echo $?
;======================================================================================

global _start

section .data
    hex db 'c'         ; letra 'c' -> 0xC = 12

section .text
_start:
    mov al, byte[hex]  ; carga el valor ASCII desde la variable 'hex' en AL
    call hex_to_int

    movzx rdi, al      ; extiende AL (8 bits sin signo) a RDI (64 bits). Los bits[63:8] toman valor 0
    mov eax, 60        ; inicializa EAX en 60, por consiguiente RAX en 60 -> syscall: exit
    syscall            ; finaliza el proceso con el código en RDI

;------------------------------------------------------------------------------------------------------
; Función      : hex_to_int
; Descripción  : Convierte un carácter ASCII hexadecimal ('0'-'9', 'a'-'f') en su
;                valor numérico correspondiente (0–15)
;
; Entrada      : AL = carácter hexadecimal válido (en minúsculas)
; Salida       : AL = valor numérico entre 0 y 15
; Restricciones: No realiza validación de caracteres fuera del rango permitido
;-----------------------------------------------------------------------------------------------------
hex_to_int:
    cmp al, '9'        ; si es menor o igual a '9', es un dígito decimal
    jbe .digit_ok
    sub al, 'a'        ; 'a' (ASCII 97) -> 10 decimal
    add al, 10
    ret

.digit_ok:
    sub al, '0'        ; convertir '0'-'9' en 0-9
    ret
```
</details>

#### Métricas

```
Opcodes: 8A 04 25 00 20 40 00 E8 0B 00 00 00 48 0F B6 F8 B8 3C 00 00 00 0F 05 3C 39 76 05 2C 61 04 0A C3 2C 30 C3
Bytes..: 35
```

```
Instrucciones ejecutadas: 10
```

#### Comparativa

| Versión  | Tamaño (bytes) | Instrucciones ejecutadas |
|----------|:--------------:|:------------------------:|
| Anterior |             36 |                       11 |
| Actual   |             35 |                       10 |

#### Diferencias

```nasm
; Versión anterior
31FF                  xor edi,edi
4088C7                mov dil,al

; Versión actual
480FB6F8              movzx rdi,al    ; 1 byte menos
```

> ### Paso 4

#### Reducir el tamaño del código

`mov al, byte[hex]` → lo reemplazamos por `mov al, byte[rel hex]`

De la forma anterior, se generaba una referencia absoluta en tiempo de ensamblado, lo que podía introducir bytes nulos y dificultar la reubicación del código. Con la nueva instrucción, se fuerza a que el acceso se codifique como una dirección relativa al `RIP` (registro de instrucción actual), mejorando la portabilidad. Esto resulta especialmente valioso en escenarios como payloads o shellcodes.

---

<details>
<summary><strong>Código corregido</strong> (expandir)</summary>

```nasm
;======================================================================================
; Archivo      : hex_to_int.asm
; Creado       : 08/04/2025
; Modificado   : 08/04/2025
; Autor        : Gastón M. González
; Plataforma   : Linux
; Arquitectura : x86-64
;======================================================================================
; Descripción  : Convierte un carácter hexadecimal en su valor numérico (0–15)
;                Se asume que el carácter está en minúsculas y es válido
;======================================================================================
; Compilar     : nasm -f elf64 -O0 hex_to_int.asm -o hex_to_int.o
; Linkear      : ld hex_to_int.o -o hex_to_int
; Ejecutar     : ./hex_to_int ; echo $?
;======================================================================================

global _start

section .data
    hex db 'c'             ; letra 'c' -> 0xC = 12

section .text
_start:
    mov al, byte[rel hex]  ; carga el valor ASCII desde la variable 'hex' en AL
    call hex_to_int

    movzx rdi, al          ; extiende AL (8 bits sin signo) a RDI (64 bits). Los bits[63:8] toman valor 0
    mov eax, 60            ; inicializa EAX en 60, por consiguiente RAX en 60 -> syscall: exit
    syscall                ; finaliza el proceso con el código en RDI

;----------------------------------------------------------------------------------------------------------
; Función      : hex_to_int
; Descripción  : Convierte un carácter ASCII hexadecimal ('0'-'9', 'a'-'f') en su
;                valor numérico correspondiente (0–15)
;
; Entrada      : AL = carácter hexadecimal válido (en minúsculas)
; Salida       : AL = valor numérico entre 0 y 15
; Restricciones: No realiza validación de caracteres fuera del rango permitido
;----------------------------------------------------------------------------------------------------------
hex_to_int:
    cmp al, '9'            ; si es menor o igual a '9', es un dígito decimal
    jbe .digit_ok
    sub al, 'a'            ; 'a' (ASCII 97) -> 10 decimal
    add al, 10
    ret

.digit_ok:
    sub al, '0'            ; convertir '0'-'9' en 0-9
    ret
```
</details>

#### Métricas

```
Opcodes: 8A 05 FA 0F 00 00 E8 0B 00 00 00 48 0F B6 F8 B8 3C 00 00 00 0F 05 3C 39 76 05 2C 61 04 0A C3 2C 30 C3
Bytes..: 34
```

```
Instrucciones ejecutadas: 10
```

#### Comparativa

| Versión  | Tamaño (bytes) | Instrucciones ejecutadas |
|----------|:--------------:|:------------------------:|
| Anterior |             35 |                       10 |
| Actual   |             34 |                       10 |

#### Diferencias

```nasm
; Versión anterior
8A042500204000        mov al,[0x402000]

; Versión actual
8A05FA0F0000          mov al,[rel 0x1000]   ; 1 byte menos 
```

> ### Paso 5

#### Reducir el tamaño del código

`mov eax, 60` → lo reemplazamos por `xor eax, eax` y `mov al, 60`

Esta sustitución logra exactamente el mismo resultado funcional, **reduciendo** la cantidad de **bytes utilizados** en el proceso. Con `xor eax, eax` inicializamos `EAX` en 0 y, gracias al [implicit zero-extension](https://github.com/Pithase/linux-asm-x86-64-tech-notes/blob/main/implicit-zero-extension.md) en modo 64 bits, se limpian automáticamente los bits superiores de `RDI`. Luego, asignamos a la parte baja del registro, `AL`, el valor correspondiente.

---

<details>
<summary><strong>Código corregido</strong> (expandir)</summary>

```nasm
;======================================================================================
; Archivo      : hex_to_int.asm
; Creado       : 08/04/2025
; Modificado   : 08/04/2025
; Autor        : Gastón M. González
; Plataforma   : Linux
; Arquitectura : x86-64
;======================================================================================
; Descripción  : Convierte un carácter hexadecimal en su valor numérico (0–15)
;                Se asume que el carácter está en minúsculas y es válido
;======================================================================================
; Compilar     : nasm -f elf64 -O0 hex_to_int.asm -o hex_to_int.o
; Linkear      : ld hex_to_int.o -o hex_to_int
; Ejecutar     : ./hex_to_int ; echo $?
;======================================================================================

global _start

section .data
    hex db 'c'             ; letra 'c' -> 0xC = 12

section .text
_start:
    mov al, byte[rel hex]  ; carga el valor ASCII desde la variable 'hex' en AL
    call hex_to_int

    movzx rdi, al          ; extiende AL (8 bits sin signo) a RDI (64 bits). Los bits[63:8] toman valor 0
    xor eax, eax           ; inicializa EAX en 0, por consiguiente RAX en 0
    mov al, 60             ; inicializa EAX en 60 -> syscall: exit
    syscall                ; finaliza el proceso con el código en RDI

;----------------------------------------------------------------------------------------------------------
; Función      : hex_to_int
; Descripción  : Convierte un carácter ASCII hexadecimal ('0'-'9', 'a'-'f') en su
;                valor numérico correspondiente (0–15)
;
; Entrada      : AL = carácter hexadecimal válido (en minúsculas)
; Salida       : AL = valor numérico entre 0 y 15
; Restricciones: No realiza validación de caracteres fuera del rango permitido
;----------------------------------------------------------------------------------------------------------
hex_to_int:
    cmp al, '9'            ; si es menor o igual a '9', es un dígito decimal
    jbe .digit_ok
    sub al, 'a'            ; 'a' (ASCII 97) -> 10 decimal
    add al, 10
    ret

.digit_ok:
    sub al, '0'            ; convertir '0'-'9' en 0-9
    ret
```
</details>

#### Métricas

```
Opcodes: 8A 05 FA 0F 00 00 E8 0A 00 00 00 48 0F B6 F8 31 C0 B0 3C 0F 05 3C 39 76 05 2C 61 04 0A C3 2C 30 C3
Bytes..: 33
```

```
Instrucciones ejecutadas: 11
```

#### Comparativa

| Versión  | Tamaño (bytes) | Instrucciones ejecutadas |
|----------|:--------------:|:------------------------:|
| Anterior |             34 |                       10 |
| Actual   |             33 |                       11 |

#### Diferencias

```nasm
; Versión anterior
B83C000000            mov eax,0x3c

; Versión actual
31C0                  xor eax,eax
B03C                  mov al,0x3c   ; con ambas instrucciones 1 byte menos
```

---

#### Resumen

Respetando la **estructura lógica** del programa original, se logró reducir **16 bytes**, pasando de **49 bytes** a **33 bytes**, lo que representa una reducción del **32,65 %** en el tamaño del código.

Cambios realizados:

```nasm
mov rdi, 0  → lo reemplazamos por mov edi, 0
mov rax, 60 → lo reemplazamos por mov eax, 60

mov edi, 0 → lo reemplazamos por xor edi, edi

xor edi, edi y mov dil, al → lo reemplazamos por movzx rdi, al

mov al, byte[hex] → lo reemplazamos por mov al, byte[rel hex]

mov eax, 60 → lo reemplazamos por xor eax, eax y mov al, 60
```

Esta mejora se logró únicamente reemplazando instrucciones por otras funcionalmente equivalentes pero más compactas, sin alterar el flujo ni la semántica del programa. Esto demuestra cómo incluso pequeñas optimizaciones a nivel de ensamblador pueden tener un impacto significativo en la eficiencia de payloads, shellcodes o binarios embebidos.

## Optimización hardcore

<details>
<summary><strong>Código</strong> (expandir)</summary>

```nasm
;======================================================================================
; Archivo      : hex_to_int.asm
; Creado       : 08/04/2025
; Modificado   : 08/04/2025
; Autor        : Gastón M. González
; Plataforma   : Linux
; Arquitectura : x86-64
;======================================================================================
; Descripción  : Convierte un carácter hexadecimal en su valor numérico (0–15)
;                Se asume que el carácter está en minúsculas y es válido
;======================================================================================
; Compilar     : nasm -f elf64 -O0 hex_to_int.asm -o hex_to_int.o
; Linkear      : ld hex_to_int.o -o hex_to_int
; Ejecutar     : ./hex_to_int ; echo $?
;======================================================================================

global _start

section .data
    hex db 'c'                ; letra 'c' → ASCII =  → 0xC = 12

section .text
_start:
    mov al, byte [rel hex]    ; carga el valor ASCII desde la variable 'hex' en AL
    sub al, '0'               ; ajusta base ASCII
    cmp al, 9                 ; ¿es decimal? 
    jbe .ok                   ; si lo es, saltar a .ok
    sub al, 39                ; si es letra, lo ajusta

.ok:
    mov edi, eax              ; copia resultado en EDI
    xor eax, eax              ; inicializa EAX en 0, por consiguiente RAX en 0
    mov al, 60                ; inicializa EAX en 60 -> syscall: exit
    syscall                   ; finaliza el proceso con el código en RDI
```
</details>

#### Métricas

```
Opcodes: 8A 05 FA 0F 00 00 2C 30 3C 09 76 02 2C 27 89 C7 31 C0 B0 3C 0F 05
Bytes..: 22
```

```
Instrucciones ejecutadas: 9
```

#### Comparativa

| Versión  | Tamaño (bytes) | Instrucciones ejecutadas |
|----------|:--------------:|:------------------------:|
| Original |             49 |                       11 |
| Última   |             33 |                       11 |
| Hardcore |             22 |                        9 |

#### Resumen

La version **hardcore** se redujo en **27 bytes** con respecto a la versión original, lo que nos da una reducción del **55,10 %** en el tamaño del código.

## Explicación del código

Para entender mejor el flujo del programa, a continuación se muestra una tabla con los valores ASCII y sus equivalentes en decimal y hexadecimal:

| Carácter | ASCII | Hexadecimal | Decimal |
|:--------:|:-----:|:-----------:|:-------:|
|      '0' |    48 |           0 |       0 |
|      '1' |    49 |           1 |       1 |
|      '2' |    50 |           2 |       2 |
|      '3' |    51 |           3 |       3 |
|      '4' |    52 |           4 |       4 |
|      '5' |    53 |           5 |       5 |
|      '6' |    54 |           6 |       6 |
|      '7' |    55 |           7 |       7 |
|      '8' |    56 |           8 |       8 |
|      '9' |    57 |           9 |       9 |
|      'a' |    97 |           a |      10 |
|      'b' |    98 |           b |      11 |
|      'c' |    99 |           c |      12 |
|      'd' |   100 |           d |      13 |
|      'e' |   101 |           e |      14 |
|      'f' |   102 |           f |      15 |

```nasm
sub al, '0'
```
Esta instrucción ajusta la base ASCII al restar el valor de `'0'` (ASCII 48) del contenido de `AL`. Con esto, si `AL` contenía `'7'` (ASCII 55), al restarle `'0'` (ASCII 48), el resultado es 7. Ya no tenemos el carácter `'7'`, sino el número 7 dentro de `AL`, lo que facilita operaciones aritméticas posteriores. De la misma manera, cualquier dígito `'0'–'9'` se convierte en su valor decimal correspondiente (0–9).

```nasm
sub al, 39
```
Resta el número decimal **39** al contenido de `AL`. Se utliza cuando el carácter **no está** en el rango `'0'–'9'` (es decir, para `'a'–'f'`). De este modo, si `AL` contenía el valor resultante de `'c' – '0'`, que es 51 (99 - 48), entonces al restar **39** se obtiene **12**, el valor decimal asociado a `'c'`. En términos generales, esto convierte las letras `'a'–'f'` en lo valores numéricos `10–15`.

## Referencias

En la [página oficial de Intel](https://www.intel.com/content/www/us/en/developer/articles/technical/intel-sdm.html) se encuentran los manuales técnicos que abarcan todos los detalles de la **arquitectura x86-64**.

**Nota:** Revisar el sitio periódicamente, ya que los manuales suelen actualizarse con frecuencia.
