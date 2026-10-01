# TP - Sortir du fichier unique : réponses

Environnement : macOS (puce Apple), `gcc` est en réalité Apple clang 21. Les messages
d'erreur ci-dessous sont donc ceux de clang et de l'éditeur de liens d'Apple (`ld`),
leur formulation diffère légèrement de celle de GCC sous Linux.

## Exercice 0 - Le dossier de projet et le premier commit

**Pourquoi ne versionne-t-on jamais les `.o` ni l'exécutable ?**

Ce sont des fichiers **produits** : on les régénère à tout moment à partir des sources
avec une commande (`gcc`, plus tard `make`). Les versionner n'apporte aucune
information et pose plusieurs problèmes :

- ils sont binaires, donc un diff ne montre rien de lisible et les conflits de fusion
  sont impossibles à résoudre ;
- ils dépendent de la machine (processeur, système, version du compilateur) : un `.o`
  produit sur un Mac arm64 ne sert à rien sur un PC Linux x86-64 ;
- ils changent à chaque recompilation, ce qui pollue chaque commit de modifications
  sans intérêt ;
- ils peuvent être en retard sur les sources : un `.o` versionné qui ne correspond plus
  au `.c` est une source de bugs incompréhensibles.

Le dépôt ne doit contenir que ce qu'un humain écrit ; le reste se reconstruit.

## Exercice 1 - Découper la liste chaînée en trois fichiers

| Question | Réponse |
| --- | --- |
| A | `main.c` manipule des `Maillon` : il déclare un `Maillon *`, et les prototypes de `liste.h` en parlent. Le compilateur traite chaque `.c` séparément et ne voit **que** ce que ce fichier inclut ; il ne lit jamais `liste.c` en compilant `main.c`. Si le type était défini dans `liste.c`, `main.c` ne le connaîtrait pas et la compilation échouerait (`unknown type name 'Maillon'`). Le `.h` est le seul endroit partagé par tous les fichiers qui utilisent le module. |
| B | Pour deux raisons. D'abord `liste.c` a lui aussi besoin de la définition de `Maillon` (et de `bool`), qui ne se trouve que dans le `.h`. Ensuite, c'est une vérification : le compilateur voit à la fois la déclaration (prototype du `.h`) et la définition (corps dans le `.c`) et signale toute incohérence entre les deux (type de retour, paramètres). Sans cette inclusion, une signature modifiée dans le `.c` mais pas dans le `.h` compilerait sans erreur et planterait à l'exécution. |

## Exercice 2 - Compiler en deux temps, à la main

Commandes :

```bash
gcc -Wall -Wextra -c main.c      # produit main.o
gcc -Wall -Wextra -c liste.c     # produit liste.o
gcc -o demo main.o liste.o       # édition de liens
./demo
```

Sortie du programme :

```
liste     : 50 -> 40 -> 30 -> 20 -> 10 -> NULL
longueur  : 5
contient 30 : oui
liberee
```

`git status` reste vide après la compilation : le `.gitignore` écarte bien `main.o`,
`liste.o` et `demo`.

| Question | Réponse |
| --- | --- |
| A | Deux : `main.o` et `liste.o`, un par fichier `.c`. Il n'y a pas de `liste.h.o`, car un `.h` ne se compile jamais seul. Ce n'est pas une unité de compilation : le préprocesseur recopie son texte dans chaque `.c` qui l'inclut, et c'est ce `.c` qui est compilé. Le `.h` ne contient d'ailleurs que des déclarations (type, prototypes), qui ne produisent aucun code machine. |
| B | Elle recompile **tous** les `.c` à chaque fois, même ceux qui n'ont pas changé, et jette les `.o` intermédiaires (fichiers temporaires effacés aussitôt). Rien n'est réutilisable : modifier une ligne de `main.c` oblige à recompiler aussi `liste.c`. Sur un projet de centaines de fichiers, cela fait passer une recompilation de quelques secondes à plusieurs minutes. En deux temps, on ne recompile que le `.c` modifié et on relie avec les `.o` déjà présents (c'est ce que `make` automatisera). Elle mélange aussi les deux phases : on voit moins facilement si une erreur vient du compilateur ou de l'éditeur de liens. |

## Exercice 3 - Provoquer les trois erreurs classiques

| Cas | Premier message exact | Compilation ou lien |
| --- | --- | --- |
| 1 | `Undefined symbols for architecture arm64: "_liste_afficher", referenced from: _main in main.o` puis `ld: symbol(s) not found for architecture arm64` | Lien |
| 2 | `main.c:5:5: error: use of undeclared identifier 'Maillon'` (13 erreurs au total) | Compilation |
| 3 | `./liste.h:4:16: error: redefinition of 'Maillon'` | Compilation |

À titre indicatif, sous GCC/Linux on obtiendrait plutôt :
`undefined reference to 'liste_inserer'` suivi de `collect2: error: ld returned 1 exit status`
(cas 1), `error: unknown type name 'Maillon'` (cas 2) et
`error: redefinition of 'struct Maillon'` (cas 3).

| Question | Réponse |
| --- | --- |
| 1 | Le cas 1. Plusieurs indices : la commande `gcc -o demo main.o` ne compile rien, elle ne fait que relier des `.o` ; le message est signé `ld` (« linker command failed », sous Linux `collect2: ... ld returned 1`) ; il ne cite aucun fichier source ni numéro de ligne, seulement un fichier objet (`main.o`) et des **symboles** (`_liste_afficher`, avec le `_` ajouté par macOS). Le compilateur avait accepté `main.c` puisque les prototypes étaient là : c'est seulement au moment d'assembler qu'on découvre qu'aucun `.o` ne contient le corps de ces fonctions. |
| 2 | Le premier : `use of undeclared identifier 'Maillon'` (GCC : `unknown type name 'Maillon'`). Il désigne la vraie cause : le type n'est pas connu, donc le `.h` qui le définit n'est pas inclus. Les douze autres en découlent : la déclaration de `liste` ayant échoué, chaque usage de `liste` est « undeclared », et les fonctions `liste_*` sont inconnues parce que leurs prototypes étaient dans le même `.h`. On corrige la première erreur, toutes les autres disparaissent : il faut toujours lire la première erreur en premier. |
| 3 | Dès qu'un `.h` en inclut un autre. Par exemple un second module `pile.h` dont les prototypes utilisent `Maillon` et qui fait donc `#include "liste.h"` : si `main.c` inclut à la fois `liste.h` et `pile.h`, `liste.h` est recopié deux fois dans la même unité de compilation sans que personne ait écrit deux fois la même ligne. Dans notre projet, chaque `.c` inclut `liste.h` une seule fois, et l'inclure dans deux `.c` différents ne pose aucun problème puisqu'ils sont compilés séparément. |

Code rétabli (`git checkout main.c liste.h`), puis recompilé de zéro avec
`-Wall -Wextra -Werror` : aucun avertissement, même sortie qu'à l'exercice 2.

## Exercice 4 - Écrire un premier Makefile

Version de make : GNU Make 3.81, celle fournie par Apple.

`make clean` puis `make` :

```
rm -f main.o liste.o demo
gcc -Wall -Wextra -std=c11 -g -c main.c
gcc -Wall -Wextra -std=c11 -g -c liste.c
gcc -Wall -Wextra -std=c11 -g -o demo main.o liste.o
```

Second `make` sans modification :

```
make: `demo' is up to date.
```

| Question | Réponse |
| --- | --- |
| A | `make` compare les dates de modification. Pour chaque cible, il regarde si le fichier existe et s'il est plus récent que toutes ses dépendances. `main.o` est plus récent que `main.c` et `liste.h`, `liste.o` plus récent que `liste.c` et `liste.h`, et `demo` plus récent que les deux `.o` : tout est à jour, aucune commande n'a besoin d'être relancée. |
| B (message exact) | Avec GNU Make 3.81 (macOS), **aucun message d'erreur** : `make: Nothing to be done for 'demo'.` et le code de sortie vaut 0. Sur un dossier propre (`make clean && make`), make compile `main.o` et `liste.o` mais ne produit jamais `demo` : la ligne indentée par des espaces est ignorée sans avertissement et la règle `demo` se retrouve sans commande (`make -p` le confirme). Avec GNU Make 4.x (Linux), on obtient à la place l'erreur attendue `Makefile:2: *** missing separator.  Stop.`. Dans les deux cas, la ligne n'est pas reconnue comme une commande ; la version d'Apple est simplement plus traîtresse puisqu'elle échoue en silence. |

## Exercice 5 - La dépendance au fichier d'en-tête

| Étape | Ce que make recompile |
| --- | --- |
| 2, avec la dépendance | `main.o` **et** `liste.o`, puis l'édition de liens de `demo` (3 commandes) |
| 4, sans la dépendance | `liste.o` seulement, puis l'édition de liens de `demo` (2 commandes) : `main.o` n'est pas recompilé |

| Question | Réponse |
| --- | --- |
| A | Avec la dépendance, modifier `liste.h` recompile les deux `.c` qui l'incluent. Sans elle, make ne sait pas que `main.c` utilise `liste.h` : il ne regarde que la date de `main.c`, qui n'a pas changé, et garde l'ancien `main.o`. Seul `liste.o` est reconstruit, et l'exécutable mélange un objet à jour et un objet périmé. make ne lit pas le code : il ne connaît que les dépendances qu'on lui écrit. |
| B | Un exécutable incohérent : `liste.o` est compilé avec la nouvelle structure, `main.o` avec l'ancienne. Les deux ne sont plus d'accord sur la taille d'un `Maillon` (`sizeof`) ni sur la position (le décalage) de chaque champ. Aucune erreur ni aucun avertissement : l'éditeur de liens ne vérifie que les **noms** des symboles, jamais les types. Dès que `main.c` lit un champ ou alloue un `Maillon`, il lit au mauvais endroit ou alloue trop peu : comportement indéfini, bug silencieux. |

Expérience faite pour la question B, sur une copie jetable : Makefile fautif, et une
ligne ajoutée dans `main.c` pour afficher `liste->valeur`. On ajoute ensuite `int id;`
avant `valeur` dans `Maillon`, puis on lance `make` : seul `liste.c` est recompilé.

```
avant : tete      : 50 (taille Maillon vue par main : 16)
apres : tete      : 0 (taille Maillon vue par main : 16)
```

`main.o` lit toujours la valeur au décalage 0, où se trouve maintenant `id`. La liste
contient bien 50, mais `main` affiche 0. La taille reste 16 octets uniquement grâce au
remplissage d'alignement : avec un champ plus gros, elle changerait aussi.

Dépendance rétablie, puis Makefile remplacé par la version complète (variables
`CC`/`CFLAGS`/`OBJ`, règle générique `%.o: %.c liste.h`). Vérifications : même
résultat qu'avant (`make clean && make` lance les 3 mêmes commandes, avec en plus un
`-o main.o` explicite), `make` relancé affiche `` `demo' is up to date ``,
`touch liste.h` recompile les deux `.o`, et `touch main.c` ne recompile que `main.o`.
