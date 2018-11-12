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

Le système de fichiers -- présentation
======================================
Il est souvent nécessaire de disposer d'un espace de stockage :
* plus important que la mémoire physique disponible ;
* plus pérenne que la durée de vie d'un processus ;
* qui peut éventuellement être partagée entre plusieurs processus ou plusieurs machines.

Cet espace de stockage est un disque dur, un disque flash, ou tout autre support de stockage de masse. L'utilisateur manipule ce stockage par le biais d'une abstraction : le fichier, et plus généralement le système de fichiers.

Redox reprend dans son système de fichiers l'ensemble des idées déjà utilisées par les autres OS : les fichiers sont organisés en une arborescence de répertoires. Ils possèdent des droits (lecture, écriture, exécution). Etc.
