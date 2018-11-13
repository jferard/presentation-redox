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

Événements
==========

Voici la définition de :code:`event::trigger` :

.. code:: rust

    pub fn trigger(scheme: SchemeId, number: usize, flags: usize) {
        let registry = registry();

        if let Some(queue_list) = registry.get(&RegKey { scheme, number }) {
            for (queue_key, queue_flags) in queue_list.iter() {
                let common_flags = flags & queue_flags;
                if common_flags != 0 {
                    let queues = queues();
                    if let Some(queue) = queues.get(&queue_key.queue) {
                        queue.queue.send(Event {
                            id: queue_key.id,
                            flags: common_flags,
                            data: queue_key.data
                        });
                    }
                }
            }
        }
    }

La défintion du registre retourné par :code:`registry()` est simple :

.. code:: rust

    type Registry = BTreeMap<RegKey, BTreeMap<QueueKey, usize>>;

Il s'agit donc d'un dictionnaire doublement ordonné : :code:`RegKey -> QueueKey -> usize`. Commençons par la fin : la valeur :code:`usize` représente un ensemble de drapeaux associés à une :code:`RegKey` et une :code:`QueueKey`.

Une :code:`RegKey` est définie par un :code:`SchemeId` (type appuyé sur :code:`usize` et qui définit un type de scheme) et un numéro. Par exemple :code:`(IRQ_SCHEME_ID, 1)` pour une interruption clavier.

La :code:`QueueKey` est quant à elle définie de la manière suivante :

.. code:: rust

    pub struct QueueKey {
        pub queue: EventQueueId,
        pub id: usize,
        pub data: usize
    }

Le champ :code:`queue` est une entrée dans une autre table de type :

.. code:: rust

    pub type EventQueueList = BTreeMap<EventQueueId, Arc<EventQueue>>;

    pub struct EventQueue {
        id: EventQueueId,
        queue: WaitQueue<Event>,
    }


Lorsque les drapeaux de l'appel et les drapeaux associés à la :code:`RegKey` et la :code:`QueueKey`trouvées ne sont pas exclusifs, on ajoute donc à cette :code:`WaitQueue` une :code:`Event`.

La WaitQueue ajoute l'événement à sa liste interne, puis notifie la WaitCondition.

kernel/syscall/src/data.rs

.. code:: rust

    pub struct Event {
        pub id: usize,
        pub flags: usize,
        pub data: usize
    }
