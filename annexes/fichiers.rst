.. This file is part of "Présentation du noyau de Redox OS".

..     Copyright (C) 2018 Julien Férard

..     "Présentation du noyau de Redox OS" is free software: you can redistribute it and/or modify
..     it under the terms of the GNU General Public License as published by
..     the Free Software Foundation, either version 3 of the License, or
..     (at your option) any later version.

..     "Présentation du noyau de Redox OS" is distributed in the hope that it will be useful,
..     but WITHOUT ANY WARRANTY; without even the implied warranty of
..     MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
..     GNU General Public License for more details.

..     You should have received a copy of the GNU General Public License
..     along with Foobar.  If not, see <https://www.gnu.org/licenses/>

Liste sélective des fichiers du noyau
=====================================

Fichiers divers
---------------
`.gitignore <https://gitlab.redox-os.org/redox-os/kernel/blob/master/.gitignore>`_, `.gitmodules <https://gitlab.redox-os.org/redox-os/kernel/blob/master/.gitmodules>`_, `ARM-AARCH64-PORT-OUTLINE.md <https://gitlab.redox-os.org/redox-os/kernel/blob/master/ARM-AARCH64-PORT-OUTLINE.md>`_, `Cargo.lock <https://gitlab.redox-os.org/redox-os/kernel/blob/master/Cargo.lock>`_, `Cargo.toml <https://gitlab.redox-os.org/redox-os/kernel/blob/master/Cargo.toml>`_, `LICENSE <https://gitlab.redox-os.org/redox-os/kernel/blob/master/LICENSE>`_, `README.md <https://gitlab.redox-os.org/redox-os/kernel/blob/master/README.md>`_, `Xargo.toml <https://gitlab.redox-os.org/redox-os/kernel/blob/master/Xargo.toml>`_, `clippy.sh <https://gitlab.redox-os.org/redox-os/kernel/blob/master/clippy.sh>`_.

* `build.rs <https://gitlab.redox-os.org/redox-os/kernel/blob/master/build.rs>`_ : sert à construire le fichier `gen.rs`. TODO

* `res/unifont.font <https://gitlab.redox-os.org/redox-os/kernel/blob/master/res/unifont.font>`_

linkers
-------
* `linkers/x86_64.ld <https://gitlab.redox-os.org/redox-os/kernel/blob/master/linkers/x86_64.ld>`_ : le script du linker n'utilise que la commande `SECTIONS` afin de définir la dispotion mémoire du fichier cible : addresse du code, des données, etc.

sous-module slab_allocator
--------------------------
L'allocation de `slab` est une technique qui permet de conserver des objets du noyau dans un pool pour le réutiliser. Cela permet d'éviter d'allouer un nouveau cadre de page pour tout nouvel objet.

* `slab_allocator/src/lib.rs <https://gitlab.redox-os.org/redox-os/slab_allocator/blob/master/src/lib.rs>`_ : ce module créé un tas et permet d'allouer des données à des objets en fonction de leur taille. On fournit un :code:`core::alloc::Layout` (en gros, une taille d'objet) et le tas sélectionne le Slab adéquat puis y prend la place nécessaire.
* `slab_allocator/src/slab.rs <https://gitlab.redox-os.org/redox-os/slab_allocator/blob/master/src/slab.rs>`_ : au départ, un :code:`Slab` est composé d'une liste de blocs libres de taille donnée.

acpi
----
ACPI signifie *Advanced Configuration and Power Interface Specification*. Il s'agit d'une interface permettant à l'OS d'nevoyer des signaux aux périphériques pour les mettre hors tension, afin d'économiser de l'énergie.

acpi/aml
~~~~~~~~
AML signifie *ACPI Machine Language*. Il s'agit d'un bytecode que l'OS doit parser pour comprendre les tables ACPI et récupérer des informations sur les CPU et les pérphériques. Par exemple : `kernel/src/arch/x86_64/start.rs`.

* `src/acpi/aml/dataobj.rs <https://gitlab.redox-os.org/redox-os/kernel/blob/master/src/acpi/aml/dataobj.rs>`_
* `src/acpi/aml/mod.rs <https://gitlab.redox-os.org/redox-os/kernel/blob/master/src/acpi/aml/mod.rs>`_
* `src/acpi/aml/namedobj.rs <https://gitlab.redox-os.org/redox-os/kernel/blob/master/src/acpi/aml/namedobj.rs>`_
* `src/acpi/aml/namespace.rs <https://gitlab.redox-os.org/redox-os/kernel/blob/master/src/acpi/aml/namespace.rs>`_
* `src/acpi/aml/namespacemodifier.rs <https://gitlab.redox-os.org/redox-os/kernel/blob/master/src/acpi/aml/namespacemodifier.rs>`_
* `src/acpi/aml/namestring.rs <https://gitlab.redox-os.org/redox-os/kernel/blob/master/src/acpi/aml/namestring.rs>`_
* `src/acpi/aml/parser.rs <https://gitlab.redox-os.org/redox-os/kernel/blob/master/src/acpi/aml/parser.rs>`_
* `src/acpi/aml/parsermacros.rs <https://gitlab.redox-os.org/redox-os/kernel/blob/master/src/acpi/aml/parsermacros.rs>`_
* `src/acpi/aml/pkglength.rs <https://gitlab.redox-os.org/redox-os/kernel/blob/master/src/acpi/aml/pkglength.rs>`_
* `src/acpi/aml/termlist.rs <https://gitlab.redox-os.org/redox-os/kernel/blob/master/src/acpi/aml/termlist.rs>`_
* `src/acpi/aml/type1opcode.rs <https://gitlab.redox-os.org/redox-os/kernel/blob/master/src/acpi/aml/type1opcode.rs>`_
* `src/acpi/aml/type2opcode.rs <https://gitlab.redox-os.org/redox-os/kernel/blob/master/src/acpi/aml/type2opcode.rs>`_

acpi/dmar
~~~~~~~~~
DMAR signifie *DMA Remapping Table*, où DMA signifie *Direct Memory Access*. Certains périphériques ont accès à la mémoire centrale indépendamment du processeur (DMA). Voir IOMMU (*Input–Output Memory Management Unit*).

* `src/acpi/dmar/drhd.rs <https://gitlab.redox-os.org/redox-os/kernel/blob/master/src/acpi/dmar/drhd.rs>`_ : DRHD : *DMA Remapping Hardware Unit Definition*
* `src/acpi/dmar/mod.rs <https://gitlab.redox-os.org/redox-os/kernel/blob/master/src/acpi/dmar/mod.rs>`_

Autres
~~~~~~
* `src/acpi/fadt.rs <https://gitlab.redox-os.org/redox-os/kernel/blob/master/src/acpi/fadt.rs>`_ : FADT (*Fixed ACPI Description Table*)
* `src/acpi/hpet.rs <https://gitlab.redox-os.org/redox-os/kernel/blob/master/src/acpi/hpet.rs>`_ : HPET (*High Precision Event Timer*)
* `src/acpi/madt.rs <https://gitlab.redox-os.org/redox-os/kernel/blob/master/src/acpi/madt.rs>`_ : MADT (*Multiple APIC Description Table*)
* `src/acpi/mod.rs <https://gitlab.redox-os.org/redox-os/kernel/blob/master/src/acpi/mod.rs>`_
* `src/acpi/rsdp.rs <https://gitlab.redox-os.org/redox-os/kernel/blob/master/src/acpi/rsdp.rs>`_ : RSDP (*Root System Description Pointer*)
* `src/acpi/rsdt.rs <https://gitlab.redox-os.org/redox-os/kernel/blob/master/src/acpi/rsdt.rs>`_ : RSDT (*Root System Description Table*)
* `src/acpi/rxsdt.rs <https://gitlab.redox-os.org/redox-os/kernel/blob/master/src/acpi/rxsdt.rs>`_ :
* `src/acpi/sdt.rs <https://gitlab.redox-os.org/redox-os/kernel/blob/master/src/acpi/sdt.rs>`_
* `src/acpi/xsdt.rs <https://gitlab.redox-os.org/redox-os/kernel/blob/master/src/acpi/xsdt.rs>`_ : XSDT (*eXtended System Descriptor Table*)

allocator
---------
Une enveloppe pour le slab-allocator, ou bien utilise une liste chaînée. Repose sur les bibliothèques de de Phil Opp. Ce module permet d'allouer des objets de taille définie dans un bloc (slab).

* `src/allocator/linked_list.rs <https://gitlab.redox-os.org/redox-os/kernel/blob/master/src/allocator/linked_list.rs>`_ : à défaut de Slab.
* `src/allocator/mod.rs <https://gitlab.redox-os.org/redox-os/kernel/blob/master/src/allocator/mod.rs>`_
* `src/allocator/slab.rs <https://gitlab.redox-os.org/redox-os/kernel/blob/master/src/allocator/slab.rs>`_ : si Slab.

arch
----
Tout ce qui est dépendant de l'architecture matérielle.

* `src/arch/mod.rs <https://gitlab.redox-os.org/redox-os/kernel/blob/master/src/arch/mod.rs>`_

arch/x86_64
~~~~~~~~~~~
* `src/arch/x86_64/debug.rs <https://gitlab.redox-os.org/redox-os/kernel/blob/master/src/arch/x86_64/debug.rs>`_ : envoie du texte sur le port COM1 et éventuellement sur un :code:`graphical_debug`.
* `src/arch/x86_64/gdt.rs <https://gitlab.redox-os.org/redox-os/kernel/blob/master/src/arch/x86_64/gdt.rs>`_ : GDT = global descriptor table, la table des segments globaux. La segmentation n'a plus de rôle actif mais doit être prise en compte (un seul segment pour tout = adresses matérielles).
* `src/arch/x86_64/idt.rs <https://gitlab.redox-os.org/redox-os/kernel/blob/master/src/arch/x86_64/idt.rs>`_ : IDT = interrupt descriptor table. Définit les différentes interruptions.
* `src/arch/x86_64/ipi.rs <https://gitlab.redox-os.org/redox-os/kernel/blob/master/src/arch/x86_64/ipi.rs>`_ : IPI = Inter-processor interrupt. Permet d'envoyer des commandes à l'APIC.
* `src/arch/x86_64/macros.rs <https://gitlab.redox-os.org/redox-os/kernel/blob/master/src/arch/x86_64/macros.rs>`_ : macros diverses.
* `src/arch/x86_64/mod.rs <https://gitlab.redox-os.org/redox-os/kernel/blob/master/src/arch/x86_64/mod.rs>`_
* `src/arch/x86_64/pti.rs <https://gitlab.redox-os.org/redox-os/kernel/blob/master/src/arch/x86_64/pti.rs>`_ : PTI = Page Table Isolation. Mesure pour ne mapper dans l'espace d'adressage utilisateur que la partie minimale du noyau, afin de se protéger de la faille Meltdown (x86).
* `src/arch/x86_64/start.rs <https://gitlab.redox-os.org/redox-os/kernel/blob/master/src/arch/x86_64/start.rs>`_ : contient la fonction :code:`kstart` qui est le lien avec le booter.
* `src/arch/x86_64/stop.rs <https://gitlab.redox-os.org/redox-os/kernel/blob/master/src/arch/x86_64/stop.rs>`_ : contient les fonctions :code:`kstop` et :code:`kreset` pour interrompre Redox.

arch/x86_64/device
******************
Concerne les périphériques

* `src/arch/x86_64/device/cpu.rs <https://gitlab.redox-os.org/redox-os/kernel/blob/master/src/arch/x86_64/device/cpu.rs>`_ : une fonction :code:`cpu_info` pour écrire les information sur le CPU.
* `src/arch/x86_64/device/hpet.rs <https://gitlab.redox-os.org/redox-os/kernel/blob/master/src/arch/x86_64/device/hpet.rs>`_ : HPET = *High Precision Event Timer*. Initialisation de l'horologe.
* `src/arch/x86_64/device/local_apic.rs <https://gitlab.redox-os.org/redox-os/kernel/blob/master/src/arch/x86_64/device/local_apic.rs>`_ : APIC = Advanced Programmable Interrupt Controller. Initialisation et commandes.
* `src/arch/x86_64/device/mod.rs <https://gitlab.redox-os.org/redox-os/kernel/blob/master/src/arch/x86_64/device/mod.rs>`_
* `src/arch/x86_64/device/pic.rs <https://gitlab.redox-os.org/redox-os/kernel/blob/master/src/arch/x86_64/device/pic.rs>`_ : PIC = Programmable Interrupt Controller. Manager des interruptions.
* `src/arch/x86_64/device/pit.rs <https://gitlab.redox-os.org/redox-os/kernel/blob/master/src/arch/x86_64/device/pit.rs>`_ : PIT = *Programmable Interval Timer*. Démarre le timer.
* `src/arch/x86_64/device/rtc.rs <https://gitlab.redox-os.org/redox-os/kernel/blob/master/src/arch/x86_64/device/rtc.rs>`_ : RTC = *Real Time Clock*.
* `src/arch/x86_64/device/serial.rs <https://gitlab.redox-os.org/redox-os/kernel/blob/master/src/arch/x86_64/device/serial.rs>`_ : les ports série COM1 et COM2.

arch/x86_64/graphical_debug
***************************
* `src/arch/x86_64/graphical_debug/debug.rs <https://gitlab.redox-os.org/redox-os/kernel/blob/master/src/arch/x86_64/graphical_debug/debug.rs>`_
* `src/arch/x86_64/graphical_debug/display.rs <https://gitlab.redox-os.org/redox-os/kernel/blob/master/src/arch/x86_64/graphical_debug/display.rs>`_
* `src/arch/x86_64/graphical_debug/mod.rs <https://gitlab.redox-os.org/redox-os/kernel/blob/master/src/arch/x86_64/graphical_debug/mod.rs>`_
* `src/arch/x86_64/graphical_debug/mode_info.rs <https://gitlab.redox-os.org/redox-os/kernel/blob/master/src/arch/x86_64/graphical_debug/mode_info.rs>`_
* `src/arch/x86_64/graphical_debug/primitive.rs <https://gitlab.redox-os.org/redox-os/kernel/blob/master/src/arch/x86_64/graphical_debug/primitive.rs>`_

arch/x86_64/interrupt
*********************
Les interruptions

* `src/arch/x86_64/interrupt/exception.rs <https://gitlab.redox-os.org/redox-os/kernel/blob/master/src/arch/x86_64/interrupt/exception.rs>`_
* `src/arch/x86_64/interrupt/ipi.rs <https://gitlab.redox-os.org/redox-os/kernel/blob/master/src/arch/x86_64/interrupt/ipi.rs>`_
* `src/arch/x86_64/interrupt/irq.rs <https://gitlab.redox-os.org/redox-os/kernel/blob/master/src/arch/x86_64/interrupt/irq.rs>`_
* `src/arch/x86_64/interrupt/mod.rs <https://gitlab.redox-os.org/redox-os/kernel/blob/master/src/arch/x86_64/interrupt/mod.rs>`_
* `src/arch/x86_64/interrupt/syscall.rs <https://gitlab.redox-os.org/redox-os/kernel/blob/master/src/arch/x86_64/interrupt/syscall.rs>`_
* `src/arch/x86_64/interrupt/trace.rs <https://gitlab.redox-os.org/redox-os/kernel/blob/master/src/arch/x86_64/interrupt/trace.rs>`_

arch/x86_64/paging
******************
La pagination

* `src/arch/x86_64/paging/entry.rs <https://gitlab.redox-os.org/redox-os/kernel/blob/master/src/arch/x86_64/paging/entry.rs>`_
* `src/arch/x86_64/paging/mapper.rs <https://gitlab.redox-os.org/redox-os/kernel/blob/master/src/arch/x86_64/paging/mapper.rs>`_
* `src/arch/x86_64/paging/mod.rs <https://gitlab.redox-os.org/redox-os/kernel/blob/master/src/arch/x86_64/paging/mod.rs>`_
* `src/arch/x86_64/paging/table.rs <https://gitlab.redox-os.org/redox-os/kernel/blob/master/src/arch/x86_64/paging/table.rs>`_
* `src/arch/x86_64/paging/temporary_page.rs <https://gitlab.redox-os.org/redox-os/kernel/blob/master/src/arch/x86_64/paging/temporary_page.rs>`_

common
------
Des outils utiles.

* `src/common/int_like.rs <https://gitlab.redox-os.org/redox-os/kernel/blob/master/src/common/int_like.rs>`_ : permet de définir des types qui s'appuient sur des entiers. Possibilité de types thread-safe.
* `src/common/mod.rs <https://gitlab.redox-os.org/redox-os/kernel/blob/master/src/common/mod.rs>`_

context
-------
Ce qui concerne les processus.

* `src/context/arch/x86_64.rs <https://gitlab.redox-os.org/redox-os/kernel/blob/master/src/context/arch/x86_64.rs>`_
* `src/context/context.rs <https://gitlab.redox-os.org/redox-os/kernel/blob/master/src/context/context.rs>`_
* `src/context/file.rs <https://gitlab.redox-os.org/redox-os/kernel/blob/master/src/context/file.rs>`_
* `src/context/list.rs <https://gitlab.redox-os.org/redox-os/kernel/blob/master/src/context/list.rs>`_
* `src/context/memory.rs <https://gitlab.redox-os.org/redox-os/kernel/blob/master/src/context/memory.rs>`_
* `src/context/mod.rs <https://gitlab.redox-os.org/redox-os/kernel/blob/master/src/context/mod.rs>`_
* `src/context/signal.rs <https://gitlab.redox-os.org/redox-os/kernel/blob/master/src/context/signal.rs>`_
* `src/context/switch.rs <https://gitlab.redox-os.org/redox-os/kernel/blob/master/src/context/switch.rs>`_
* `src/context/timeout.rs <https://gitlab.redox-os.org/redox-os/kernel/blob/master/src/context/timeout.rs>`_

devices
-------
Les périphériques.

* `src/devices/mod.rs <https://gitlab.redox-os.org/redox-os/kernel/blob/master/src/devices/mod.rs>`_
* `src/devices/uart_16550.rs <https://gitlab.redox-os.org/redox-os/kernel/blob/master/src/devices/uart_16550.rs>`_

memory
------
La mémoire.

* `src/memory/bump.rs <https://gitlab.redox-os.org/redox-os/kernel/blob/master/src/memory/bump.rs>`_
* `src/memory/mod.rs <https://gitlab.redox-os.org/redox-os/kernel/blob/master/src/memory/mod.rs>`_
* `src/memory/recycle.rs <https://gitlab.redox-os.org/redox-os/kernel/blob/master/src/memory/recycle.rs>`_

scheme
------
Un `Scheme` est la désignation d'une ressource. Cette ressource peut-être ouverte, liée ou déliée, ou bien se voir ajouter une sous-structure.

* `src/scheme/debug.rs <https://gitlab.redox-os.org/redox-os/kernel/blob/master/src/scheme/debug.rs>`_ : le scheme :code`"debug:"` est associé à la console (stdin, stdout, stderr) ;
* `src/scheme/event.rs <https://gitlab.redox-os.org/redox-os/kernel/blob/master/src/scheme/event.rs>`_ : le scheme :code:`"event:"` est associé aux événements du noyau ;
* `src/scheme/initfs.rs <https://gitlab.redox-os.org/redox-os/kernel/blob/master/src/scheme/initfs.rs>`_ : le scheme :code:`"initfs:"` est associé au système de fichiers à l'initialisation (en lecture seule) ;
* `src/scheme/irq.rs <https://gitlab.redox-os.org/redox-os/kernel/blob/master/src/scheme/irq.rs>`_ : le scheme :code:`"irq:"` sert à interagir avec les interruptions :
* `src/scheme/live.rs <https://gitlab.redox-os.org/redox-os/kernel/blob/master/src/scheme/live.rs>`_ : ?
* `src/scheme/memory.rs <https://gitlab.redox-os.org/redox-os/kernel/blob/master/src/scheme/memory.rs>`_ : .?
* `src/scheme/mod.rs <https://gitlab.redox-os.org/redox-os/kernel/blob/master/src/scheme/mod.rs>`_
* `src/scheme/pipe.rs <https://gitlab.redox-os.org/redox-os/kernel/blob/master/src/scheme/pipe.rs>`_ : le scheme :code:`"pipe:"` est associé aux tubes du noyau ;
* `src/scheme/root.rs <https://gitlab.redox-os.org/redox-os/kernel/blob/master/src/scheme/root.rs>`_ : ?
* `src/scheme/time.rs <https://gitlab.redox-os.org/redox-os/kernel/blob/master/src/scheme/time.rs>`_ : ?
* `src/scheme/user.rs <https://gitlab.redox-os.org/redox-os/kernel/blob/master/src/scheme/user.rs>`_ : ?

scheme/sys
~~~~~~~~~~
Les informations sur le système.

* `src/scheme/sys/context.rs <https://gitlab.redox-os.org/redox-os/kernel/blob/master/src/scheme/sys/context.rs>`_
* `src/scheme/sys/cpu.rs <https://gitlab.redox-os.org/redox-os/kernel/blob/master/src/scheme/sys/cpu.rs>`_ : une fonction :code:`resource` pour récupérer les information sur le CPU.
* `src/scheme/sys/exe.rs <https://gitlab.redox-os.org/redox-os/kernel/blob/master/src/scheme/sys/exe.rs>`_
* `src/scheme/sys/iostat.rs <https://gitlab.redox-os.org/redox-os/kernel/blob/master/src/scheme/sys/iostat.rs>`_
* `src/scheme/sys/mod.rs <https://gitlab.redox-os.org/redox-os/kernel/blob/master/src/scheme/sys/mod.rs>`_
* `src/scheme/sys/scheme.rs <https://gitlab.redox-os.org/redox-os/kernel/blob/master/src/scheme/sys/scheme.rs>`_
* `src/scheme/sys/scheme_num.rs <https://gitlab.redox-os.org/redox-os/kernel/blob/master/src/scheme/sys/scheme_num.rs>`_
* `src/scheme/sys/syscall.rs <https://gitlab.redox-os.org/redox-os/kernel/blob/master/src/scheme/sys/syscall.rs>`_
* `src/scheme/sys/uname.rs <https://gitlab.redox-os.org/redox-os/kernel/blob/master/src/scheme/sys/uname.rs>`_

Sync
----
* `src/sync/mod.rs <https://gitlab.redox-os.org/redox-os/kernel/blob/master/src/sync/mod.rs>`_
* `src/sync/wait_condition.rs <https://gitlab.redox-os.org/redox-os/kernel/blob/master/src/sync/wait_condition.rs>`_
* `src/sync/wait_map.rs <https://gitlab.redox-os.org/redox-os/kernel/blob/master/src/sync/wait_map.rs>`_
* `src/sync/wait_queue.rs <https://gitlab.redox-os.org/redox-os/kernel/blob/master/src/sync/wait_queue.rs>`_

Syscall
-------
* `src/syscall/debug.rs <https://gitlab.redox-os.org/redox-os/kernel/blob/master/src/syscall/debug.rs>`_
* `src/syscall/driver.rs <https://gitlab.redox-os.org/redox-os/kernel/blob/master/src/syscall/driver.rs>`_
* `src/syscall/fs.rs <https://gitlab.redox-os.org/redox-os/kernel/blob/master/src/syscall/fs.rs>`_
* `src/syscall/futex.rs <https://gitlab.redox-os.org/redox-os/kernel/blob/master/src/syscall/futex.rs>`_
* `src/syscall/mod.rs <https://gitlab.redox-os.org/redox-os/kernel/blob/master/src/syscall/mod.rs>`_
* `src/syscall/privilege.rs <https://gitlab.redox-os.org/redox-os/kernel/blob/master/src/syscall/privilege.rs>`_
* `src/syscall/process.rs <https://gitlab.redox-os.org/redox-os/kernel/blob/master/src/syscall/process.rs>`_
* `src/syscall/time.rs <https://gitlab.redox-os.org/redox-os/kernel/blob/master/src/syscall/time.rs>`_
* `src/syscall/validate.rs <https://gitlab.redox-os.org/redox-os/kernel/blob/master/src/syscall/validate.rs>`_

* `src/tests/mod.rs <https://gitlab.redox-os.org/redox-os/kernel/blob/master/src/tests/mod.rs>`_
* `src/time.rs <https://gitlab.redox-os.org/redox-os/kernel/blob/master/src/time.rs>`_
* `src/panic.rs <https://gitlab.redox-os.org/redox-os/kernel/blob/master/src/panic.rs>`_

Syscall
-------

* `.gitignore <https://gitlab.redox-os.org/redox-os/syscall/blob/master/.gitignore>`_
* `Cargo.toml <https://gitlab.redox-os.org/redox-os/syscall/blob/master/Cargo.toml>`_
* `LICENSE <https://gitlab.redox-os.org/redox-os/syscall/blob/master/LICENSE>`_
* `README.md <https://gitlab.redox-os.org/redox-os/syscall/blob/master/README.md>`_
* `rust-toolchain <https://gitlab.redox-os.org/redox-os/syscall/blob/master/rust-toolchain>`_

Syscall/Arch
~~~~~~~~~~~~
* `src/arch/arm.rs <https://gitlab.redox-os.org/redox-os/syscall/blob/master/src/arch/arm.rs>`_
* `src/arch/x86.rs <https://gitlab.redox-os.org/redox-os/syscall/blob/master/src/arch/x86.rs>`_
* `src/arch/x86_64.rs <https://gitlab.redox-os.org/redox-os/syscall/blob/master/src/arch/x86_64.rs>`_

* `src/call.rs <https://gitlab.redox-os.org/redox-os/syscall/blob/master/src/call.rs>`_
* `src/data.rs <https://gitlab.redox-os.org/redox-os/syscall/blob/master/src/data.rs>`_
* `src/error.rs <https://gitlab.redox-os.org/redox-os/syscall/blob/master/src/error.rs>`_
* `src/flag.rs <https://gitlab.redox-os.org/redox-os/syscall/blob/master/src/flag.rs>`_

Syscall/I/O
~~~~~~~~~~~
* `src/io/dma.rs <https://gitlab.redox-os.org/redox-os/syscall/blob/master/src/io/dma.rs>`_
* `src/io/io.rs <https://gitlab.redox-os.org/redox-os/syscall/blob/master/src/io/io.rs>`_
* `src/io/mmio.rs <https://gitlab.redox-os.org/redox-os/syscall/blob/master/src/io/mmio.rs>`_
* `src/io/mod.rs <https://gitlab.redox-os.org/redox-os/syscall/blob/master/src/io/mod.rs>`_
* `src/io/pio.rs <https://gitlab.redox-os.org/redox-os/syscall/blob/master/src/io/pio.rs>`_

Syscall/Scheme
~~~~~~~~~~~~~~
* `src/scheme/generate.sh <https://gitlab.redox-os.org/redox-os/syscall/blob/master/src/scheme/generate.sh>`_
* `src/scheme/mod.rs <https://gitlab.redox-os.org/redox-os/syscall/blob/master/src/scheme/mod.rs>`_
* `src/scheme/scheme.rs <https://gitlab.redox-os.org/redox-os/syscall/blob/master/src/scheme/scheme.rs>`_
* `src/scheme/scheme_block.rs <https://gitlab.redox-os.org/redox-os/syscall/blob/master/src/scheme/scheme_block.rs>`_
* `src/scheme/scheme_block_mut.rs <https://gitlab.redox-os.org/redox-os/syscall/blob/master/src/scheme/scheme_block_mut.rs>`_
* `src/scheme/scheme_mut.rs <https://gitlab.redox-os.org/redox-os/syscall/blob/master/src/scheme/scheme_mut.rs>`_

Syscall/Targets
~~~~~~~~~~~~~~~
* `targets/aarch64-unknown-none.json <https://gitlab.redox-os.org/redox-os/syscall/blob/master/targets/aarch64-unknown-none.json>`_
* `targets/arm-unknown-none.json <https://gitlab.redox-os.org/redox-os/syscall/blob/master/targets/arm-unknown-none.json>`_
* `targets/x86_64-unknown-none.json <https://gitlab.redox-os.org/redox-os/kernel/blob/master/targets/x86_64-unknown-none.json>`_

* `src/consts.rs <https://gitlab.redox-os.org/redox-os/kernel/blob/master/src/consts.rs>`_
* `src/elf.rs <https://gitlab.redox-os.org/redox-os/kernel/blob/master/src/elf.rs>`_
* `src/event.rs <https://gitlab.redox-os.org/redox-os/kernel/blob/master/src/event.rs>`_
* `src/externs.rs <https://gitlab.redox-os.org/redox-os/kernel/blob/master/src/externs.rs>`_
* `src/lib.rs <https://gitlab.redox-os.org/redox-os/kernel/blob/master/src/lib.rs>`_

* `src/lib.rs <https://gitlab.redox-os.org/redox-os/syscall/blob/master/src/lib.rs>`_
* `src/number.rs <https://gitlab.redox-os.org/redox-os/syscall/blob/master/src/number.rs>`_
