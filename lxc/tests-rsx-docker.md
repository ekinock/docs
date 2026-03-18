# Priorisation dans l'attribution des interfaces réseaux

## Avant-propos
## Stratégie de test

> **Windows 11 Entreprise**  
> - Version: 10.0.22631  
> - PSVersion: 5.1.22621.6345  
> **WSL**  
> - Version: 2.4.13.0  
> - Version noyau: 5.15.167.4-1  

> **Docker Desktop**  
> - Version: 4.58.0  
> - Engine: 29.1.5  
> - Compose: v5.0.1  
> **VLAN bureautique standard**  

## Environnement
Les tests sont réalisés depuis une **station bureautique**, avec un **compte utilisateur** classique ayant été ajouté au préalable au **groupe local docker-users**.  

Les commandes sont lancées **sans élévation de droits**.  

## Image utilisée
Comme je n'ai pas trouvé d'image officielle ou soutenue par Docker adaptée à mes tests, j'en build une qui est légère et ne contient que les commandes dont j'ai besoin.  

```dockerfile
FROM debian:12-slim

LABEL description="Tests - Priorisation attribution interfaces réseaux conteneurs"

RUN apt-get update && apt-get install -y --no-install-recommends \
    iproute2 \
    iputils-ping \
    curl \
    dnsutils \
    net-tools \
 && rm -rf /var/lib/apt/lists/*

CMD ["tail", "-f", "/dev/null"]
```
## Tests

### Première phase


**Tests :**
- t1 : impact des **noms** & de l'**ordre** dans le docker-compose.yml  
- t2 : impact des **alias**  
- t3 et t4 : impact de l'attribut **`internal=[true;false]`**  
- t5 : impact de l'attribut **`internal=[true;false]`** par rapport au **tri alphabétique**  
- t6 : impact de l'attribut **`external=[true;false]`**  
- t7 et t8 : impact de l'**ordre** dans le docker-compose.yml (dans `networks` puis dans `services`)


> **Conclusions intermédiaires :**
>> Dans certains cas l'**ordre alphabétique** a l'air de primer sur l'ordre du docker-compose, dans d'autres c'est l'inverse
>> Les réseaux `internal=true` ne sont **jamais route par défaut**
>> Lorsque `external=false`, les **interfaces** semblent être attribuées en suivant l'**ordre alphabétique**
>> Seul l'ordre dans l'**attribut de niveau 1** "networks" a l'air d'avoir un impact, pas celui dans `services > <nom_service> > networks`
>> L'attribution des interfaces lorsque sont mélangés des réseaux internal et non-internal semble aléatoire
>> Les alias n'ont pas d'effet


### Deuxième phase

**Tests :**
- t9 et t10 : impact de l'attribut **`external=[true;false]`** par rapport à **l'ordre** dans le docker-compose  
- t11 à t13 : impact des **alias**  
- t14 et t15 : impact de l'**ordre** sur l'attribution des interfaces dans un mix `internal=true` et `internal=false`  
- t16 à t18 : impact de l'**ordre dans le fichier** par rapport à l'ordre **alphabétique**, comparaison selon forme de liste ou non

> **Conclusions intermédiaires :**
>> L'**absence d'effet des alias** se confirme
>> Le fait que l'**ordre** dans le docker-compose et l'ordre **alphabétique** aient un **impact** se confirme, bien que les règles restent obscures
>> Il est impossible d'identifier les règles d'attribution des interfaces
>> L'impact du paramètre `internal=true/false` sur l'attribution de la route par défaut est **confirmé**
>> **Un paramètre négligé jusqu'à présent apparaît comme significatif** : la forme de liste (avec des "-") ou non
>>> Plus de tests doivent être réalisés sur ce point

### Troisième phase

**Tests t19 à t22 :**
- t19 et t21 : ordre fichier = ordre alphabétique  
- t20 et t22 : ordre fichier ≠ alphabétique  
- t19 et t20 : réseau `internal` = dernier dans le fichier  
- t21 et t22 : réseau `internal` ≠ dernier dans le fichier

> **Conclusions intermédiaires :**
>> Si un seul réseau est `internal=true`, c'est à lui qu'est **attribuée une interface en dernier** (ethX avec le X le plus élevé)
>> La **première adresse réseau** est attribuée au **dernier réseau dans le fichier** docker-compose
>> Si ce réseau est `internal=false`, il devient la **route par défaut**
>> Sinon, la route par défaut est attribuée selon des modalités non explicites

## Traitement des données

- Jeu de données des 22 tests adapté à un **traitement qualitatif**
- La route par défaut est toujours un réseau `internal=false`
- Aucune tendance majoritaire ne se confirme, données inexploitables autrement

Le jeu de données obtenu avec les 22 tests est adapté à un **traitement qualitatif** *(trop restreint pour un traitement quantitatif)*. \
À ce stade, on peut déjà affirmer que **la route par défaut est forcément un réseau ayant le paramètre `internal=false`**. \
Mis à part cela, ***aucune tendance majoritaire ne semble se confirmer*** ; les données obtenues sont toujours inexploitables en l'état.

### Analyse statistique
- Études sur : **eth0**, **route par défaut**, **première adresse réseau**
- Mesures : ordre alphabétique, ordre sur fichier, attributs `external` et `internal`, forme de liste
- Modes : échelle 1 à 3 + booléens

### Influence de l'attribut external
- Seuls 6 tests prennent en compte l'attribut
- La **première adresse** attribuée correspond toujours à un réseau `external=true`
- L'attribut `external=true` influence l'ordre de traitement par le daemon, plus que la hiérarchie des interfaces

## Variabilité des résultats

### Périmètre
- `external` et `internal` non pris en compte ici  
- Tous les autres attributs sont considérés

### Méthodologie
- Calcul de **l'indice de Simpson** pour chaque critère (nom, ordre, …)  
- Interprétation :  
  - 1 -> homogénéité totale  
  - [0,65;1[ -> catégorie domine  
  - [0,4;0,65[ -> équilibré  
  - [0,3;0,4[ -> très hétérogène

### Analyse proportionnée globale
- Proportion de réseaux selon : position fichier, ordre alphabétique, `external=true`, `internal=true`  
- Observation selon forme de liste

### Résultats
**Légende :**
- significatif & indépendant de liste  
- significatif & dépendant de liste  
- notable mais difficile à interpréter  

**Hypothèse nulle :**
- booléen : 50%  
- valeurs échelonnées : 33%

## Traitement par IA
- Résultats bruts et traités soumis à IA (ChatGPT et Perplexity)
- Identification de **patterns** et **règles**  
- Plusieurs prompts successifs pour affiner et corriger les incompréhensions

### Conclusions
- Au moins une règle proposée était toujours fausse
- IA convergent sur tendances majoritaires mais difficultés aux mêmes endroits
- Difficultés reflètent l'absence de consensus côté éditeurs, pros IT et amateurs éclairés
- Docker développé en Go, langage objet  
- Certaines aberrations peuvent venir du fonctionnement du langage qui réordonne ses entrées

## Influence du langage GO sur les résultats

### Analyses

### Notes & références
- [Open Container Initiative](https://opencontainers.org/)  
- Traitement des données :  
  - [Indice de Simpson](https://fr.wikipedia.org/wiki/Indice_de_Simpson#:~:text=L'indice%20de%20Simpson%20est,d'individus%20class%C3%A9s%20en%20cat%C3%A9gories.)  
  - [Matrices de corrélations](https://pdfs.semanticscholar.org/4c44/d84171b8681b7153a020129f29dd1e78ede4.pdf)  
- Codes sources :  
  - [Docker Engine](https://github.com/moby/moby) : daemon Docker  
  - [CLI Docker](https://github.com/docker/cli) : parsing commandes, user CLI, appels API  
  - [Docker Compose](https://github.com/docker/compose) : parsing yaml, orchestration  
  - [compose-spec](https://github.com/compose-spec/compose-spec) : spécification docker-compose.yml  
  - [compose-go](https://github.com/compose-spec/compose-go) : lib pour Compose  
  - [libnetwork](https://github.com/moby/libnetwork) : gestion des réseaux Docker  
  - [containerd](https://github.com/containerd/containerd) : daemon conteneurs, runtime haut niveau  
  - [runc](https://github.com/opencontainers/runc) : daemon conteneurs, runtime bas niveau  
- Langage Go :  
  - [Documentation officielle Go](https://go.dev/doc/)  
  - [Get started with Go](https://go.dev/doc/tutorial/getting-started)  
  - [Slices](https://go.dev/blog/slices-intro) : traitement ordonné  
  - [Maps](https://go.dev/blog/maps) : traitement non ordonné  
  - [Go Memory Model](https://go.dev/ref/mem) : gestion données mises en cache

