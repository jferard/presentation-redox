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

*Page Table Isolation*
======================
https://fedoramagazine.org/kpti-new-kernel-feature-mitigate-meltdown/

pti::map et pti::unmap : change le pointeur de pile utiliser avant de déclencher la fonction d'interruption.

Le pointeur de pile passe la PTI_CPU_STACK propre à chaque CPU à la PTI_CONTEXT_STACK avant de déclencher la fonction. (Copie de la pile CPU -> CONTEXT).
