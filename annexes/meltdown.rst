La faille Meltdown
==================
Principe
--------
Soit `a` une adresse dans le noyau, contenu une valeur `X` (:code:`*a = X`). Que se passe-t-il si on, en mode utilisateur, demande au processeur :
* de charger dans le registre R1 la valeur à l'adresse `a` (:code:`R1 <- *a = X`);
* puis de charger dans le registre R2 la valeur à l'adresse R1 (:code:`R2 <- *R1 = *X`).

L'intuition nous dit que la première instruction va échouer en raison des permissions marquées dans la table des pages, et que la seconde ne sera donc pas exécutée. Eh bien non. Le CPU dispose de différentes unités de calcul qui prennent parfois les instructions simultanément. Il est donc possible que la seconde instruction soit exécutée avant que le refus de charger la valeur soit effectif. Le processeur jette alors au rebut les résultats des deux instructions et tout se passe comme l'intuition nous l'avait suggéré.

Mais il reste un point intéressant : le processeur a chargé la valeur à l'adresse R1, soit l'adresse `X`. Et ceci a un impact sur le TLB (*Translation Lookahaed Buffer*) : les accès futurs à l'adresse `X` seront plus rapides. Il ne reste à l'attaquant qu'à tester la vitesse d'accès de toutes les pages pour obtenir la valeur `X`.

Cette méthode permet en théorie d'accéder aux pages dont les permissions en lecture requièrent un privilège Ring 0.

Réponse
-------
Ceci soulève une question intéressante : mais pourquoi `X` se trouvait dans la table des pages d'un obscur processus utilisateur ? Pourquoi l'adresse `a`, du noyau, est mappée dans l'espace utilisateur ? Pour des raisons de performance. Pour ne pas avoir à recharger une table des pages et invalider le TLB.

Une réponse simple est donc la *Page Table Isolation*. Pour désamorcer toute la séquence, il faut que `X` ne soit pas là. Donc que l'adresse `a` ne soit pas mappée dans l'espace utilisateur.

L'impact en terme de performance existe. Au niveau du code, on verra une modification du mappage au moment des interruptions, avec les fonctions :code:`pti::map()` (avant l'exécution d'une routine en mode superviseur) et :code:`pti::unmap()` (avant le retour en mode utilisateur) :

.. code:: rust

    pub unsafe fn map() {
        // Switch to per-context stack
        switch_stack(PTI_CPU_STACK.as_ptr() as usize + PTI_CPU_STACK.len(), PTI_CONTEXT_STACK);
    }

    pub unsafe fn unmap() {
        // Switch to per-CPU stack
        switch_stack(PTI_CONTEXT_STACK, PTI_CPU_STACK.as_ptr() as usize + PTI_CPU_STACK.len());
    }

    unsafe fn switch_stack(old: usize, new: usize) {
        let old_rsp: usize;
        asm!("" : "={rsp}"(old_rsp) : : : "intel", "volatile");

        let offset_rsp = old - old_rsp;

        let new_rsp = new - offset_rsp;

        ptr::copy_nonoverlapping(
            old_rsp as *const u8,
            new_rsp as *mut u8,
            offset_rsp
        );

        asm!("" : : "{rsp}"(new_rsp) : : "intel", "volatile");
    }


En pratique, on voit qu'il s'agit d'une copie de la pile. Attention, car :
* :code:`PTI_CPU_STACK` désigne un tableau destiné à stocker une pile. Ce tableau qui grandit vers les adresses hautes : :code:`PTI_CPU_STACK.as_ptr()` est donc le bas de la pile et :code:`PTI_CPU_STACK.as_ptr() + PTI_CPU_STACK.len()` le sommet.
* :code:`PTI_CONTEXT_STACK` désigne le sommet de la pile.

Voici ce qui se passe à l'initialisation :

.. code:: rust

    pub unsafe fn set_tss_stack(stack: usize) {
        use arch::x86_64::pti::{PTI_CPU_STACK, PTI_CONTEXT_STACK};
        TSS.rsp[0] = (PTI_CPU_STACK.as_ptr() as usize + PTI_CPU_STACK.len()) as u64;
        PTI_CONTEXT_STACK = stack;
    }

Ici, `stack` désigne le haut de la pile fourni par le chargeur. :code:`TSS.rsp[0]` est le pointeur de pile en mode Ring 0. On le fait pointer sur la pile temporaire, tandis que la pile du contexte pointe sur la pile telle que vue par le chargeur.

La fonction :code:`switch_stack` est donc claire : on copie les octets `old_rsp..old` (où `old` est le haut de la pile) à la position `new_rsp..new` (où `new` est le haut de pile). Par conséquent :
* en début d'interruption, :code:`pti::map()` copie la pile de sauvegarde :code:`PTI_CPU_STACK` à la place de la pile :code:`PTI_CONTEXT_STACK`.
* en fin d'interruption, :code:`pti::unmap()` sauvegarde à nouveau la pile :code:`PTI_CONTEXT_STACK` dans le tableau :code:`PTI_CPU_STACK`.

Bibliographie
-------------
https://www.ovh.com/fr/blog/failles-de-securite-spectre-meltdown-explication-3-failles-mesures-correctives-public-averti/
