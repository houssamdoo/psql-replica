
# PostgreSQL Replication

## Streaming mode
Cette approche de la réplication est basée sur le déplacement des fichiers WAL de la base de données primaire vers la base de données cible.

### 01- Configuration du nœud primaire
#### STEP 01: Initialiser la base de données
Créez un nouvel utilisateur avec des privilèges de réplication en utilisant la commande suivante :
```bash
CREATE USER <example_username> REPLICATION LOGIN ENCRYPTED PASSWORD 'example_password';
```
Le mot clé de réplication est utilisé pour donner à l'utilisateur les privilèges requis.

#### STEP 02: Configurer les propriétés de Streaming
Ensuite, vous pouvez configurer les propriétés de streaming avec le fichier de configuration PostgreSQL (postgresql.conf) qui peut être modifié comme suit:
```
wal_level = logical
wal_log_hints = on
max_wal_senders = 8
max_wal_size = 1GB
hot_standby = on
```
Voici un petit aperçu des paramètres utilisés dans l'extrait précédent :

`wal_log_hints`: ce paramètre est requis pour la capacité pg_rewind qui est pratique lorsque le serveur de secours n'est pas synchronisé avec le serveur principal.

`wal_level`: vous pouvez utiliser ce paramètre pour activer la réplication en continu PostgreSQL, avec des valeurs possibles telles que minimal, replica ou logical.

`max_wal_size`: Ceci peut être utilisé pour spécifier la taille des fichiers WAL qui peuvent être conservés dans les fichiers journaux.

`hot_standby`: vous pouvez utiliser ce paramètre pour une connexion en lecture avec le secondaire lorsqu'il est défini sur ON.

`max_wal_senders`: vous pouvez utiliser max_wal_senders pour spécifier le nombre maximal de connexions simultanées pouvant être établies avec les serveurs de secours.

#### STEP 03 : Créer une nouvelle entrée
Après avoir modifié les paramètres dans le fichier `postgresql.conf`, une nouvelle entrée de réplication dans le fichier `pg_hba.conf` peut permettre aux serveurs d'établir une connexion entre eux pour la réplication.

Vous pouvez généralement trouver ce fichier dans le répertoire de données de PostgreSQL. Vous pouvez utiliser l'extrait de code suivant pour la même chose :

```bash
host replication <example_username> <IPaddress> md5
```
Une fois l'extrait de code exécuté, le serveur principal permet à un utilisateur appelé rep_user de se connecter et d'agir en tant que serveur de secours en utilisant l'adresse IP spécifiée pour la réplication.

### 01- Configuration du nœud secondiare
Pour configurer le nœud de secours pour la réplication en continu, procédez comme suit :
#### STEP 01: Back Up le nœud primaire
Pour configurer le nœud de secours, utilisez l'utilitaire `pg_basebackup` pour générer une sauvegarde du nœud principal. Cela servira de point de départ pour le nœud de secours. Vous pouvez utiliser cet utilitaire avec la syntaxe suivante :
```bash
pg_basebackp -D  -h  -X stream -c fast -U <example_username> -W
```

Les paramètres utilisés dans la syntaxe mentionnée ci-dessus sont les suivants :

- `-h`: Vous pouvez l'utiliser pour mentionner l'hôte principal.
- `-D`: Ce paramètre indique le répertoire sur lequel vous travaillez actuellement.
- `-C`: Vous pouvez l'utiliser pour définir les points de contrôle.
- `-X`: Ce paramètre peut être utilisé pour inclure les fichiers journaux transactionnels nécessaires.
- `-W`: Vous pouvez utiliser ce paramètre pour inviter l'utilisateur à entrer un mot de passe avant de se connecter à la base de données.

#### STEP 02: Configurer le fichier de configuration de la réplication
Ensuite, vous devez vérifier si le fichier de configuration de réplication existe. Si ce n'est pas le cas, vous pouvez générer le fichier de configuration de réplication en tant que recovery.conf.

Vous devez créer ce fichier dans le répertoire de données de l'installation de PostgreSQL. Vous pouvez le générer automatiquement en utilisant l'option -R dans l'utilitaire pg_basebackup.

Le fichier recovery.conf doit contenir les commandes suivantes :
```
standby_mode = 'on'

primary_conninfo = 'host=<master_host> port=<postgres_port> user=<replication_user> password=<password> application_name="host_name"'

recovery_target_timeline = 'latest'
```
Les paramètres utilisés dans les commandes susmentionnées sont les suivants :
- `primary_conninfo` : Vous pouvez l'utiliser pour établir une connexion entre les serveurs principal et secondaire en utilisant une chaîne de connexion.
- `standby_mode` : Ce paramètre peut entraîner le démarrage du serveur principal en tant que serveur de secours lorsqu'il est activé.
- `recovery_target_timeline`: Vous pouvez l'utiliser pour définir le temps de récupération.
Pour configurer une connexion, vous devez fournir le nom d'utilisateur, l'adresse IP et le mot de passe comme valeurs pour le paramètre primary_conninfo.

#### STEP 03 :: Redémarrer le serveur secondaire
Enfin, vous pouvez redémarrer le serveur secondaire pour terminer le processus de configuration.

Cependant, la réplication en continu s'accompagne de plusieurs défis, tels que :

- Divers clients PostgreSQL (écrits dans différents langages de programmation) conversent avec un seul point de terminaison. Lorsque le nœud principal échoue, ces clients continuent de réessayer avec le même nom DNS ou IP. Cela rend le basculement visible pour l'application.
- La réplication PostgreSQL n'est pas livrée avec un basculement et une surveillance intégrés. Lorsque le nœud principal tombe en panne, vous devez promouvoir un nœud secondaire en tant que nouveau nœud principal. Cette promotion doit être exécutée de manière à ce que les clients écrivent sur un seul nœud principal et qu'ils n'observent pas les incohérences des données.
- PostgreSQL réplique tout son état. Lorsque vous devez développer un nouveau nœud secondaire, le secondaire doit récapituler l'historique complet des changements d'état à partir du nœud principal, ce qui nécessite beaucoup de ressources et rend coûteux l'élimination des nœuds dans la tête et la création de nouveaux.
