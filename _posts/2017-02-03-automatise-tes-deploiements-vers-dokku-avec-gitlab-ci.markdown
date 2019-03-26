---
layout: post
title: Automatise tes déploiements vers Dokku avec GitLab CI
date: 2017-02-03
---

J'ai longtemps déployé mon code hébergé sur [Github](https://www.github.com/) vers [Heroku](https://www.heroku.com/). Et d'ailleurs, je continue de le faire.  
Je suis totalement tombé amoureux des [pipelines Heroku](https://devcenter.heroku.com/articles/pipelines) afin de gérer plusieurs environnements. En dehors des traditionnels environnements de staging et de production, on a également aussi les [review apps](https://devcenter.heroku.com/articles/github-integration-review-apps). En deux mots, cela te permet d'avoir un environnement de testing dédié à chaque pull request sur Github. Le truc parfait quand tu veux faire tester une nouvelle feature au product owner sans avoir à pourrir l'environnement de staging.  
Tout ça c'est sympa et très bien intégré si tu es sur Github et Heroku. Toutefois, j'ai quelques projets qui sont sur [Gitlab](https://www.gitlab.com/). Et Heroku, c'est bien mais des fois, j'ai besoin d'auto-héberger des projets.

Histoire de m'amuser un peu, j'ai essayé de reproduire toute la magie autour de cette intégration parfaite entre Github et Heroku.  

![GitLab CI et Dokku pour tout automatiser](images/automatise-tes-deploiements-vers-dokku-avec-gitlab-ci/gitlab-ci-dokku.png)

## Au fait, pourquoi automatiser tes déploiements ?

En deux mots :

### Productivité

Je préfère largement prendre un peu de temps, au début du projet, pour écrire des scripts que tout le monde pourra utiliser et maintenir, plutôt que d'écrire une documentation, qui ne sera jamais à jour, expliquant comment mettre en production.  
En plus de ça, je vais pouvoir ré-utiliser mes scripts.  
« Combien de déploiements sont effectués chaque année, sur combien de serveurs et combien de systèmes » permet de se faire une idée de la charge de travail.

### Fiabilité

Automatiser permet de réduire les erreurs liées aux interventions humaines. Cela permet également de traçer les différentes opération.  
Si le déploiement est bien fait, on peut également limiter les impacts négatifs sur la production, l’arrêt d’une application métier pendant une journée ouvrable par exemple.  
L’utilisation d’outils permet de coupler plus facilement les tâches de déploiement et des tâches de mises à jour des environnements. Ces dernières sont souvent disjointes quand elles sont effectuées manuellement. 

## Dokku, le Heroku auto-hébergé

[Dokku](http://dokku.viewdocs.io/dokku/) te permet de mettre en place un [PaaS](https://en.wikipedia.org/wiki/Platform_as_a_service) très rapidement. Et en plus, ça tombe bien, il s'appuie sur [Docker](https://imjufu.github.io/docker).

Pour faire tourner Dokku, je l'ai installé sur un Ubuntu serveur 16.04, le tout hébergé sur un serveur dédié [Online.net](https://www.online.net/) à ~10 € par mois.  
En une ligne de commande et quelques configurations, Dokku est opérationnel.

### Quelques commandes utiles pour la suite

```bash
dokku apps # Lister les applications existantes
dokku apps:create <app> # Créer une application
dokku apps:destroy <app> # Détruire une application

# Installer un plugin
#   Quelques exemples de plugins depuis https://github.com/dokku :
#     dokku-letsencrypt : Mettre en place automatiquement des certificats Let's Encrypt TLS
#     dokku-postgres : Plugin postgres
#     dokku-mysql : Plugin mysql
dokku plugin:install <git-url> 

dokku config <app> # Récupérer les variables d'environnement de l'application
dokku config:set <app> KEY1=VALUE1 # Définir une variable d'environnement pour l'application
```

## GitLab CI pour tester, builder et déployer vers Dokku

GitLab CI est une réponse de Gitlab aux différents outils d'intégration continue qui gravitent et sont pleinement intégrés avec Github.

### Pipelines

On retrouve la notion de pipeline proposée par Heroku.  
Celle-ci va nous permettre de définir différentes actions réparties sur différentes étapes avant la mise en production de l'application. On peut imaginer avoir les étapes suivantes :

* Test
  * Exécuter les tests unitaires
  * Exécuter un linter
* Staging
  * Déployer une nouvelle version du code
  * Migrer la base de données
  * Avertir (par mail, slack...) les personnes du projet de la mise en staging
* Production (débloquer par une validation manuelle)
  * Déployer une nouvelle version du code
  * Migrer la base de données
  * Avertir (par mail, slack...) les personnes du projet de la mise en production

Si l'étape test échoue, alors le pipeline s'arrête. Sinon, on passe à l'étape staging puis, en cliquant sur un bouton sur l'interface de Gitlab, on passe à l'étape production.

Et tout cela se configure via un simple fichier `gitlab-ci.yml` à la racine du projet.  
Voici un premier exemple pour lancer les tests automatiquement à chaque intégration sur `master` ou à chaque merge request :

```yml
image: ruby:2.3.3

stages:
  - test

rspec:
  stage: test
  services:
    - mysql:5.7
  variables:
    MYSQL_RANDOM_ROOT_PASSWORD: 'yes'
    MYSQL_DATABASE: hello_world_test
    MYSQL_USER: hello_world_test
    MYSQL_PASSWORD: hello_world_test
  script:
    - bundle install
    - cp config/database.gitlab-ci.yml config/database.yml
    - bundle exec rspec

rubocop:
  stage: test
  script:
    - bundle install
    - bundle exec rubocop
```

### Automatiser la mise en staging

**Première étape**, la création de l'application Dokku :

```bash
dokku apps:create hello-world
```

Mon projet `hello-world` a besoin d'une base de données MySQL :

```bash
# Installation du plugin mysql
dokku plugin:install https://github.com/dokku/dokku-mysql.git
# Création de la base de données
dokku mysql:create hello-world-database
# Création du lien de la base de données avec mon application
dokku mysql:link hello-world-database hello-world
```

**Deuxième étape**, autoriser GitLab à communiquer avec mon serveur Dokku.  
Sur GitLab, il suffit de définir une variable `SSH_PRIVATE_KEY` qui contient la valeur d'une clé privée créée uniquement pour cet usage.  

**Troisième étape**, compléter le fichier `gitlab-ci.yml` :

```yml
image: ruby:2.3.3

stages:
  - test
  - staging

rspec:
  stage: test
  services:
    - mysql:5.7
  variables:
    MYSQL_RANDOM_ROOT_PASSWORD: 'yes'
    MYSQL_DATABASE: hello_world_test
    MYSQL_USER: hello_world_test
    MYSQL_PASSWORD: hello_world_test
  script:
    - bundle install
    - cp config/database.gitlab-ci.yml config/database.yml
    - bundle exec rspec

rubocop:
  stage: test
  script:
    - bundle install
    - bundle exec rubocop

deploy_staging:
  stage: staging
  before_script:
    - mkdir -p ~/.ssh
    - echo "$SSH_PRIVATE_KEY" | tr -d '\r' > ~/.ssh/id_rsa
    - chmod 600 ~/.ssh/id_rsa
    - ssh-keyscan -H 'mon-serveur.com' >> ~/.ssh/known_hosts
  script:
    - git push dokku@mon-serveur.com:hello-world master
  environment:
    name: staging
    url: http://hello-world.mon-serveur.com
  only:
    - master
```

À partir de là, à chaque intégration de code dans `master`, le code va être testé, validé puis déployé automatiquement en staging.

**Quatrième étape**, gérer les actions post-déploiement. Par exemple, la migration de la base de données. Rien de plus simple, il suffit de créer un fichier de configuration `app.yml` à la racine du projet pour expliquer à Dokku les actions à effectuer suite au déploiement. Par exemple :

```yml
{
  "scripts": {
    "dokku": {
      "postdeploy": "bundle exec rails db:migrate"
    }
  },
  "addons": [
    "dokku-mysql"
  ]
}
```

## Bonus, les review apps

Avant l'intégration d'une merge request dans `master`, je vois deux points à valider :

* Le code est-il de bonne qualité ? Cela sera déterminé suite à la relecture de la branche par un membre de l'équipe technique.
* La fonctionnalité répond-t-elle au besoin ? Cela sera déterminé par le product owner.

Toutefois, le product owner ne va pas s'amuser à regarder la code pour valider ça. Il a besoin d'un environnement pour tester la fonctionnalité. C'est là qu'interviennent les review apps.

Imagine une merge request nommée `ajout-du-prenom` qui permet de rajouter le prénom de l'utilisateur sur l'interface de l'application `hello-world`.  
Le product owner pourrait se rendre sur http://ajout-du-prenom.mon-serveur.com pour tester et valider la fonctionnalité.

### Comment faire ça ?

Très simple, il suffit d'ajouter une étape dans notre fichier de configuration `gitlab-ci.yml` afin de lui expliquer quoi faire lors de merge request.

```yml
image: ruby:2.3.3

stages:
  - test
  - review
  - staging

rspec:
  stage: test
  services:
    - mysql:5.7
  variables:
    MYSQL_RANDOM_ROOT_PASSWORD: 'yes'
    MYSQL_DATABASE: hello_world_test
    MYSQL_USER: hello_world_test
    MYSQL_PASSWORD: hello_world_test
  script:
    - bundle install
    - cp config/database.gitlab-ci.yml config/database.yml
    - bundle exec rspec

rubocop:
  stage: test
  script:
    - bundle install
    - bundle exec rubocop

start_review:
  stage: review
  before_script:
    - mkdir -p ~/.ssh
    - echo "$SSH_PRIVATE_KEY" | tr -d '\r' > ~/.ssh/id_rsa
    - chmod 600 ~/.ssh/id_rsa
    - ssh-keyscan -H 'mon-serveur.com' >> ~/.ssh/known_hosts
  script:
    - ssh dokku@mon-serveur.com apps:create $CI_BUILD_REF_SLUG
    - ssh dokku@mon-serveur.com config:set $CI_BUILD_REF_SLUG MYSQL_DATABASE_SCHEME=mysql2
    - ssh dokku@mon-serveur.com mysql:create $CI_BUILD_REF_SLUG-database
    - ssh dokku@mon-serveur.com mysql:link $CI_BUILD_REF_SLUG-database $CI_BUILD_REF_SLUG
    - git push dokku@mon-serveur.com:$CI_BUILD_REF_SLUG HEAD:refs/heads/master
  environment:
    name: review/$CI_BUILD_REF_NAME
    url: http://$CI_BUILD_REF_SLUG.mon-serveur.com
    on_stop: stop_review
  only:
    - branches
  except:
    - master

stop_review:
  stage: review
  before_script:
    - mkdir -p ~/.ssh
    - echo "$SSH_PRIVATE_KEY" | tr -d '\r' > ~/.ssh/id_rsa
    - chmod 600 ~/.ssh/id_rsa
    - ssh-keyscan -H 'mon-serveur.com' >> ~/.ssh/known_hosts
  script:
    - ssh dokku@mon-serveur.com apps:destroy $CI_BUILD_REF_SLUG --force
    - ssh dokku@mon-serveur.com mysql:destroy $CI_BUILD_REF_SLUG-database --force
  variables:
    GIT_STRATEGY: none
  when: manual
  environment:
    name: review/$CI_BUILD_REF_NAME
    action: stop
  only:
    - branches
  except:
    - master

deploy_staging:
  stage: staging
  before_script:
    - mkdir -p ~/.ssh
    - echo "$SSH_PRIVATE_KEY" | tr -d '\r' > ~/.ssh/id_rsa
    - chmod 600 ~/.ssh/id_rsa
    - ssh-keyscan -H 'mon-serveur.com' >> ~/.ssh/known_hosts
  script:
    - git push dokku@mon-serveur.com:hello-world master
  environment:
    name: staging
    url: http://hello-world.mon-serveur.com
  only:
    - master
```

À chaque merge request, GitLab va automatiquement :

* Créer une nouvelle application
* Créer et faire le lien d'une nouvelle base de données à l'application
* Déployer le code de la branche sur cette application
* Migrer la base de données

Une fois la merge request intégrée dans master, GitLab va automatiquement :

* Supprimer l'application
* Supprimer la base de données

Quelques points à noter :

* `$CI_BUILD_REF_SLUG` est une variable disponible automatiquement dans le runner GitLab. Il s'agit du nom de la branche en minuscule, de maximum 63 octets, et comportant uniquement des caractères valides pour une url.
* `ssh dokku@mon-serveur.com` permet de piloter Dokku à travers ssh. On a accès à toutes les commandes Dokku disponibles directement depuis la ligne de commande mais en ssh.
