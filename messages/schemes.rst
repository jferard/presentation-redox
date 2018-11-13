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

Un exemple avec les Schemes
===========================

L'entrée :code:`stdin` est un Scheme de type :code:`DebugScheme` : :code:`open` l'ajoute à la liste des handles, :code:`read` appelle `receive_into` sur une WaitQueue. C'est une lecture bloquante sur une WaitCondition.

Les WaitCondition
Tout wait bloque le contexte courant et l'ajoute à la liste des contextes. Tout notify débloque un contexte et l'enlève de la liste courante.
