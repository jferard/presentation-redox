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

Le chargement
=============

Ce texte se concentre sur le chargeur UEFI.

/home/jferard/prog/rust/redox/bootloader-efi/src/rt.rs

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

Lancement de `kstart`
---------------------
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

Le chargeur `inner` de `bootloader-efi/src/loader/mod.rs` charge le noyau et se place à cette position :

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
