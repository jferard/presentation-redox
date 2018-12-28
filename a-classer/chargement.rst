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

Le chargement
=============
Ce texte se concentre sur le chargeur UEFI (*Unified Extensible Firmware Interface*). En effet, la spécification UEFI offre des services bien plus étendus qu'un classique BIOS. En particulier, UEFI :
* initialise les segments et une pagination "identité"
* ouvre la porte A20
* peut charger un code arbitraire pour l'exécuter
* permet de découvrir les périphériques
* ...

Comment démarre un ordinateur ?
-------------------------------
BIOS
````
Avant UEFI, le démarrage d'un ordinateur était géré en totalité par le BIOS (Basic Input/Output System), un logiciel embarqué (firmware). Ce BIOS commence par une vérification du fonctionnement et de la configuration matérielle (Power-On Self-Test), ainsi que l'initialisation du matériel (configuration de la mémoire, du bus PCI, etc.). La vérification permet de lancer la recherche d'un périphérique amorçable (CD, clé USB, réseau, disque dur). Le BIOS lit ensuite le MBR du périphérique d'amorçage qui contient une routine d'amorçage chargée, dans le cas le plus simple, de charger l'OS, et dans le cas le plus courant de charger le bootloader (chargeur d'amorçage) présent sur la partition active. En effet, le MBR est le premier secteur du disque et sa taille est de 512 octets. Il doit donc passer la main à un chargeur d'amorçage qui va : 1. charger l'OS en mémoire ; 2. passer en mode protégé ; 3. mettre en place le runtime pour le noyau (notamment la pile).

UEFI
````
UEFI est un firmware qui propose des services normalisés et plus étendus que BIOS. L'initialisation ouvre la porte A20 et prépare la mémoire continue. UEFI permet de charger directement un fichier exécutable de taille quelconque (soit un bootloader, soit l'OS directement). UEFI fournit un accès direct et unique aux tables ACPI et de nombreuses fonctions clé en main (par oppositions aux interruptions du BIOS).

De manière pratique, un effet important de la standardisation de UEFI est la possibilté de programmer le chargement de l'OS en un langage de haut niveau (C en général, mais Rust dans le cas de Redox).

Le chargeur UEFI
----------------
Comme il faut définir un point d'entrée pour l'OS, il faut également définir un point d'entrée pour le chargeur. Le `Makefile` `redox/bootloader-efi/Makefile` montre ce point d'entrée :

.. code:: make

    $(BUILD)/boot.efi: $(BUILD)/boot.o $(LD)
    	$(LD) \
            ...
            --oformat pei-x86-64 \
            ...
    		--entry _start \
            ...
    		$< -o $@

Il s'agit d'un point d'entrée similaire à celui présent dans tout exécutable. Deux remarques : le fichier `boot.efi` est le fichier qui sera exécuté par le formware UEFI ; ce fichier est au format PEI.

La fonction :code:`_start` se trouve dans `redox/bootloader-efi/src/rt.rs` :

.. code:: rust

    #[no_mangle]
    pub extern "win64" fn _start(handle: uefi::Handle, uefi: &'static mut uefi::system::SystemTable) -> isize {
        unsafe {
            ::HANDLE = handle;
            ::UEFI = uefi;

            let _ = (uefi.BootServices.SetWatchdogTimer)(0, 0, 0, ptr::null());

            if let Err(err) = set_max_mode(uefi.ConsoleOut).into_result() {
                println!("Failed to set max mode: {:?}", err);
            }

            let _ = (uefi.ConsoleOut.SetAttribute)(uefi.ConsoleOut, 0x0F);

            uefi_alloc::init(::core::mem::transmute(&mut *::UEFI));
        }

        main();

        0
    }

Elle commence par récupérer les paramètres dans deux variables statiques. Puis on voit comment se réalise l'appel aux services UEFI : le watchdog timer est désactivé, le mode console est mis au mode le plus élevé et la couleur est blanc sur noir. Il y a ensuite sauvegarde de la table système UEFI, puis lancement de :code:`main`.

La fonction main se trouve dans `bootloader-efi/src/lib.rs`:

.. code:: rust

    fn main() {
        let uefi = unsafe { &mut *::UEFI };

        if let Err(err) = loader::main() {
            println!("Loader error: {:?}", err);
            let _ = io::wait_key();
        }

        unsafe {
            (uefi.RuntimeServices.ResetSystem)(ResetType::Cold, Status(0), 0, ptr::null());
        }
    }

On suit dans `bootloader-efi/src/loader/mod.rs`:

.. code:: rust

    pub fn main() -> Result<()> {
        ...
        pretty_pipe(&splash, inner)?;
        ...
    }

La fonction :code:`pretty_pipe` appelle :code:`TextDisplay::pipe` qui se trouve dans `/home/jferard/prog/rust/redox/bootloader-efi/src/text.rs` et qui appelle la fonction passée en argument. Ce chaînage permet d'afficher une image et du texte avant d'exécuter la fonction.


Lancement de `kstart`
---------------------
C'est donc la fonction :code:`inner` qui est appelée. Elle réalise les opérations suivantes:

* trouver le noyau (:code:`find(KERNEL)`) dans le système de fichier simple `bootloader-efi/src/fs.rs` ou dans un système de fichiers `bootloader-efi/src/loader/redoxfs/mod.rs` et le charger dans un vecteur.
* trouver le point d'entrée et copier le noyau à l'adresse `0x100000`. Ce point d'entrée du noyau est la fonction :code:`_kstart` (voir démarrage du noyau).
* copier l'environnement (`REDOXFS_UUID`) sur la pile
* appeler successivement les fonctions :code:`vesa`, :code:`exit_boot_services`, :code:`paging` et :code:`enter`.

Voyons en détail ces éléments.

Chargement du noyau
```````````````````
Voir les systèmes de fichiers.

Trouver le point d'entrée
`````````````````````````
Dans le format ELF, l'entry point se trouve à la position `0x18`. Initialisation de KERNEL ENTRY.

.. code:: rust

    println!("Copying Kernel...");
    unsafe {
        KERNEL_SIZE = kernel.len() as u64;
        println!("Size: {}", KERNEL_SIZE);
        KERNEL_ENTRY = *(kernel.as_ptr().offset(0x18) as *const u64);
        println!("Entry: {:X}", KERNEL_ENTRY);
        ptr::copy(kernel.as_ptr(), KERNEL_PHYSICAL as *mut u8, kernel.len());
    }

Les fonctions
`````````````
enter :

.. code:: rust

    unsafe fn enter() -> ! {
        let args = KernelArgs {
            kernel_base: KERNEL_PHYSICAL,
            kernel_size: KERNEL_SIZE,
            stack_base: STACK_VIRTUAL,
            stack_size: STACK_SIZE,
            env_base: STACK_VIRTUAL,
            env_size: ENV_SIZE,
        };

        let entry_fn: extern "C" fn(args_ptr: *const KernelArgs) -> ! = mem::transmute(KERNEL_ENTRY);
        entry_fn(&args);
    }
