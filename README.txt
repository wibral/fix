Les bibliothèques partagées sont du code compilé destiné à être partagé entre plusieurs différents programmes. Ils sont distribués en tant que fichiers .so dans /usr/lib/.

Une bibliothèque exporte des symboles qui sont les versions compilées de fonctions, de classes et de variables. Une bibliothèque possède ce qui s’appelle un SONAME comprenant un numéro de version. Cette version de SONAME ne correspond pas nécessairement au numéro de version publique. Un programme est compilé avec une version SONAME donnée de la bibliothèque. Si l’un des symboles est retiré ou modifié, alors le numéro de version doit être changé, ce qui oblige la recompilation de tous les paquets utilisant cette bibliothèque avec la nouvelle version. Les numéros de version sont généralement fixés par l’amont et nous les suivons dans nos noms de paquets binaires à l’aide d’un nombre ABI, mais parfois les amonts n’utilisent pas de numéros logiques de version et les empaqueteurs doivent conserver des numéros de version séparés.

Les bibliothèques sont généralement distribuées par l’amont sous forme de versions autonomes. Parfois, elles sont distribuées comme partie d’un programme. Dans ce cas, elles peuvent être incluses dans le paquet binaire avec le programme (c’est ce qu’on appelle le regroupement) lorsqu’il n’est pas prévu que d’autres programmes utilisent la bibliothèque. Le plus souvent, elles sont divisées en plusieurs paquets binaires séparés.

Les bibliothèques elles-mêmes sont mises dans un paquet binaire nommé libfoo1 où foo est le nom de la bibliothèque et 1 la version de SONAME. Les fichiers de développement issus du paquet, tels que les fichiers d’en-tête, nécessaires pour compiler des programmes avec la bibliothèque sont placés dans un paquet appelé libfoo-dev.
8.1. Un exemple

Nous allons utiliser libnova comme exemple :

$ bzr branch ubuntu:trusty/libnova
$ sudo apt-get install libnova-dev

Pour trouver le SONAME de la bibliothèque, lancez :

$ readelf -a /usr/lib/libnova-0.12.so.2 | grep SONAME

Le SONAME est libnova-0.12.so.2, qui correspond au nom du fichier (généralement c’est le cas, mais pas toujours). Ici, l’amont a mis le numéro de version comme partie du SONAME et lui a donné 2 comme version d’ABI. Les noms de paquets de bibliothèque devraient suivre le SONAME de la bibliothèque les contenant. Le paquet binaire de bibliothèque s’appelle libnova-0.12-2, où libnova-0.12 est le nom de la bibliothèque et 2 est notre ABI.

Si l’amont apporte des modifications incompatibles avec leur bibliothèque, il devra revoir la version de SONAME et nous devrons renommer notre bibliothèque. Tous les autres paquets utilisant notre paquet de bibliothèque devront être recompilés à la nouvelle version, c’est ce qu’on appelle une transition et peut demander un certain travail. Heureusement, notre nombre ABI continuera à correspondre au SONAME des amonts, mais parfois ils introduisent des incompatibilités sans changer leurs numéros de version et nous devrons changer les nôtres.

En regardant dans debian/libnova-0.12-2.install, nous voyons qu’il comprend deux fichiers :

usr/lib/libnova-0.12.so.2
usr/lib/libnova-0.12.so.2.0.0

Le dernier est la bibliothèque réelle, se terminant par un numéro de version mineur et un point. Le premier est un lien symbolique pointant vers la bibliothèque réelle. Le lien symbolique est ce que les programmes utilisant la bibliothèque rechercheront, les programmes en cours d’exécution ne se souciant pas du numéro de version mineur.

Libnova-dev.install inclut tous les fichiers nécessaires pour compiler un programme utilisant cette bibliothèque. Les fichiers d’en-tête, un binaire de configuration, le fichier .la d’utilitaires bibliothèque et libnova.so qui est un autre lien symbolique pointant vers la bibliothèque, les programmes compilant avec cette bibliothèque ne se souciant pas du numéro de version majeur (même si le binaire qu’ils compilent le feront).

.la libtool files are needed on some non-Linux systems with poor library support but usually cause more problems than they solve on Debian systems. It is a current Debian goal to remove .la files and we should help with this.
8.2. Les bibliothèques statiques

Les paquets -dev délivrent aussi usr/lib/libnova.a. Il s’agit d’une bibliothèque statique, une alternative à la bibliothèque partagée. Tout programme compilé avec la bibliothèque statique comprendra le répertoire du code en lui-même. Cela contourne le souci de la compatibilité binaire de la bibliothèque. Mais cela signifie aussi que tous les bogues, y compris les problèmes de sécurité, ne seront pas mis à jour avec la bibliothèque jusqu’à ce que le programme soit recompilé. Pour cette raison, l’usage de programmes utilisant des bibliothèques statiques est déconseillé.
8.3. Les fichiers de symboles

Quand un paquet est construit à partir d’une bibliothèque, le mécanisme shlibs ajoute une dépendance de paquet sur cette bibliothèque. C’est pourquoi de nombreux programmes auront Depends: ${shlibs:Depends} dans debian/control. Ceci est remplacé par les dépendances de bibliothèque à la compilation. Cependant shlibs peut seulement le faire dépendre du numéro de version majeure ABI, 2 dans notre exemple libnova. Si de nouveaux symboles sont ajoutés à libnova 2.1, un programme utilisant ces symboles pourrait toujours être installé avec libnova ABI 2.0, ce qui provoquerait son plantage.

Pour rendre les dépendances de bibliothèques plus précises, nous gardons les fichiers .symbols qui répertorient tous les symboles d’une bibliothèque et la version à laquelle ils sont apparus.

libnova n’a pas de fichier de symboles, nous pouvons donc en créer un. Commencez par la compilation du paquet :

$ bzr builddeb -- -nc

Le -nc terminera la compilation sans supprimer les fichiers de construction. Modifiez le fichier compilé et exécutez dpkg-gensymbols pour le paquet de bibliothèque :

$ cd ../build-area/libnova-0.12.2/
$ dpkg-gensymbols -plibnova-0.12-2 > symbols.diff

Cela crée un fichier diff que vous pouvez rendre automatiquement applicable :

$ patch -p0 < symbols.diff

Ce qui va créer un fichier nommé de manière similaire à dpkg-gensymbolsnY_WWI listant tous les symboles. Il répertorie également la version actuelle du paquet. Nous pouvons supprimer la version d’empaquetage de celles indiquées dans le fichier de symboles car de nouveaux symboles ne sont généralement pas ajoutés par de nouvelles versions d’empaquetage, mais par les développeurs de l’amont :

$ sed -i s,-0ubuntu2,, dpkg-gensymbolsnY_WWI

Maintenant, déplacez le fichier à sa place, soumettez et faites un test de compilation :

$ mv dpkg-gensymbolsnY_WWI ../../libnova/debian/libnova-0.12-2.symbols
$ cd ../../libnova
$ bzr add debian/libnova-0.12-2.symbols
$ bzr commit -m "add symbols file"
$ bzr builddeb

Si la compilation s’achève avec succès, le fichier de symboles est correct. Avec la prochaine version amont de libnova, vous devrez exécuter dpkg-gensymbols à nouveau et cela vous donnera un fichier diff pour mettre à jour le fichier de symboles.
8.4. Les fichiers C++ de la bibliothèque de symboles

C++ has even more exacting standards of binary compatibility than C. The Debian Qt/KDE Team maintain some scripts to handle this, see their Working with symbols files page for how to use them.
8.5. Lectures complémentaires

Junichi Uekawa’s Debian Library Packaging Guide goes into this topic in more detail.
