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

Le démarrage du noyau
=====================
Le chargement du noyau est un processus complexe qui implique un programme présent dans la ROM (BIOS ou UEFI), le chargement et démarrage d'un chargeur qui va charger le système lui-même. Je me place ici après cette partie, lorsque le noyau est présent en mémoire.

Le point d'entrée
-----------------
Dans le script pour le linker du kernel (`kernel/linkers/x86_64.ld`), on trouve la commande suivante :

::

    ENTRY(kstart)

La commande `ENTRY` désigne le point d'entrée (*entry point*) de l'exécutable produit, c'est-à-dire la première instruction du programme. Allons voir la signature de cette fonction, dans :code:`kernel/src/arch/x86_64/start.rs` :

.. code-block:: rust

    #[no_mangle]
    pub unsafe extern fn kstart(args_ptr: *const KernelArgs) -> ! {
        ...
    }

Comme on le remarque, la fonction prend des paramètres. Elle ne peut donc pas être lancée telle quelle. On y reviendra avec l'analyse du bootloader, mais qu'il suffise pour l'instant de constater que le chargeur peut récupérer la fonction désignée par l'*entry point* pour la lancer en lui fournissant les bons paramètres.

C'est donc la fonction :code:`kstart` avec les paramètres qui conviennent qui est appelée à la fin du chargement du kernel.

Nous allons suivre cette fonction comme plan de notre exploration. Dans :code:`kernel/src/arch/x86_64/start.rs` on trouve l'entry point du noyau compilé :

.. code-block:: rust

    #[no_mangle]
    pub unsafe extern fn kstart(args_ptr: *const KernelArgs) -> ! {
        let env = {
            ...
            // Set up GDT before paging
            gdt::init();

            // Set up IDT before paging
            idt::init();

            // Initialize memory management
            memory::init(0, kernel_base + ((kernel_size + 4095)/4096) * 4096);

            // Initialize paging
            let (mut active_table, tcb_offset) = paging::init(0, kernel_base, kernel_base + kernel_size, stack_base, stack_base + stack_size);

            // Set up GDT after paging with TLS
            gdt::init_paging(tcb_offset, stack_base + stack_size);

            // Set up IDT
            idt::init_paging();

            ...
            // Setup kernel heap
            allocator::init(&mut active_table);

            // Use graphical debug
            #[cfg(feature="graphical_debug")]
            graphical_debug::init(&mut active_table);

            // Initialize devices
            device::init(&mut active_table);

            // Read ACPI tables, starts APs
            #[cfg(feature = "acpi")]
            acpi::init(&mut active_table);

            // Initialize all of the non-core devices not otherwise needed to complete initialization
            device::init_noncore();

            // Initialize memory functions after core has loaded
            memory::init_noncore();

            // Stop graphical debug
            #[cfg(feature="graphical_debug")]
            graphical_debug::fini(&mut active_table);

            BSP_READY.store(true, Ordering::SeqCst);

            slice::from_raw_parts(env_base as *const u8, env_size)
        };

        ::kmain(CPU_COUNT.load(Ordering::SeqCst), env);
    }

Le plan est donc simple :

1. La segmentation : Global Descriptor Table (GDT)
2. La Interrupt Descriptor Table (IDT)
3. L'initialisation de la mémoire et la pagination
4. Le kernel heap
5. Les périphériques
6. Advanced Configuration & Power Interface
7. La boucle principale :code:`kmain`
