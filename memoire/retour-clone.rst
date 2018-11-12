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

Mémoire -- retour sur clone
===========================
On vient de voir que le processeur x86 utilise une pagination à quatre niveaux. La table `P4`, dont l'adresse est stockée dans le registre de contrôle `cr3`, est le point d'entrée. L'adresse est codée, pour un processeur 64 bits, sur 48 bits : l'index dans chaque table sur 9 bits (4 * 9 = 36, 2 ** 9 = 512 entrées de 8 o) et l'offset dans la page de 4 ko sur 12 bits (2**12 = 4096).

Deux problèmes liés existent :

1. Le volume total des tables est : 1 table P4 + 512 tables P3 + 262 144 tables P2 + 134 217 728 tables P1. Cela fait 134 480 385 tables de 4kio, soit 513 Mio. Cela représente un volume important, et il serait intéressant de pouvoir paginer la table des pages elle-même.
2. Plus grave, lors d'un appel à `clone`, il faut construire la nouvelle table des pages pour le processus fils. La question est la suivante : comment accéder aux adresses physiques du noyau (qui font partie de l'espace d'adressage du processus père et doivent être recopiées dans l'espace d'adressage du fils) ? Il faut accéder à ces adresses via l'adressage virtuel du père. Il est bien sûr possible de récupérer l'adresse physique de la table `P4` dans le registre `cr3`, mais ceci ne sera d'aucun intérêt pour travailler sur les tables, à moins de pouvoir aller des adresse physiques vers les adresses virtuelles.

Le blog `os.phil-opp.com <https://os.phil-opp.com/page-tables/>`_ montre comment résoudre ces deux problèmes en suivant un processus d'amélioration progressive. Je vais partir des solutions directement, ce qui sera plus simple à comprendre.

La pagination de la table des pages
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
On va attribuer une place à la table P4 dans l'espace d'adressage. Ce sera l'adresse :

.. code:: rust

    pub const P4: *mut Table = 0xffffffff_fffff000 as *mut _;

Ce qui donne en binaire : `0b1111111111111111 111111111 111111111 111111111 111111111 000000000000`. Les 16 premiers bits sont toujours à 1. Pour le reste, on voit que c'est la table P1 qui correspond à la dernière entrée de la table P2 qui correspond à la dernière entrée de la table P3 qui correspond à la dernière entrée de la table P4. On a pris à chaque fois la dernière entrée, qu'on branche sur l'adresse physique de la table P4.

Comment créer une InactivePageTable ?
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
Il faut écrire un cadrede page qui va servir de table P4. Comment écrire un cadre de page. La pagination étant activée, il est impossible de sélectionner un cadre de page libre pour y écrire (avec des adresses physiques) la table récursive.

Il faut prendre une adresse libre dans l'espace d'adressage du process courant, mapper un cadre libre, et écrire les données d'une table des pages minimale.

Création d'une nouvelle :code:`InactivePageTable` :

.. code:: rust

        pub fn new(frame: Frame, active_table: &mut ActivePageTable, temporary_page: &mut TemporaryPage) -> InactivePageTable {
            {
                let table = temporary_page.map_table_frame(frame.clone(), EntryFlags::PRESENT | EntryFlags::WRITABLE | EntryFlags::NO_EXECUTE, active_table);
                // now we are able to zero the table
                table.zero();
                // set up recursive mapping for the table
                table[::RECURSIVE_PAGE_PML4].set(frame.clone(), EntryFlags::PRESENT | EntryFlags::WRITABLE | EntryFlags::NO_EXECUTE);
            }
            temporary_page.unmap(active_table);

            InactivePageTable { p4_frame: frame }
        }

La première ligne effectue un mappage de la page temporaire nouvellement créée vers un cadre de page nouvellement créé, dans la table des pages active. On vide la table, puis la dernière entrée se voit ajouter le cadre.

Changement de table des pages
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Le changement de table des pages se trouve dans `kernel/src/arch/x86_64/paging/mod.rs` :

.. code:: rust

    pub fn switch(&mut self, new_table: InactivePageTable) -> InactivePageTable {
        let old_table = InactivePageTable {
            p4_frame: Frame::containing_address(
                PhysicalAddress::new(unsafe { control_regs::cr3() } as usize)
            ),
        };
        unsafe {
            control_regs::cr3_write(new_table.p4_frame.start_address().get() as u64);
        }
        old_table
    }

On crée une nouvelle :code:`InactivePageTable` à partir de l'adresse du registre `cr3`, puis on écrit dans ce même registre l'adresse du cadre `p4_frame` de la nouvelle table.

Il s'agit juste de changer le contenu du registre `cr3`.


.. code:: rust

    // Copy kernel image mapping
    {
        let frame = active_table.p4()[::KERNEL_PML4].pointed_frame().expect("kernel image not mapped");
        let flags = active_table.p4()[::KERNEL_PML4].flags();
        active_table.with(&mut new_table, &mut temporary_page, |mapper| {
            mapper.p4_mut()[::KERNEL_PML4].set(frame, flags);
        });
    }

ActivePageTable.with :

.. code:: rust

    pub fn with<F>(&mut self, table: &mut InactivePageTable, temporary_page: &mut TemporaryPage, f: F)
        where F: FnOnce(&mut Mapper)
    {
        {
            let backup = Frame::containing_address(PhysicalAddress::new(unsafe { control_regs::cr3() as usize }));

            // map temporary_page to current p4 table
            let p4_table = temporary_page.map_table_frame(backup.clone(), EntryFlags::PRESENT | EntryFlags::WRITABLE | EntryFlags::NO_EXECUTE, self);

            // overwrite recursive mapping
            self.p4_mut()[::RECURSIVE_PAGE_PML4].set(table.p4_frame.clone(), EntryFlags::PRESENT | EntryFlags::WRITABLE | EntryFlags::NO_EXECUTE);
            self.flush_all();

            // execute f in the new context
            f(self);

            // restore recursive mapping to original p4 table
            p4_table[::RECURSIVE_PAGE_PML4].set(backup, EntryFlags::PRESENT | EntryFlags::WRITABLE | EntryFlags::NO_EXECUTE);
            self.flush_all();
        }

        temporary_page.unmap(self);
    }




Etape 1 : /home/jferard/prog/rust/redox/kernel/src/consts.rs contient les positions des différents éléments dans l'espace d'adressage virtuel d'un processus.

::

    pub const P4: *mut Table<Level4> = 0xffff_ffff_ffff_f000 as *mut _; = recursion : P4 -> P4 -> P4 -> P1

Création d'une page temporaire correspondant à l'adresse virtuelle USER_TMP_MISC_OFFSET
