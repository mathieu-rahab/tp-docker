# tp-docker

## Exercice 1 — Comprendre le cycle de vie des processus dans un conteneur

### Questions

1. Quel est le rôle du processus principal dans un conteneur ?
Le processus principal (PID 1) exécute l'application du conteneur.

2. Quelle différence existe-t-il entre `docker stop` et `docker kill` ?
`docker stop` demande un arrêt propre et `docker kill` arrête immédiatement le conteneur.

3. Pourquoi un conteneur s'arrête-t-il lorsque son processus principal se termine ?
Un conteneur dépend de son processus principal : quand celui-ci se termine, le conteneur s’arrête.

4. Quelle différence existe-t-il entre `SIGTERM` et `SIGKILL` ?
`SIGTERM` demande un arrêt propre et `SIGKILL` force l’arrêt sans possibilité de le gérer.

5. Pourquoi est-il important qu'une application gère correctement les signaux d'arrêt ?
Gérer les signaux permet de terminer proprement les tâches et de préserver les données.


## Exercice 2 — Comprendre les couches d'une image et le système de fichiers

### Questions

1. Quelle différence existe-t-il entre les couches d'une image et la couche inscriptible d'un conteneur ?
Les couches de l’image sont en lecture seule. Le conteneur ajoute une couche inscriptible.


2. À quoi sert le mécanisme *copy-on-write* ?
Le *copy-on-write* copie un fichier dans la couche inscriptible seulement lorsqu’on le modifie.


3. Pourquoi les modifications effectuées dans un conteneur ne modifient-elles pas directement l'image d'origine ?
L’image reste inchangée et peut être partagée entre plusieurs conteneurs.


4. À quoi sert la commande `docker diff` ?
`docker diff` affiche les fichiers ajoutés, modifiés ou supprimés dans le conteneur.


5. Pourquoi les modifications apportées à la couche inscriptible sont-elles perdues lorsqu'on supprime le conteneur ?
La couche inscriptible appartient au conteneur : sa suppression efface ses modifications.


## Exercice 3 — Construire une image avec plusieurs étapes (Multi-stage build)

Sources : [tp-multistage/main.go](tp-multistage/main.go) et [tp-multistage/Dockerfile](tp-multistage/Dockerfile).

### Questions

1. Pourquoi utiliser plusieurs instructions `FROM` ?
Chaque `FROM` démarre une étape distincte : on compile dans une image complète, puis on ne garde que le résultat dans l'image finale.

2. Quelle différence existe-t-il entre l'étape `builder` et l'étape finale ?
`builder` contient Go et les sources pour compiler, l'étape finale ne contient que l'exécutable.

3. Pourquoi l'image finale ne contient-elle pas le compilateur Go ?
Seul ce qui est copié avec `COPY --from=builder` arrive dans l'image finale ; le reste de l'étape `builder` est abandonné.

4. Quels sont les avantages d'une image finale minimale ?
Elle est plus légère, donc plus rapide à télécharger et à déployer, et plus sûre car elle ne contient ni shell ni paquets inutiles.

5. Quelles précautions faut-il prendre avant d'utiliser une image `scratch` en production ?
Compiler un binaire statique (`CGO_ENABLED=0`), ajouter ce qui manque si besoin (certificats CA, fuseaux horaires), utiliser un utilisateur non root et prévoir le diagnostic sans shell (`docker exec sh` est impossible).


## Exercice 4 — Renforcer la sécurité d'un conteneur

### Questions

1. Pourquoi est-il préférable d'exécuter une application avec un utilisateur non privilégié ?
Si l'application est compromise, l'attaquant n'a que des droits limités, dans le conteneur comme en cas d'évasion vers l'hôte.

2. Que sont les capacités Linux ?
Un découpage des privilèges de root en droits unitaires (ex. `CHOWN`, `NET_BIND_SERVICE`) que l'on peut retirer un par un.

3. Quelle différence existe-t-il entre `--read-only` et `--tmpfs` ?
`--read-only` interdit l'écriture sur le système de fichiers racine ; `--tmpfs` monte un espace inscriptible en mémoire, non persistant.

4. Pourquoi un conteneur peut-il avoir besoin de zones inscriptibles malgré un système de fichiers racine en lecture seule ?
Beaucoup d'applications écrivent des fichiers temporaires, des caches, des fichiers PID ou des sockets pour fonctionner.

5. Ces options suffisent-elles à garantir une isolation parfaite ? Justifiez votre réponse.
Non : le conteneur partage le noyau de l'hôte, donc une faille noyau ou un montage dangereux (socket Docker, volume sensible) peut casser l'isolation. Ces options réduisent le risque sans l'annuler.


## Exercice 5 — Diagnostiquer un conteneur qui se termine anormalement

### Questions

1. Quelle différence existe-t-il entre un conteneur arrêté et un conteneur supprimé ?
Un conteneur arrêté existe encore (fichiers, journaux, métadonnées) et peut être inspecté ou redémarré ; un conteneur supprimé a tout perdu.

2. Que signifie un code de sortie différent de zéro ?
Le processus principal s'est terminé en erreur ; la valeur précise la cause (ici `42`, choisi par l'application ; `137` pour un `SIGKILL`).

3. Quelle différence existe-t-il entre les sorties standard et d'erreur ?
`stdout` porte les messages normaux et `stderr` les erreurs ; ce sont deux flux séparés, tous deux capturés par Docker.

4. Pourquoi les journaux sont-ils utiles lorsqu'un conteneur se termine trop rapidement ?
On ne peut plus faire de `docker exec` dans un conteneur arrêté, alors que ses journaux restent consultables tant qu'il n'est pas supprimé.

5. Pourquoi faut-il examiner le code de sortie au lieu de se fier uniquement au message d'erreur ?
Le message peut être absent ou trompeur, alors que le code est fiable et exploitable automatiquement (scripts, politiques de redémarrage, orchestrateurs).


## Exercice 6 — Configurer la rotation des journaux

### Questions

1. Pourquoi des journaux trop volumineux peuvent-ils poser problème ?
Ils peuvent saturer le disque de l'hôte, ce qui bloque Docker et les autres services, et ils deviennent difficiles à exploiter.

2. À quoi sert un pilote de journaux Docker ?
Il définit où et comment les sorties des conteneurs sont stockées ou envoyées (`local`, `json-file`, `syslog`, `journald`…).

3. Quelle différence existe-t-il entre la consultation des journaux avec `docker logs` et leur stockage sur le long terme ?
`docker logs` lit les journaux locaux du conteneur, tronqués par la rotation et perdus à sa suppression ; le long terme demande une centralisation externe.

4. Pourquoi la rotation des journaux est-elle importante sur un serveur ?
Un serveur tourne longtemps : sans rotation les fichiers grossissent sans limite (le pilote `json-file` par défaut n'en impose aucune).

5. Pourquoi faut-il choisir une stratégie de journalisation adaptée à l'environnement de production ?
Il faut arbitrer entre volume, durée de rétention, coût et obligations légales, et pouvoir rechercher dans les journaux de plusieurs hôtes.


## Exercice 7 — Explorer les processus et les modifications d'un conteneur en fonctionnement

### Questions

1. Quelle différence existe-t-il entre `docker exec` et `docker run` ?
`docker run` crée un nouveau conteneur à partir d'une image ; `docker exec` lance un processus supplémentaire dans un conteneur déjà actif.

2. À quoi sert `docker top` ?
Il liste les processus du conteneur vus depuis l'hôte, même si l'image ne contient pas `ps`.

3. Pourquoi `docker diff` peut-il être utile pendant un diagnostic ?
Il montre ce qui a changé par rapport à l'image : fichier inattendu, configuration modifiée, données écrites hors volume.

4. Quelles informations fournit `docker stats` ?
L'utilisation du CPU, de la mémoire (usage et limite), du réseau, des entrées/sorties disque et le nombre de processus.

5. Pourquoi faut-il éviter de modifier manuellement un conteneur en production sans documenter les changements ?
Ces changements sont perdus à la recréation du conteneur et ne sont pas reproductibles, il faut corriger l'image ou la configuration puis redéployer.


## Conclusion

Grâce au build multi-stage, l'image finale ne contient que le binaire Go : elle pèse 12,1 Mo contre 388 Mo pour l'image de compilation.
Pour la sécurité, l'utilisateur non root, le système de fichiers en lecture seule et la suppression des capacités limitent les dégâts si l'application est compromise.