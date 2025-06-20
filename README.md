# architecture-micro-service-tp3

Travail collaboratif (Romain Courbaize & Jodie Monterde)

## CONTEXTE
2.1 Cohérence concurrente et synchronisation
— Quels types de problèmes de concurrence peuvent apparaître dans ce système multi-clients ?
On peut se retrouver dans une situation de problème de 'Race Condition'. C'est-à-dire que deux opérations arrivent en même temps : on ne sait pas laquelle traiter en priorité.

On peut également avoir des problèmes si le serveur fait une opération en même temps qu'un client : le client envoie un message en même temps que le serveur charge son état. Le message envoyé par le client n'est donc pas traité/mal traité, ou le chargement se fait mal, ou un ensemble des différentes options...

On peut également se retrouver avec des incohérences d'états : les opérations peuvent se mélanger, se corrompre, les données peuvent perdre leurs sens (dans les deux sens du terme)

— Que peut-il arriver si deux clients rejoignent ou quittent un canal en même temps ?
Si les connexions ne sont pas protégés (verrous), on peut avoir des problèmes de gestion de la déconnexion/connexion, qui peuvent ne pas aboutir ou engendrer des problèmes.

2.2 Modularité et séparation des responsabilités

Quelles sont les grandes responsabilités fonctionnelles de votre application serveur (gestion client, traitement commande, envoi message, logs, persistance…) ?

On peut imaginer avoir :

    la gestion des connexions clients
    l'analyse et exécution des commandes
    la diffusion des messages

Peut-on tracer une frontière claire entre logique métier et logique d’entrée/sortie réseau ?

On peut distinguer la logique métier (traitement des commandes, gestion des canaux) de la logique réseau (écoute socket, send/recv).
En cas d’erreur dans une commande, quelle couche doit réagir ?

La couche métier (l'analyse et exécution des commandes) doit signaler l’erreur. Et la couche réseau peut se charger de transmettre le message d’erreur au client.
2.3 Scalabilité et capacité à évoluer
Si vous deviez ajouter une nouvelle commande (ex : /topic, /invite,/ban), quelle partie du système est concernée ?

Pour ajouter des commandes, il faut modifier l'analyse et l'exécution des commandes.
Que faudrait-il pour que ce serveur fonctionne à grande échelle (plusieurs centaines de clients) ?

Il faudrait distribuer l'application sur plusieurs serveurs.
Quelles limitations structurelles du code actuel empêchent une montée en charge ?

Il n'y a pas d'espace partagé et de lock.
2.4 Portabilité de l’architecture
Ce serveur TCP pourrait-il être adapté en serveur HTTP ? Quelles parties seraient conservées, quelles parties changeraient ?

On garde la partie métier et on change toute la partie réseau.
Dans une perspective micro-services, quels modules seraient candidats naturels pour devenir des services indépendants ?

Il y aurait :

    Authentification
    Gestion des canaux
    Messages & gestion commande

Est-il envisageable de découpler la gestion des utilisateurs de celle des canaux ? Comment ?

On peut découpler l'ensemble en utilisant des APIs qui feront office de services.
2.5 Fiabilité, tolérance aux erreurs, robustesse
Le serveur sait-il détecter une déconnexion brutale d’un client ? Peut-il s’en remettre ?

Il faut qu'il try les exceptions des sockets et donc faire le nécessaire dans ce cas.
Si un message ne peut pas être livré à un client (socket cassée), le système le détecte-t-il ?

La socket lève une erreur.
Peut-on garantir une livraison ou au moins une trace fiable de ce qui a été tenté/envoyé ?

On peut imaginer d'avoir un historique de tous les messages envoyés.

2.6 Protocole : structuration et évolutivité
Quelles sont les règles implicites du protocole que vous utilisez ? Une ligne = une commande, avec un préfixe (/msg, /join, etc.) et éventuellement des arguments : est-ce un protocole explicite, documenté, formalisé ?

## LIMITES

Synthèse sur les limites de l’architecture actuelle du serveur IRC
1. Ce que ce serveur fait bien

Le serveur présente plusieurs qualités qui en font un bon point de départ pédagogique ou expérimental :

   Simplicité d’implémentation : le traitement des commandes est direct, basé sur du texte, ce qui facilite la lecture et la modification rapide du code.

   Réactivité : les messages sont traités de manière synchrone, sans mise en file ou latence importante, tant que le nombre de clients reste faible.

   Modèle unique : tout est centralisé, ce qui permet une compréhension globale de l’état du système et de son fonctionnement sans architecture distribuée complexe.

2. Ce que le serveur cache ou gère mal

Le passage à l’échelle, à l’interopérabilité ou à la fiabilité révèle plusieurs faiblesses :
Couplage fort et non modularité

    Les fonctions manipulent à la fois le réseau, l’état métier et la journalisation, ce qui rend difficile la réutilisation ou le remplacement de composants (comme le protocole).

    Le protocole est implémenté de manière implicite (pas de grammaire formelle, pas de séparation commande/effet), rendant l’adaptation à un autre protocole (WebSocket, HTTP) coûteuse.

Erreurs mal gérées

    Aucune structuration des erreurs (pas de types d’erreur explicites, ni de code ou format unifié).

    Des erreurs peuvent passer inaperçues (ex : client non connecté qui envoie /msg).

    L’état du système peut devenir incohérent si une commande échoue à mi-exécution.

Testabilité insuffisante

    Impossible d’écrire des tests unitaires sans simuler une socket.

    De nombreux comportements sont implicites et non testés (disparition de canaux, déconnexion de clients fantômes…).

Aucune tolérance aux pannes

    Le système ne supporte pas les pannes partielles (échec d’écriture, perte de socket, etc.).

    Une défaillance d’un élément peut bloquer l’ensemble de l’application.

Manque de scalabilité

    L’état est uniquement en mémoire, non répliqué.

    Une montée en charge (ex : 1 000 clients) rend le serveur rapidement inopérant.

    Aucune file d’attente ni boucle événementielle n’est mise en œuvre.

3. Ce qu’il faudrait refactorer pour évoluer vers un système web ou micro-service

Plusieurs pistes de refonte sont nécessaires pour moderniser l’architecture :
1. Séparation claire des responsabilités

    Créer des modules indépendants pour : le protocole, les commandes métier, la persistance, les logs.

    Introduire un moteur de commande avec une API interne propre.

2. Refactorisation du protocole

    Définir une grammaire explicite du protocole.

    Encapsuler les messages dans un format structuré (JSON) avec métadonnées (type, version, timestamp).

    Ajouter un champ de version pour permettre l’évolution.

3. Ajout de robustesse et testabilité

    Découpler la logique métier des sockets pour écrire des tests unitaires simples.

    Simuler des cas d’erreur pour valider la résilience.

    Implémenter des exceptions contrôlées, des vérifications d’état, et un comportement par défaut en cas d’échec.

4. Préparation à la scalabilité

    Externaliser l’état vers une base de données ou une file de messages.

    Implémenter une architecture orientée messages (pub/sub) ou événements.

    Gérer la montée en charge avec une boucle asyncio, des workers, ou des brokers (ex : RabbitMQ, Kafka).

5. Transition vers une architecture micro-services

    Identifier des services autonomes : gestion des utilisateurs, des canaux, des messages.

    Créer des APIs REST ou RPC internes.

    Conteneuriser l’application avec Docker, ajouter des fichiers de configuration externes.

    Ajouter du monitoring (Prometheus, Grafana), des logs structurés, et des tests de charge.

Conclusion
Le serveur actuel est un bon démonstrateur pédagogique. Cependant, il souffre d’un couplage fort, d’un protocole implicite, et d’une faible résilience. En refactorant le code vers une architecture modulaire, en structurant le protocole, et en introduisant une couche de services, ce projet pourrait devenir la base d’un système robuste, scalable et devops-compatible, orienté micro-services.
