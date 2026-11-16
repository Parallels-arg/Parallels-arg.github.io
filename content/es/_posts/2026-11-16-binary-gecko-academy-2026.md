---
title: "Binary Gecko Academy 2026: el challenge de clasificación"
description: "Cuatro niveles encadenados en un crackme MinGW: hashes del usuario, RC4, anti-debugging y una VM de bytecode, resueltos sin parchear y con un keygen por nivel."
date: 2026-11-16 12:00:00 -0300
categories: [Writeups, Reversing]
tags: [crackme, windows, keygen, x64dbg, binary-ninja, anti-debugging, vm]
translated: true
media_subpath: /assets/img/posts/binary-gecko-academy-2026
---

Este es el writeup que mandé para el challenge de clasificación de la Academy 2026 de [Binary Gecko](https://binarygecko.com/), publicado ahora que terminó el proceso de selección. El análisis es el mismo que envié (originalmente en inglés); para el blog pasé las capturas de mis scripts a bloques de código que se pueden copiar y dejé afuera el binario del challenge.

## Resumen

`GeckoAcademy2026_7_challenge.exe` es un programa de consola de 64 bits compilado con MinGW, con cuatro niveles encadenados y chequeos anti-debugging en el camino. Resolví los cuatro sin parchear el binario y escribí un keygen en Python para cada uno:

| Nivel | Qué chequea | Lo que ingresé |
| --- | --- | --- |
| 1 | Un serial de 8 dígitos hex que tiene que coincidir con un hash del usuario | Usuario `Bruno`, serial `0038D6D8` |
| 2 | Un serial XXXX-XXXX-XXXX-YYYY: tres hashes de 16 bits del mismo usuario con semillas distintas, más el XOR de los tres | `D31B-84B1-76D4-217E` |
| 3 | Un login: la contraseña se arma a partir de un usuario nuevo y una clave de 8 bytes descifrada con RC4 | Usuario `Bruno`, contraseña `AFB0685485B42B8B` |
| 4 | Una contraseña de 12 caracteres validada por una pequeña VM de bytecode. Hay muchas contraseñas posibles. | `Bruno_W!59*.` |

Código de finalización: `937F16DE-EB8C1201`

El código de finalización depende de todo lo que se ingresó en la corrida, así que solo es válido para los datos de arriba (los detalles están en la sección [el código de finalización](#codigo-de-finalizacion)).

## Primer análisis

Bajé el challenge por primera vez el 21 de septiembre. El zip (contraseña `a`) traía `GeckoAcademy2026_7_challenge.exe` y un `notas.txt`:

![Notepad++ mostrando notas.txt de la primera descarga](img-001.webp){: w="886" h="253" }

Cuando lo volví a bajar el 22 de septiembre, la nota había sido reemplazada por `Qualifying requirements.txt`, así que parece que la nota original fue un descuido de los organizadores, ¡ups! El ejecutable también tenía otra fecha de archivo, así que comparé los hashes SHA-256 para ver si el binario en sí había cambiado:

![Hashes SHA-256 de los ejecutables de las dos primeras descargas](img-002.webp){: w="1880" h="248" }

Las dos copias tenían el mismo SHA-256: `C7868C66050F75AE9BAE7E8116D74BF04BA8B5A1C5D0A1B9CD0B7457BB02ED2F`. Una tercera descarga, el 14 de octubre, también coincidía:

![Hash SHA-256 de la tercera descarga, igual a los anteriores](img-003.webp){: w="1880" h="267" }

Los tres ejecutables eran idénticos. Para el análisis usé la copia de la segunda descarga.

### Reglas de juego

Antes de empezar me puse algunas reglas:

1. **Entender a fondo**. Este año agarré el challenge temprano y hay tiempo de sobra hasta la fecha límite, así que me lo voy a tomar con calma. En cada nivel voy a intentar entender del todo qué hace el chequeo, documentarlo y encontrar todas las entradas válidas, no solo una que justo pase.
1. **Nada de parchear como solución**. El año pasado NOPeé algunas instrucciones para esquivar un crash del ejecutable del challenge, sin llegar a entender nunca por qué se producía. Este año no voy a tocar el binario, y si algo crashea o se me cruza en el camino, el objetivo es entender por qué.
1. **Confirmar con análisis dinámico**. Primero leo el código en el desensamblador y después uso el debugger para comprobar lo que supongo.
1. **Entorno**. Windows 11 x64. Dada la reputación de los organizadores, me siento cómodo corriendo el ejecutable directo en mi máquina, sin VM ni otras medidas de seguridad.

### Herramientas

| Herramienta | Para qué la usé |
| --- | --- |
| Detect It Easy 3.21 | Identificación inicial del archivo |
| PEStudio 9.61 | Identificación inicial del archivo |
| Binary Ninja 6.0 | Desensamblador / decompilador principal |
| Calltree (plugin de Binary Ninja) | Árboles de llamadores y llamados para ver el panorama general del programa |
| x64dbg | Análisis dinámico |
| Python 3.13 | Keygens y scripts de la solución |

### Documentación en la que me apoyé

| Fuente | Para qué la usé |
| --- | --- |
| [mingw-w64 CRT source, stdio/](https://mingw.googlesource.com/mingw-w64/+/refs/heads/master/mingw-w64-crt/stdio/) | Las implementaciones propias de MinGW-w64 de las funciones stdio de C (printf, sscanf, snprintf...), para identificarlas en el binario |
| [Stack Overflow: Using msvcrt with Mingw-w64](https://stackoverflow.com/questions/64158676/using-msvcrt-with-mingw-w64) | Por qué MinGW-w64 usa sus propias implementaciones en lugar de las de msvcrt.dll (msvcrt.dll solo cubre C89, GCC compila como C11) |
| [Binary Ninja docs: basic types](https://docs.binary.ninja/guide/types/basictypes.html) | Cambiar el tipo de una función |
| [binaryninja-printf](https://github.com/nshp/binaryninja-printf) | Confirmar que Binary Ninja necesita ayuda con las funciones variádicas tipo printf |
| [cppreference: scanf](https://en.cppreference.com/w/c/io/fscanf) | Cómo parsea `%x` un número hex (espacios, prefijo 0x, dónde corta) |
| [Microsoft Learn: chcp](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/chcp) | La página de códigos de la consola, y por qué las letras con tilde llegan al binario en cp850 |
| [Microsoft Learn: GetOEMCP](https://learn.microsoft.com/en-us/windows/win32/api/winnls/nf-winnls-getoemcp) | La página de códigos OEM del sistema que usó mi primer arreglo, que no es necesariamente la página de entrada de la consola |
| [Microsoft Learn: GetConsoleCP](https://learn.microsoft.com/en-us/windows/console/getconsolecp) | Leer la página de códigos de entrada actual de la consola en los keygens |
| [Wikipedia: RC4](https://en.wikipedia.org/wiki/RC4) | Reconocer el key scheduling y el generador de RC4 en el nivel 3 |
| [x64dbg docs: HideDebugger](https://help.x64dbg.com/en/latest/commands/misc/HideDebugger.html) | Ocultar x64dbg de IsDebuggerPresent en el nivel 3 |
| [Cracking a virtualized binary with Angr](https://noh.am/en/posts/vm/) | Cómo funciona una VM dentro de un crackme (bytecode, dispatcher, handlers) y la forma manual de revertirla, para el nivel 4 |
| [x64dbg docs: conditional breakpoints](https://help.x64dbg.com/en/latest/introduction/ConditionalBreakpoint.html) | Breakpoints que escriben una línea en el log en vez de frenar, para trazar la VM del nivel 4 |
| [x64dbg docs: string formatting](https://help.x64dbg.com/en/latest/introduction/Formatting.html) | La sintaxis `{expresión}` del texto del log, que por defecto imprime en hex |
| [x64dbg docs: values](https://help.x64dbg.com/en/latest/introduction/Values.html) | Leer memoria en una expresión con `[addr]` y `dword:[addr]` |
| [Baeldung: meet-in-the-middle attack](https://www.baeldung.com/cs/security-meet-in-the-middle-attack) | El truco para encontrar contraseñas del nivel 4, yendo hacia adelante y hacia atrás hasta encontrarse en el medio |
| [Wikipedia: ASCII printable characters](https://en.wikipedia.org/wiki/ASCII) | El rango de caracteres que se pueden tipear en una contraseña del nivel 4 (0x21 a 0x7E) |
| [Wikipedia: Fowler–Noll–Vo hash function](https://en.wikipedia.org/wiki/Fowler%E2%80%93Noll%E2%80%93Vo_hash_function) | Reconocer las constantes de FNV-1a (0x811c9dc5, 0x1000193) en el código de finalización |

### Análisis básico

Primera parada, Detect It Easy:

![Detect It Easy: PE de consola de 64 bits, MinGW GCC 13, C](img-004.webp){: w="727" h="532" }

Un PE de consola de 64 bits, pero esta vez compilado con `MinGW (GCC 13)`, lenguaje C, y **stripped**.

Antes de seguir revisé la entropía, para asegurarme de no estar ante un binario empaquetado:

![Vista de entropía de Detect It Easy, 6.16 y sin empaquetar](img-005.webp){: w="2560" h="1032" }

6,16 en total, y DIE dice "not packed". Sección por sección, la más alta es `.text` con 6,27. Nada se acerca a 7-8, que es lo esperable para datos comprimidos o cifrados.

Después, las secciones:

![Tabla de secciones del binario](img-006.webp){: w="1566" h="269" }

Una búsqueda rápida en Google mostró que es la distribución de secciones estándar de un binario compilado con MinGW, así que no hay nada agregado a mano.

El binario también tiene **TLS callbacks**, dos, y los dos apuntan a `.text`:

![Detect It Easy listando dos TLS callbacks](img-007.webp){: w="2560" h="1032" }

Anoté `0x140002960` y `0x140002930` para revisarlos después en el desensamblador.

PEStudio confirma lo que mostró DIE y además da la fecha de compilación:

![Resumen de PEStudio con la fecha de compilación](img-008.webp){: w="1234" h="741" }

### Imports

Solo dos librerías: `KERNEL32.dll` y `msvcrt.dll`. Los imports de KERNEL32 son donde se pone interesante:

![Imports de KERNEL32.dll en PEStudio, con las APIs anti-debugging marcadas](img-009.webp){: w="854" h="405" }

- `IsDebuggerPresent` y `CheckRemoteDebuggerPresent`: los nombres lo dicen todo, chequeos de debugger. Muy en línea con el challenge del año pasado.
- `GetThreadContext` junto con `GetCurrentThread`: buscando, es una forma de que un programa lea sus propios registros de debug (DR0-DR3), que es donde viven los breakpoints por hardware.
- `QueryPerformanceCounter` / `QueryPerformanceFrequency`: lo más probable es que también sean para detectar el debugger, ya que el challenge del año pasado también tenía chequeos de tiempo.
- `VirtualProtect` y `VirtualQuery` aparecen marcados, pero todavía no puedo decir mucho de ellos.

Del lado de `msvcrt.dll` está el runtime de C:

![Imports de msvcrt.dll en PEStudio](img-010.webp){: w="854" h="781" }

### Strings

PEStudio encontró 1129 strings, la mayoría ruido:

![Strings en PEStudio con los banners de los niveles y los mensajes anti-debugging](img-011.webp){: w="1178" h="781" }

Mirándolos, se ve que el challenge tiene 4 niveles, además de mensajes anti-debugger, strings de formato y la existencia de un "completion code".

Con esto ya tengo una buena idea de a qué me enfrento: 4 niveles en C, uno o más chequeos anti-debugging, dos TLS callbacks pendientes de revisar y un código de finalización. Hora de abrirlo en Binary Ninja.

## Orientándome

### Preparando Binary Ninja

Binary Ninja encontró `main` solo, apenas terminó el análisis, lo cual es un buen comienzo. Pero antes de leer cualquier nivel quería una forma de ver todo el programa de un vistazo: quién llama a quién, qué helpers comparten los niveles, etcétera. Para eso instalé **Calltree** desde el Extension Manager. Agrega una barra lateral con las llamadas entrantes y salientes de la función actual, y un botón que convierte esos árboles en un grafo:

![Binary Ninja con la barra lateral de Calltree](img-012.webp){: w="1282" h="832" }

### Un primer vistazo a `main`

Así se veía `main` recién abierto el binario:

![main decompilado antes de renombrar, con llamadas anidadas](img-013.webp){: w="1282" h="832" }

La estructura ya está ahí. Algo de inicialización, un banner, cuatro llamadas (las dos primeras comparten el mismo `var_88`) y una última función que imprime el código de finalización. Pero las llamadas a `sub_14000fb30` terminan anidadas unas dentro de otras, e incluso dentro de los argumentos del primer nivel, lo cual no tiene sentido para un banner.

Cada línea del banner pasa por la misma función, `sub_14000fb30`. Al hacer clic en ella se resaltan todos sus usos, y el panel de Cross References muestra que se llama 40 veces en todo el programa, siempre con un string como primer argumento: el banner, `"Username: "`, `"Serial (8 hex digits): "`, `"\n[!] %s\n"`, y algunos tienen especificadores de formato como `%s` o `%08X`:

![Referencias cruzadas de sub_14000fb30, siempre llamada con un string de formato](img-014.webp){: w="1210" h="638" }

Eso parece algún tipo de printf, así que la abrí:

![sub_14000fb30 y los dos wrappers que tiene debajo](img-015.webp){: w="670" h="459" }

Guarda los argumentos extra en el stack y llama a otra función con `data_140010100(1)`, el string de formato y un puntero a los argumentos. Las dos funciones que tiene justo debajo tienen exactamente la misma forma: reciben un formato más una cantidad variable de argumentos y le pasan todo a otra función. Así que parecía toda una familia de funciones stdio, pero ninguna es un import. Volviendo a los imports de `msvcrt.dll` en PEStudio, los únicos de esa familia eran `fprintf` y `vfprintf`.

Buscando, descubrí que MinGW-w64 trae sus propias implementaciones de las funciones stdio de C, que se linkean dentro del binario en lugar de usar las de `msvcrt.dll` (el código está en el [mingw-w64 CRT, stdio/](https://mingw.googlesource.com/mingw-w64/+/refs/heads/master/mingw-w64-crt/stdio/)). Con el código a mano, distinguirlas fue cuestión de comparar cada wrapper con cómo se usa en el binario:

- `sub_14000fb30` es `printf`: todos los mensajes de consola pasan por ella, y como la versión de mingw-w64, solo le pasa un stream (`data_140010100(1)`), el formato y los argumentos a la función que hace el formateo de verdad. MinGW-w64 usa su propia versión porque la de `msvcrt.dll` no sigue del todo el estándar C99, así que su [stdio.h](https://mingw.googlesource.com/mingw-w64/+/refs/heads/master/mingw-w64-headers/crt/stdio.h) redirige `printf` a `__mingw_printf` cuando se pide conformidad con C99.
- `sub_14000fb90` es `sscanf`. En el primer nivel, justo después de leer el serial, se la llama con lo que ingresó el usuario, `"%8x"` y un puntero a una variable, y sus referencias cruzadas muestran que también se usa con `"%4x-%4x-%4x-%4x"`. El `sscanf` de mingw-w64 hace exactamente eso: le pasa el string de entrada, el formato y los argumentos a `__mingw_vsscanf`. Está ahí por la misma razón: el mismo `stdio.h` redirige `sscanf` a `__mingw_sscanf` para que los formatos de entrada también sigan C99 en lugar del comportamiento de `msvcrt.dll`.

![sscanf identificada en Binary Ninja](img-016.webp){: w="1686" h="923" }

- `sub_14000fbe0` es `snprintf`: sus referencias cruzadas muestran que recibe un buffer, un tamaño (`0x20`) y `"%08X"`, y el `snprintf` de mingw-w64 le pasa el buffer, su tamaño, el formato y los argumentos a `__vsnprintf`. La misma historia que las otras dos: `stdio.h` redirige `snprintf` a `__mingw_snprintf` para que su salida siga C99 en lugar del comportamiento de `msvcrt.dll`.

![snprintf identificada en Binary Ninja](img-017.webp){: w="1686" h="923" }

Aun después de identificar estas funciones, seguía sin entender por qué `main` se veía anidado. Investigando un poco, descubrí que a Binary Ninja hay que avisarle que una función recibe una cantidad variable de argumentos. En un binario stripped solo puede adivinar cuántos argumentos tiene cada llamada, y con estas funciones adivinó mal, así que terminó tratando algunas llamadas como argumentos de otras. Es tan común que hasta existe un plugin, [binaryninja-printf](https://github.com/nshp/binaryninja-printf), cuyo único trabajo es anotar funciones variádicas tipo printf, incluidas las que vienen linkeadas estáticamente en el binario como acá. Con solo tres funciones lo hice a mano: las renombré y les cambié el tipo con `y` ([Binary Ninja docs](https://docs.binary.ninja/guide/types/basictypes.html)) a `int32_t printf(char const* fmt, ...)` y similares, y el anidamiento desapareció. Después recorrí las cuatro llamadas de `main` para confirmar qué era cada una antes de nombrarlas:

- `sub_140001877` imprime *"LEVEL 1 / 4 - Difficulty: MEDIUM"*.
- `sub_140001ae5` no imprime su título directamente, pero dice *"using the same username from Level 1"* y pide un serial XXXX-XXXX-XXXX-YYYY, así que claramente es el nivel 2.
- `sub_140001fca` imprime *"LEVEL 3 / 4 - Difficulty: HARD (login)"*.
- `sub_140002468` imprime *"LEVEL 4 / 4 - Difficulty: VERY HARD (final)"*.
- `sub_14000262e` imprime *"CHALLENGE COMPLETE"* y el código de finalización.

Después de renombrar, `main` se lee mucho mejor:

![main después de renombrar las funciones de stdio y los niveles](img-018.webp){: w="1686" h="923" }

Igual había algo que me seguía molestando. El banner dice que los niveles están encadenados, pero en el `main` decompilado nada chequea lo que devuelve cada nivel. Solo llama a los cuatro y devuelve 0, así que decidí revisar el código en modo desensamblado:

![Desensamblado de main, que chequea el valor de retorno después de cada nivel](img-019.webp){: w="1090" h="1052" }

Después de cada `call level_N` hay un `xor eax, 0x1` / `test al, al` / `je`: si un nivel devuelve cualquier cosa distinta de 1, `main` sale con `eax = 1`. Así que los chequeos están, el decompilador simplemente los simplificó. Bueno saberlo antes de confiar a ciegas en el decompilador dentro de los niveles.

También le eché un vistazo rápido a la primera llamada, `sub_140002900`, ya que corre antes que cualquier nivel. Recorre una pequeña tabla de punteros a funciones y llama a cada una, que es simplemente código de inicialización sin nada que ver con el challenge, así que la llamé `run_init_table` y seguí.

### Los TLS callbacks

De vuelta a los dos TLS callbacks que anoté en el análisis básico:

![Los dos TLS callbacks en Binary Ninja](img-020.webp){: w="1686" h="923" }

Los dos terminan en `sub_140003400`, y Calltree muestra que solo baja un par de llamadas más, a `free` y a dos funciones chicas. Las recorrí por arriba y nada me llamó la atención. No hay APIs anti-debugging ni nada relacionado con los niveles, así que lo que sea que haga el binario contra los debuggers, no pasa antes de `main`.

### ¿Dónde están los chequeos anti-debugging?

Para encontrarlos fui al revés, empezando por los imports. Seleccionar cada entrada de la tabla de imports muestra dónde se usa:

![Entradas de la tabla de imports y dónde se usa cada una](img-021.webp){: w="1686" h="923" }

Cada API anti-debugging se llama desde exactamente una función chica. Las nombré según lo que hacen:

![Los helpers anti-debugging después de renombrarlos](img-022.webp){: w="1686" h="923" }

- `check_debugger`: solo `IsDebuggerPresent()`.
- `check_remote_debugger`: `CheckRemoteDebuggerPresent()` sobre su propio proceso.
- `check_hw_breakpoints`: obtiene el contexto del thread actual y devuelve 1 si alguno de Dr0-Dr3 está seteado.

![check_hw_breakpoints leyendo Dr0 a Dr3](img-023.webp){: w="1686" h="923" }

- `check_timing`: toma un `QueryPerformanceCounter` antes y después de un loop de `0x30d40` iteraciones, y devuelve 1 si tardó más de 250 ms. Un loop así debería tardar muchísimo menos, así que solo debería saltar si alguien está ejecutando paso a paso (o tiene un breakpoint) adentro.

Cuando alguno devuelve true, el nivel llama a otra función chica que imprime `"\n[!] %s\n"` con el mensaje y llama a `exit(1)`. La llamé `die`. El comienzo de `level_4` muestra el patrón completo:

![Comienzo de level_4 llamando a los chequeos y a die](img-024.webp){: w="1686" h="923" }

Con las referencias cruzadas de cada helper armé qué nivel usa qué chequeo:

| Nivel | `check_debugger` | `check_remote_debugger` | `check_hw_breakpoints` | `check_timing` |
| --- | --- | --- | --- | --- |
| 1 |  |  |  |  |
| 2 | SÍ |  |  |  |
| 3 | SÍ |  | SÍ | SÍ |
| 4 | SÍ | SÍ | SÍ | SÍ |

El nivel 1 no tiene ningún chequeo, mientras que el nivel 4 corre los cuatro antes de imprimir siquiera su banner. O sea que el debugger va a ser cada vez menos bienvenido a medida que avance.

### Otros helpers

Mientras recorría los niveles para nombrarlos, aparecían una y otra vez algunas funciones chicas:

| Función | Qué hace | Nombre |
| --- | --- | --- |
| `sub_1400015bf` | `fgets` desde `stdin` y corta el string en el primer `\r` o `\n` | `read_line` |
| `sub_140001555` | Hace XOR de cada byte de un buffer con una clave y lo copia a un destino | `xor_decode` |
| `sub_140001480` | Binary Ninja ya la muestra como un ROLD, una rotación a la izquierda | `rol32` |

Dos de ellas conviene tenerlas presentes para más adelante:

- `read_line` es como todos los niveles leen su entrada, siempre en un buffer de `0x80` bytes. Como `fgets` lee un carácter menos que el tamaño, todo lo que pase de 127 caracteres se corta, y el string termina en el primer salto de línea. Eso va a importar a la hora de buscar casos borde.
- `rol32` no la usa solo `check_timing`. Sus referencias cruzadas la muestran en `sub_140001801` (a la que `level_1` llama con el usuario), en dos de las funciones que cuelgan de `level_3` y en `sub_140002161`, la función a la que `level_4` le pasa la contraseña. Así que las rotaciones van a aparecer bastante.

`xor_decode` explica el título faltante del nivel 2. `level_2` la llama dos veces con la clave `0x5a`: una para decodificar su banner (que imprime enseguida con `" %s\n"`) y otra para decodificar el mensaje que le pasa a `die` si hay un debugger. Por eso no aparecía *"LEVEL 2 / 4"* entre los strings.

![level_2 decodificando sus strings con xor_decode](img-025.webp){: w="1686" h="923" }

Con la clave y los datos ahí mismo, no hacía falta ejecutar nada para leerlos: copié los bytes de los dos buffers desde Binary Ninja y les hice XOR con `0x5a` en un script de Python cortito.

```python
# level 2 strings, xor key 0x5a
title = bytes.fromhex("1b191b1e1f17037a081f770d13147a60607a163f2c3f367a68")
msg = bytes.fromhex("1e3f382f3d3d33343d7a3f342c33283534373f342e7a3e3f2e3f392e3f3e")

for buf in (title, msg):
    print(bytes(b ^ 0x5a for b in buf).decode())
```
{: file="level2_strings.py" }

```text
ACADEMY RE-WIN :: Level 2
Debugging environment detected
```

El título del nivel 2 es *"ACADEMY RE-WIN :: Level 2"*, y el mensaje con el que muere `level_2` cuando encuentra un debugger es *"Debugging environment detected"*.

### El mapa

Con todo renombrado, esta es la foto final que tengo antes de empezar con los niveles, el grafo de Calltree de `main`:

![Grafo de Calltree de main con todo renombrado](img-027.webp){: w="1297" h="876" }

Los cuatro niveles cuelgan de `main`, y los helpers compartidos quedan debajo, con flechas que llegan desde varios niveles. Hora de empezar con el nivel 1.

## Nivel 1

### Primer vistazo

El nivel 1 es bastante corto:

![level_1 decompilado](img-028.webp){: w="823" h="764" }

Después del banner, `level_1` entra en un loop infinito pidiendo dos cosas, ambas leídas con `read_line` en buffers de `0x80` bytes: un usuario en `_Source` y un serial, `"8 hex digits"`, en `var_118`.

![level_1 leyendo el usuario y el serial](img-029.webp){: w="636" h="98" }

Justo después viene el chequeo en sí:

![El chequeo del serial en level_1](img-030.webp){: w="767" h="322" }

Llama a `sub_140001801` con el usuario, toma el `strlen` del serial y, solo si tiene exactamente 8 caracteres, lo parsea con `sscanf(&var_118, "%8x", &var_11c)`. El loop solo se corta cuando se cumplen las tres condiciones:

- el serial tiene 8 caracteres (`rax_2 == 8`),
- sscanf pudo leer un número hex (`rax_3 == 1`),
- y ese valor es igual a lo que devolvió `sub_140001801` para el usuario (`rax_1 == var_11c`).

Si no, imprime *"Invalid serial. Try again."* y vuelve a pedir los dos. No hay contador de intentos ni anti-debugging, lo que coincide con la tabla que armé en la sección anterior.

![level_1 copiando el usuario y guardando el hash](img-031.webp){: w="669" h="98" }

Cuando el chequeo pasa, copia el usuario en `arg1`, un buffer que recibe de `main`, guarda el valor de `sub_140001801` en la global `data_140015040` y devuelve 1. La copia explica por qué el nivel 2 decía que usaba *"the same username from Level 1"*.

Así que el serial es simplemente un número calculado a partir del usuario, escrito en hex. Todo depende de `sub_140001801`.

### El hash del usuario

![sub_140001801, el hash del usuario](img-032.webp){: w="823" h="220" }

Es corta y la salida del decompilador se lee casi como C plano. Recorre el usuario y, por cada carácter, suma su valor (casteado a `uint8_t`) multiplicado por su posición empezando en 1. Después le hace XOR a la suma de 32 bits con `0xc001c0de` (¿cool code?) y rota el resultado 5 bits a la izquierda con `rol32`, el helper que ya había encontrado en la sección anterior:

`hash = rol32( sum(name[i] * (i + 1)) ^ 0xc001c0de, 5 )`

Como es un hash del usuario, la renombré `username_hash`.

Dos detalles que anoté para los casos borde:

- Nada valida el usuario: no hay largo mínimo ni conjunto de caracteres permitido. Incluso un usuario vacío debería funcionar: la suma queda en 0, `0 ^ 0xc001c0de` es simplemente `0xc001c0de`, y rotarlo 5 a la izquierda da `0x00381BD8`, así que el serial sería `00381BD8`.
- Cada carácter se lee como un byte sin signo (`uint8_t`), un número de 0 a 255. Para letras y dígitos comunes es su código ASCII, pero no estoy seguro de qué pasaría con letras que no son ASCII plano, como á o ñ, así que es algo para probar en ejecución.

Con el hash entendido, renombré el resto de las variables de `level_1`:

![level_1 con sus variables renombradas](img-033.webp){: w="823" h="743" }

Quedaba una cosa suelta: justo antes de volver, `level_1` guarda el hash en una variable global, `data_140015040`, y nada en el nivel 1 lo vuelve a leer.

![Referencias cruzadas de data_140015040](img-034.webp){: w="731" h="153" }

Las referencias cruzadas muestran que solo dos lugares la tocan: `level_1` la escribe y `print_completion_code` la lee con `snprintf(&buf, 0x20, "%08X", ...)`. O sea que el hash del nivel 1 termina siendo parte del código de finalización que se imprime al final. Renombré la global `level1_hash`.

A esta altura el nivel 1 se reduce a calcular `username_hash` para cualquier usuario y escribirlo como 8 dígitos hex, así que el paso siguiente es un keygen.

### Keygen

El keygen es simplemente `username_hash` reescrita en Python: la misma suma, el mismo XOR y la misma rotación. Lo escribí como un pequeño programa de consola, `keygen1.py`, que pide el usuario e imprime el serial:

```python
def rol32(x, n):
    return ((x << n) | (x >> (32 - n))) & 0xffffffff

def lvl1(name):
    s = 0
    for i, c in enumerate(name):
        s += c * (i + 1)
    return rol32(s ^ 0xc001c0de, 5)

name = input("Username: ")
print("Serial: %08X" % lvl1(name.encode()))
input("Press Enter to exit")
```
{: file="keygen1.py" }

Corriéndolo con mi nombre:

```text
Username: Bruno
Serial: 0038D6D8
Press Enter to exit
```

Para probarlo contra el binario primero ingresé un serial cualquiera, para ver el error, y después el del keygen:

![El nivel 1 rechaza 12345678 y acepta 0038D6D8](img-037.webp){: w="965" h="380" }

`12345678` es rechazado y el loop vuelve a preguntar, mientras que `0038D6D8` pasa y el binario sigue al nivel 2, que ya muestra *"Bruno"* como el usuario que tomó del nivel 1.

### Casos borde

Con un keygen funcionando volví a las notas que había tomado mientras leía el código.

#### Usuario vacío

Nada lo impide, así que apreté Enter en el prompt del usuario y usé `00381BD8`, el serial que había calculado a mano para el nombre vacío (el keygen da lo mismo si solo apretás Enter):

![El nivel 1 pasa con usuario vacío y 00381BD8](img-038.webp){: w="983" h="166" }

Pasa, y el nivel 2 muestra un usuario vacío entre los paréntesis.

#### ¿Qué tan estricto es `"%8x"`?

El chequeo de largo solo cuenta caracteres, y el número en sí lo lee sscanf. Según la [referencia de scanf](https://en.cppreference.com/w/c/io/fscanf), `%x` saltea los espacios del principio, acepta el prefijo `0x` y deja de leer en el primer carácter que no es hex, así que lo probé. Mi serial, `0038D6D8`, empieza con dos ceros, así que puedo sacarlos y usar esos dos caracteres para otra cosa. Cada uno de estos tiene 8 caracteres y pasó:

- `0x38d6d8`, con el prefijo `0x` (y en minúsculas):

![El nivel 1 acepta 0x38d6d8](img-039.webp){: w="983" h="70" }

- `38D6D8`, con dos espacios adelante:

![El nivel 1 acepta el serial con dos espacios adelante](img-040.webp){: w="983" h="70" }

- `38D6D8zz`, con basura al final:

![El nivel 1 acepta 38D6D8zz](img-041.webp){: w="983" h="70" }

Así que, para cualquier hash que empiece con 00, hay varios seriales válidos además del "oficial". Con un hash que usa los 8 dígitos, solo la forma plana entra en 8 caracteres (más las variantes de mayúsculas y minúsculas).

#### Usuarios de más de 127 caracteres

Acá volví a `read_line`:

![read_line decompilada](img-042.webp){: w="823" h="352" }

Llama a `fgets` con el tamaño que recibe (`0x80`) y después corta el string en el primer `\r` o `\n`. Si la línea tiene más de 127 caracteres, `fgets` corta ahí y el resto de la línea queda en el buffer de entrada, y nada lo limpia. Así que el siguiente `read_line`, el del serial, recibe lo que sobró sin esperarme. Para probarlo corrí el keygen con 127 letras A, que da `0139E3D8`, y pegué esas 127 A seguidas directamente de ese serial, todo en una línea (`AAAA...AAAA0139E3D8`). Los primeros 127 caracteres pasan a ser el usuario y los 8 restantes son lo que `read_line` devuelve como serial:

![127 letras A seguidas del serial en una sola línea, y el nivel 1 pasa](img-043.webp){: w="983" h="183" }

El binario imprime el prompt del serial pero no espera, porque el serial ya estaba ahí, y el nivel pasa. El nivel 2 también muestra las 127 A como usuario. Lo mismo pasaría con cualquier usuario de más de 127 caracteres: los caracteres de más se convierten en el serial, lo que la mayoría de las veces solo significa un *"Invalid serial"*.

#### Letras con tilde

Para `Brunó` el keygen me dio `00399C18`:

```text
Username: Brunó
Serial: 00399C18
Press Enter to exit
```

y el binario lo rechazó. Mi keygen codificaba el nombre con el default de Python, UTF-8, donde la ó ocupa dos bytes. La consola que usé para estas pruebas estaba en la página de códigos 850 ([docs de chcp](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/chcp)), donde la ó es un solo byte, `0xA2`. Codificando el nombre con cp850 da `0038F738`, que pasó:

![Brunó rechazado con el serial de UTF-8 y aceptado con el de cp850](img-045.webp){: w="983" h="147" }

O sea que el serial de un nombre con tildes depende de la página de códigos de la consola donde se tipea. Mi primer arreglo usaba [GetOEMCP](https://learn.microsoft.com/en-us/windows/win32/api/winnls/nf-winnls-getoemcp), llamada desde Python con ctypes, para no dejar cp850 fijo en el código. Eso daba el resultado correcto acá, pero cerca del final del challenge me di cuenta, de casualidad, de que había confundido la página de códigos OEM del sistema con la página de entrada de la consola. No sabía que pueden ser distintas, por ejemplo después de cambiar la página de la consola con `chcp`.

Cuando lo revisé, la documentación de [GetConsoleCP](https://learn.microsoft.com/en-us/windows/console/getconsolecp) me llevó a la página de códigos de entrada de la consola asociada al proceso que la llama. Cambié los cuatro keygens a eso. Corro cada keygen y el challenge en la misma consola, sin cambiar su página de códigos entre uno y otro, así que el keygen codifica la entrada con la misma página que recibe el challenge. Esta es la versión corregida:

```python
import ctypes

def rol32(x, n):
    return ((x << n) | (x >> (32 - n))) & 0xffffffff

def lvl1(name):
    s = 0
    for i, c in enumerate(name):
        s += c * (i + 1)
    return rol32(s ^ 0xc001c0de, 5)

# input code page of the current console
cp = "cp%d" % ctypes.windll.kernel32.GetConsoleCP()

name = input("Username: ")
print("Serial: %08X" % lvl1(name.encode(cp)))
input("Press Enter to exit")
```
{: file="keygen1.py" }

Con la consola todavía en cp850, el serial para `Brunó` sigue siendo el mismo:

```text
Username: Brunó
Serial: 0038F738
Press Enter to exit
```

Nivel 1 terminado. Sigue el nivel 2, que reutiliza este usuario.

## Nivel 2

### Primer vistazo

El comienzo de `level_2` es la parte que ya había visto mientras buscaba su título faltante:

![Comienzo de level_2](img-048.webp){: w="1216" h="345" }

Decodifica sus dos strings con `xor_decode` y después llama a `check_debugger`. Si hay un debugger, muere con *"Debugging environment detected"*, así que es el primer nivel que no quiere correr bajo un debugger, aunque por ahora solo usa `IsDebuggerPresent`. Después imprime el título decodificado y el usuario, que llega en `arg1` y es el mismo buffer que `main` le pasó a `level_1`, donde el nivel 1 copió el usuario al final.

Después viene el loop:

![Loop del serial de level_2](img-049.webp){: w="1216" h="535" }

- Lee el serial con `read_line`, de nuevo en un buffer de `0x80` bytes.
- `sscanf` lo parsea con `"%4x-%4x-%4x-%4x"` y tiene que devolver 4, así que el serial son cuatro grupos de hasta 4 dígitos hex separados por guiones.
- Cada valor se recorta a 16 bits (`int16_t`) y se guarda en cuatro variables locales que están una al lado de la otra: `var_120`, `var_11e_1`, `var_11c_1` y `var_11a_1`. Eso es un array de cuatro palabras de 16 bits, y su dirección es lo que se le pasa a `sub_140001a34`, junto con el usuario.
- Si `sub_140001a34` devuelve algo distinto de cero, el nivel pasa, la primera palabra se guarda en la global `data_140015044` y devuelve 1. Si no, imprime *"Invalid serial. Try again."* y vuelve a pedir solo el serial, ya que el usuario quedó fijo desde el nivel 1.

### El chequeo del serial

![sub_140001a34 comparando los cuatro grupos](img-050.webp){: w="1216" h="251" }

`sub_140001a34` llama tres veces a `sub_1400019b1` con el usuario y una constante de 16 bits distinta cada vez: `0x1234`, `0xabcd` y `0x7777`. El serial es válido cuando los tres primeros grupos son iguales a esos tres resultados y el cuarto es el XOR de los tres. Eso explica el `YYYY` del prompt del nivel, ya que el último grupo es un checksum de los otros tres. La renombré `check_serial2`.

### El hash de 16 bits

![sub_1400019b1, el hash de 16 bits](img-051.webp){: w="1216" h="235" }

`sub_1400019b1` es un hash chico. Empieza desde la constante que recibe y, por cada carácter del usuario, lo mezcla con un XOR, multiplica por `0x9e37` y pasa el resultado por `sub_14000149c`. Todo trabaja con valores de 16 bits.

`sub_14000149c` está justo debajo de `rol32` y se le parece mucho: también es una rotación, pero de 16 bits.

![sub_14000149c al lado de rol32](img-052.webp){: w="1216" h="274" }

Una línea del código decompilado me confundió: `arg1` es un `int16_t` (un número de 16 bits con signo), y antes de desplazarlo el código lo castea a `uint32_t`. No estaba seguro de qué hace ese cast con un valor negativo. ¿Rellena los 16 bits nuevos con ceros, o con unos para mantener el signo? Si rellena con unos, esos unos se meterían en el resultado al desplazar y ya no sería una rotación de verdad. Miré el desensamblado:

![Desensamblado de la rotación, que usa movzx](img-053.webp){: w="1216" h="444" }

La instrucción que usa es `movzx`, "move with zero extension", así que el valor se rellena con ceros y es una rotación de 16 bits común y corriente. Renombré `sub_14000149c` a `rol16` y `sub_1400019b1` a `name_hash16`. Después de renombrar también sus variables, `name_hash16` se lee así, donde `start` es la constante que recibe:

![name_hash16 después de renombrar](img-054.webp){: w="1216" h="213" }

Así que el serial es el usuario hasheado tres veces, empezando desde `0x1234`, `0xabcd` y `0x7777`, más un cuarto grupo con los tres hashes combinados con XOR.

Como en el nivel 1, el nivel guarda algo en una global antes de volver, así que revisé quién usa `data_140015044`:

![Referencias cruzadas de data_140015044](img-055.webp){: w="954" h="154" }

De nuevo, solo dos lugares. `level_2` escribe la primera palabra del serial, y `print_completion_code` la lee con `snprintf(&buf, 0x20, "%04X", ...)`. O sea que el primer grupo del serial del nivel 2 también se usa para el código de finalización. Renombré la global `level2_word`.

Con las funciones y la global nombradas, renombré también las variables de `level_2` y convertí las cuatro variables locales de 16 bits en un array, `int16_t words[4]`, cambiando el tipo de la primera con `y`. Así se lee el loop del serial después de eso:

![Loop del serial de level_2 con int16_t words[4]](img-056.webp){: w="965" h="649" }

### Keygen

El keygen sigue la misma idea que el del nivel 1: una réplica de `name_hash16` reescrita en Python (con las 3 constantes) más el XOR de los tres resultados. Mantuve la detección de la página de códigos del nivel 1, ya que el nivel 2 hashea el mismo usuario:

```python
import ctypes

def rol16(x, n):
    return ((x << n) | (x >> (16 - n))) & 0xffff

def name_hash16(name, start):
    h = start
    for c in name:
        h = rol16(((h ^ c) * 0x9e37) & 0xffff, 3)
    return h

# console input code page, like in level 1
cp = "cp%d" % ctypes.windll.kernel32.GetConsoleCP()

name = input("Username: ").encode(cp)
a = name_hash16(name, 0x1234)
b = name_hash16(name, 0xabcd)
c = name_hash16(name, 0x7777)
print("Serial: %04X-%04X-%04X-%04X" % (a, b, c, a ^ b ^ c))
input("Press Enter to exit")
```
{: file="keygen2.py" }

Corriéndolo con mi nombre:

```text
Username: Bruno
Serial: D31B-84B1-76D4-217E
Press Enter to exit
```

Y contra el binario, primero con un serial cualquiera y después con el del keygen:

![El nivel 2 rechaza un serial al azar y acepta el del keygen](img-059.webp){: w="986" h="178" }

### Casos borde

#### Usuario vacío

El nivel 1 acepta un usuario vacío y el nivel 2 simplemente lo reutiliza. Sin caracteres, `name_hash16` nunca entra a su loop y devuelve la constante de la que partió, así que el serial son directamente las tres constantes y su XOR:

```text
Username:
Serial: 1234-ABCD-7777-CE8E
Press Enter to exit
```

![El nivel 2 acepta 1234-ABCD-7777-CE8E con usuario vacío](img-061.webp){: w="986" h="113" }

Pasa, así que para un usuario vacío el serial del nivel 2 está ahí mismo, a la vista, en `check_serial2`.

#### ¿Qué tan estricto es `"%4x-%4x-%4x-%4x"`?

Como en el nivel 1, `%x` saltea los espacios antes de cada número y no le importan las mayúsculas o minúsculas. Y esta vez nada chequea el largo del serial, así que una vez que `sscanf` leyó sus cuatro números, lo que venga después se ignora:

- En minúsculas y con espacios después de los guiones:

![El nivel 2 acepta el serial en minúsculas y con espacios](img-062.webp){: w="986" h="50" }

- Con algo de texto al final:

![El nivel 2 acepta el serial con texto al final](img-063.webp){: w="986" h="54" }

Nivel 2 terminado. Sigue el nivel 3, que pide un usuario nuevo y una contraseña.

## Nivel 3

### Primer vistazo

![Comienzo de level_3 con tres chequeos anti-debugging](img-064.webp){: w="1282" h="516" }

El nivel 3 empieza llamando, uno detrás del otro, a tres de los helpers anti-debugging que encontré mientras me orientaba. Por ahora no estoy usando un debugger, pero es algo para tener en cuenta si después lo necesito.

Después de los chequeos copia una global de 8 bytes, `data_140010000`, a una variable local y llama a `sub_140001c74`. Después imprime el banner y entra al loop de login:

![Loop de login de level_3](img-065.webp){: w="1282" h="498" }

- Pide un usuario, que tiene que tener al menos 5 caracteres (`strlen > 4`); si no, imprime *"Invalid username (minimum length 5)."*.
- Después pide una contraseña, de nuevo con `read_line` y `0x80`.
- `sub_140001f2b` recibe el usuario y la variable local de 8 bytes, y escribe algo en un buffer de `0x20` bytes, `_Str2`. Si la contraseña es igual a ese buffer (`strcmp`), el loop termina.
- Después de *"Level 3 PASSED"*, copia 16 caracteres de la contraseña a otra global, `data_140015050`, y devuelve 1.

Así que la contraseña se arma a partir del usuario y de esos 8 bytes. A diferencia de los dos primeros niveles, esta vez la comparación es un `strcmp` contra un string, así que no hay un `sscanf` que sea permisivo con la entrada, y la contraseña tiene que coincidir exactamente.

### La clave de 8 bytes

Antes de mirar el login, quería saber qué hace `sub_140001c74` con esos 8 bytes, ya que corre antes que todo lo demás:

![sub_140001c74 antes de renombrar](img-066.webp){: w="1282" h="669" }

Lo primero que me llamó la atención fueron los loops hasta `0xff`. El primero solo llena una tabla de 256 bytes con los números del 0 al 255, en orden. El segundo vuelve a recorrer esa tabla e intercambia sus posiciones, y para decidir qué intercambiar usa los bytes del primer argumento, el string `"RE-WIN!!%08X%08X"`. El último loop corre una vez por cada byte de los datos, 8 veces acá, y le hace XOR a cada byte con un valor sacado de la tabla.

Una tabla de 256 bytes llena con 0 a 255 y después mezclada con una clave: casualmente, me había cruzado con algo así en un CTF chico en la DEF CON de este año. Buscando, confirmé que es RC4, y el pseudocódigo de [Wikipedia](https://en.wikipedia.org/wiki/RC4) tiene los mismos tres loops.

Después de renombrar los argumentos y las variables, la función se lee así:

![rc4 después de renombrar](img-067.webp){: w="1282" h="630" }

Un detalle: la posición en la clave es `i_1 % key_len`, y `key_len` es el `8` que le pasa `level_3`. Así que el índice vuelve al principio después del octavo carácter, y de todo el string `"RE-WIN!!%08X%08X"` solo se usa `RE-WIN!!`. La parte `%08X%08X` nunca se lee, al menos no acá.

Estos son los 8 bytes cifrados:

![Los 8 bytes cifrados mostrados como un valor de 64 bits](img-068.webp){: w="653" h="45" }

Binary Ninja los muestra como un único número de 64 bits, `0x5c044c603c508d4a`. Como RC4 trabaja con bytes, pasé a la vista hex para verlos en el orden en que están en memoria:

![Los 8 bytes cifrados en la vista hexadecimal](img-069.webp){: w="1143" h="24" }

En vez de descifrarlos a mano, quería ver al programa descifrarlos, así que era el momento de abrir x64dbg, teniendo en cuenta que el nivel empieza con los chequeos anti-debugging.

Arranqué el challenge como siempre, pasé el nivel 1 y adjunté x64dbg mientras el nivel 2 esperaba su serial. Mi primera idea fue ir directo a la llamada a `rc4` con la dirección de Binary Ninja, `0x14000204f`, pero a x64dbg no le gustó:

![x64dbg rechazando la dirección de Binary Ninja](img-070.webp){: w="542" h="123" }

La pestaña Symbols mostraba que x64dbg tenía el programa cargado en `0x7FF77E120000`, no en `0x140000000` como Binary Ninja, así que las direcciones no coincidían. En vez de hacer la cuenta, busqué el string `RE-WIN!!`.

![El string RE-WIN!! en x64dbg](img-071.webp){: w="1159" h="54" }

Con doble clic llegué a `level_3`, con la llamada a `rc4` dos instrucciones más abajo, así que puse un breakpoint ahí. También seguí una de las llamadas a `die` desde `level_3` y puse otro breakpoint al principio, para que, si algún chequeo me agarraba, al menos supiera cuál. Después de ingresar el serial del nivel 2, uno me agarró:

![Breakpoint alcanzado en die después de IsDebuggerPresent](img-072.webp){: w="1118" h="134" }

El mensaje dice que fue `IsDebuggerPresent`. Según la [documentación de x64dbg](https://help.x64dbg.com/en/latest/commands/misc/HideDebugger.html), su comando `hide` modifica el proceso para que `IsDebuggerPresent()` devuelva false. Así que reinicié el challenge, volví a adjuntarme y corrí `hide` antes de ingresar el serial del nivel 2:

![Ejecutando hide en x64dbg](img-073.webp){: w="931" h="149" }

Esta vez los tres chequeos pasaron y el programa frenó en la llamada a `rc4`. El tercer argumento, `r8`, es la dirección de los 8 bytes, así que lo seguí en el dump. Ahí estaban los bytes cifrados, en el mismo orden que en la vista hex:

![Bytes cifrados en el dump de x64dbg antes de rc4](img-074.webp){: w="1004" h="75" }

Pasé por encima de la llamada y los mismos 8 bytes cambiaron:

![Los mismos bytes descifrados después de rc4](img-075.webp){: w="1004" h="75" }

La clave descifrada es `13 37 c0 ff ee 42 99 01` (¿elite coffee?).

### Armando la contraseña

Ahora la función que arma la contraseña, `sub_140001f2b`. Es corta:

![sub_140001f2b, que arma la contraseña](img-076.webp){: w="1282" h="214" }

Hace tres cosas:

1. Llama a `sub_140001e9f` con el usuario y se queda con el resultado, un número de 32 bits.
1. Por cada uno de los 8 bytes descifrados, llama a `sub_140001ee9` con ese número y el byte, y se queda con el nuevo resultado.
1. Escribe dos números de 32 bits en hex en el buffer con `"%08X%08X"`: el resultado final y una variación, `rol32(result ^ 0xdec0ded1, 11)`. Por eso la contraseña tiene 16 caracteres hex.

![sub_140001e9f, djb2 con XOR](img-077.webp){: w="1282" h="271" }

`sub_140001e9f` recorre el usuario carácter por carácter. Empieza en `0x1505` y, por cada carácter, multiplica por `0x21` y le hace XOR con el carácter. Buscando `0x1505` descubrí que es `djb2`, en una variante que usa XOR en lugar de una suma. La renombré `djb2`.

![sub_140001ee9, que mezcla cada byte de la clave](img-078.webp){: w="1282" h="116" }

`sub_140001ee9` es una sola línea, pero le hace varias cosas al número: le aplica XOR con el byte, lo rota 7 bits a la izquierda, le resta `0x61c88647` y le hace XOR consigo mismo desplazado 13 bits a la derecha. No encontré un algoritmo que coincidiera buscando en internet, aunque la constante `0x61c88647` aparece en muchos algoritmos de hashing. La renombré `mix_key_byte`, y `sub_140001f2b` pasó a ser `make_password3`.

Como en los niveles anteriores, también revisé quién usa la global a la que se copia la contraseña:

![Referencias cruzadas de data_140015050](img-079.webp){: w="954" h="175" }

Además de `level_3`, solo la usa `print_completion_code`, que se la pasa a `sub_1400025c6`. O sea que la contraseña del nivel 3 también forma parte del código de finalización. La renombré `level3_password`, y a los bytes cifrados `level3_key_enc`. Con todo renombrado, `level_3` se lee así:

![level_3 después de renombrar](img-080.webp){: w="1282" h="612" }

x64dbg seguía adjuntado y frenado justo después de la llamada a `rc4`, así que lo usé para conseguir una contraseña correcta con la que probar mi keygen más adelante. Bajé desde la llamada a `rc4` hasta la llamada a `strcmp`, puse un breakpoint ahí, dejé correr el programa e ingresé `Bruno` con una contraseña cualquiera:

![Registros en strcmp, con la contraseña esperada en rdx](img-081.webp){: w="1494" h="67" }

`rcx` tiene la contraseña que tipeé y `rdx` la que el nivel espera para Bruno: `AFB0685485B42B8B`.

Para ver el nivel completo de un vistazo, este es su grafo de Calltree con todo renombrado:

![Grafo de Calltree de level_3](img-082.webp){: w="1249" h="764" }

Todo lo que es específico de este nivel cuelga de `rc4` y `make_password3`; el resto son los mismos helpers que usan los otros niveles.

### El nivel 3, paso a paso

1. Antes que nada, busca un debugger de tres formas distintas y sale si encuentra uno.
1. Descifra con `RC4` 8 bytes guardados en el binario, usando `RE-WIN!!` como clave. Esos bytes son siempre los mismos, así que funcionan como un secreto fijo que solo aparece mientras el programa corre.
1. Pide un usuario de al menos 5 caracteres y después una contraseña.
1. Arma la contraseña esperada a partir del usuario, en tres pasos:
   - `djb2` recorre el usuario carácter por carácter y lo convierte en un único número de 32 bits.
   - `mix_key_byte` toma ese número y le mezcla los 8 bytes descifrados, uno por uno. Así el resultado depende tanto del usuario como de la clave.
   - El número final se escribe en hex dos veces, 8 caracteres cada vez: primero tal cual, y después con XOR `0xdec0ded1` y rotado 11 bits a la izquierda. Juntos forman los 16 caracteres de la contraseña.
1. Si la contraseña que tipeé es exactamente ese string, el nivel pasa y la contraseña se guarda para el código de finalización.

Lo bueno es que nada depende de la máquina ni de la hora. La única entrada es el usuario, así que un keygen solo tiene que repetir estos pasos en Python.

### Keygen

Este keygen tiene algunas piezas más que los anteriores, pero cada una es una copia directa de lo que vi en Binary Ninja: `rc4` con sus tres loops, `djb2`, `mix_key_byte` y los dos `%08X` del final de `make_password3`. Los 8 bytes cifrados van tal cual aparecen en la vista hex, y además hice que imprima la clave descifrada, para compararla con lo que vi en x64dbg. Mantuve la detección de la página de códigos de los niveles anteriores, porque acá también se hashea el usuario byte por byte:

```python
import ctypes

def rol32(x, n):
    return ((x << n) | (x >> (32 - n))) & 0xffffffff

def rc4(key, data):
    S = list(range(256))
    j = 0
    for i in range(256):
        j = (j + S[i] + key[i % len(key)]) & 0xff
        S[i], S[j] = S[j], S[i]
    out = []
    i = j = 0
    for b in data:
        i = (i + 1) & 0xff
        j = (j + S[i]) & 0xff
        S[i], S[j] = S[j], S[i]
        out.append(b ^ S[(S[i] + S[j]) & 0xff])
    return bytes(out)

def djb2(name):
    h = 0x1505
    for c in name:
        h = (c ^ (h * 0x21)) & 0xffffffff
    return h

def mix_key_byte(h, b):
    t = (rol32(h ^ b, 7) - 0x61c88647) & 0xffffffff
    return t ^ (t >> 13)

# level3_key_enc, decrypted with RE-WIN!!
key = rc4(b"RE-WIN!!", bytes.fromhex("4a8d503c604c045c"))
print("Key:", key.hex())

cp = "cp%d" % ctypes.windll.kernel32.GetConsoleCP()

name = input("Username: ").encode(cp)
h = djb2(name)
for b in key:
    h = mix_key_byte(h, b)
print("Password: %08X%08X" % (h, rol32(h ^ 0xdec0ded1, 11)))
input("Press Enter to exit")
```
{: file="keygen3.py" }

Corriéndolo con mi nombre:

```text
Key: 1337c0ffee429901
Username: Bruno
Password: AFB0685485B42B8B
Press Enter to exit
```

Tanto la clave como la contraseña coinciden con lo que me mostró x64dbg.

Y contra el binario, primero con una contraseña cualquiera y después con la del keygen:

![El nivel 3 rechaza una contraseña al azar y acepta la del keygen](img-085.webp){: w="736" h="199" }

### Casos borde

A diferencia de los dos primeros niveles, acá no hay `sscanf`, así que la contraseña tiene mucho menos margen para jugar. Igual probé algunas cosas:

#### Usuario corto

El mínimo de 5 caracteres se chequea antes de pedir la contraseña, así que con 4 caracteres el nivel ni siquiera llega:

![El nivel 3 rechaza un usuario de 4 caracteres](img-086.webp){: w="736" h="52" }

#### Contraseña en minúsculas

En los niveles 1 y 2 a `%x` no le importaban las mayúsculas o minúsculas, pero acá la contraseña se compara con `strcmp`, carácter por carácter. La contraseña esperada se imprime en mayúsculas con `%08X`, así que la misma contraseña en minúsculas falla:

![El nivel 3 rechaza la contraseña en minúsculas](img-087.webp){: w="736" h="68" }

#### Un usuario hecho solo de espacios

El único chequeo sobre el usuario es su largo, y `read_line` solo corta el salto de línea, así que cinco espacios son un usuario válido. Los seleccioné en la consola para que se vean:

![Salida del keygen para un usuario de cinco espacios, seleccionados](img-088.webp){: w="982" h="107" }

![El nivel 3 acepta un usuario de cinco espacios](img-089.webp){: w="736" h="68" }

Nivel 3 terminado. Sigue el nivel 4, el último, donde la contraseña la chequea una VM interna.

## Nivel 4

### Primer vistazo

![Comienzo de level_4 con los cuatro chequeos](img-090.webp){: w="1282" h="557" }

El último nivel empieza con los cuatro chequeos anti-debugging, después imprime el banner, esta vez con una advertencia y una pista, y entra a su loop:

![Loop de la contraseña de level_4](img-091.webp){: w="1282" h="550" }

- Pide una contraseña, con `read_line` y `0x80` como los otros niveles.
- Le pasa la contraseña y la dirección de una variable local, `var_89`, a `sub_140002161`. El nivel solo pasa si esa función devuelve algo distinto de 0 y `var_89` también termina en algo distinto de 0.
- Después de *"LEVEL 4 PASSED"*, copia hasta `0x3f` caracteres de la contraseña a otra global, `data_140015080`, y devuelve 1.

El banner dice que la contraseña la valida una VM interna, y la pista pide entender el bytecode. Tenía una idea aproximada de lo que significaba, pero antes de seguir busqué cómo funciona una VM en un crackme. La explicación más clara que encontré fue [Cracking a virtualized binary with Angr](https://noh.am/en/posts/vm/), de Noham. La idea es que el chequeo real no está escrito como código x86 normal, sino que se convierte en bytecode para un set de instrucciones inventado por los autores, y el programa trae un pequeño intérprete que lo ejecuta. Ese intérprete es un loop que lee un opcode, lo busca en una tabla y salta al pedazo de código (el handler) de esa instrucción, guardando el estado de la VM en algunas variables propias. El post resuelve su binario con angr, pero también describe una forma manual: recorrer cada handler, entender el formato del bytecode y escribir un desensamblador para él. Seguí el mismo enfoque para entender los handlers y el formato del bytecode, y después decodifiqué el bytecode a mano.

Esta vez no hay usuario ni `strcmp` contra una contraseña esperada. Todo se decide adentro de `sub_140002161`, que, por lo que dice el banner, tiene que ser la VM.

### Adentro de `sub_140002161`

![sub_140002161 antes de renombrar](img-092.webp){: w="1282" h="557" }

Esta función es más larga que las otras, así que la recorrí por partes, de arriba hacia abajo.

**Primero, un loop que llena un buffer:**

![Loop que descifra el bytecode](img-093.webp){: w="1000" h="153" }

Va de 0 a `0x88`, así que corre `0x89` (137) veces. En cada vuelta toma un byte de `data_140011380`, le hace XOR con `var_9` y guarda el resultado en la misma posición de un buffer local, `var_b8`. `var_9` empieza en `0xa5` y sube de a uno después de cada byte (`0xa5`, `0xa6`, `0xa7`...), así que cada byte se combina con un valor distinto. Dicho de otra forma, `data_140011380` guarda 137 bytes cifrados, y después de este loop `var_b8` los tiene descifrados.

**Después, un chequeo:**

![Chequeo del CRC contra 0x69f27969](img-094.webp){: w="1000" h="39" }

Llama a `sub_1400014d4` con `data_140011380` y `0x89`, los mismos bytes y el mismo largo, y si el resultado no es `0x69f27969` la función devuelve 0 ahí mismo. Todavía no sabía qué calcula `sub_1400014d4`, pero parecía una forma de asegurarse de que esos bytes son los esperados.

**Después, algunas variables arrancan en 0:**

![Variables inicializadas en 0](img-095.webp){: w="1000" h="77" }

Dos variables locales, `var_14_1` y `var_18_1`, se ponen en 0. El largo de la contraseña se guarda en `rax_10`, y un contador, `var_20_1`, también arranca en 0. Todavía no sabía para qué eran las dos primeras, pero eso iba a salir de ver cómo se usan.

**Por último, el loop principal:**

![Loop principal de la VM con el switch](img-096.webp){: w="1000" h="172" }

Sigue mientras `var_20_1` no pase de `0x88`, la última posición de `var_b8`. En cada vuelta lee el byte de `var_b8` en la posición de `var_20_1`, avanza `var_20_1` una posición y usa el byte que acaba de leer para elegir un case de un switch.

Esa última parte es exactamente el loop que describía el post: leer un opcode, elegir su handler con el switch, ejecutarlo, repetir. Así que `var_b8` debería ser el bytecode del que habla el banner. Estos son los bytes de los que sale:

![Bytecode cifrado en data_140011380](img-097.webp){: w="1282" h="114" }

Por sí solos no dicen mucho, ya que lo que está guardado en el archivo es la versión cifrada. Podía descifrarlos a mano con `var_9`, pero esta vez quería ver al programa descifrarlos, así que era el momento de volver a x64dbg.

`level_4` empieza con los chequeos anti-debugging, cuatro esta vez, incluido el nuevo `check_remote_debugger`. Mirando de nuevo `level_4`, vi que corren al principio de todo de la función, antes del loop de la contraseña.

![Grafo de level_4, con los chequeos antes del loop de la contraseña](img-098.webp){: w="1591" h="428" }

Eso significaba que, en vez de pelearme con los chequeos, podía simplemente llegar después de ellos. Corrí el challenge normalmente, sin ningún debugger, pasé los niveles 1 a 3 con mis keygens y recién adjunté x64dbg cuando el nivel 4 ya mostraba `Password:`

![El challenge esperando la contraseña del nivel 4](img-099.webp){: w="736" h="225" }

![x64dbg adjuntado, con el proceso todavía corriendo](img-100.webp){: w="996" h="193" }

El proceso siguió corriendo después de adjuntarme, sin ningún `die`. Para ese momento los cuatro chequeos ya habían corrido sin nada adjuntado, así que no tenían nada que encontrar, y esta vez ni siquiera necesité `hide`.

Por el nivel 3 sabía que x64dbg carga el programa en `0x7FF77E120000`, así que fui a la instrucción justo después del loop de descifrado (`0x1400021b3` en Binary Ninja, `0x7FF77E1221B3` en x64dbg), puse un breakpoint ahí y tipeé `test` como contraseña. Esta es la misma instrucción en las dos herramientas, el `mov edx, 0x89` que viene justo después del `jle` del loop:

![mov edx, 0x89 en Binary Ninja](img-101.webp){: w="794" h="78" }

![La misma instrucción en x64dbg](img-102.webp){: w="1118" h="34" }

Cuando frenó, `var_b8` ya estaba descifrado, y como el desensamblado accede a él como `rbp-0xB0`, abrí esa dirección en el dump:

![El bytecode descifrado en el dump de x64dbg](img-103.webp){: w="996" h="123" }

Ahí están, los 137 bytes descifrados. Vuelvo a ellos una vez que sepa qué hace cada instrucción.

**Después, el chequeo del paso 2:**

![sub_1400014d4, CRC32 con 0xedb88320](img-104.webp){: w="1282" h="287" }

`sub_1400014d4` recorre los bytes que recibe y, por cada uno, hace 8 rondas de desplazamientos y XOR con `0xedb88320`. Buscar esa constante en Google me dijo directamente que es el polinomio de CRC32, así que la renombré `crc32`.

¿Para qué sirve este chequeo?

1. Antes de ejecutar nada, `sub_140002161` calcula el CRC de `data_140011380`, el bytecode cifrado tal como está guardado en el archivo.
1. Compara el resultado con un valor fijo escrito en el código, `0x69f27969`.
1. Si no coinciden, devuelve 0 antes de ejecutar una sola instrucción, y el nivel falla.

Eso protege al bytecode contra cambios. A alguien se le podría ocurrir editar esos bytes en el exe para que la VM acepte cualquier contraseña, pero eso cambiaría el CRC y la VM ni siquiera arrancaría. A mí no me afecta, porque igual había decidido no parchear el binario, pero confirma que la forma de pasar este nivel es entender el bytecode, tal como dice la pista.

Como estaba frenado justo antes de esa llamada, la pasé por encima con F8 para ver el resultado:

![eax igual a 0x69F27969 después de la llamada](img-105.webp){: w="1494" h="157" }

`eax` es exactamente `0x69F27969`, así que el chequeo pasa y la VM sigue.

Con eso, renombré `var_b8` a `code`, `data_140011380` a `code_enc` y `var_9` a `key`.

### Las instrucciones

Lo siguiente eran las variables del loop. Empecé por las dos que eran más fáciles de descifrar:

- `var_20_1` arranca en 0, el loop para cuando pasa de `0x88` (la última posición de `code`), y en cada vuelta se usa para leer el siguiente byte de `code` y después avanza una posición. Mirando los cases por arriba, muchos lo avanzan algunas posiciones más (`+= 4`, `+= 2`), y dos le dan un valor completamente nuevo. Así que siempre apunta a la siguiente instrucción a ejecutar, que es lo que en una CPU se suele llamar el program counter. La renombré `pc`.
- `rax_10` es el resultado de `strlen(arg1)`, y `arg1` es la contraseña que `level_4` leyó con `read_line`, así que guarda el largo de la contraseña. La renombré `pass_len`.

Para entender `var_14_1` hice clic en ella, y el panel de Cross References listó todos los lugares que la usan:

![Referencias cruzadas de var_14_1](img-106.webp){: w="557" h="250" }

Mirando la lista, casi todas las líneas le ponen un valor nuevo a `var_14_1`. Recibe un número (`= rax_18`), se le suma algo (`+=`), se le aplica XOR (`^=`), se rota (`rol32`) o se multiplica (`*=`), o recibe el largo de la contraseña. Solo una línea, el `if`, la lee para compararla con algo. Así que las instrucciones no trabajan con varias variables distintas, sino que todas van actualizando esta misma, que funciona básicamente como un registro de CPU. La llamé `acc`.

`var_18_1` fue más simple. Tiene solo cuatro referencias: arranca en 0, `case 6` escribe en ella el resultado de una comparación y `case 7` la lee para decidir algo. Un valor que solo recuerda si una comparación fue verdadera es un flag, así que pasó a ser `flag`.

Los argumentos fueron más fáciles. `arg1` es lo que `level_4` leyó con `read_line`, así que pasó a ser `password`. `arg2` solo recibe un 1 o un 0, y `level_4` necesita que sea 1 para pasar, así que la llamé `accepted`. La función en sí pasó a ser `vm_run`:

![vm_run después de renombrar](img-107.webp){: w="1282" h="557" }

Después venía el `switch`. Este es completo, ya renombrado, en tres partes:

![El switch de la VM, parte 1 de 3](img-108.webp){: w="1282" h="732" }

![El switch de la VM, parte 2 de 3](img-109.webp){: w="1282" h="693" }

![El switch de la VM, parte 3 de 3](img-110.webp){: w="1282" h="732" }

Es mucho código de golpe, así que lo leí de a un case. En cada uno miré qué lee del bytecode, qué hace con `acc` y cuánto avanza `pc`, y fui renombrando sus variables temporales.

`case 1:`

![Handler de la VM para case 1](img-111.webp){: w="900" h="133" }

Cuando empieza `case 1`, el loop principal ya leyó el opcode y avanzó `pc` una posición, así que `pc` apunta al byte que sigue al opcode. Entonces case 1:

1. Lee los 4 bytes que empiezan en `pc` como un único número de 32 bits. Eso significa `*(uint32_t*)(pc + &code)`, y Binary Ninja ya había llamado al resultado `acc_1`.
1. Avanza `pc` 4 posiciones.
1. Pone ese número en `acc`.

El paso 2 fue el que me mostró que esos 4 bytes no son instrucciones sino datos que le pertenecen a esta, y que la siguiente instrucción empieza justo después de ellos, así que una instrucción puede ocupar más de un byte. Primero viene el opcode, que dice qué hacer, y después su argumento, el valor con el que trabaja.

`case 2:`

![Handler de la VM para case 2](img-112.webp){: w="900" h="285" }

Este tiene un argumento de 1 byte. Paso a paso, case 2:

1. Lee el byte en `pc` y avanza `pc` una posición.
1. Usa ese byte como una posición dentro de la contraseña. Como es un índice en la contraseña, lo llamé `idx`.
1. Chequea que `idx` no pase del final de la contraseña (`idx >= pass_len`). Si pasa, el resultado es 0. Si no, el resultado es el carácter de la contraseña en esa posición. Como guarda un carácter, lo llamé `ch`.
1. Suma `ch` a `acc`.

Es el primer case que mira la contraseña. Con él, el bytecode puede elegir cualquier carácter de la contraseña y sumarlo a `acc`.

`case 3:`

![Handler de la VM para case 3](img-113.webp){: w="900" h="133" }

Igual que case 1, pero en vez de reemplazar `acc` con el número de 4 bytes, le aplica XOR. A ese número lo llamé `imm`, de inmediato.

`case 4:`

![Handler de la VM para case 4](img-114.webp){: w="900" h="133" }

Lee un byte y rota `acc` a la izquierda esa cantidad de bits, con el mismo `rol32` de los niveles anteriores.

`case 5:`

![Handler de la VM para case 5](img-115.webp){: w="900" h="96" }

Sin ningún argumento, solo pone el largo de la contraseña en `acc`.

`case 6:`

![Handler de la VM para case 6](img-116.webp){: w="900" h="210" }

Lee 4 bytes, que llamé `expected`, y los compara con `acc`. El decompilador lo escribe al revés (`acc != expected ? 0 : 1`), pero significa que `flag` vale 1 si son iguales y 0 si no. No cambia nada más.

`case 7:`

![Handler de la VM para case 7](img-117.webp){: w="900" h="252" }

Lee 2 bytes, que llamé `target`, y avanza `pc` más allá de ellos. Después mira `flag`. Si vale 1, no hace nada más y la VM sigue con la próxima instrucción. Si vale 0, pone `pc` en `target`, así que la próxima instrucción sale de otro lugar del bytecode. En otras palabras, es un salto que solo ocurre cuando la última comparación falló.

`case 8:`

![Handler de la VM para case 8](img-118.webp){: w="900" h="115" }

También es un salto, pero este siempre ocurre: lee 2 bytes y los pone directamente en `pc`.

`case 9:`

![Handler de la VM para case 9](img-119.webp){: w="900" h="115" }

Escribe un 1 en `accepted` y devuelve 1. Es el único lugar donde la contraseña puede ser aceptada.

`case 0xb:`

![Handler de la VM para case 0xb](img-120.webp){: w="900" h="134" }

Como case 1 y case 3, pero multiplica `acc` por el número de 4 bytes, que llamé `factor`.

`case 0xc:`

![Handler de la VM para case 0xc](img-121.webp){: w="900" h="271" }

El gemelo de case 2. Lee una posición, toma ese carácter de la contraseña (o 0) y se lo aplica a `acc` con XOR en vez de sumarlo. Sus variables pasaron a ser `idx2` y `ch2`.

`case 0xa:`

![Handler de la VM para case 0xa](img-122.webp){: w="900" h="96" }

Lo dejé para el final porque me confundió. Lo único que muestra el decompilador son dos `break`, así que la pregunta era adónde llevan esos break. Al final de `vm_run` hay dos lugares posibles:

![Las dos salidas posibles al final de vm_run](img-123.webp){: w="849" h="142" }

1. `return 0`, justo después del switch pero todavía dentro del loop. Si case `0xa` termina acá, `vm_run` devuelve 0.
1. `accepted = 0; return 1`, después de todo el loop. Si termina acá, `vm_run` devuelve 1, pero con `accepted` en 0.

Las dos hacen fallar el nivel, pero son resultados distintos, y con los dos break solos no podía saber qué salida toma case `0xa`. Así que pasé al desensamblado, donde Binary Ninja marca adónde cae cada case de la tabla de saltos:

![Destinos de la tabla de saltos en el desensamblado](img-124.webp){: w="1081" h="338" }

Así que case `0xa` toma la segunda salida. Escribe un 0 en `accepted`, pone `eax` (el valor de retorno) en 1 y salta directo al final de la función.

Juntando todo, este es el set de instrucciones de la VM:

| Opcode | Argumento | Qué hace |
| --- | --- | --- |
| 0x01 | 4 bytes | acc = valor |
| 0x02 | 1 byte, i | acc += password[i] (0 si i pasa del final) |
| 0x03 | 4 bytes | acc ^= valor |
| 0x04 | 1 byte, n | acc = rol32(acc, n) |
| 0x05 | - | acc = largo de la contraseña |
| 0x06 | 4 bytes | flag = (acc == valor) |
| 0x07 | 2 bytes | si flag es 0, salta a esa posición |
| 0x08 | 2 bytes | salta a esa posición |
| 0x09 | - | aceptar: accepted = 1, return 1 |
| 0x0a | - | rechazar: accepted = 0, return 1 |
| 0x0b | 4 bytes | acc *= valor |
| 0x0c | 1 byte, i | acc ^= password[i] (0 si i pasa del final) |
| 0x00 | - | return 0 |

Me llevó bastante ver cómo encajaban todas estas piezas, pero al final entendí algunas cosas:

1. La única instrucción que acepta la contraseña es `0x09`, así que el bytecode tiene que llegar a una para pasar.
1. La única instrucción que puede elegir entre dos caminos es `0x07`, el salto condicional. Todas las demás hacen siempre lo mismo: las matemáticas se ejecutan y siguen con la próxima, y `0x08` siempre salta.
1. `0x07` decide mirando `flag`, y la única instrucción que escribe `flag` es `0x06`, comparando `acc` con un número del bytecode.
1. La contraseña solo entra en `acc` a través de `0x02` y `0x0c`, un carácter a la vez, y de `0x05`, que carga su largo.

Juntando eso, el programa tiene que verse más o menos así: calcular un número a partir de la contraseña en `acc`, compararlo con `0x06` y usar `0x07` para saltar a otro lado cuando no coincide, probablemente a un `0x0a`.

Con la tabla en la mano, volví al dump y leí los bytes de la misma forma que lo haría la VM. No podía agarrar cualquier byte y buscarlo en la tabla, porque un byte puede ser un opcode o parte de un argumento según desde dónde se empiece a leer. Por ejemplo, el tercer byte es un `0C`: solo, parecería el opcode `0x0c`, pero en realidad es parte del número que viene después del `06`. Así que empecé donde empieza `pc`, en la posición 0, y avancé de a una instrucción, usando la tabla para saber cuántos bytes ocupa cada una:

1. **La posición 0** es un `05`. En la tabla, `0x05` no tiene argumento, así que esta instrucción ocupa 1 byte: `acc = largo de la contraseña`. La siguiente empieza en la posición 1.
1. **La posición 1** es un `06`, que tiene un argumento de 4 bytes: `0C 00 00 00`, el número 12 (x86 guarda primero el byte menos significativo, así que los bytes se leen al revés). Entonces `flag = (acc == 12)`. La instrucción ocupa 5 bytes, y la siguiente empieza en la posición 6.
1. **La posición 6** es un `07`, con un argumento de 2 bytes: `88 00`, el número `0x88`. Si `flag` es 0, salta a la posición `0x88`. Ocupa 3 bytes, así que la siguiente empieza en la posición 9.
1. **La posición 9** es un `01`, con un argumento de 4 bytes: `05 15 00 00`, el número `0x1505`. Pone ese número en `acc`.

Seguí igual hasta el final de los 137 bytes, siempre con la misma regla: la próxima instrucción empieza en la posición actual, más 1 por el opcode, más el tamaño de su argumento. Según la tabla de opcodes, eso significa que una instrucción ocupa 1 byte sin argumento (0x05, 0x09, 0x0a), 2 bytes con un argumento de 1 byte (0x02, 0x04, 0x0c), 3 bytes con uno de 2 (0x07, 0x08) y 5 bytes con uno de 4 (0x01, 0x03, 0x06, 0x0b). La mayor parte del programa resultó ser las mismas tres instrucciones repetidas, 0x04, 0x02 y 0x03, que ocupan 2 + 2 + 5 = 9 bytes, así que cada ronda empieza 9 posiciones después de la anterior (0E, 17, 20, 29...). Y la cuenta cerró justo: la última instrucción está en 0x88, así que el programa termina en 0x89, los 137 bytes. Si me hubiera equivocado en algún tamaño, me habría dado cuenta, porque habría terminado antes o pasado del final.

Escrito como tabla, con la posición (en hex), los bytes y lo que hace cada instrucción, este es el programa completo:

| Posición | Bytes | Qué hace |
| --- | --- | --- |
| 00 | 05 | acc = largo de la contraseña |
| 01 | 06 0C 00 00 00 | flag = (acc == 0xC) |
| 06 | 07 88 00 | si flag == 0: salta a 88 |
| 09 | 01 05 15 00 00 | acc = 0x1505 |
| 0E | 04 05 | acc = rol32(acc, 5) |
| 10 | 02 00 | acc += password[0] |
| 12 | 03 00 01 00 00 | acc ^= 0x100 |
| 17 | 04 05 | acc = rol32(acc, 5) |
| 19 | 02 01 | acc += password[1] |
| 1B | 03 01 01 00 00 | acc ^= 0x101 |
| 20 | 04 05 | acc = rol32(acc, 5) |
| 22 | 02 02 | acc += password[2] |
| 24 | 03 02 01 00 00 | acc ^= 0x102 |
| 29 | 04 05 | acc = rol32(acc, 5) |
| 2B | 02 03 | acc += password[3] |
| 2D | 03 03 01 00 00 | acc ^= 0x103 |
| 32 | 04 05 | acc = rol32(acc, 5) |
| 34 | 02 04 | acc += password[4] |
| 36 | 03 04 01 00 00 | acc ^= 0x104 |
| 3B | 04 05 | acc = rol32(acc, 5) |
| 3D | 02 05 | acc += password[5] |
| 3F | 03 05 01 00 00 | acc ^= 0x105 |
| 44 | 04 05 | acc = rol32(acc, 5) |
| 46 | 02 06 | acc += password[6] |
| 48 | 03 06 01 00 00 | acc ^= 0x106 |
| 4D | 04 05 | acc = rol32(acc, 5) |
| 4F | 02 07 | acc += password[7] |
| 51 | 03 07 01 00 00 | acc ^= 0x107 |
| 56 | 04 05 | acc = rol32(acc, 5) |
| 58 | 02 08 | acc += password[8] |
| 5A | 03 08 01 00 00 | acc ^= 0x108 |
| 5F | 04 05 | acc = rol32(acc, 5) |
| 61 | 02 09 | acc += password[9] |
| 63 | 03 09 01 00 00 | acc ^= 0x109 |
| 68 | 04 05 | acc = rol32(acc, 5) |
| 6A | 02 0A | acc += password[10] |
| 6C | 03 0A 01 00 00 | acc ^= 0x10A |
| 71 | 04 05 | acc = rol32(acc, 5) |
| 73 | 02 0B | acc += password[11] |
| 75 | 03 0B 01 00 00 | acc ^= 0x10B |
| 7A | 03 BE BA FE CA | acc ^= 0xCAFEBABE |
| 7F | 06 5B 17 EB 71 | flag = (acc == 0x71EB175B) |
| 84 | 07 88 00 | si flag == 0: salta a 88 |
| 87 | 09 | aceptar |
| 88 | 0A | rechazar |

Leído así, el programa es más corto de lo que parecía:

1. **Posiciones 00 a 06:** el `05` en `00` carga el largo de la contraseña, el `06` en `01` lo compara con 12, y el `07` en `06` salta a `88` si son distintos. En `88` hay un `0x0a` que la rechaza. Así que la contraseña tiene que tener exactamente 12 caracteres, y lo que había supuesto del salto era correcto.
1. **Posición 09:** el `01` arranca `acc` en `0x1505`. Ese número es el mismo valor inicial que tenía `djb2` en el nivel 3.
1. **Posiciones 0E a 75:** una ronda por carácter, siempre las mismas tres instrucciones. En la primera ronda, el `04` en `0E` rota `acc` 5 bits a la izquierda, el `02` en `10` suma `password[0]` y el `03` en `12` le aplica XOR `0x100` a `acc`. La segunda ronda hace lo mismo en `17`, `19` y `1B` con `password[1]` y `0x101`, y así sucesivamente, hasta la última ronda en `71`, `73` y `75`, con `password[11]` y `0x10B`.
1. **Posiciones 7A a 88:** el `03` en `7A` le aplica XOR `0xCAFEBABE` a `acc` (segunda mención al café), el `06` en `7F` lo compara con `0x71EB175B`, y el `07` en `84` salta a `88` si son distintos. Si coinciden, el salto no se toma y la VM llega al `09` en `87`, que acepta la contraseña. Si no, cae en el `0A` en `88` y la contraseña se rechaza.

Para comprobar mi decodificación a mano, quería ver correr a la VM. Mi primera idea fue un breakpoint normal en el switch, mirando el opcode y `acc` cada vez que frenara. Pero con una contraseña de 12 caracteres la VM podía ejecutar decenas de instrucciones, y apretar F9 y anotar valores tantas veces sonaba lento y fácil de equivocarse. Así que busqué una forma de que x64dbg tomara las notas por mí. Esto es lo que encontré:

- [Conditional breakpoints](https://help.x64dbg.com/en/latest/introduction/ConditionalBreakpoint.html): un breakpoint puede tener una condición de corte, y si da 0 el programa no frena. También puede tener un texto de log, que se escribe en la pestaña Log cada vez que se pasa por el breakpoint.
- [String formatting](https://help.x64dbg.com/en/latest/introduction/Formatting.html): para poner valores en ese texto, todo lo que va entre llaves, como `{eax}`, se evalúa y se imprime, en hex por defecto.
- [Values](https://help.x64dbg.com/en/latest/introduction/Values.html): dentro de esas llaves, `[addr]` lee memoria, y `dword:[addr]` lee exactamente 4 bytes.

Con eso, armé el breakpoint así:

- **Dónde:** en el `cmp eax, 0xc` del switch, donde `eax` ya tiene el opcode.
- **Condición de corte:** 0, para que nunca frene.
- **Texto del log:** la posición de la instrucción, el opcode y `acc` antes de ejecutarla. Me llevó algunas corridas dar con el formato exacto. En el desensamblado, `pc` está en `rbp-0x18` (en ese punto ya está uno más allá del opcode, de ahí el -1) y `acc` está en `rbp-0xC`.

De vuelta en el programa, tipeé la contraseña `test`:

![Traza del breakpoint de log para la contraseña test](img-125.webp){: w="735" h="70" }

Solo cuatro instrucciones, exactamente las que había decodificado a mano: `0x05` carga el largo (4), `0x06` lo compara con 12, `0x07` salta a la posición `0x88` porque son distintos, y ahí la VM encuentra un `0x0a`, el rechazo. Probé de nuevo con 12 caracteres, `abcdefghijkl`:

![Traza del breakpoint de log para abcdefghijkl](img-126.webp){: w="735" h="634" }

Esta vez el chequeo de largo pasa y corre el programa completo, en el mismo orden que mi listado. Las posiciones del log son las mismas que había anotado a mano. Después de cargar `0x1505` en `acc` con `0x01`, las mismas tres instrucciones se repiten 12 veces, `0x04`, `0x02` y `0x03`, una ronda por carácter. Al final hay un `0x03` más, la comparación `0x06`, el salto `0x07` y el mismo `0x0a` en `0x88`, así que `vm_run` devuelve 1 con `accepted` en 0, y `level_4` imprime el mensaje *"Incorrect password"*.

Como en los otros niveles, también revisé quién usa la global a la que se copia la contraseña:

![Referencias cruzadas de data_140015080](img-127.webp){: w="627" h="256" }

`print_completion_code` se la pasa a `sub_1400025c6` y también a `crc32`, así que la contraseña del nivel 4 también forma parte del código de finalización. La renombré `level4_password`, y con eso `level_4` se lee así:

![level_4 después de renombrar](img-128.webp){: w="1282" h="555" }

Con el programa entendido completo, lo que queda es encontrar una contraseña que lo pase.

### Yendo hacia atrás

Algo importante de este programa: nunca compara la contraseña con una contraseña guardada. Convierte los 12 caracteres en un único número de 32 bits, `acc`, y lo único que chequea es si ese número termina siendo `0x71EB175B`. Cualquier combinación de 12 caracteres que produzca ese número pasa. Eso también significa que no hay una sola contraseña válida. 12 caracteres son 96 bits comprimidos en un número de 32 bits, así que, como con un hash, muchísimas contraseñas distintas terminan en el mismo resultado. El objetivo no era recuperar "la" contraseña, sino encontrar cualquier contraseña que produzca ese número.

El chequeo pasa al final de todo, así que empecé por ahí. Para pasar, `acc` tiene que valer `0x71EB175B` justo después del XOR con `0xCAFEBABE`. Hacer el mismo XOR dos veces lo deshace, así que antes de ese XOR, al final de las 12 rondas, `acc` tiene que valer `0x71EB175B ^ 0xCAFEBABE`.

Una ronda también se puede deshacer, siempre que sepa qué carácter usó. Cada uno de sus tres pasos tiene un opuesto:

1. `acc = rol32(acc, 5)` se deshace rotando 5 bits a la derecha.
1. `acc += password[i]` se deshace restando el carácter.
1. `acc ^= 0x100 + i` se deshace con el mismo XOR.

Yendo hacia atrás, se deshacen en orden inverso: primero el XOR, después la resta, después la rotación. Para probarlo, escribí un script con una función, `undo`, que toma un valor de `acc` al final de las 12 rondas y una contraseña, y deshace las rondas una por una de la última a la primera, para obtener el valor que tenía `acc` al principio. Después lo corrí dos veces con `abcdefghijkl`:

```python
def ror32(x, n):
    return ((x >> n) | (x << (32 - n))) & 0xffffffff

def undo(acc, pw):
    # undo the 12 rounds, from the last one to the first
    for i in range(11, -1, -1):
        acc = ror32(((acc ^ (0x100 + i)) - pw[i]) & 0xffffffff, 5)
    return acc

pw = b"abcdefghijkl"

# value the trace showed after the rounds for abcdefghijkl
print("from the trace:  %08X" % undo(0x8E7FBD34, pw))

# value needed to pass: 0x71EB175B before the xor with 0xCAFEBABE
print("from the target: %08X" % undo(0x71EB175B ^ 0xCAFEBABE, pw))
```
{: file="backwards.py" }

```text
from the trace:  00001505
from the target: C75EFFD6
```

La primera línea es una buena noticia: partiendo de la traza y deshaciendo las 12 rondas, volví exactamente a `0x1505`, así que el deshacer está bien. La segunda muestra el problema: partiendo del valor necesario para pasar, termino en `0xC75EFFD6` en lugar de `0x1505`. Es lo que esperaba, ya que `abcdefghijkl` no es la contraseña, pero muestra que no puedo elegir cualquier carácter en el camino de vuelta.

### Fuerza bruta

Mi siguiente idea fue elegir yo los caracteres mientras iba hacia atrás. Eso funciona hasta el primer carácter: para entonces, `acc` antes de él tiene que ser exactamente `0x1505`, así que su valor ya no lo elijo yo, lo fuerzan los otros 11. Para ver cómo es ese valor forzado, escribí otro script. Elige los últimos 11 caracteres al azar, va hacia atrás y calcula cuál tendría que ser el primero:

```python
import random
import string

def rol32(x, n):
    return ((x << n) | (x >> (32 - n))) & 0xffffffff

def ror32(x, n):
    return ((x >> n) | (x << (32 - n))) & 0xffffffff

goal = 0x71EB175B ^ 0xCAFEBABE

for attempt in range(10):
    # pick the last 11 characters at random
    rest = "".join(random.choice(string.ascii_letters) for _ in range(11)).encode()
    acc = goal
    for i in range(11, 0, -1):
        acc = ror32(((acc ^ (0x100 + i)) - rest[i - 1]) & 0xffffffff, 5)
    # now acc is the value after the first round, so the first char is forced
    first = ((acc ^ 0x100) - rol32(0x1505, 5)) & 0xffffffff
    print("?" + rest.decode(), "-> the first char would have to be %08X" % first)
```
{: file="first_char.py" }

```text
?dWFIYfIjTba -> the first char would have to be 4EE20312
?cUBeeKtimgs -> the first char would have to be 6788CE98
?mwAtcdkYDgU -> the first char would have to be 13732425
?UKiAddquIJG -> the first char would have to be AECF27A5
?JUUsMBaUVAP -> the first char would have to be E30DC49F
?KJxoYWylWVL -> the first char would have to be 0A874669
?OKOkSqqrczo -> the first char would have to be E3A1BFE1
?xbYvfyLtmzj -> the first char would have to be AF9F1754
?mGvzAGDboYf -> the first char would have to be 2489B68E
?UkSbdFExWDI -> the first char would have to be 36A99B13
```

Diez intentos y ninguno es un carácter. Son números de 32 bits como `4EE20312`, cuando un carácter que puedo tipear está entre `0x21` (`!`) y `0x7E` (`~`), los caracteres imprimibles de la [tabla ASCII](https://en.wikipedia.org/wiki/ASCII), sin el espacio. Como `acc` es un número de 32 bits, el valor forzado puede ser cualquiera de 2^32 = 4.294.967.296 valores, y solo 94 de ellos son caracteres que puedo tipear.

Igual, nada me impide probar una y otra vez hasta que alguno caiga en ese rango, así que escribí un script que hace exactamente eso y medí cuánto tarda:

```python
import random
import time

def rol32(x, n):
    return ((x << n) | (x >> (32 - n))) & 0xffffffff

def ror32(x, n):
    return ((x >> n) | (x << (32 - n))) & 0xffffffff

goal = 0x71EB175B ^ 0xCAFEBABE
start = rol32(0x1505, 5)
chars = range(0x21, 0x7f)  # printable, without the space

t = time.time()
tries = 0
while True:
    tries += 1
    rest = bytes(random.choices(chars, k=11))
    acc = goal
    for i in range(11, 0, -1):
        acc = ror32(((acc ^ (0x100 + i)) - rest[i - 1]) & 0xffffffff, 5)
    first = ((acc ^ 0x100) - start) & 0xffffffff
    if 0x21 <= first <= 0x7e:
        break

print("password: " + chr(first) + rest.decode())
print("tries: %d" % tries)
print("time: %.1f seconds" % (time.time() - t))
```
{: file="brute_first_char.py" }

```text
password: }p:AS~h+TV/)
tries: 3723181
time: 22.4 seconds
```

Encontró `}p:AS~h+TV/)` después de 3,7 millones de intentos y 22 segundos, pero esa corrida tuvo suerte. Con 94 valores buenos sobre 4.294.967.296, el promedio ronda los 45 millones de intentos, y a unos 8 microsegundos por intento en Python, son varios minutos por cada contraseña. Así que, técnicamente, ya tenía una forma de conseguir contraseñas.

### Una contraseña con mi nombre

Pero había dos cosas que no me convencían. La primera es que las contraseñas salen como basura al azar, tipo `}p:AS~h+TV/)`, mientras que los keygens de los otros niveles partían de mi nombre. Quería algo parecido acá, una contraseña que empiece con `Bruno_`. La segunda es que cada contraseña cuesta minutos.

Fijar un prefijo planteaba una pregunta: si 6 de los 12 caracteres ya están decididos, ¿alcanzan los otros 6 para que exista una contraseña válida? Cada combinación de los caracteres libres termina en uno de los 4.300 millones de valores posibles de `acc`, más o menos al azar, así que en promedio solo una de cada 4.300 millones de combinaciones es una contraseña válida. Dividiendo la cantidad de combinaciones por eso, pude estimar cuántas contraseñas válidas esperar según la cantidad de caracteres libres:

| Caracteres libres | Combinaciones posibles (94^n) | Contraseñas válidas esperables (÷ 4.300 millones) |
| --- | --- | --- |
| 1 | 94 | prácticamente ninguna |
| 4 | unos 78 millones | unas 0,02, o sea, casi seguro ninguna |
| 5 | unos 7.300 millones | unas 2 |
| 6 | unos 690.000 millones | unas 160 |

Con 4 caracteres libres probablemente no exista ni una contraseña válida, por mejor que busque. Con 6 debería haber alrededor de 160. Así que dejé 6 libres y usé los otros 6 para mi nombre.

Después la pregunta era cómo encontrarlas rápido. Adivinar esos 6 caracteres al azar llevaría unos 4.300 millones de intentos en promedio, horas en Python. Calcular uno de ellos a partir de los otros, como en la fuerza bruta, lo dejaría otra vez en minutos por contraseña.

Acá me trabé un poco. No veía cómo hacerlo mejor que en minutos. Seguía pensando en lo único que sabía con seguridad, que las rondas se podían deshacer, pero no lograba convertir eso en un plan. Así que me puse a leer cómo se ataca este tipo de problema, donde conocés el principio y el final de un cálculo pero no los valores del medio, y así encontré el ataque meet-in-the-middle. La [explicación de Baeldung](https://www.baeldung.com/cs/security-meet-in-the-middle-attack) tiene una frase que encajaba con mi problema:

> The attack takes advantage of the fact that the encryption and decryption processes are inverse operations of each other.

Era exactamente mi situación. Las rondas de la VM hacen de cifrado, y mi función `undo` es su inversa. Así que, en lugar de ir de punta a punta desde un extremo, podía ir hacia adelante desde el principio y hacia atrás desde el final, probando por fuerza bruta algunos caracteres de cada lado, y buscar el punto donde se encuentran. Aplicado a esta contraseña, queda así:

```text
posición:     0 1 2 3 4 5 | 6 7 8 | 9 10 11
contraseña:   B r u n o _ | ? ? ? | ?  ?  ?
                             -->      <--
```

1. **Hacia adelante:** desde `0x1505`, ejecuto las rondas de `Bruno_`, después pruebo todas las combinaciones para las posiciones 6 a 8 y guardo cada `acc` resultante en una tabla.
1. **Hacia atrás:** desde el valor necesario para pasar, deshago las rondas de las posiciones 11, 10 y 9, probando todas las combinaciones de esos 3 caracteres.
1. **El punto de encuentro:** cuando un valor del lado de atrás está en la tabla, las dos mitades encajan, y eso es una contraseña válida.

Lo escribí como un script, con `Bruno_` como prefijo:

```python
import time

def rol32(x, n):
    return ((x << n) | (x >> (32 - n))) & 0xffffffff

def ror32(x, n):
    return ((x >> n) | (x << (32 - n))) & 0xffffffff

goal = 0x71EB175B ^ 0xCAFEBABE
prefix = b"Bruno_"
chars = range(0x21, 0x7f)  # printable, without the space

t = time.time()

# forwards: the prefix, then every combination for positions 6 to 8
acc = 0x1505
for i, c in enumerate(prefix):
    acc = ((rol32(acc, 5) + c) & 0xffffffff) ^ (0x100 + i)

middle = {}
for a in chars:
    acc6 = ((rol32(acc, 5) + a) & 0xffffffff) ^ 0x106
    for b in chars:
        acc7 = ((rol32(acc6, 5) + b) & 0xffffffff) ^ 0x107
        for c in chars:
            acc8 = ((rol32(acc7, 5) + c) & 0xffffffff) ^ 0x108
            middle[acc8] = bytes([a, b, c])

# backwards: undo positions 11, 10 and 9 with every combination
found = 0
for z in chars:
    acc11 = ror32(((goal ^ 0x10B) - z) & 0xffffffff, 5)
    for y in chars:
        acc10 = ror32(((acc11 ^ 0x10A) - y) & 0xffffffff, 5)
        for x in chars:
            acc9 = ror32(((acc10 ^ 0x109) - x) & 0xffffffff, 5)
            if acc9 in middle:
                pw = prefix + middle[acc9] + bytes([x, y, z])
                print("%s  meets at %08X" % (pw.decode(), acc9))
                found += 1

print("found %d passwords in %.1f seconds" % (found, time.time() - t))
input("Press Enter to exit")
```
{: file="mitm.py" }

La parte hacia adelante llena una tabla, `middle`, con el valor de `acc` después de la posición 8 para cada combinación de las posiciones 6 a 8. La parte hacia atrás deshace las posiciones 11, 10 y 9 para cada combinación, y cada vez que el resultado está en la tabla, las dos mitades encajan e imprime la contraseña. Corriéndolo:

```text
Bruno_ZCu9*.  meets at 90817621
Bruno_ZCtY*.  meets at 90817620
Bruno_ZCcy*.  meets at 9081761F
Bruno_ZCu6J.  meets at 90817621
Bruno_ZCtVJ.  meets at 90817620
Bruno_ZCcvJ.  meets at 9081761F
Bruno_ZCu7j.  meets at 90817621
Bruno_ZCtWj.  meets at 90817620
Bruno_ZCcwj.  meets at 9081761F
Bruno_ZCu9-N  meets at 90817621
Bruno_ZCtY-N  meets at 90817620
Bruno_ZCcy-N  meets at 9081761F
Bruno_ZCu6MN  meets at 90817621
Bruno_ZCtVMN  meets at 90817620
Bruno_ZCcvMN  meets at 9081761F
Bruno_ZCu7mN  meets at 90817621
Bruno_ZCtWmN  meets at 90817620
Bruno_ZCcwmN  meets at 9081761F
Bruno_ZCu9,n  meets at 90817621
Bruno_ZCtY,n  meets at 90817620
Bruno_ZCcy,n  meets at 9081761F
Bruno_ZCu6Ln  meets at 90817621
Bruno_ZCtVLn  meets at 90817620
Bruno_ZCcvLn  meets at 9081761F
Bruno_ZCu7ln  meets at 90817621
Bruno_ZCtWln  meets at 90817620
Bruno_ZCcwln  meets at 9081761F
found 27 passwords in 0.7 seconds
Press Enter to exit
```

27 contraseñas en 0,7 segundos, en vez de minutos por una sola. Tomando la primera, `Bruno_ZCu9*.`: yendo hacia adelante con `Bruno_ZCu`, `acc` vale `0x90817621` después de la posición 8, y yendo hacia atrás desde el valor necesario para pasar, deshaciendo `.`, `*` y `9`, `acc` también vale `0x90817621` antes de la posición 9. Las dos mitades se encuentran en el mismo valor, así que la contraseña completa funciona.

Igual, dos cosas me llamaron la atención. Todas empiezan con `Bruno_ZC`, y son solo 27, cuando esperaba alrededor de 160. Después de dar algunas vueltas, me di cuenta de que mi tabla guarda una sola combinación por cada valor de `acc`, así que si varias combinaciones terminan en el mismo valor, la última pisa a las demás. Para comprobarlo, cambié la tabla para que guarde una lista de combinaciones por valor, e hice que el script imprima cuántos valores distintos hay y una muestra de las contraseñas que encuentra:

```python
import time

def rol32(x, n):
    return ((x << n) | (x >> (32 - n))) & 0xffffffff

def ror32(x, n):
    return ((x >> n) | (x << (32 - n))) & 0xffffffff

goal = 0x71EB175B ^ 0xCAFEBABE
prefix = b"Bruno_"
chars = range(0x21, 0x7f)  # printable, without the space

t = time.time()

# forwards: the prefix, then every combination for positions 6 to 8
acc = 0x1505
for i, c in enumerate(prefix):
    acc = ((rol32(acc, 5) + c) & 0xffffffff) ^ (0x100 + i)

middle = {}
for a in chars:
    acc6 = ((rol32(acc, 5) + a) & 0xffffffff) ^ 0x106
    for b in chars:
        acc7 = ((rol32(acc6, 5) + b) & 0xffffffff) ^ 0x107
        for c in chars:
            acc8 = ((rol32(acc7, 5) + c) & 0xffffffff) ^ 0x108
            # keep every combination, not just the last one
            middle.setdefault(acc8, []).append(bytes([a, b, c]))

print("combinations: %d, different values: %d" % (94 ** 3, len(middle)))

# backwards: undo positions 11, 10 and 9 with every combination
found = []
for z in chars:
    acc11 = ror32(((goal ^ 0x10B) - z) & 0xffffffff, 5)
    for y in chars:
        acc10 = ror32(((acc11 ^ 0x10A) - y) & 0xffffffff, 5)
        for x in chars:
            acc9 = ror32(((acc10 ^ 0x109) - x) & 0xffffffff, 5)
            for mid in middle.get(acc9, []):
                found.append(prefix + mid + bytes([x, y, z]))

print("found %d passwords in %.1f seconds" % (len(found), time.time() - t))
for pw in found[::25]:
    print(pw.decode())
input("Press Enter to exit")
```
{: file="mitm_all.py" }

```text
combinations: 830584, different values: 102704
found 243 passwords in 0.8 seconds
Bruno_W!59*.
Bruno_ZBCy*.
Bruno_YccvJ.
Bruno_Ya#wj.
Bruno_W"Cy-N
Bruno_ZCtVMN
Bruno_ZA4WmN
Bruno_YbTY,n
Bruno_W#tVLn
Bruno_W!4Wln
Press Enter to exit
```

Era eso. Las 830.584 combinaciones del lado de adelante dan solo 102.704 valores distintos, así que muchas comparten valor, y la primera versión tiraba la mayoría. Guardándolas todas, la misma búsqueda encuentra 243 contraseñas en 0,8 segundos, mucho más cerca de la estimación, y con todo tipo de caracteres después de `Bruno_`.

Al final me di cuenta de que podría haber usado el ataque meet-in-the-middle sin prefijo, pero tener un prefijo hace que la contraseña sea mucho más significativa, así que esa es la versión que usé para el keygen.

### El nivel 4, paso a paso

Antes de escribir el keygen, este es el nivel completo:

1. `level_4` corre sus cuatro chequeos anti-debugging una vez, después pide una contraseña y se la pasa a `vm_run`.
1. `vm_run` descifra 137 bytes de bytecode (XOR con `0xa5`, `0xa6`, `0xa7`...), verifica el CRC32 de los bytes cifrados originales y ejecuta el bytecode descifrado en una VM chica con un registro, `acc`, un `flag` y 12 instrucciones.
1. El bytecode chequea que la contraseña tenga exactamente 12 caracteres.
1. Arranca `acc` en `0x1505` y, por cada carácter, rota `acc` 5 bits a la izquierda, suma el carácter y le aplica XOR con `0x100` más la posición del carácter.
1. Al final, le aplica XOR `0xCAFEBABE` a `acc`, y la contraseña pasa solo si el resultado es `0x71EB175B`.
1. Las rondas se pueden deshacer, así que una búsqueda meet-in-the-middle puede encontrar contraseñas que empiecen con 6 caracteres a mi elección.

### Keygen

El keygen es la segunda versión de la búsqueda, con dos cambios: pide los primeros 6 caracteres en lugar de usar siempre `Bruno_`, y usa la página de códigos de la consola, como los keygens de los otros niveles, para que los caracteres que tipeo en el keygen sean los mismos bytes que recibe el challenge. También avisa cuando no encuentra ninguna contraseña, pero a eso vuelvo en los casos borde:

```python
import ctypes

def rol32(x, n):
    return ((x << n) | (x >> (32 - n))) & 0xffffffff

def ror32(x, n):
    return ((x >> n) | (x << (32 - n))) & 0xffffffff

# acc right before the final xor with 0xCAFEBABE
goal = 0x71EB175B ^ 0xCAFEBABE
chars = range(0x21, 0x7f)    # printable, without the space

# console input code page, like in the other levels
cp = "cp%d" % ctypes.windll.kernel32.GetConsoleCP()
prefix = input("First 6 characters: ").encode(cp)

if len(prefix) != 6:
    print("It has to be exactly 6 characters")
else:
    # forwards: the prefix, then every combination for positions 6 to 8
    acc = 0x1505
    for i, c in enumerate(prefix):
        acc = ((rol32(acc, 5) + c) & 0xffffffff) ^ (0x100 + i)

    middle = {}
    for a in chars:
        acc6 = ((rol32(acc, 5) + a) & 0xffffffff) ^ 0x106
        for b in chars:
            acc7 = ((rol32(acc6, 5) + b) & 0xffffffff) ^ 0x107
            for c in chars:
                acc8 = ((rol32(acc7, 5) + c) & 0xffffffff) ^ 0x108
                middle.setdefault(acc8, []).append(bytes([a, b, c]))

    # backwards: undo positions 11, 10 and 9 and look for a match
    found = []
    for z in chars:
        acc11 = ror32(((goal ^ 0x10B) - z) & 0xffffffff, 5)
        for y in chars:
            acc10 = ror32(((acc11 ^ 0x10A) - y) & 0xffffffff, 5)
            for x in chars:
                acc9 = ror32(((acc10 ^ 0x109) - x) & 0xffffffff, 5)
                for mid in middle.get(acc9, []):
                    found.append(prefix + mid + bytes([x, y, z]))

    if not found:
        print("No password starts with that, try other 6 characters")
    else:
        print("Found %d passwords, for example:" % len(found))
        for pw in found[::max(1, len(found) // 10)][:10]:
            print(pw.decode(cp))

input("Press Enter to exit")
```
{: file="keygen4.py" }

Corriéndolo con mi nombre:

```text
First 6 characters: Bruno_
Found 243 passwords, for example:
Bruno_W!59*.
Bruno_ZA#y*.
Bruno_Ya#vJ.
Bruno_W!#wj.
Bruno_ZA4Y-N
Bruno_Ya4VMN
Bruno_W!4WmN
Bruno_ZA59,n
Bruno_Ya56Ln
Bruno_W!57ln
Press Enter to exit
```

Prueba en el binario, primero con una contraseña cualquiera y después con la primera que me dio el keygen:

![El nivel 4 rechaza una contraseña al azar y acepta Bruno_W!59*.](img-141.webp){: w="736" h="193" }

Nivel 4 superado, y con él todo el challenge. Justo después, el programa imprime el código de finalización, al que vuelvo al final:

![Challenge completo, con el código de finalización](img-142.webp){: w="736" h="141" }

### Casos borde

#### Prefijos sin contraseñas

Probando el keygen con otros prefijos, encontré algo que no esperaba. Algunos no dan ninguna contraseña, como `Gecko!`:

```text
First 6 characters: Gecko!
No password starts with that, try other 6 characters
Press Enter to exit
```

Después de probar algunos más, resultó ser todo o nada. `Bruno_` da 243 contraseñas, `abcdef` 216, `Gecko_` 213 y `RE-WIN` 162, pero `Gecko!`, `Binary`, `Hello!`, `AAAAAA`, `Bruno!` y `Academ` no dan ninguna. O sea que la estimación de unas 160 era solo un promedio, y las contraseñas válidas no se reparten parejo entre los prefijos. Por eso ajusté el keygen para que, cuando no encuentra nada, lo diga y pida otros 6 caracteres, en lugar de imprimir una lista vacía.

#### Largo incorrecto

Lo primero que chequea el bytecode es el largo, así que una contraseña del keygen con un carácter menos, o uno más, falla de entrada:

![El nivel 4 rechaza contraseñas de 11 y 13 caracteres](img-144.webp){: w="736" h="111" }

#### Muchas contraseñas válidas

Como no hay una sola contraseña, cualquiera de las que imprime el keygen debería funcionar, no solo la primera. `Bruno_Ya#vJ.`, otra de la misma lista, también pasa:

![El nivel 4 acepta Bruno_Ya#vJ. con otro código de finalización](img-145.webp){: w="736" h="168" }

El código de finalización es distinto al de la corrida anterior. Incluye la contraseña del nivel 4, así que cambia con cada contraseña. Vuelvo a esto al final.

#### Caracteres con tilde

Como en los otros niveles, el keygen codifica lo que tipeo con la página de códigos de la consola. Con `Brunó_` también encuentra 243 contraseñas:

```text
First 6 characters: Brunó_
Found 243 passwords, for example:
Brunó_W!55'6
Brunó_ZA#u'6
Brunó_Ya#rG6
Brunó_W!#sg6
Brunó_ZA4U&V
Brunó_Ya4RFV
Brunó_W!4SfV
Brunó_ZA55!v
Brunó_Ya52Av
Brunó_W!53av
Press Enter to exit
```

Y `Brunó_Ya#rG6` pasa en el binario:

![El nivel 4 acepta Brunó_Ya#rG6](img-147.webp){: w="736" h="49" }

En cp850 la ó es un solo byte, así que la contraseña tiene 12 bytes. Si el keygen hubiera usado UTF-8, la ó ocuparía dos bytes, y la contraseña de 13 bytes ni siquiera pasaría el chequeo de largo. Nivel 4 terminado, y con él todo el challenge.

## El código de finalización {#codigo-de-finalizacion}

Probando el nivel 4 obtuve tres códigos de finalización distintos, uno por cada contraseña, así que antes de elegir cuál enviar quería saber exactamente qué entra en él. De vuelta en Binary Ninja, `print_completion_code` ya se leía fácil, porque todas las globales que toca las había renombrado mientras recorría los niveles:

![print_completion_code](img-148.webp){: w="1586" h="380" }

Arranca un valor de 32 bits, `var_c`, en `0x811c9dc5`, y después le pasa un puntero a él a `sub_1400025c6` cinco veces, cada vez con un string distinto:

1. `arg1`, que en `main` es `&var_88`, el mismo buffer que reciben `level_1` y `level_2` (el usuario).
1. `level1_hash`, impreso con `"%08X"`, o sea, el serial del nivel 1 en hex en mayúsculas.
1. `level2_word`, impreso con `"%04X"`, o sea, el primer grupo del serial del nivel 2.
1. `level3_password`.
1. `level4_password`.

Al final imprime `var_c` y, después de un guion, el `crc32` de la contraseña del nivel 4.

`sub_1400025c6` era la única parte que todavía no había visto:

![sub_1400025c6, FNV-1a](img-149.webp){: w="1586" h="384" }

Recorre el string byte por byte hasta el terminador nulo y, por cada byte, le aplica XOR al valor con ese byte y lo multiplica por `0x1000193`. Igual que con el CRC32 del nivel 4, una búsqueda rápida de `0x811c9dc5` y `0x1000193` me dijo que son el offset basis y el primo del [hash FNV-1a](https://en.wikipedia.org/wiki/Fowler%E2%80%93Noll%E2%80%93Vo_hash_function) de 32 bits. Como el mismo valor pasa por las cinco llamadas, la primera mitad del código es un único hash FNV-1a de los cinco strings uno detrás del otro. La renombré `fnv1a`.

Así que el código de finalización es algo así como una huella de toda la corrida. Depende del usuario y de lo que ingresé en cada nivel, y la segunda mitad depende solo de la contraseña del nivel 4. Eso explica por qué cada contraseña del nivel 4 me daba un código distinto. Para asegurarme de haberlo entendido, lo reconstruí en Python con los valores de mi primera corrida completa:

```python
import zlib

def fnv(h, s):
    for c in s:
        h = ((h ^ c) * 0x1000193) & 0xffffffff
    return h

username = b"Bruno"
lvl1 = b"0038D6D8"           # level1_hash with %08X
lvl2 = b"D31B"               # level2_word with %04X
lvl3 = b"AFB0685485B42B8B"
lvl4 = b"Bruno_W!59*."

h = 0x811c9dc5
for s in [username, lvl1, lvl2, lvl3, lvl4]:
    h = fnv(h, s)

print("%08X-%08X" % (h, zlib.crc32(lvl4)))
```
{: file="completion.py" }

```text
937F16DE-EB8C1201
```

`937F16DE-EB8C1201`, el mismo código que imprimió el binario después de `Bruno_W!59*.`. Ese es el que envié, de esta corrida:

| Nivel | Lo que ingresé |
| --- | --- |
| 1 | Usuario `Bruno`, serial `0038D6D8` |
| 2 | Serial `D31B-84B1-76D4-217E` |
| 3 | Usuario `Bruno`, contraseña `AFB0685485B42B8B` |
| 4 | Contraseña `Bruno_W!59*.` |

Código de finalización: `937F16DE-EB8C1201`

## Conclusiones

El nivel 4 fue, de lejos, la parte más difícil del challenge. Entender la VM ya me llevó un buen rato: tuve que descifrar el bytecode, deducir las instrucciones de a un case, decodificarlo a mano y trazarlo en x64dbg. Pero incluso una vez que entendí exactamente qué hacía el bytecode, pasé horas buscando una forma de sacarle una contraseña. Ir hacia atrás me permitió calcular el primer carácter a partir de los otros once. La fuerza bruta terminó encontrando una contraseña válida, pero yo quería una forma más rápida de conseguir una que empezara con mi nombre. Estuve bastante perdido hasta que me crucé con el ataque meet-in-the-middle. Poder elegir los primeros 6 caracteres y obtener cientos de contraseñas válidas en menos de un segundo fue, sin duda, el momento más satisfactorio de todo el challenge.

Lo que me llevo:

- MinGW-w64 linkea sus propias funciones stdio dentro del binario, así que `printf` y compañía aparecen como funciones locales en lugar de imports.
- A Binary Ninja hay que avisarle cuando una función es variádica; si no, las llamadas terminan anidadas unas dentro de otras.
- Una VM de bytecode se puede leer de a un handler, y una vez armada la tabla de opcodes, el bytecode se puede decodificar a mano.
- Los breakpoints de log de x64dbg permiten trazar una corrida completa de la VM sin frenarla.
- La página de códigos de la consola decide qué bytes recibe un programa, así que un keygen tiene que codificar su entrada de la misma forma.

El método importó tanto como cualquiera de esas cosas: leer el código primero, confirmar cada suposición en el debugger, no parchear el binario y buscar todas las entradas válidas en lugar de una sola. La mayoría de los casos borde de este writeup salieron de esa última regla.

Por último, gracias al equipo de Binary Gecko por armar este challenge. Fue muy divertido de resolver, y aprendí más de lo que esperaba.
