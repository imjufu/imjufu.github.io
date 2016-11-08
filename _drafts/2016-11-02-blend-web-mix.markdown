---
layout: post
title: Blend Web Mix 2016
date: 2016-11-02
---

# Duik

https://rainboxprod.coop/fr/outils/duik/

La boîte à outils complète d’animation et de setup pour After Effects !

Démarre en 2009 avec 1000 lignes de code
En 2015, 15k lignes de code

Codé en js

822 téléchargements (~6Go) / jour
Utilisé au niveau mondial

320k téléchargements
10k ouvertures quotidiennes

Logiciel libre (mais pas gratuit)
À la base, il n'est pas codeur, il s'est formé sur le tas

Pourquoi libre ?

Vendre un logiciel, c'est compliqué :
- Rédiger une licence
- Créer une boutique en ligne
- Lutter contre le piratage
- Réparer les bugs
- Aider les utilisateurs

Il est sous GPLv3 :
- Liberté d'utiliser le logiciel à n'importe quelle fin
- Modifier le programme pour l'adapter à ses besoins
- Redistribuer des copies à ses amis
- Partager avec les autres ses modifications

Choisir une licence libre, c'est avant tout des valeurs, une vision du monde.
C'est une manière de remercier ceux qui m'ont inspirés.

Tout cela prend du temps et coûte de l'argent :
- Développement
- Support (forum, emails)
- Documentation
- Tutoriels
- Frais technique (hébergement web)

Comment financer tout ça ?

Quel est le coût du logiciel ?

2 visions : définir la valeur par le travail ou par ce que l'utilisateur est prêt à payer.

Actuellement, il faut 15k € au minimum pour maintenir le logiciel.
Plusieurs moyens de financement :
- La pub : pas éthique
- "Choisissez votre prix" : incompatible avec le logiciel libre
- Location : pas libre

Il reste le crowdfunding et la vente de produits dérivés.

En 2015, 7175 € (258 contributeurs). Ce qui est faible en nombre de contributeurs.
Vente de produits dérivés. 2016, 1000 €.

Vivre du logiciel libre, ça fonctionne mais il ne faut pas vouloir gagner des millions d'euros.
Mais en terme d'ego, c'est très gratifiant. Il est contacté par Nickelodeon, Dreamworks.

Il existe des plateformes comme patreon.com pour financer le développement.

Idée de projet :

Ce serait top d'avoir une plateforme de crowdfunding uniquement dédiée au logiciel libre.
On fincance sans contrepartie et une fois le seuil atteint, le développeur commence à coder.
Pendant cette phase, les utilisateurs peuvent participer aux beta tests.
Une fois terminé, distibution en licence libre.

Voir opencollective.com


# De startup à scale up

Les leçons d'Evernote

Plusieurs étapes dans la création d'une startup :
- Formation : phase de création
- Validation : phase de rentabilité
- Growth : phase de recrutement et de développement

Il est Product Manager. Il est entre la techno, le design et le business.
Il a travaillé chez Evernote et maintenant chez SirupeLab.

5 minutes, temps pour convaincre de revenir

3 leçons :
- Collectez vos data : pour faire de l'apprentissage, prendre des décisions stratégiques
- Experimentez : le produit est le laboratoire, testez sur des échantillons d'utilisateurs
- Restez simple : se concentrer sur ce qui fonctionne et jeter le reste


# Designer pour le sub/conscient


L'attention se focalise plus facilement sur des éléments en mouvement.
Plus on est stressé et plus le locus de l'attention rétrécit.
La mémoire n'apparait que si il y a attention.

Si on veut obtenir quelque chose de nos utilisateurs il faut attirer l'attention dessus.
Exemple du formulaire et du message d'erreur en haut d'écran.

Le subconscient est la zone où les habitudes se développent.
Exemple : la marche, conduire


Le bouton back android ne marche pas car il ne respecte pas les attentes utilisateurs

La fenetre modal ne fonctionne pas car c'est un automatisme que notre cerveau développe et du coup, on clique sans vraiment le vouloir.
Solution : pas de modal de confirmation mais un bouton annuler

Le multi-tache est un mythe.
Si je marche en écoutant de la musique et en mangeant un sandwich et en réfléchissant à un problème mathématique. En vrai, seul le problème de maths capte notre attention. Le reste est en mode automatique.

État de flow : buts clairs à chaque étape, une retour immédiat à chaque action, l'activité devient une fin en soi
On devrait tendre vers ça.

Technologie calme, elle ne sur-sollicite pas mon attention.
On devrait tendre vers ça.

Conférence inspirée de "The Human Interface"


# Progressive Web Apps

Les apps natives ont plus de fonctionnalités et sont plus solides que les sites web mais touchent moins d'utilisateurs que les sites web mobiles.

PWA = site web mobile enrichi

Pourquoi progressif ?

Car j'ai une logique progressive de l'application. Je vais délivrer la meilleure expérience possible à mon utilisateur final.

Safari sur iOS ne sait pas gérer correctement les PWA pourtant le site est opérationnel.
Chrome sur Android sait gérer le offline, les notifications... et du coup propose tout ça aux utilisateurs finaux.

The Washington Post -> pionier des PWA : 80ms de temps de chargement d'un article
Flipkart : accès au contenu Hors-ligne et dans les zones de mauvaise connexion réseau

Quelles sont les limites ?

- HTTPS obligatoire
- PWA pas disponible sur iPhone mais cela fonctionne quand même



# strlen("pizza") != 1

ASCII publié en 1963, 7 bits par caractère, 127 possibilités pour encoder du texte
Avant ASCII, c'est le chaos !

Autrement dit, uniquement les caractères américains
Mais très problématique pour le reste du monde

Ajout d'1 bit par la suite pour gérer plus de caractères

Puis internet arriva. Deviner l'encodage d'un document, c'est pas top et hacker ASCII a ses limites.

L'age d'or du Mojibake.
Problème d'encodage, c'est compliqué

Unicode publié en 1991
"Un encodage pour les gouverner tous"

Unicode 1.0 stocké sur 16bits, 1 caractère = 2 octets
Adopté par JS, C, JAva
Mais pas suffisament de caractères !


UTF et Unicode 2, publié en 1996
On ne parle plus de caractères mais de code point
Unicode = la liste des caractères
UTF = l'encodage

3 différents UTF
UTF8 : le plus connus et utilisé en europe
UTF16 : compatible unicode 1
UTF32 : pas utilisé

Les emoji sotn des caractères unicode et ont été ajouté en 2010

JS est resté sur le standard en 16bits (UTF16)


Tips:
- Ne pas limiter les caractères valides
- Attention à la normalisation
- Attention aux homoglyphes (github s'est fait attaquer comme ça)
- Attention au charset UTF8 dans mysql, il faut utiliser utf8mb4 !



# Ne mourrez pas idiots, faites des jeux vidéos

Nous sommes le côté sérieux de l'informatique.
Le bingo : phygital, scale-up, design thinking, venture capital...

Les préjugés sur le monde du jeu vidéo :
* Truc de graphistes
* Truc de programmeurs
* Complexe & inaccessible
* La créativité

Il faut se lancer même si les premiers jeux qu'on fait sont pourris ! C'est normal et pas grave !

La productivité, c'est :
* Efficacité
* Force de proposition
* Innovation

Dans nos travails respectifs, on rentre dans une routine car on est bon dans ce qu'on fait et personne ne va nous demander de faire autre chose.

La spécialisation est mauvaise. On s'enferme nous même dans un monde de routine.
Et en plus c'est dangereux, un langage ça disparait...

La meilleure manière d'avancer, d'apprendre c'est de se vautrer !

Le but n'est pas de tout plaquer pour devenir game developer mais bien de faire du jeu vidéo pour s'amuser et s'améliorer.

https://itch.io/

Quelques vidéos intéressantes :
* Indie Game : The movie
* Minecraft : The story of Mojang
* Double Fine Adventure (sur Youtube)


# Design Fiction : anticiper les usages, prévoir l’expérience utilisateur





# À voir plus tard

* En janvier 2017, HTTPS obligatoire pour surfer sur chrome
* https://speakerdeck.com/romain/headers-http-et-securite-blindez-votre-appli-web

