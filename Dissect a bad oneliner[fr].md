## Disséquer une mauvaise One-Liner ##


```$ ls *.zip | while read i; do j=`echo $i | sed 's/.zip//g'`; mkdir $j; cd $j; unzip ../$i; cd ..; done```

>Il s'agit d'un One-Liner que quelqu'un a demandé dans `#bash`. **Il y a plusieurs choses incorrectes dans celle-ci. Nous allons les décomposer !**

This is an actual one-liner someone asked about in `#bash`. **There are several things wrong with it. Let's break it down!**

```$ ls *.zip | while read i; do ...; done```

>(Veuillez lire http://mywiki.wooledge.org/ParsingLs) Cette commande exécute `ls` sur l'expansion de `*.zip`. En supposant qu'il existe des noms de fichiers dans le répertoire courant qui se terminent par '.zip', `ls` donnera une liste explicite de ces noms. La sortie de `ls` n'est pas pour l'analyse. Mais tout comme dans bash et sh nous pouvons en toute sécurité appliquer une boucle sur la glob elle-même :

(Please read http://mywiki.wooledge.org/ParsingLs) This command executes `ls` on the expansion of `*.zip`. Assuming there are filenames in the current directory that end in '.zip', `ls` will give a human-readable list of those names. The output of `ls` is not for parsing. But in sh and bash alike we can loop safely over the glob itself:

```$ for i in *.zip; do j=`echo $i | sed 's/.zip//g'`; mkdir $j; cd $j; unzip ../$i; cd ..; done```

>Décomposons un peu plus !

Let's break it down some more!

```j=`echo $i | sed 's/.zip//g'` # where $i is some name ending in '.zip'```

>Le but ici semble être juste d'obtenir le nom du fichier sans l'extension `.zip`. En fait, il y a déjà une commande POSIX(r)-conforme pour ce faire : `basename` ! La mise en œuvre ici n'est pas optimale de plusieurs façons, mais la seule chose vraiment faillible qui est sujette à cela est "`echo $i`". La résonance (Faisant écho) d'une valeur sans guillemets signifie que [wordsplitting](http://wiki.bash-hackers.org/syntax/expansion/wordsplit) aura lieu, de sorte que toute les espaces dans `$i` seront essentiellement normalisés. En `sh` il est nécessaire d'utiliser une commande externe et un sous-Shell pour atteindre l'objectif, mais nous pouvons éliminer un tube (pipelines) (Sous-Shell, commande externe, et tubes véhiculent surcharge supplémentaire lorsqu'ils se lancent, de sorte qu'ils peuvent vraiment nuire aux performances dans une boucle). Pour faire bonne mesure, nous allons utiliser le plus lisible et [moderne](http://wiki.bash-hackers.org/syntax/expansion/cmdsubst) `$()` en lieu et place des classiques apostrophes inversées :

The goal here seems to be just get the filename without its `.zip` extension. In fact, there is a POSIX(r)-compliant command to do this already: `basename`! The implementation here is suboptimal in several ways, but the only thing that's genuinely error-prone with this is "`echo $i`". Echoing an *unquoted* value means that [wordsplitting](http://wiki.bash-hackers.org/syntax/expansion/wordsplit) will take place, so any whitespace in `$i` will essentially be normalized. In `sh` it is necessary to use an external command and a subshell to achieve the goal, but we can eliminate a pipe (subshells, external commands, and pipes carry extra overhead when they launch, so they can really hurt performance in a loop). Just for good measure, let's use the more readable, [modern](http://wiki.bash-hackers.org/syntax/expansion/cmdsubst) `$()` instead of oldschool backticks:

```sh $ for i in *.zip; do j=$(basename "$i" ".zip"); mkdir $j; cd $j; unzip ../$i; cd ..; done```

>Nul besoin en Bash d'un sous-shell ou de la commande externe `basename`. Voir [élimination de sous-chaînes avec l'expansion de paramètres](http://wiki.bash-hackers.org/syntax/pe#substring_removal):

In Bash we don't need the subshell or the external `basename` command. See [Substring removal with parameter expansion](http://wiki.bash-hackers.org/syntax/pe#substring_removal):

```bash $ for i in *.zip; do j="${i%.zip}"; mkdir $j; cd $j; unzip ../$i; cd ..; done```

>Continuons :
Let's keep going:

```$ mkdir $j; cd $j; ...; cd ..```

>En tant que programmeur, vous ne savez **jamais** dans quelle situation votre programme sera exécuté. Même si vous les bonnes pratiques suivante, elles ne feront pas de mal: Chaque fois que votre commande suivante dépend de la réussite de votre commande précédente, vérifier la présence de succès !  Vous pouvez le faire avec la conjonction "`&&`", ce qui signifie que si la commande précédente échoue, bash n'essaiera pas d'exécuter la commande suivante(s). Cela est totalement compatible POSIX(r). Oh, et ne oubliez pas ce que j'ai dit sur [wordsplitting](http://wiki.bash-hackers.org/syntax/expansion/wordsplit) à l'étape précédente, si vous n'entourez pas `$j` entre guillemets, wordsplitting peut arriver à nouveau.

As a programmer, you **never** know the situation under which your program will run. Even if you do, the following best practice will never hurt: Whenever your following command depends on the success of your previous command(s), check for success! You can do this with the "`&&`" conjunction, which means that if the previous command fails, bash will not try to execute the following command(s). It's fully POSIX(r). Oh, and remember what I said about [wordsplitting](http://wiki.bash-hackers.org/syntax/expansion/wordsplit) in the previous step? Well, if you don't quote `$j`, wordsplitting can happen again.

```$ mkdir "$j" && cd "$j" && ... && cd ..```

>C'est presque juste, mais il y a un problème -- que se passe-t-il si `$j` contient une barre oblique ? Après `cd ..` cela ne retournera pas dans le répertoire d'origine. Incorrecte ! Sauf avec `cd -` qui provoque le retour au répertoire de travail précédent, ce qui est un bien meilleur choix : 

That's almost right, but there's one problem -- what happens if `$j` contains a slash? Then `cd ..` will not return to the original directory. That's wrong! `cd -` causes cd to return to the previous working directory, so it's a much better choice:

```$ mkdir "$j" && cd "$j" && ... && cd -```

>(S'il vous est venu à l'esprit que j'ai oublié de vérifier la présence de succès après `cd -`, bravo ! Vous pourriez le faire avec `{ cd - || break; }`, mais je vais l'omettre parce que c'est verbeux et je crois qu'il est probable que nous serons en mesure de revenir à notre répertoire de travail d'origine sans problème.)

(If it occurred to you that I forgot to check for success after `cd -`, good job! You could do this with `{ cd - || break; }`, but I'm going to leave that out because it's verbose and I think it's likely that we will be able to get back to our original working directory without a problem.)

>Ainsi, maintenant nous avons :

So now we have:

```sh $ for i in *.zip; do j=$(basename "$i" ".zip"); mkdir "$j" && cd "$j" && unzip ../$i && cd -; done```

```bash $ for i in *.zip; do j="${i%.zip}"; mkdir "$j" && cd "$j" && unzip ../$i && cd -; done```

>Lançons depuis un répertoire enfant la commande `unzip` :

Let's throw the `unzip` command back in the mix:

```mkdir "$j" && cd "$j" && unzip ../$i && cd -```

>Eh bien, outre wordsplitting, il n'y a rien de terriblement mal avec cela. Pourtant, vous est-il venu à l'esprit qu'`unzip` peut déjà être en mesure de cibler un répertoire ? Cela n'est pas un standard pour la commande `unzip`, mais toutes les implémentations que j'ai vu peut le faire avec le drapeau -d. Donc, nous pouvons entièrement abandonner la commande `cd` :

Well, besides the wordsplitting, there's nothing terribly wrong with this. Still, did it occur to you that `unzip` might already be able to target a directory? There isn't a standard for the `unzip` command, but all the implementations I've seen can do it with the -d flag. So we can drop the cd commands entirely:

```$ mkdir "$j" && unzip -d "$j" "$i"```

```sh $ for i in *.zip; do j=$(basename "$i" ".zip"); mkdir "$j" && unzip -d "$j" "$i"; done```

```bash $ for i in *.zip; do j="${i%.zip}"; mkdir "$j" && unzip -d "$j" "$i"; done```

There! That's as good as it gets.
