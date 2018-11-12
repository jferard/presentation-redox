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

Messages
========
Transformer l'asynchrone en bloquant.

/redox/relibc/src/header/stdio/mod.rs

.. code:: rust

    /// Get a single char from `stdin`
    #[no_mangle]
    pub extern "C" fn getchar() -> c_int {
        fgetc(unsafe { &mut *stdin })
    }
