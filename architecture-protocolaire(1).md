# Fiche de protocole – IRC CanaDuck

Complétez cette fiche pour décrire comment les clients interagissent avec le serveur IRC actuel.

## Format général

- Chaque commande est envoyée par le client sous forme d’une **ligne texte** terminée par `\n`.

## Commandes supportées

| **Commande**  | **Syntaxe exacte**        | **Effet attendu**              | **Réponse du serveur**                       | **Responsable côté serveur**  |
|---------------|---------------------------|--------------------------------|----------------------------------------------|-------------------------------|
| `/nick`       | `/nick <pseudo>`          | Attribue un pseudo unique      | Message de bienvenue ou erreur               | `set_pseudo()`                |
| `/join`       | `/join <nom du canal>`    | Rejoins le canal spécifié      | Message de confirmation ou erreur            | `rejoindre_canal()`           |
| `/msg`        | `/msg <message>`          | Envoie le message sur un canal | Message envoyé (dans canal) ou erreur        | `envoyer_message()`           |
| `/read`       | `/read`                   | Rien (que du direct)           | Explication non support                      | `lire_messages()`             |
| `/log`        | `/log`                    | Ecrit tous les fichiers de log | Messages de log ou erreur                    | `lire_logs()`                 |
| `/alert`      | `/alert <message alerte>` | Envoie d'un message d'alerte   | Message envoyé (tous utilisateurs) ou erreur | `envoyer_alerte()`            |
| `/quit`       | `/quit`                   | Déconnexion du serveur         | Message d'adieu                              |  Gérée dans boucle principale |

## Exemples d’interactions

### Exemple 1 : choix du pseudo

```
Client > /nick ginette
Serveur > Bienvenue, ginette !
```

### Exemple 2 :
```

Client > /join general
Serveur > Canal general rejoint.

```
...

# Structure interne – Qui fait quoi ?

| Élément                      | Rôle dans l’architecture                            |
|------------------------------|-----------------------------------------------------|
| `IRCHandler.handle()`        | Lit et traite les lignes de commande                |
| `etat_serveur`               | Stocke l'état des utilisateurs et des canaux        |
| `log()`                      | Enregistre les événements dans un fichier           |
| `broadcast_system_message()` | Difffuse un message système à tous les utilisateurs |
| `changer_etat()`             | Charge l'état depuis le fichier JSON                |
| `sauvegarder_etat()`         | Sauvegarde l'état dans un fichier JSON              |

# Points de défaillance potentiels

> Complétez cette section à partir de votre lecture du code.

| **Zone fragile**                 | **Cause possible**             | **Conséquence attendue**         | **Présence de gestion d’erreur ?**  |
|----------------------------------|--------------------------------|----------------------------------|-------------------------------------|
| `wfile.write(...)`               | Connexion interrompue          | Message non délivré              | Oui, try et except                  |
| Modification d’`etat_serveur`    | Accès concurrent               | Incohérence des données          | Oui, verrou threading.Lock()        |
| Lecture du fichier log           | Fichier inexistant ou corrompu | Erreur de lecture                | Oui, try et except                  |
| Pseudo déjà pris (`/nick`)       | Pseudo dupliqué                | Message d'erreur                 | Oui, vérification explicite         |
| Utilisateur sans canal courant   | Commande /msg sans canal       | Message d'erreur                 | Oui, vérification explicite         |
|                                  | Role insuffisant               | Message d'erreur                 | Oui, vérification des roles         |
