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
..     along with "Présentation du noyau de Redox OS".  If not, see <https://www.gnu.org/licenses/>

Mémoire -- segmentation
=======================

La segmentation est en train de devenir obsolète, notamment car elle n'est pas disponible sur toutes les plateformes matérielles. Linux par exemple, pour être portable, ne fait usage quasiment que de la pagination. Il en va de même pour Redox.

Cependant, la segmentation fait encore partie de l'architecture x64_64 et ne peut être totalement évitée à ce titre.
Lorsque le processeur x86 démarre, chaque adresse logique est composée d'une adresse de base d'un segment et d'un décalage. Il faut donc avoir défini les segments avant la mise en place de la pagination.

Les entrées de la GDT (*Global Descriptor Table*)
-------------------------------------------------
La structure clé est :code:`GdtEntry` (`kernel/src/arch/x86_64/gdt.rs <https://gitlab.redox-os.org/redox-os/kernel/tree/master/src/kernel/src/arch/x86_64/gdt.rs>`_). Elle décrit un segment (une entrée de la GDT = Global Descriptor Table, soit la table des segments globaux).

.. code-block:: rust

    #[repr(packed)]
    pub struct GdtEntry {
        pub limitl: u16,        // la limite basse du segment
        pub offsetl: u16,       // le décalage du segment
        pub offsetm: u8,        // suite décalage 1
        pub access: u8,         // les droits d'accès (ring 0-3, code/données)
        pub flags_limith: u8,   // en o/ko, et suite limite
        pub offseth: u8         // suite décalage 2
    }

Cette partie est évidemment totalement dépendante de l'architecture. A ce titre, elle se trouve dans un répertoire `arch` et la structure utilise la directive :code:`#[repr(packed)]`.

Dans le même fichier, la variable :code:`INIT_GDT` décrit 4 segments initiaux : Null, le code noyau (ring 0 = privilégié), les données noyau (ring 0) et le Thread Local Storage noyau (ring 3 : utilisateur). Ces segments sont accesibles via les registres de segments (cs, ds, ...).

La variable :code:`GDT` reprend les 4 segments précédent en y ajoutant le code utilisateur, les données utilisateur, le TLS utilisateur, le TSS (Task State Segment) utilisateur.

L'initialisation des segments
-----------------------------
Elle a pour but de neutraliser la segmentation.

La fonction :code:`init` construit une table de descripteurs de segments. Elle commence par charger la table des descripteurs de segments :code:`INIT_GDTR`, puis insère les descripteurs de segments (registres contenant les décalages) dans cette table : :code:`GDT_KERNEL_CODE` dans le segment `cs` et :code:`GDT_KERNEL_DATA` dans les autres segments (`ds`, etc.). Les sélecteurs de segment sont des décalages dans la GDT.

Lorsqu'une instruction est utilisée, le processeur détermine quel registre est concerné, en fonction du type de l'instruction. Il charge le descripteur en fonction de cette instruction.

Exemple d'utilisation : pour décoder une adresse dans le code, le matériel cherche le décalage du descripteur dans le registre du segment de code (`cs`). Ce registre contient :code:`GDT_KERNEL_CODE` (= 1), donc le matériel cherche dans l'entrée 1 de la table :code:`INIT_GDTR`, qui vaut :code:`GdtEntry::new(0, 0, GDT_A_PRESENT | GDT_A_RING_0 | GDT_A_SYSTEM | GDT_A_EXECUTABLE | GDT_A_PRIVILEGE, GDT_F_LONG_MODE)`.

Autre exemple : pour décoder n'importe quelle autre adresse, le décalage dans la GDT sera de :code:`GDT_KERNEL_DATA` (= 2) et le segment sera :code:`GdtEntry::new(0, 0, GDT_A_PRESENT | GDT_A_RING_0 | GDT_A_SYSTEM | GDT_A_PRIVILEGE, GDT_F_LONG_MODE)`.

Il faut noter que les champs `offset` et  `limit` sont tous à 0 pour une raison simple : il n'y a qu'un seul espace d'adressage en mode 64-bit (drapeau :code:`GDT_F_LONG_MODE`). Seuls les bits de protection sont considérés [#f1]_.

Il apparaît donc que toutes les adresses sont interprétées comme des adresses physiques.

La seconde initialisation des segments
--------------------------------------
Elle utilise le *Thread Local Storage* (*TLS*). Cette fois-ci, la table de descripteur des pointeurs peut-être différente selon les threads [#f2]_.

TODO.

.. [#f1] Dans le manuel `Intel® 64 and IA-32 Architectures Software Developer’s Manual, Volume 3A: System Programming Guide, Part 1 <https://www.intel.com/content/dam/www/public/us/en/documents/manuals/64-ia-32-architectures-software-developer-vol-3a-part-1-manual.pdf>`_, la section 3.2.4 précise:

        In 64-bit mode, segmentation is generally (but not completely) disabled, creating a flat 64-bit linear-address space. The processor treats the segment base of CS, DS, ES, SS as zero, creating a linear address that is equal to the effective address. The FS and GS segments are exceptions. [...] Note that the processor does not perform segment limit checks at runtime in 64-bit mode.

.. [#f2] L'idée du TLS est qu'une variable globale au programme ne soit en réalité qu'un pointeur vers un espace de stockage local au thread. Ainsi, elle peut avoir une valeur différente selon le thread qui y accède.
