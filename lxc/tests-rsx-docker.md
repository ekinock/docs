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


> **Conclusions intermédiaires :** \
> Dans certains cas l'**ordre alphabétique** a l'air de primer sur l'ordre du docker-compose, dans d'autres c'est l'inverse \
> Les réseaux `internal=true` ne sont **jamais route par défaut** \
> Lorsque `external=false`, les **interfaces** semblent être attribuées en suivant l'**ordre alphabétique** \
> Seul l'ordre dans l'**attribut de niveau 1** "networks" a l'air d'avoir un impact, pas celui dans `services > <nom_service> > networks` \
> L'attribution des interfaces lorsque sont mélangés des réseaux internal et non-internal semble aléatoire \
> Les alias n'ont pas d'effet


### Deuxième phase

**Tests :**
- t9 et t10 : impact de l'attribut **`external=[true;false]`** par rapport à **l'ordre** dans le docker-compose  
- t11 à t13 : impact des **alias**  
- t14 et t15 : impact de l'**ordre** sur l'attribution des interfaces dans un mix `internal=true` et `internal=false`  
- t16 à t18 : impact de l'**ordre dans le fichier** par rapport à l'ordre **alphabétique**, comparaison selon forme de liste ou non

> **Conclusions intermédiaires :** \
> L'**absence d'effet des alias** se confirme \
> Le fait que l'**ordre** dans le docker-compose et l'ordre **alphabétique** aient un **impact** se confirme, bien que les règles restent obscures \
> Il est impossible d'identifier les règles d'attribution des interfaces \
> L'impact du paramètre `internal=true/false` sur l'attribution de la route par défaut est **confirmé** \
> **Un paramètre négligé jusqu'à présent apparaît comme significatif** : la forme de liste (avec des "-") ou non \
> -> Plus de tests doivent être réalisés sur ce point

### Troisième phase

**Tests t19 à t22 :**
- t19 et t21 : ordre fichier = ordre alphabétique  
- t20 et t22 : ordre fichier ≠ alphabétique  
- t19 et t20 : réseau `internal` = dernier dans le fichier  
- t21 et t22 : réseau `internal` ≠ dernier dans le fichier

> **Conclusions intermédiaires :** \
> Si un seul réseau est `internal=true`, c'est à lui qu'est **attribuée une interface en dernier** (ethX avec le X le plus élevé) \
> La **première adresse réseau** est attribuée au **dernier réseau dans le fichier** docker-compose \
> Si ce réseau est `internal=false`, il devient la **route par défaut** \
> Sinon, la route par défaut est attribuée selon des modalités non explicites

## Traitement des données

Le jeu de données obtenu avec les 22 tests est adapté à un **traitement qualitatif** *(trop restreint pour un traitement quantitatif)*. \
À ce stade, on peut déjà affirmer que **la route par défaut est forcément un réseau ayant le paramètre `internal=false`**. \
Mis à part cela, ***aucune tendance majoritaire ne semble se confirmer*** ; les données obtenues sont toujours inexploitables en l'état.

### Analyse statistique
#### Mise en forme des données
J'ai étudié 3 aspects : 
- les critères d'attribution de **eth0**
- les critères d'attribution de la **route par défaut**
- les critères d'attribution de la **première adresse réseau**

Pour chacun j'ai mesuré : 
- l'**ordre alphabétique** du nom du réseau
- l'**ordre sur le fichier** docker-compose.yml
- la valeur de l'**attribut "external"** *(booléen)*
- si les réseaux sont **organisés sous forme de liste** (avec des "-") dans le docker-compose *(booléen)*

Ainsi que les 2 autres aspects non étudiés (par exemple, j'ai relevé le numéro de l'interface et de l'adresse réseau pour chaque test lorsque j'étudiais l'attribution de la route par défaut).

J'ai utilisé 2 modes de mesure : 
- une **échelle de 1 à 3**, pour identifier l'ordre
    1. premier
    2. n'importe lequel au milieu
    3. dernier
- des **booléens**

#### Influence de l'attribut external
Les attributs `internal` et `external` ne sont étudiés que dans 6 tests. Pour tous les autres, ils prennent la valeur `false` par défaut. \
En termes statistiques, il est difficile de différencier les occurrences où les réseaux qui ressortent ont un attribut `external = false` ou `internal = false` parce que l'attribut n'a pas été pris en compte et a conservé sa valeur par défaut de ceux où ce résultat a du sens lorsque l'on traite le jeu de données en entier. \
C'est pourquoi je vais traiter ces cas en amont et les ignorer par la suite.

➡️ *à noter que l'on a déjà des conclusions solides sur l'impact de l'attribut `internal = true`, seul l'impact de l'attribut `external = true` reste à étudier.*

##### Méthodologie
Pour évaluer l'impact de `external = true`, j'ai d'abord mesuré la proportion de tests qui incluaient au moins un réseau ayant cette valeur d'attribut. \
J'ai ensuite fait, pour chaque aspect étudié, le rapport entre le nombre de fois où un réseau `external = true` ressortait et le nombre de fois où cet attribut était testé. 

➡️ *à noter que le **nombre de tests** qui étudient cet attribut est **particulièrement petit**.*

> **Conclusions intermédiaires :** \
> Seuls 6 tests prennent en compte cet attribut \
> Sur cet échantillon, la **première adresse** attribuée correspond **toujours** à un réseau `external = true` \
> Une tendance similaire semble se dessiner pour les autres aspects, mais pas aussi nettement \
> Il semble que l'attribut `external = true` ait une influence sur l'**ordre dans lequel le daemon traite les réseaux** plus que sur la manière dont il les hiérarchise

#### Variabilité des résultats

##### Périmètre
- Les attributs `external` et `internal` ne sont **pas pris en compte** ici.
- **Tous les autres attributs** sont pris en compte.

##### Méthodologie
Pour mesurer **l'hétérogénéité** des résultats, j'ai calculé **[l'indice d'équitabilité de Simpson](https://www.bonobosworld.org/fr/glossaire/indice-d-equitabilite-de-simpson)** de chaque critère (nom, ordre, …) pour chaque aspect traité (attribution de la première interface, adresse réseau ou de la route par défaut).

Dans ce contexte, cet indice s'interprète de la manière suivante : 
- 1 -> **homogénéité** totale
- valeurs échelonnées *(3 catégories)* :
  - ∈ [0,65;1[ -> **une catégorie domine** largement
  - ∈ [0.4;0.65[ -> catégories **équilibrées** (légère domination d'une catégorie)
  - ∈ [0.3;0.4[ -> très **hétérogène** *(maximum théorique)*
- booléens :
  - ∈ [0,75;1[ -> **une catégorie domine** largement
  - ∈ [0.55;0.75[ -> catégories **équilibrées** (légère domination d'une catégorie)
  - ∈ [0.48;0.55[ -> très **hétérogène** *(maximum théorique)* 

> **Conclusions intermédiaires :** \
> la **route par défaut est le 1er par ordre alphabétique** dans la *majorité* des cas \
> les autres critères sont **très hétérogènes** *(indice < 55)*

#### Analyse proportionnée globale

##### Méthodologie
Pour chacun des 3 aspects, j'ai mesuré la proportion de réseaux :
- selon leur **positionnement dans le fichier docker-compose.yml**
- selon leur **ordre alphabétique**
- pour lesquels `external = true`
- pour lesquels `internal = true`
- pour lesquels 2 ou plus aspects sont vérifiés

➡️ *à noter que les résultats ont été séparés en fonction de si les réseaux sont **renseignés sous forme de liste** (avec des "-") dans le fichier docker-compose ou non* 

##### Résultats
> **Légende :** \
> 🟣 significatif & indépendant de liste \
> 🔴 significatif & dépendant de liste \
> 🟤 notable mais difficile à interpréter 

|                 | eth0    | eth0   | Default route | Default route | 1ère addr rsx | 1ère addr rsx |
| --------------- | ------- | ------ | ------------- | ------------- | ------------- | ------------- |
| liste ordonnée  | list=0  | list=1 | list=0        | list=1        | list=0        | list=1        |
| external = true | 🔴 100% | 60%    | 🟤 0%         | 60%           | 🟣 100%       | 🟣 100%       |
| internal = true | 🔴 100% | 🟤 0%  | 🟣 0%         | 🟣 0%         | 🟤 0%         | 50%           |
| ordre dc = 1    | 10%     | 🟤 67% | 🟤 60%        | 44%           | 20%           | 33%           |
| ordre dc = 2    | 20%     | 11%    | 10%           | 33%           | 🟤 50%        | 0%            |
| ordre dc = 3    | 🟤 70%  | 22%    | 30%           | 22%           | 30%           | 🟤 67%        |
| ordre alpha = 1 | 40%     | 22%    | 🟣 90%        | 🟣 89%        | 10%           | 🟤 67%        |
| ordre alpha = 2 | 10%     | 11%    | 10%           | 11%           | 🟤 70%        | 0%            |
| ordre alpha = 3 | 50%     | 🟤 67% | 0%            | 0%            | 20%           | 33%           |
| eth0            | —       | —      | 30%           | 33%           | 20%           | 33%           |
| default         | 30%     | 33%    | —             | —             | 10%           | 56%           |
| 1er addr rsx    | 20%     | 33%    | 10%           | 56%           | —             | —             |

> **Hypothèse nulle :** \
> booléen : **50%** \
> valeurs échelonnées : **33%**

### Traitement par IA

##### Méthodologie
Les résultats bruts ainsi que les résultats traités ont été formatés en **CSV**, puis soumis à l'analyse de 2 IA de **type LLM** (*ChatGPT* et *Perplexity*). \
Le prompt initial contenait également l'explication du *cadre* des tests, de l'*infrastructure* de test ainsi que des *hypothèses* et *modalités de traitement* des données. \
Il a été demandé d'**identifier des patterns**, soit des attributs évoluant ensemble de manière notable, soit des **règles** qui avaient l'air d'être vraies dans la majorité des cas.

➡️ *à noter que **plusieurs prompts** successifs ont été utilisés afin d'**affiner** l'analyse et de **corriger les incompréhensions**.*

> **Conclusions intermédiaires :** \
> À chaque fois que les IA ont proposé, avec beaucoup d'assurance, un pool de règles, **au moins une était toujours visiblement fausse**, et ce malgré les correctifs apportés par la suite. Les deux IA s'accordent sur des tendances qui ont l'air majoritaires et sont mises en difficulté aux mêmes endroits. \
> Au vu des ressources disponibles sur le sujet, il apparaît que ces difficultés reflètent l'**absence de consensus** sur le sujet, que ce soit du côté des éditeurs, de celui des professionnels de l'IT ou de celui des amateurs éclairés.
>
> ChatGPT apporte cependant une remarque intéressante : les **briques fonctionnelles Docker** (compose, engine, ...etc) sont **développées en Go**, un *langage objet*. Il apparaît donc qu'**une partie des "aberrations" constatées pourrait être due au fonctionnement même du langage**, qui réordonne ses entrées pour optimiser leur traitement.

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

