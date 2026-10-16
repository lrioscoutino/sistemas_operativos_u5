# Unidad 5 — Sistemas de archivos

**Materia:** Sistemas Operativos (AEC-1061)
**Grupo:** S5C — Aula E10
**Horario:** Martes 18:00–20:00 · Jueves 17:00–19:00
**Enfoque:** teórico-conceptual, con comandos reales sobre un sistema de archivos `ext4` de Ubuntu — sin simulaciones

Continuación directa de la [Unidad 4](../sistemas_operativos_u4/README.md): ahí viste cómo el núcleo administra los dispositivos de E/S. Esta unidad entra al sistema de archivos — la capa que convierte una tira de bloques sin estructura en archivos con nombre, carpetas, permisos, y mecanismos para sobrevivir a una falla de energía a la mitad de una escritura.

## Contenido

- [02_practica_sistemas_archivos.md](02_practica_sistemas_archivos.md) — Práctica guiada paso a paso, **100% sobre Ubuntu real** (ningún código simulado): 5.3 inodos reales con hard links, 5.4/5.6 archivos dispersos y asignación bajo demanda, 5.4/5.6 fragmentación real medida con `filefrag`, 5.8 el journal de ext4 (`jbd2`) corriendo de verdad en el kernel de esta máquina, y 5.6 inodos como recurso agotable (`df -i`)

## Nota sobre el código

A diferencia de las Unidades 3 y 4 (simulaciones en Python verificadas), esta práctica usa **comandos reales de Ubuntu sobre el sistema de archivos `ext4` de la propia máquina** — nada es una simulación. Todos los comandos corren con permisos de usuario normal, sin `sudo`, incluida la evidencia del journal del kernel vía `/sys/fs/ext4/`. Cada salida mostrada en el documento es la salida real obtenida al ejecutar los comandos.

## Fuentes

- Programa sinóptico oficial AEC-1061 — TecNM
- Tanenbaum, A. — *Sistemas Operativos Modernos* (3ra ed.) — inodos, asignación de espacio, journaling
- Stallings, W. — *Sistemas Operativos* (5ta ed.)
- `man 5 ext4`, documentación del kernel de Linux (`Documentation/filesystems/ext4/`)
