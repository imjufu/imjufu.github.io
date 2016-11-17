---
layout: post
title: Gère tes tâches de fond avec Sidekiq
date: 2015-01-18
---

[Sidekiq](http://sidekiq.org/) est un gestionnaire de tâches de fond écrit en Ruby. Il est parfois très intéressant de gérer des jobs en parallèle et de façon asynchrone.  
Exemples : envoi de mailings, stockage de fichier en différé…

![Sidekiq](images/sidekiq/logo.png)

## Pourquoi ne pas choisir Resque ? Il est plus populaire !

Certes.  
Toutefois, contrairement à Sidekiq qui s'appuie sur du multithreading pour traiter les jobs, Resque fork un nouveau processus à chaque traitement. Pour effectuer le même travail, Resque a besoin de 10Gb de RAM alors que Sidekiq en utilise seulement 300Mb. Sidekiq ne consomme pas plus de CPU pour autant.

## Ok, c'est cool. Mais comment ça marche ?

Sidekiq se décompose en trois parties : le serveur, le client et le stockage.  
Le moteur de stockage utilisé est Redis. Une base clé-valeur simple mais très riche et performante.

### Regardons le client de plus près…

Le client tourne dans le même processus que notre application web (Rails). Il va nous permettre de pusher des tâches en fond.

Prenons un worker simple :

```ruby
# app/workers/hard_worker.rb
class HardWorker
  include Sidekiq::Worker

  def perform(user, count)
    puts 'Doing hard work'
  end
end
```

Si l'on souhaite programmer un job pour l'utilisateur `PKoin`, il suffit d'écrire :

```ruby
HardWorker.perform_async('PKoin', 5)
```

Notre worker va utiliser le client Sidekiq. Celui-ci va reçevoir un Hash qui représente le job à traiter.  
La ligne du dessus peut également être écrite de la façon suivante :

```ruby
# Lower-level generic API
Sidekiq::Client.push('class' => HardWorker, 'args' => ['PKoin', 5])
```

Lorsqu'on pousse un job en background, le client reçoit une représentation du job sous forme d'un Hash, le sérialise en Json et le push dans une queue dans Redis.

Étant donné que le client sérialise ses paramètres, ce dernier doit reçevoir des objets simples :

* numbers, strings, boolean, array, hash… <strong>OK!</strong>
* Date, Time, ActiveRecord instances… <strong>KO!</strong>

### Regardons maintenant le serveur de plus près…

Dans le but de gérer correctement le multithreading, Sidekiq s'appuie sur [Celluloid](https://github.com/celluloid/celluloid).

Quand on démarre le serveur Sidekiq, celui-ci crée un processus dans lequel il va créer plusieurs threads.  
Il va initialiser un Manager Actor (controller). Ce Manager va initialiser un Fetcher Actor (connector) et N Processor Actor (worker).  
Le N est configurable afin d'ajuster la concurrence souhaitée, autrement dit le nombre de jobs qui seront traités en parallèle.

En vrai, ce n'est pas très compliqué. Voici un petit schéma pour illustrer tout ça.

![Serveur Sidekiq](images/sidekiq/server.jpg)

Le Fetcher Actor permet de maintenir une connexion unique et active avec Redis. Celui-ci va puller, à intervalle de temps régulier (par défaut 15 secondes), les jobs en attente de traitement.  
Si il trouve un job en attente, il le récupère, le supprime de la base Redis et l'envoie au Manager Actor.

Le Manager Actor va dispatcher les différents jobs, reçus du Fetcher, aux Processor Actors libres.

Le Processor Actor reçoie un job en entrée, le traite et retourne une réponse positive ou négative au Manager. Dans le cas où le job échoue, il en profite pour mettre l'erreur dans des logs. Le Manager prendra alors la décision de renvoyer le job vers un Processor Actor (pas obligatoirement le même) afin de ré-essayer.

### Regardons ce que Sidekiq stocke dans Redis…

J'ai codé rapidement une petite application Rails pour faire des tests.  
Cette application comporte 3 éléments notables :

* 1 worker : `app/workers/hard_worker.rb`
* 1 tâche rake : `lib/tasks/workers.rake` pour nous aider à empiler des jobs
* 1 controller : `app/controllers/welcome_controller.rb` pour visualiser la base Redis

L'application est disponible sur github : [sidekiq-wat-is-da-shit](https://github.com/PKoin/sidekiq-wat-is-da-shit). Lisez le README pour le faire tourner en local.

Commençons par un test simple. Le serveur Sidekiq est éteint et on empile un job grâce à la commande :

```
$ bundle exec rake job:work_hard['PKoin',45]
# 55722c1724796339f0fd361b
```

Dans notre application Rails de test, via l'interface Redis, on y retrouve :

```
> COMMAND Redis: KEYS sidekiq-wat-da-shit*
sidekiq-wat-da-shit:queue:default : list
sidekiq-wat-da-shit:queues : set
```

Le client a créé deux clés.  
La première comporte la liste des jobs à dépiler dans la queue par défaut.  
La seconde est un ensemble des différentes queues disponibles.

Regardons maintenant ce que contient la queue par défaut.

```
> COMMAND Redis: LRANGE sidekiq-wat-da-shit:queue:default 0 -1
{"retry":1,"queue":"default","class":"HardWorker","args":["PKoin","45"],"jid":"55722c1724796339f0fd361b","enqueued_at":1423177060.093096}
```

Bonne nouvelle, on retrouve bien notre job.

Allumons maintenant le serveur afin de dépiler ce fameux job et voyons ce que contient Redis.

```
$ bundle exec sidekiq
# 2015-02-05T23:06:21.850Z 17500 TID-ovfc0w2jk HardWorker JID-55722c1724796339f0fd361b INFO: start
# 2015-02-05T23:06:21.850Z 17500 TID-ovfc0w2jk HardWorker JID-55722c1724796339f0fd361b DEBUG: PKoin: I'm doing hard work… ZzZ
```

Côté Redis,

```
> COMMAND Redis: KEYS sidekiq-wat-da-shit*
sidekiq-wat-da-shit:MacBook-Air-de-Julien.local:17500 : hash
sidekiq-wat-da-shit:MacBook-Air-de-Julien.local:17500:workers : hash
```

Deux nouvelles clés font leur apparition…  
Regardons ce qu'elles contiennent.

```
> COMMAND Redis: HGETALL sidekiq-wat-da-shit:MacBook-Air-de-Julien.local:17500
["beat", "1423177341.735065"]
["info", "{\"hostname\":\"MacBook-Air-de-Julien.local\",\"started_at\":1423177276.662422,\"pid\":17178,\"tag\":\"sidekiq-wat-is-da-shit\",\"concurrency\":3,\"queues\":[\"default\"],\"labels\":[]}"]
["busy", "1"]
```

Cette première clé contient un Hash contenant les détails du serveur Sidekiq.  
On peut également voir la variable `busy` qui est égale à 1. Ce qui est tout à fait normal étant donné que le serveur traite notre job.

```
> COMMAND Redis: HGETALL sidekiq-wat-da-shit:MacBook-Air-de-Julien.local:17500:workers
["ovfc0w2jk", "{\"queue\":\"default\",\"payload\":{\"retry\":1,\"queue\":\"default\",\"class\":\"HardWorker\",\"args\":[\"PKoin\",\"45\"],\"jid\":\"55722c1724796339f0fd361b\",\"enqueued_at\":1423177560.7913868},\"run_at\":1423177581}"]
```

Cette deuxième clé contient un Hash décrivant les workers en cours de traitement.  
On peut retrouver ici notre job, qu'on a ajouté plus haut, avec une variable supplémentaire : `run_at`.  
On peut constater également que ce job n'existe plus dans la clé `sidekiq-wat-da-shit:queue:default`.

```
# … 45 secondes plus tard, côté console …
2015-02-05T23:07:06.855Z 17500 TID-ovfc0w2jk HardWorker JID-55722c1724796339f0fd361b DEBUG: PKoin: IT'S OK. I'M NOT SLEEPING!
2015-02-05T23:07:06.856Z 17500 TID-ovfc0w2jk HardWorker JID-55722c1724796339f0fd361b INFO: done: 45.005 sec
```

Côté Redis :

```
> COMMAND Redis: GET sidekiq-wat-da-shit:stat:processed
1
```

Une nouvelle clé fait son apparition et contient un compteur des jobs terminés.

Que se passe-t-il maintenant lorsqu'un worker échoue ?

```
$ bundle exec rake job:work_hard['britney',45]
# 1379216acfecac655c5c753f
```

Côté Redis :

```
> COMMAND Redis: ZRANGE sidekiq-wat-da-shit:retry 0 -1
{"retry":1,"queue":"default","class":"HardWorker","args":["britney","45"],"jid":"1379216acfecac655c5c753f","enqueued_at":1423178069.9739678,"error_message":"Oops! I did it again.","error_class":"RuntimeError","failed_at":1423178069.980392,"retry_count":0}
```

Une nouvelle clé fait son apparition et contient les détails du job qui a échoué.  
On retrouve le job id, le message d'erreur ainsi que la date à laquelle il a échoué.  
Deux éléments importants :

* retry : nombre d'essai max pour notre job
* retry_count : nombre d'essai total

Il me parait également important d'expliquer comment Sidekiq calcule le temps à attendre entre deux essais.  
Entre chaque essai, Sidekiq temporise de manière exponentielle. En effet, il applique la formule suivante :

```
(retry_count ** 4) + 15 + (rand(30) * (retry_count + 1)) (i.e. 15, 16, 31, 96, 271, ... seconds + a random amount of time)
```

Sidekiq va donc mettre approximativement 21 jours pour faire 25 essais sur un même job. 25 étant le nombre d'essai max par défaut. Cela nous laisse suffisamment de temps pour corriger le bug et déployer son correctif en production !

Toutefois, que se passe-t-il si on atteint le nombre d'essai max ?

Sidekiq fait mourir le job.

Côté Redis :

```
> COMMAND Redis: ZRANGE sidekiq-wat-da-shit:dead 0 -1
{"retry":1,"queue":"default","class":"HardWorker","args":["britney","45"],"jid":"1379216acfecac655c5c753f","enqueued_at":1423178069.9739678,"error_message":"Oops! I did it again.","error_class":"RuntimeError","failed_at":1423178069.980392,"retry_count":1,"retried_at":1423178103.004996}
```

Une nouvelle clé fait son apparition et contient les détails du job mort.

Voilà globalement comment fonctionne Sidekiq.

### Quelques conseils…

* Il faut absolument passer des objects simples aux workers. Ne passez pas directement vos instances Active Record !
* Un worker doit être idempotent (peut être rejoué plusieurs fois) et transactionnel.
* Plusieurs workers doivent pouvoir tourner en parallèle.
* Utilisez un service d'erreur externe type Honeybadger, Airbrake, Rollbar, BugSnag, Sentry, Exceptiontrap, Raygun…
* Monitorez vos serveurs qui font tourner Sidekiq. Par exemple, [Inspeqtor](http://contribsys.com/inspeqtor/) est très simple et fait parfaitement le job.

### Pour finir, quelques avertissements…

* Ne faites pas des dizaines de files. Sidekiq n'aime pas ça !
* Il est impossible de traiter une queue en série.
* Sidekiq ne garantit pas l'ordre de traitement des jobs dans une queue.
* Sidekiq doit être redémarré quand le code source change.
* Sidekiq fait du nettoyage tous les 6 mois : dead jobs…
* Si Sidekiq part en segfault ou que la VM Ruby crash, tous les jobs qui étaient en cours d'exécution sont perdus !
* Attention de bien configurer le pool de connexion d'ActiveRecord… ainsi que le nombre de connexion max du serveur de DB.
