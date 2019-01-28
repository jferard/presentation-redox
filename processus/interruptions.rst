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

Processus -- Les interruptions
==============================
Présentation
------------
Les interruptions sont un mécanisme fondamental utilisé par l'OS. Lorsqu'un événement survient :

* au niveau des périphériques : touche enfoncée au clavier, fin de lecture ou d'écriture sur disque, ... Il s'agit d'une **interruption matérielle**, asynchrone par rapport à l'exécution du programme courant qu'elle vient littéralement interrompre ;
* au niveau du processeur ou de la mémoire en cas d'opération exceptionnelle : division par 0, overflow, défaut de page, lecture ou écriture interdite. On nomme cette interruption une **trappe** ;
* au niveau du processus : **appel système**.

Les deux derniers cas se produisent de manière synchrone au déroulement du programme, le premier de manière asynchrone, mais pour le système d'exploitation, cela ne fait aucune différence. Les routines d'interruption sont enregistrées par l'OS dans un vecteur d'interruption.

Les enjeux sont multiples :

1. Comment enregistrer une routine dans le vecteur d'interruption ?
2. Comment passer en mode noyau (cas de l'appel système) ?
3. Comment appeler la routine et lui passer des arguments ?
4. Comment retourner en mode utilisateur ?

Je vais essayer de répondre à ces questions dans le contexte de l'appel :code:`clone`.

Interrupt descriptor table
--------------------------
Le vecteur dinterruption (Interrupt descriptor table, IDT) est défini dans `src/arch/x86_64/idt.rs <https://gitlab.redox-os.org/redox-os/kernel/blob/master/src/arch/x86_64/idt.rs>`_. En particulier, l'interruption `0x80` dédiée aux appels système.

.. code-block:: rust

    pub static mut IDT: [IdtEntry; 256] = [IdtEntry::new(); 256];

    ...

    pub unsafe fn init_paging() {
        ...
        // Set syscall function
        IDT[0x80].set_func(syscall::syscall);
        IDT[0x80].set_flags(IdtFlags::PRESENT | IdtFlags::RING_3 | IdtFlags::INTERRUPT);
        ...
    }

Il faut bien noter que c'est le matériel qui fournit ce dispositif de vecteur d'interruption. Le processeur impose le format des entrées du vecteur d'interruption [#] _:

.. code-block:: rust

    #[derive(Copy, Clone, Debug)]
    #[repr(packed)]
    pub struct IdtEntry {
        offsetl: u16,
        selector: u16,
        zero: u8,
        attribute: u8,
        offsetm: u16,
        offseth: u32,
        zero2: u32
    }

Le processeur fournit également l'instruction :code:`lidt` [#]_) qui permet de charger le vecteur d'interruption:

.. code-block:: rust

    /// Load IDT table.
    pub unsafe fn lidt(idt: &DescriptorTablePointer<IdtEntry>) {
        asm!("lidt ($0)" :: "r" (idt) : "memory");
    }

:code:`src/arch/x86_64/syscall:syscall`
---------------------------------------
Toute interruption aboutit donc à l'appel de la fonction :code:`syscall::syscall` de `src/arch/x86_64/interrupt/syscall.rs <https://gitlab.redox-os.org/redox-os/kernel/blob/master/src/arch/x86_64/interrupt/syscall.rs>`_. C'est une fonction sans argument :

.. code-block:: rust

    #[naked]
    pub unsafe extern fn syscall() {
        #[inline(never)]
        unsafe fn inner(stack: &mut SyscallStack) -> usize {
            let rbp;
            asm!("" : "={rbp}"(rbp) : : : "intel", "volatile");

            syscall::syscall(stack.rax, stack.rbx, stack.rcx, stack.rdx, stack.rsi, stack.rdi, rbp, stack)
        }

        asm!("push rax
             ... (push rbx...r11)
             push fs
             mov r11, 0x18
             mov fs, r11"
             : : : : "intel", "volatile");

        // Get reference to stack variables
        let rsp: usize;
        asm!("" : "={rsp}"(rsp) : : : "intel", "volatile");

        // Map kernel
        pti::map();

        let a = inner(&mut *(rsp as *mut SyscallStack));

        // Unmap kernel
        pti::unmap();

        asm!("" : : "{rax}"(a) : : "intel", "volatile");

        // Interrupt return
        asm!("pop fs
              (pop r11...rbx)
              add rsp, 8
              iretq"
              : : : : "intel", "volatile");
    }

Cette routine commence par définir une fonction :code:`inner`, puis met en place les données sur la pile qui seront transmises à la fonction :code:`syscall` de `src/syscall/mod.rs` et enfin nettoie la pile et retourne en mode utilisateur. Voyons cela en détail.

:code:`inner`
~~~~~~~~~~~~~
La fonction :code:`inner` prend une référence à une structure :code:`SyscallStack` définie dans le même fichier. Elle commence par récupérer le registre :code:`rbp` (*Stack Base Pointer*) qui pointera sur le haut de la pile avant l'appel, c'est-à-dire sur la dernière variable locale de `syscall`. Elle appelle la fonction :code:`syscall` avec cette référence et le pointeur :code:`rbp`. Les paramètres de l'appel système (dont évidemment son code) doivent donc se trouver dans cette structure :code:`SyscallStack` :

.. code :: rust

    #[allow(dead_code)]
    #[repr(packed)]
    pub struct SyscallStack {
        pub fs: usize,
        pub r11: usize,
        pub r10: usize,
        pub r9: usize,
        pub r8: usize,
        pub rsi: usize,
        pub rdi: usize,
        pub rdx: usize,
        pub rcx: usize,
        pub rbx: usize,
        pub rax: usize,
        pub rip: usize,
        pub cs: usize,
        pub rflags: usize,
    }

Le passage de paramètres
~~~~~~~~~~~~~~~~~~~~~~~~
Avant de déclencher l'interruption, l'appel système doit mettre les paramètres utiles dans les registres. Le seul paramètre nécessaire dans tous les cas est :code:`eax/rax` qui contient le numéro de code de l'appel système.

Avant la mise en place de ces paramètres sur la pile, la pile contenait les valeurs de quelques registres, valeur sauvegardées par le matériel au moment de l'interruption (voir [2]_) :

* :code:`rflags` : résultat des opérations et état du processus courant (analogue au PSW, Program Status Word de l'IBM System/360)
* :code:`cs`: pointeur sur le code
* :code:`rip` : pointeur sur la prochaine instruction

La routine complète la pile par les valeurs d'autres registres, pour obtenir une pile comme suit (elle croit vers les adresses basses) :

.. code

    [ rflags ]
    [ cs     ]
    [ rip    ]
    [ rax    ]
    [ rbx    ]
    [ rcx    ]
    [ rdx    ]
    [ rdi    ]
    [ rsi    ]
    [ r8     ]
    [ r9     ]
    [ r10    ]
    [ r11    ]
    [ fs     ]

Ce qui correspond exactement à la structure attendue :code:`SyscallStack`. Le registre :code:`rsp` (*Stack Pointer*) contient donc exactement l'adresse d'une telle structure.
Le registre :code:`fs` est mis à :code:`0x18`.

Appel et nettoyage
##################
La fonction :code:`inner` est appelée (voir ci-dessous). En pratique, cela permet d'appeler une fonction avec toutes les informations nécessaires, pour peu qu'elles ait été initialisées avant l'appel de :code:`syscall`.

La valeur de retour est stockée dans le registre :code:`rax`, puis la pile est nettoyée pour réattribuer les valeurs aux registres (instruction :code:`pop`). Attention, il n'y a pas d'instruction :code:`pop rax` qui écraserait la valeur de retour, mais un déplacement du pointeur du haut de la
pile (:code:`add rsp, 8`, la pile décroit vers les adresses basses).

Il est temps d'effectuer le retour d'interruption.

:code:`src/syscall/mod.rs:syscall`
----------------------------------

Voyons maintenant la fonction :code:`syscall` avec paramètres. Le numéro de l'appel système est le paramètre :code:`a`. Par exemple, pour l'appel à :code:`clone` (`src/syscall/mod.rs <https://gitlab.redox-os.org/redox-os/kernel/blob/master/src/syscall/mod.rs>`_) on trouve:

.. code-block:: rust

    pub fn syscall(a: usize, b: usize, c: usize, d: usize, e: usize, f: usize, bp: usize, stack: &mut SyscallStack) -> usize {
        ...
        match a {
            ...
            SYS_CLONE => clone(b, bp).map(ContextId::into),
            ...
        }
    }

Le mécanisme d'interruption en détail
-------------------------------------
Les routine du vecteur d'interruption sont sont déclarées dans Redox, pour une raison qui va s'éclairer, au sein d'une macro qui créé une fonction enveloppe. Les macros en question se trouvent dans le fichier `kernel/src/arch/x86_64/macros.rs <https://gitlab.redox-os.org/redox-os/kernel/tree/master/src/kernel/arch/x86_64/macros.rs>`_. Il en existe actuellement cinq : :code:`interrupt`, :code:`interrupt_stack`, :code:`interrupt_error`, :code:`interrupt_stack_p` et :code:`interrupt_error_p`.

Au niveau matériel
~~~~~~~~~~~~~~~~~~
Il faut à présent entrer dans le détail [#]_: au moment de l'interruption et lorsque la routine invoquée est exécutée à un niveau inférieurle processeur change la pile d'exécution. Typiquement, c'est ce qui se produit quand un processeur utilisateur interrompu et une routine en mode noyau activée. Redox déclare toutes les routines d'interruption au niveau 0 (espace noyau).

Le processeur trouve le sélecteur de segment et le pointeur de pile cible dans le `TaskStateSegment` et pousse sur la nouvelle pile :
1. le sélecteur de segment de pile du processus courant
2. le pointeur de pile du processus courant
3. l'état courant des EFLAGS
4. le sélecteur de segment de pile
5. le pointeur de pile
6. un code d'erreur optionnel.

Ces informations sont nécessaires au processeur pour effectuer un retour d'interruption.

Au niveau de Redox
~~~~~~~~~~~~~~~~~~
Redox a maintenant la main. Le code de Redox contient un certain nombre de macros permettant de créer facilement un :code:`syscall`. Examinons la macro la plus simple, à savoir :code:`interrupt` :

.. code:: rust

    #[macro_export]
    macro_rules! interrupt {
        ($name:ident, $func:block) => {
            #[naked]
            pub unsafe extern fn $name () {
                #[inline(never)]
                unsafe fn inner() {
                    $func
                }

                // Push scratch registers
                scratch_push!();
                fs_push!();

                // Map kernel
                $crate::arch::x86_64::pti::map();

                // Call inner rust function
                inner();

                // Unmap kernel
                $crate::arch::x86_64::pti::unmap();

                // Pop scratch registers and return
                fs_pop!();
                scratch_pop!();
                iret!();
            }
        };
    }

Cette macro vaut la peine d'être étudiée en détail. Le bloc fonction est intégré dans une fonction :code:`inner`.

La sauvegarde du contexte consiste à pousser les informations sur la pile : Redox pousse également sur la pile les "scratch registers", à savoir les registres qui peuvent être utilisés librement et le registre `fs`, qu'il remplace par le Thread Local Storage du noyau :

.. code:: rust

    macro_rules! fs_push {
        () => (asm!(
            "push fs
            mov rax, 0x18
            mov fs, ax"
            : : : : "intel", "volatile"
        ));
    }

Ici, `0x18` représente l'indice `GDT_KERNEL_TLS` multiplié par 8 (la taille en octets d'une entrée dans la table) auquel on additione 0 (pour le mode d'exécutiuon Ring 0) [#]_.

Vient ensuite :code:`pti::unmap()`. Cette fonction est liée à des questions de sécurité (la faille Meltdown) [#]_.

Vient ensuite l'exécution de la fonction enveloppée. Celle-ci peut récupérer ce qui est déposé sur la pile, mais doit remettre la pile en été avant de se terminer.

Enfin, les informations empilées par Redox sont dépilées, et le matériel reprend la main. Il retrouve les informations qu'il avait empilées initialement (sélecteurs, EFLAGS, etc.) et retourne au processus interrompu.

Quelques interruptions intéressantes
------------------------------------
La table des vecteurs d'interruption dans `kernel/src/arch/x86_64/idt.rs <https://gitlab.redox-os.org/redox-os/kernel/tree/master/src/kernel/arch/x86_64/idt.rs>`_ contient la déclaration de toutes les interruptions. La fonction :code:`idt::init_paging()` associe une fonction Rust à un vecteur d'interruption. Par exemple :
* l'interruption n°14, qui correspond à un défaut de page, déclenche la fonction `exception::page` ;
* l'interruption n°32, qui correspond à un "tick" du timer, déclenche la fonction `irq::pit` ;
* l'interruption n°33, qui correspond au clavier, déclenche la fonction `irq::keyboard` ;
* comme déjà vu, l'interruption n°0x80 (= 128), qui correspond à un appel système, déclenche la fonction `syscall::syscall` ;

Les défauts de page
-------------------
Habituellement, le défaut de page est utilisé pour réaliser la pagination à la demande. Dans ce cas, aucune page n'est mappée à la mémoire physique à la création d'un processus. Evidemment, le premier accès à la mémoire créé un défaut de page. A charge pour la fonction associée d'associer un cadre de page de la mémoire physique à cette page et de charger la page.

Redox n'adopte pas, dans sa version 0.4.1, ce mécanisme.

.. code-block:: rust

    IDT[14].set_func(exception::page);

Allons voir la fonction `exception::page <https://gitlab.redox-os.org/redox-os/kernel/tree/master/src/kernel/arch/x86_64/interrupt/exception.rs>`_ :

.. code::rust

    interrupt_error_p!(page, stack, {
        let cr2: usize;
        asm!("mov rax, cr2" : "={rax}"(cr2) : : : "intel", "volatile");
        println!("Page fault: {:>016X}", cr2);
        stack.dump();
        stack_trace();
        ksignal(SIGSEGV);
    });

Qui est levée en cas de défault de page. Le principe en est simple : on déplace le contenu du registre `cr2` (l'adresse de la page appelée) sur le registre `rax` pour le lire. Le :code:`dump` affiche la pile et :code:`stack_trace` la suite des appels. Enfin, le signal `SIGSEGV` est envoyé au processus. On verra le détail des signaux dans la partie sur la communication inter-processus.

La différente entre :code:`interrupt` et :code:`interrupt_p` est simple :
* Redox pousse sur la pile, en plus de "scratch register", les "preserved registers" dont les valeurs sont préservées au moment de l'appel de fonction.
* Redox récupère le pointeur de pile et le transmet en argument à la fonction.

Le timer
--------
Le timer est initialisé dans la fonction :code:`pti::init()`. La fréquence choisie est un diviseur de 1,193182 MHz. Dans le cas de Redox, ce diviseur est 2685, soit une fréquence de 444,38 Hz, à savoir un tick toutes les 2,25 ms.

.. code:: rust

    interrupt!(pit, {
        // Saves CPU time by not sending IRQ event irq_trigger(0);

        const PIT_RATE: u64 = 2_250_286;

        {
            let mut offset = time::OFFSET.lock();
            let sum = offset.1 + PIT_RATE;
            offset.1 = sum % 1_000_000_000;
            offset.0 += sum / 1_000_000_000;
        }

        pic::MASTER.ack();

        // Wake up other CPUs
        ipi(IpiKind::Pit, IpiTarget::Other);

        // Any better way of doing this?
        timeout::trigger();

        if PIT_TICKS.fetch_add(1, Ordering::SeqCst) >= 10 {
            let _ = context::switch();
        }
    });

Dasn un premier temps, on ajoute la durée du tick, soit 2 250 286 ns à l'offset (qui possède une partie 0 en secondes et partie 1 en nanosecondes) pour mettre à jour le temps système.

On envoie au chip 8259 un ACK pour signifier que l'interruption a bien été reçue.

On envoie aux autres CPU l'information, puis on déclenche les événements dont la date est dépasée (à détailler).

Enfin, un nouveau processus est choisi.

Le clavier
----------

.. code:: rust

    interrupt!(keyboard, {
        trigger(1);
    });

La fonciton :code:`trigger` se contente d'envoyer un ACK au périhphérique (1) et de lancer un :code:`irq_trigger` avec 1 pour paramètre. On a donc une translation. Le code de :code:`irq_trigger` est :

.. code:: rust

    #[no_mangle]
    pub extern fn irq_trigger(irq: u8) {
        COUNTS.lock()[irq as usize] += 1;
        event::trigger(IRQ_SCHEME_ID.load(Ordering::SeqCst), irq as usize, EVENT_READ);
    }

On reviendra sur le passage de message dans Redox, qui est une des particularités de cet OS.

.. [#] 2.8.1 Figure 6-7. 64-Bit IDT Gate Descriptors dans <https://www.intel.com/content/dam/www/public/us/en/documents/manuals/64-ia-32-architectures-software-developer-vol-3a-part-1-manual.pdf>_.

.. [#] 2.8.1 Loading and Storing System Registers dans <https://www.intel.com/content/dam/www/public/us/en/documents/manuals/64-ia-32-architectures-software-developer-vol-3a-part-1-manual.pdf>_.

.. [#] Pour le détail, voir : `6.12.1 Exception- or Interrupt-Handler Procedures <https://www.intel.com/content/dam/www/public/us/en/documents/manuals/64-ia-32-architectures-software-developer-vol-3a-part-1-manual.pdf>`_.

.. [#] :code:`GDT_KERNEL_TLS << 3 | 0 = 0x18`

.. [#] Ceci est vu en annexe.
