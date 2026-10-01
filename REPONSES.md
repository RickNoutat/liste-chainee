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
