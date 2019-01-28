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

La macro :code:`asm!`
=====================

Redox fait un usage intensif de la macro :code:`asm!`. En effet, une partie du travail ne peut-être réalisé qu'en manipulant certains registres du processeurs. C'est le cas lors d'un chargement de table de pages, invalidation du TLB, manipulation directe de la pile (par :code:`clone`), etc. L'utilisation de cette macro est limitée à des parties de l'OS dépendant de l'architecture [1]_

La macro prend cinq arguments :

* Le motif est une chaîne qui peut prendre des paramètres (préfixés par le symbole `$`).
* La valeur de retour, sous la forme :code:`={<reg>}(<var>)`.
* Les valeurs en entrée, sous la forme :code:`{<reg>}(<var>)` ou :code:`<value>`. Attention, ces valeurs ne se trouvent pas forcément dans le motif.
* Les clobbers, c'est-à-dire les registres susceptibles d'être écrasés par l'opération. La valeur "memory" signifie que certaines zones mémoire peuvent être également écrasées (hors valeur de retour)
* Des options. Dans Redox, ce sera toujours "volatile" (ne sera pas effacée) et "intel" (syntaxe Intel).

Les valeurs d'entrée et de retour assurent le lien entre le code Rust et le code assembleur.

.. [1] Dans la version courante:

   .. code-block :: bash

        jferard\@jferard-Z170XP-SLI:~/prog/rust/redox/kernel/src$ grep -r "asm!" . | cut -f 1 -d " " | uniq
        ./context/arch/x86_64.rs:
        ./arch/x86_64/interrupt/syscall.rs:
        ./arch/x86_64/interrupt/trace.rs:
        ./arch/x86_64/interrupt/mod.rs:
        ./arch/x86_64/interrupt/exception.rs:
        ./arch/x86_64/start.rs:
        ./arch/x86_64/pti.rs:
        ./arch/x86_64/graphical_debug/primitive.rs:
        ./arch/x86_64/macros.rs:
        ./arch/x86_64/stop.rs:
