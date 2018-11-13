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

Processus -- Création
=====================
Présentation
------------
Redox conserve la philosophie UNIX de création des processus : le processus père est d'abord dupliqué (au moyen d'un appel :code:`fork`) puis l'environnement ainsi dupliqué sert de point de départ à l'exécution d'un programme par le processus fils (au moyen de :code:`exec`). Cette séparation en deux appels distincts permet au fils d'hériter de l'environnement du père.

L'appel à :code:`clone`
-----------------------
Le point de départ est l'appel système :code:`clone` de la bibliothèque standard de Redox.

La fonction :code:`clone`
~~~~~~~~~~~~~~~~~~~~~~~~~
UPour créer un processus, il faut utiliser un appel système afin de passer en mode superviseur. Ce point est aussi l'occasion de comprendre le fonctionnement d'un appel système (interruption `0x80`).

Dans l'API des appels système de Redox (fichier `syscall/src/call.rs <https://gitlab.redox-os.org/redox-os/syscall/blob/master/src/call.rs>`_), on trouve l'appel suivant :

.. code-block:: rust

    /// Produce a fork of the current process, or a new process thread
    pub unsafe fn clone(flags: usize) -> Result<usize> {
        syscall1_clobber(SYS_CLONE, flags)
    }

Dans le fichier :code:`syscall/src/arch/x86_64.rs <https://gitlab.redox-os.org/redox-os/syscall/blob/master/src/arch/x86_64.rs>_`, on trouve la définition de :code:`syscall1_clobber` :

.. code-block:: rust

    // Clobbers all registers - special for clone
    pub unsafe fn syscall1_clobber(mut a: usize, b: usize) -> Result<usize> {
        asm!("int 0x80"
            : "={eax}"(a)
            : "{eax}"(a), "{ebx}"(b)
            : "memory", "ebx", "ecx", "edx", "esi", "edi"
            : "intel", "volatile");

        Error::demux(a)
    }

Traduction : passage de contrôle au vecteur d'interruption :code:`0x80`. En entrée, a et b, à savoir le code de l'appel :code:`SYS_CLONE` et les drapeaux :code:`flags`, la sortie est écrite sur :code:`a`. Enfin, le résulat est un :code:`Result<usize>` selon que l'appel a réussi (attention : :code:`Error::demux`  peut renvoyer :code:`Ok`).

Les interruptions
~~~~~~~~~~~~~~~~~
La question des interruptions est assez complexe et fait l'objet d'une section à part. Le seul point à comprendre ici est que l'interruption (gérée au niveau matériel) réalise deux opérations simultanément :
* passer en mode superviseur, avec un accès complet à la mémoire et aux registres ;
* réaliser un appel de fonction en mode superviseur.

Ici, la fonction appelée s'appelle également :code:`clone`. Cette fonction est la suivante :

.. code-block:: rust

    // kernel/src/syscall/process.rs
    pub fn clone(flags: usize, stack_base: usize) -> Result<ContextId> {
        // - récupération du contexte actuel
        // - copie du contexte actuel
        // - retour du nouveau numéro de contexte
    }

Le contexte
-----------
Le contexte contient toutes les informations sur le processus en cours d'exécution.

Par définition, un processus possède son propre espace d'adressage, un ou plusieurs fils d'exécution (compteur ordinal), ses signaux, sa pile, etc. Tout ceci est rassemblé dans Redox sous l'appellation :code:`Context` (:code:`task_struct` dans Linux, mais le nom générique est plutôt *Process control block*).

La description de ce contexte est séparée en deux partie : une partie générique (:code:`context::Context`), une partie liée à l'architecture (:code:`context::arch::Context`).

La structure :code:`contexte::Context` (`kernel/src/context/context.rs <https://gitlab.redox-os.org/redox-os/kernel/blob/master/src/context/context.rs>`_) contient ce qu'on peut attendre d'un PCB.  :
* les identifiants (du processus, de son père, de l'utilisateur (réel ou effectif [#f1]_)) ;
* l'état ;
* les fichiers ouverts ;
* les signaux ;
* l'espace d'adressage et la pile ;
* des informations liées à l'architecture.

.. [#f1] L'utilsateur réel est celui qui lance le processus, l'utilisateur effectif celui qui est considéré pour les autorisations. Sous Linux, on peut penser au bit `setuid` : l'utilisateur qui lance le programme (identité reélle) `usr/bin/password` endosse temporairement l'identité de `root` (identité effective).

La partie liée à l'architecture est décrite dans :code:`kernel/src/context/arch/x86_64.rs  <https://gitlab.redox-os.org/redox-os/kernel/blob/master/src/context/arch/x86_64.rs>_`:

.. code-block:: rust

    pub struct Context {
        /// FX valid?
        loadable: bool,
        /// FX location
        fx: usize,
        /// Page table pointer
        cr3: usize,
        /// RFLAGS register
        rflags: usize,
        /// RBX register
        rbx: usize,
        /// R12 register
        r12: usize,
        /// R13 register
        r13: usize,
        /// R14 register
        r14: usize,
        /// R15 register
        r15: usize,
        /// Base pointer
        rbp: usize,
        /// Stack pointer
        rsp: usize
    }

Récupération du contexte actuel
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
La méthode pour récupérer le contexte est toujours la même :

.. code-block:: rust

    let contexts = context::contexts();
    let context_lock = contexts.current().ok_or(Error::new(ESRCH))?;
    let context = context_lock.read();

La fonction :code:`contexts()` renvoie un :code:`RwLockReadGuard` (verrou qui garantit qu'il n'y a pas de writer) sur une :code:`ContextList`, qui est déréférencée à la ligne suivante avec l'appel de :code:`current()`, retournant une :code:`Option` sur un verrou sur un :code:`Context`. Avec :code:`read()` le contexte est récupéré.

Copie du contexte actuel
------------------------
Cette partie est plus complexe et nécessite des notions plus avancées concernant la mémoire. Elle sera vue au moment de la description de la mémoire.

Pourquoi une interruption ?
---------------------------
Pourquoi l'appel à :code:`clone` ne peut-il se faire totalement dans l'espace utilisateur ? La question qui se pose ici est : qu'est-ce qui empêche un processus :code:`Cloner` de faire le travail à la place du noyau : recopier la représentation du processus père dans une zone mémoire et d'exécuter cette zone mémoire sans demander son avis au noyau ? A quel moment précis le rôle du noyau est-il essentiel ?

La réponse est évidente et banale à la fois : le noyau dispose de l'accès à la table des processus. Il s'agit de la :code:`ContextList` qui est utilisée au moment de l'ordonnancement. Le noyau, puisqu'il peut accéder à toute la mémoire, peut évidemment accéder à la représentation en mémoire du process père et en faire une copie (virtuelle). Puis ajouter le nouveau contexte à la liste des contextes pour un ordonnancement prochain.
