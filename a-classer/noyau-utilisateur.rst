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

Mode noyau - mode utilisateur
=============================
Un OS effectue constamment des va et vient entre le mode noyau et le mode utilisateur. Lorsqu'on consièdre le processeur, il s'agit d'un passage entre le ring 0 et le ring 3 (souvenir de Multics). Lorsqu'on considère l'ensemble du contexte (au sens large, pas au sens Redox) on parle de passer du *Kernel Space* au *User space*. Ces passages on lieu a différents moments :
* premier passage du mode noyau au mode utilisateur, lorsque l'initialisation est terminée ;
* passage du mode utilisateur au mode noyau lors d'une interruption ;
* passage du mode noyau au mode utilisateur à la fin d'une interruption.

Le passage mode utilisateur au mode noyau est assuré par le matériel. Il reste donc les deux premières bascules à étudier.

Retourner en mode utilisateur
-----------------------------
Commençons par le plus simple : le retour en mode utilisateur s'effectue avec une instruction de type :code:`iret` (retour d'interruption).
La première entrée en mode utilisateur
--------------------------------------

La fonction qui permet le passage en mode utilisateur est évidemment en assembleur. Elle se trouve dans `kernel/src/arch/x86_64/start.rs <https://gitlab.redox-os.org/redox-os/kernel/blob/master/arch/x86_64/start.rs>`_ :

.. code:: rust

    #[naked]
    pub unsafe fn usermode(ip: usize, sp: usize, arg: usize) -> ! {
        asm!(
            // pousse sur la pile : data segment, stack pointer, interrupt flags, code segment, interrupt code et arguments
        );

        // Unmap kernel
        pti::unmap();

        // Go to usermode
        asm!(
            // se place sur les segments utilisateur
            // met rax, rbx, rcx, rdx, rsi, rdi, rbp, r8..r15 à 0
            fninit
            pop rdi
            iretq"
            : // No output because it never returns
            :   "{r14}"(gdt::GDT_USER_DATA << 3 | 3), // Data segment
                "{r15}"(gdt::GDT_USER_TLS << 3 | 3) // TLS segment
            : // No clobbers because it never returns
            : "intel", "volatile");
        unreachable!();
    }

Le premier point intéressant de cette fonction est la signature : la type valeur de retour est un point d'exclamation. il s'agit du type :code:`never` (`never <https://doc.rust-lang.org/nightly/std/primitive.never.html>`_. La fonction ne retourne jamais de valeur. Ceci est confirmé par la macro :code:`unreachable` en fin de corps de la fonction.

La fonction est divisée en deux parties : sauvegarder le contexte actuel, passer en mode utilisateur.


fninit : Sets the FPU control, status, tag, instruction pointer, and data pointer registers to their default states.


Les appels à usermode
---------------------
La fonction :code:`usermode` est appelée dans deux situations :
* /home/jferard/prog/rust/redox/kernel/src/context/signal.rs : :code:`signal_handler` qui gère la réception des signaux par le processus courant.
* /home/jferard/prog/rust/redox/kernel/src/syscall/process.rs : :code:`fexec_noreturn`, appelée par :code:`fexec` et qui réalise le lancement d'un programme.
