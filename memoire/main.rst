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

La mémoire - présentation
=========================

Le problème
-----------
La gestion de la mémoire est une des composantes essentielles d'un OS. Pour simplifier, il s'agit d'un ensemble de case où le programme en cours d'exécution (le processus) va pouvoir stocker de l'information au cours de sa vie.

Dans le cadre de la multiprogrammation, où plusieurs processus se partagent les ressources (mémoire, processeur, périphériques), c'est à l'OS de veiller au bon partage de la mémoire. Chaque processus doit avoir l'illusion qu'il est seul à disposer d'une mémoire.

De plus, il faut assurer le partage d'une ressource limitée entre des processus dont les besoins en mémoire cumulés peuvent excéder la mémoire physique disponible.

Les solutions
-------------
La première contrainte, à savoir la création d'espaces de mémoire indépendants entre processus, est résolue par la segmentation de la mémoire. Les processus disposent chacun d'un ou plusieurs segments de mémoire protégés contre la lecture, l'écriture ou l'exécution par d'autres processus.

La seconde contrainte, celle d'une mémoire illimitée, est résolue par la pagination. L'astuce consiste à utiliser le fait qu'un processus n'utilise pas toute la mémoire qui lui a été fournie. On divise l'espace mémoire alloué (virtuellement) au processus en pages et la mémoire physique en cadres de page. Lorsqu'une de ces pages est référencée, soit elle possède un cadre de page correspondant dans la mémoire physique, soit il faut charger le contenu de la page depuis un support de stockage secondaire vers la mémoire physique et réaliser l'association page demandée - nouveau cadre de page.

Ces deux solution impliquent une participation du matériel :
* les segments sont gérés par le matériel (accès et protection) grâce à des registres ;
* la pagination assurée par la MMU (Memory Management Unit) au moyen d'une table des pages fournie  par l'OS.
