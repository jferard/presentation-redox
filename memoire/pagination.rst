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

Mémoire -- pagination
=====================
Présentation
------------
L'idée de base est que chaque processus possède un espace d'adressage unique. Cet espace d'adressage est mappé sur la mémoire physique, ce qui signifie qu'à chaque page de 4096 octets de cet espace d'adressage, l'OS va essayer de faire correspondre un cadre de page, c'est-à-dire une portion de 4096 octets de la mémoire. Cela suppose que l'OS gère à la fois :

* une table des pages pour faire correspondre un cadre de page à une adresse virtuelle utilisée par un programme ;
* une liste des cadres de page libres en cas d'allocation d'un cadre de page à une nouvelle page.

Quelques remarques :
* l'adresse virtuelle (de la page) et l'adresse physique (du cadre) sont complètement indépendantes ;
* un cadre peut correspondre à plusieurs pages, d'un ou plusieurs processus, ce qui permet un partage de la mémoire (bibliothèques partagées, espace commun de données) ;

Redox ne pratique pas la pagination à la demande. Un défaut de page est donc une erreur fatale.

Initialisation du module mémoire
--------------------------------

La mémoire est initialisée dans :code:`kernel/src/memory/mod.rs`:

.. code-block:: rust

    static mut MEMORY_MAP: [MemoryArea; 512]

Chaque :code:`MemoryArea` contient une base, une limite, un type et des informations ACPI (Advanced Configuration and Power Interface) :

.. code-block:: rust

    pub struct MemoryArea {
        pub base_addr: u64,
        pub length: u64,
        pub _type: u32,
        pub acpi: u32
    }

Toutes les entrées sont initialisées avec une adresse de base de 0x500. Enfin l'allocateur est initialisé de la manière suivante :

.. code-block:: rust

    *ALLOCATOR.lock() = Some(RecycleAllocator::new(BumpAllocator::new(kernel_start, kernel_end, MemoryAreaIter::new(MEMORY_AREA_FREE))));

Reste à savoir ce que sont ces :code:`RecycleAllocator` et :code:`BumpAllocator`. D'abord le :code:`BumpAllocator` (`kernel/src/memory/bump.rs`) : il prend un :code:`kernel_start`, un :code:`kernel_end` et un iterateur sur les zoones libres de la :code:`MEMORY_MAP`. L'initialisation à défini le :code:`kernel_start` à 0 et le :code:`kernel_end` à :code:`kernel_base + kernel_size` (en gros).

La structure :code:`Frame` (`kernel/src/memory/mod.rs`) représente un cadre de page, associé à une adresse pĥysique (`PhysicalAddress` dans :code:`kernel/src/arch/x86_64/paging/mod.rs`).

Pagination
~~~~~~~~~~
Le coeur de la pagination est le :code:`Mapper` (kernel/src/arch/x86_64/paging/mapper.rs) :

.. code-block:: rust

    impl Mapper {
        ...

        /// Map a page to the next free frame
        pub fn map(&mut self, page: Page, flags: EntryFlags) -> MapperFlush {
            ...
        }

        pub fn translate_page(&self, page: Page) -> Option<Frame> {
            ...
        }

        ...
    }

A une :code:`Page` sont associé des index successif dans des tables organisées en hiérarchie : le début de l'adresse virtuelle donne l'index dans la table 4, la suite dans la table 3, la suite dans la table 2, et enfin l'index dans la table 1.

Le plus simple est la table 1, de type :code:`Table<L> where L: TableLevel` car elle est indexable et contient des :code:`Entry` (cadres de 4 ko). Une :code:`Entry` est représentée par un :code:`u64` : les bits 12 à 51 contiennent l'adresse physique, les autres bits sont des drapeaux variés.

controlregs::cr3_write(PT_BASE);




kernel/src/arch/x86_64/paging/mod.rs

VirtualAdress (= usize) / PAGE_SIZE -> Page
