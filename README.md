# DLL-Hijack-Windows
## Mise en œuvre d'un DLL Hijack et exposition de celui-ci à l'aide de Sysmon et de EventViewer.
|     | Description |
|-----|-------------|
| 1-  | Installation de Sysmon | 
| 2-  | Exemple d'un fonctionnement normal du programme calc.exe avec EventViewer | 
| 3-  | Telechargement et Execution du Reflective Dll |
| 4-  | Detection du DLL Hijacking avec EventViewer |

## 1- Installation de Sysmon

A) Pour commencer, vous pouvez installer Sysmon en le téléchargeant depuis la documentation officielle de Microsoft (https://docs.microsoft.com/fr-fr/sysinternals/downloads/sysmon). Une fois téléchargé, ouvrez une invite de commandes en tant qu'administrateur et exécutez la commande suivante pour installer Sysmon :

<code>sysmon64 -i -accepteula -h md5, sha256,imphash -l -n</code>

B) Dans notre cas d'utilisation spécifique, nous visons à détecter une attaque de détournement de DLL. Les identifiants de journal d'événements Sysmon pertinents pour les détournements de DLL peuvent être trouvés dans la documentation de Sysmon (https://docs.microsoft.com/en-us/sysinternals/downloads/sysmon). Pour détecter un détournement de DLL, nous devons nous concentrer sur l'événement de type 7, qui correspond aux événements de chargement de module. Pour cela, nous devons modifier le fichier de configuration Sysmon sysmonconfig-export.xml que nous avons téléchargé depuis https://github.com/SwiftOnSecurity/sysmon-config.

Lorsque le fichier est ouvert, naviguez jusqu'à l'événement de type 7 dans le fichier et procédez en remplaçant "include" par "exclude".

![](https://imgur.com/uvYBagS.png)

N'oubliez pas d'éxecuter ce qui suit pour utiliser la configuration Sysmon mise à jour:

<code>sysmon.exe -c sysmonconfig-export.xml</code>

## 2- Exemple d'un fonctionnement normal du programme calc.exe avec EventViewer

A) Veuillez exécuter le programme "calc.exe" depuis votre terminal. Le programme devrait s'ouvrir normalement.

B) Avec la configuration Sysmon modifiée, nous pouvons commencer à observer les événements de chargement d'image. Pour afficher ces événements, accédez à l'Observateur d'événements et sélectionnez "Applications et services" -> "Microsoft" -> "Windows" -> "Sysmon"

Pour filtrer uniquement les événements de chargement d'image de fichiers, accédez à **"Filtrer Current Log"** et saisissez 7 à la place de **\<All Event IDs>**.

![](https://imgur.com/zgF3765.png)

Ensuite, nous recherchons des instances de "calc.exe" en cliquant sur "Find..." afin d'identifier le chargement de DLL associé à notre application.

![](https://imgur.com/88nhRGT.png)

En cliquant avec le bouton droit sur l'événement et en accédant aux propriétés de l'événement, nous pouvons voir le chargement authentifié de "wininet.dll" par "calc.exe".

![](https://imgur.com/hirE8Gx.png)

## 3- Telechargement et Execution du Reflective Dll

A) Nous allons effectuer le Hijack en utilsant "calc.exe" et "WININET.dll". Pour simplifier nous allons utiliser le reflective DLL créé par Stephen Fewer appellé "hello world" que vous pouvez trouver ici : https://github.com/stephenfewer/ReflectiveDLLInjection/tree/master/bin

B) Une fois téléchargé, renommez **"reflective_dll.x64.dll"** par **"WININET.dll"** en utilisant la commande suivante:

<code>rename reflective_dll.x64.dll WININET.dll</code>

![](https://imgur.com/kL3OxoW.png)

C) Nous devons maintenant déplacer calc.exe depuis C:\Windows\System32 ainsi que WININET.dll vers un répertoire inscriptible (tel que le dossier Bureau(Desktop)).

![](https://imgur.com/5nBxZRU.png)

Si vous ne pouvez pas déplacer calc.exe, il faudra modifier les autorisations du répertoire System32. Pour ce faire, naviguez jusqu'au dossier System32, faites un clic droit dessus et sélectionnez "Propriétés". Dans les propriétés, cliquez sur "Sécurité" où vous verrez une option avancée pour effectuer les modifications nécessaires.

![](https://imgur.com/efMAMsM.png)

D) Maintentant lors de l'execution de calc.exe:

![](https://imgur.com/ao5hROO.png)

## 4- Detection du DLL Hijacking avec EventViewer

Recherchons des instances de "calc.exe" en cliquant sur "Find..." afin d'identifier le chargement de DLL associé à notre hijack.

![](https://imgur.com/zWmAn0x.png)
![](https://imgur.com/2rXgrus.png)

Les résultats de Sysmon fournissent des informations précieuses. Maintenant, nous pouvons observer plusieurs indicateurs de compromission (IOCs) pour créer des règles de détection efficaces:

1) "calc.exe", initialement situé dans System32, ne devrait pas être trouvé dans un répertoire inscriptible. Par conséquent, une copie de "calc.exe" dans un répertoire inscriptible constitue un IOC, car il devrait toujours résider dans System32 ou éventuellement Syswow64.

2) "WININET.dll", initialement situé dans System32, ne doit pas être chargé en dehors de System32 par calc.exe. Si des chargements de "WININET.dll" se produisent en dehors de System32 avec "calc.exe" en tant que processus parent, cela indique une détournement de DLL dans calc.exe. Bien que la prudence soit nécessaire lors de l'alerte sur toutes les instances de chargement de "WININET.dll" en dehors de System32 (car certaines applications peuvent empaqueter des versions spécifiques de DLL pour la stabilité), dans le cas de "calc.exe", nous pouvons affirmer avec confiance un détournement en raison du nom immuable de la DLL, que les attaquants ne peuvent pas modifier pour échapper à la détection

3) La DLL d'origine "WININET.dll" est signée par Microsoft, tandis que notre DLL injectée reste non signée.
