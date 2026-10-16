# Práctica guiada: Sistemas de archivos, directo sobre Ubuntu real

A diferencia de las prácticas de Memoria (Unidad 3) y E/S (Unidad 4), que simulaban el comportamiento con código Python, aquí **no vas a simular nada** — vas a usar comandos reales de Ubuntu para observar, con tus propios ojos, cómo el sistema de archivos `ext4` de tu propia máquina organiza, asigna y protege tus datos. Todo lo que ves en esta práctica ya está pasando en tu computadora en este momento, aunque nunca lo hayas mirado.

Requisito: una terminal de Ubuntu (o cualquier Linux con `ext4`). **No necesitas `sudo`** — todo se hace con permisos de usuario normal, a propósito, para que puedas correrlo exactamente como está sin pedirle la contraseña a nadie.

---

## Antes de empezar: ¿qué es un sistema de archivos, de verdad?

Cuando guardas un archivo, tu disco no sabe nada de "carpetas" ni "nombres" — un disco es, por dentro, solo una tira larguísima de bloques numerados donde se pueden escribir y leer bytes. El **sistema de archivos** (ext4, NTFS, APFS, etc.) es la capa de software que inventa, sobre esa tira de bloques sin estructura, la ilusión ordenada de archivos con nombre, carpetas, permisos y tamaños — y que además se las arregla para que, si se va la luz a la mitad de una escritura, no se corrompa todo el disco.

Tres piezas que vas a tocar hoy:

- **Inodo:** la ficha real de un archivo — dueño, permisos, tamaño, fecha, y **dónde están sus bloques de datos en el disco**. El nombre del archivo (`foto.jpg`) no es el archivo: es solo una entrada en un directorio que apunta a un número de inodo.
- **Bloque:** la unidad mínima de espacio que el sistema de archivos reparte (4096 bytes en este `ext4`). Un archivo de 1 byte igual ocupa un bloque completo.
- **Journal (bitácora):** un diario de "voy a hacer esto" que el sistema de archivos escribe *antes* de modificar los datos de verdad, para poder recuperarse si la máquina se apaga a medio camino.

```
Tú escribes:  /home/usuario/documentos/tarea.txt
                     │
                     ▼
El directorio "documentos" solo guarda:  "tarea.txt" -> inodo 7345447
                                                              │
                                                              ▼
                                          El inodo 7345447 guarda:
                                          dueño, permisos, tamaño,
                                          fechas, y la lista de
                                          bloques donde están
                                          los bytes reales.
```

---

## Paso 1 — El inodo es el archivo real; el nombre es solo una etiqueta

### El problema

Mucha gente asume que "borrar un archivo" borra sus datos de inmediato, y que el nombre y el contenido son la misma cosa. Ninguna de las dos cosas es cierta. Vamos a comprobarlo.

### Lo que vas a correr

```bash
mkdir -p ~/practica_fs && cd ~/practica_fs

echo "hola mundo" > archivo1.txt
ls -i archivo1.txt                          # -i muestra el numero de inodo
stat archivo1.txt | head -4

echo "--- creamos un hard link: otro nombre apuntando AL MISMO inodo ---"
ln archivo1.txt archivo1_link.txt
ls -li archivo1.txt archivo1_link.txt
stat -c "%n inode=%i nlink=%h" archivo1.txt archivo1_link.txt

echo "--- borramos el nombre original ---"
rm archivo1.txt
cat archivo1_link.txt                        # el contenido SIGUE AHI
stat -c "%n inode=%i nlink=%h" archivo1_link.txt
```

### Salida real (verificada en esta misma máquina)

```
7345447 archivo1.txt
  Fichero: archivo1.txt
  Tamaño: 11        	Bloques: 8          Bloque E/S: 4096   fichero regular
Device: 259,2	Inode: 7345447     Links: 1
--- creamos un hard link: otro nombre apuntando AL MISMO inodo ---
7345447 -rw-rw-r-- 2 luiscoutino luiscoutino 11 oct 15 18:47 archivo1_link.txt
7345447 -rw-rw-r-- 2 luiscoutino luiscoutino 11 oct 15 18:47 archivo1.txt
archivo1.txt inode=7345447 nlink=2
archivo1_link.txt inode=7345447 nlink=2
--- borramos el nombre original ---
hola mundo
archivo1_link.txt inode=7345447 nlink=1
```

### Qué acaba de pasar

`archivo1.txt` y `archivo1_link.txt` **comparten el mismo número de inodo** (`7345447`): no son dos copias del archivo, son **dos nombres distintos apuntando al mismo contenido**. El campo `nlink` ("número de enlaces") cuenta cuántos nombres apuntan a ese inodo — subió a `2` al crear el link.

Al borrar `archivo1.txt`, lo único que pasó fue que **desapareció esa entrada del directorio** (el nombre) — `nlink` bajó a `1`, pero el inodo y sus datos siguen vivos porque todavía hay un nombre (`archivo1_link.txt`) apuntándole. Esto es, literalmente, cómo funciona `rm` en Linux: **borrar un archivo de verdad significa que `nlink` llega a 0** (y nadie lo tiene abierto). Mientras quede al menos un nombre o un proceso con el archivo abierto, los datos permanecen en el disco.

### Ejercicio para ti

Corre `rm archivo1_link.txt` y luego intenta hacer `cat` sobre ese inodo — ya no hay forma de acceder a él por nombre (aunque, hasta que el sistema reutilice esos bloques, los datos físicos siguen ahí; esto es precisamente lo que aprovechan las herramientas de "recuperación de archivos borrados"). ¿Por qué crees que un archivo que sigue *abierto* por un proceso en el momento de borrarlo (`nlink=0` pero con un *file descriptor* activo) no libera su espacio hasta que ese proceso lo cierra?

---

## Paso 2 — Archivos dispersos (*sparse*): el sistema de archivos no reserva lo que no usas

### El problema

¿Qué significa realmente que un archivo "mida 1 GB"? ¿El sistema de archivos reserva 1 GB de bloques reales de inmediato, o solo anota "este archivo llega hasta el byte 1,073,741,824" y reserva bloques **solo cuando de verdad escribes algo ahí**?

### Lo que vas a correr

```bash
cd ~/practica_fs

truncate -s 1G archivo_disperso.img          # "crea" un archivo de 1GB, sin escribir nada

echo "=== ls -lh: tamaño APARENTE ==="
ls -lh archivo_disperso.img

echo "=== du -h: bloques REALES ocupados en disco ==="
du -h archivo_disperso.img

echo "=== ahora escribimos 4KB de verdad, en medio del archivo ==="
dd if=/dev/urandom of=archivo_disperso.img bs=4096 count=1 seek=100000 conv=notrunc status=none

echo "=== tras escribir 4KB reales ==="
ls -lh archivo_disperso.img
du -h archivo_disperso.img
```

### Salida real (verificada)

```
=== ls -lh: tamaño APARENTE ===
-rw-rw-r-- 1 luiscoutino luiscoutino 1,0G oct 15 18:47 archivo_disperso.img
=== du -h: bloques REALES ocupados en disco ===
0	archivo_disperso.img
=== ahora escribimos 4KB de verdad, en medio del archivo ===
=== tras escribir 4KB reales ===
-rw-rw-r-- 1 luiscoutino luiscoutino 1,0G oct 15 18:47 archivo_disperso.img
4,0K	archivo_disperso.img
```

### Qué acaba de pasar

`ls -lh` dice que el archivo mide **1 GB** — ese es su tamaño *lógico*, lo que el inodo declara como "hasta dónde llega el archivo". Pero `du -h` (que suma los bloques *reales* asignados en disco) dice **0 bytes** antes de escribir nada, y **4 KB** después de escribir un solo bloque en medio del archivo.

El sistema de archivos nunca reservó el GB completo: solo creó un hueco lógico, y fue asignando bloques reales únicamente donde de verdad hubo una escritura. Esto se llama **archivo disperso (*sparse file*)**, y es exactamente la técnica que usan las máquinas virtuales (un disco virtual de "40 GB" que en realidad ocupa 3 GB reales) y las bases de datos que pre-reservan espacio de archivo sin "gastarlo" todavía.

### Ejercicio para ti

Escribe otro bloque de 4KB con `seek=500000` (mucho más adelante en el archivo) y vuelve a medir con `du -h`. ¿Cuánto espacio real ocupa ahora? ¿Por qué no es la diferencia entre 500000 y 100000 multiplicada por el tamaño de bloque?

---

## Paso 3 — Fragmentación real, medida con `filefrag`

### El problema

Cuando un archivo se escribe de corrido, el sistema de archivos puede darle bloques consecutivos ("contiguos"). Pero si varios archivos crecen **al mismo tiempo, intercalados**, el sistema de archivos no tiene de otra que repartirles bloques en zonas separadas del disco — el archivo queda "fragmentado" en varios pedazos (*extents*). Vamos a provocarlo a propósito y medirlo con la herramienta real que usan los administradores de sistemas.

### Lo que vas a correr

```bash
cd ~/practica_fs

echo "=== Archivo escrito de corrido (sin interrupciones) ==="
dd if=/dev/zero of=ordenado.dat bs=1M count=20 status=none
filefrag ordenado.dat

echo ""
echo "=== Dos archivos escritos INTERCALADOS, forzando al disco a repartir bloques ==="
rm -f frag_a.dat frag_b.dat
: > frag_a.dat; : > frag_b.dat
for i in $(seq 1 30); do
  dd if=/dev/urandom bs=1M count=1 >> frag_a.dat 2>/dev/null
  sync frag_a.dat                 # fuerza a escribir YA, para que no se pueda re-ordenar despues
  dd if=/dev/urandom bs=1M count=1 >> frag_b.dat 2>/dev/null
  sync frag_b.dat
done
filefrag frag_a.dat frag_b.dat
```

### Salida real (verificada)

```
=== Archivo escrito de corrido (sin interrupciones) ===
ordenado.dat: 1 extent found

=== Dos archivos escritos INTERCALADOS, forzando al disco a repartir bloques ===
frag_a.dat: 30 extents found
frag_b.dat: 30 extents found
```

### Qué acaba de pasar

El archivo escrito de un tirón (`ordenado.dat`, 20 MB) quedó en **1 sola extensión** — el sistema de archivos le dio bloques consecutivos porque no tuvo que atender a nadie más mientras lo escribía.

Los dos archivos escritos **alternando** un MB de uno, un MB del otro, forzando cada escritura al disco con `sync` antes de seguir, quedaron en **30 extensiones cada uno** — una por cada MB escrito. Cada vez que el sistema de archivos tuvo que decidir "¿dónde pongo este MB nuevo?", el espacio contiguo que acababa de usar ya se lo había quedado el otro archivo, así que tuvo que buscar hueco en otra parte del disco.

Esto es fragmentación real, no un concepto de libro: un archivo fragmentado tarda más en leerse completo porque el cabezal del disco (en un HDD mecánico) o el controlador (en un SSD) tiene que saltar entre zonas distintas en vez de leer todo seguido. Es exactamente lo que resuelven los "desfragmentadores" de Windows de hace años — algo que casi nunca hace falta en Linux con ext4 porque su algoritmo de asignación (con *delayed allocation*, que agrupa escrituras antes de decidir dónde van) evita fragmentar en la mayoría de los casos normales, como viste en el archivo `ordenado.dat`.

### Ejercicio para ti

Corre `filefrag -v frag_a.dat` (con `-v` de verboso) y mira la columna `physical_offset` de cada extensión — son los números de bloque reales en el disco. ¿Los bloques de `frag_a.dat` y `frag_b.dat` quedaron intercalados en el disco, o agrupados por separado? Usa `filefrag -v frag_b.dat` para comparar.

---

## Paso 4 — El journal de ext4, corriendo ahora mismo en tu máquina

### El problema

Si se va la luz a la mitad de una escritura al disco, ¿qué evita que el sistema de archivos quede en un estado corrupto a medias? La respuesta es el **journal** (bitácora): antes de modificar los datos reales, `ext4` escribe primero en una zona especial del disco "voy a hacer este cambio" — y si la máquina se reinicia a medio camino, al arrancar de nuevo revisa el journal y termina o deshace lo que quedó pendiente, en vez de dejar el sistema de archivos en un estado inconsistente.

Esto normalmente se investiga con herramientas que piden `sudo` (`dumpe2fs`, `tune2fs`) — aquí vamos a encontrar evidencia real **sin pedir contraseña**, usando lo que el kernel ya expone en `/sys`.

### Lo que vas a correr

```bash
# Encuentra el disco donde esta montado "/" (ajusta el nombre si el tuyo es distinto)
df -T / 
DISCO=$(findmnt -n -o SOURCE / | xargs basename)   # ej. nvme0n1p2

echo "=== PID del hilo del kernel que mantiene el journal de este disco ==="
cat /sys/fs/ext4/$DISCO/journal_task

echo "=== que proceso es ese PID, de verdad ==="
PID=$(cat /sys/fs/ext4/$DISCO/journal_task)
ps -p $PID -o pid,comm
cat /proc/$PID/comm
```

### Salida real (verificada en esta máquina)

```
=== PID del hilo del kernel que mantiene el journal de este disco ===
509
=== que proceso es ese PID, de verdad ===
    PID COMMAND
    509 jbd2/nvme0n1p2-8
jbd2/nvme0n1p2-8
```

### Qué acaba de pasar

`/sys/fs/ext4/<disco>/journal_task` no es un archivo normal — es una ventana que el kernel de Linux deja abierta hacia su propio estado interno. Te devolvió el PID `509`, y al preguntarle al sistema "¿qué proceso es el 509?", la respuesta fue `jbd2/nvme0n1p2-8`.

**`jbd2`** significa *Journaling Block Device 2* — es el subsistema del kernel de Linux encargado de llevar el journal de los sistemas de archivos `ext3`/`ext4`. No es una simulación ni una explicación de libro: es un hilo de kernel **corriendo ahora mismo en tu computadora**, protegiendo cada escritura que haces a `/` mientras lees esto. Cada vez que modificas un archivo, `jbd2` escribe primero un registro en el journal antes de dejar que el cambio real llegue a su lugar definitivo — por eso, si tu laptop se queda sin batería a la mitad de guardar un archivo, lo normal es que al prender de nuevo el sistema arranque limpio (recuperando desde el journal) en vez de pedirte un `fsck` largo y arriesgado como pasaba con sistemas de archivos sin journal (como `ext2`, el antecesor de `ext4`).

### Ejercicio para ti

Corre `cat /proc/$PID/status | head -5` sobre el PID que encontraste — ¿qué tipo de proceso parece ser (fíjate en el campo `State` y en que no aparece en `ps aux` como un programa normal con un ejecutable en disco)? Investiga (sin `sudo`) si tu sistema tiene más de un hilo `jbd2` corriendo con `ps aux | grep jbd2` — ¿por qué podría haber más de uno si tienes más de un disco o partición con `ext4`?

---

## Paso 5 — Cuántos archivos le quedan a tu disco (inodos, no espacio)

### El problema

"Se quedó sin espacio" normalmente significa que los bloques de datos se acabaron. Pero hay una segunda forma, menos conocida, de quedarse sin espacio: **acabarse los inodos**. Como cada archivo necesita su propia ficha (inodo) sin importar qué tan chiquito sea, un disco puede tener bloques libres de sobra y aun así no poder crear ni un archivo más, si ya usó todos sus inodos — típicamente porque hay millones de archivos diminutos (cachés, sesiones, logs rotados).

### Lo que vas a correr

```bash
echo "=== espacio en bloques (lo que normalmente ves) ==="
df -h /

echo "=== espacio en INODOS (la ficha de cada archivo) ==="
df -i /
```

### Salida real (verificada)

```
=== espacio en INODOS (la ficha de cada archivo) ===
S.ficheros      Nodos-i NUsados  NLibres NUso% Montado en
/dev/nvme0n1p2 31195136 4155785 27039351   14% /
```

### Qué acaba de pasar

Esta máquina tiene **31,195,136 inodos en total** en la partición raíz, de los cuales ya se usaron 4,155,785 (14%). Fíjate que este número **no tiene nada que ver con cuántos bytes de datos hay** — es el límite de *cuántos archivos* puede tener este sistema de archivos, sin importar qué tan pequeños sean. Ese número de inodos totales se decide **al crear el sistema de archivos** (con `mkfs.ext4`) y normalmente no se puede cambiar después sin reformatear — por eso un servidor que maneja millones de archivos pequeños (por ejemplo, una caché de miniaturas de imágenes) a veces falla con "no space left on device" con `df -h` mostrando espacio de sobra: el problema real es `df -i` mostrando 100% de inodos usados.

### Ejercicio para ti

Compara: si esta partición usa bloques de 4096 bytes y tiene 31,195,136 inodos, ¿cuántos archivos como mínimo (de 1 byte cada uno) podrían llenar todos los inodos antes de llenar todo el espacio en bloques? (Pista: compara el total de inodos contra el total de bloques que viste en el Paso 1 con `stat -f /`.)

---

## Resumen de lo que cubriste

| Tema del programa oficial | Dónde lo viste |
|---|---|
| 5.1-5.2 Concepto, archivo real | Introducción — el diagrama nombre→inodo→bloques |
| 5.3 Componentes de un sistema de archivos | Paso 1 — el inodo como la ficha real del archivo |
| 5.4 Organización lógica y física | Paso 2 (tamaño lógico vs. bloques físicos reales) y Paso 3 (extensiones/fragmentación) |
| 5.5 Mecanismos de acceso a los archivos | Paso 1 (nombre→inodo es el mecanismo de acceso por nombre) |
| 5.6 Manejo de espacio en memoria secundaria | Paso 2 (asignación bajo demanda), Paso 3 (fragmentación), Paso 5 (inodos como recurso agotable) |
| 5.7 Modelo jerárquico | Diagrama de introducción — directorio→nombre→inodo |
| 5.8 Mecanismos de recuperación en caso de falla | Paso 4 — el journal (`jbd2`) corriendo de verdad en el kernel |
