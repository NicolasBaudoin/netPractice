# Guide réseau — comprendre et finir NetPractice vite

Ce guide n'est pas un cours de réseau complet : c'est le strict nécessaire pour
comprendre ce que le simulateur te demande, plus une méthode mécanique à appliquer
niveau après niveau. Tout NetPractice tient dans **trois questions** :

1. Ces deux interfaces sont-elles dans le **même sous-réseau** ?
2. Cette IP est-elle une **adresse d'hôte valide** ?
3. Si la destination n'est pas dans mon sous-réseau, **par quelle passerelle** je sors ?

Le reste, c'est du calcul binaire.

> Pendant la soutenance, **aucun outil externe n'est autorisé** (`bc` toléré). Les
> calculs doivent donc être faisables de tête. Ce guide t'apprend à les faire de tête.

---

## Sommaire

1. [Une IP, c'est 32 bits](#1-une-ip-cest-32-bits)
2. [Le masque de sous-réseau](#2-le-masque-de-sous-réseau)
3. [Adresse réseau, broadcast, plage utilisable](#3-adresse-réseau-broadcast-plage-utilisable)
4. [Calcul mental : la méthode du "bloc"](#4-calcul-mental--la-méthode-du-bloc)
5. [Switch vs routeur](#5-switch-vs-routeur)
6. [La passerelle par défaut](#6-la-passerelle-par-défaut)
7. [Les tables de routage](#7-les-tables-de-routage)
8. [Adresses privées et Internet](#8-adresses-privées-et-internet)
9. [Les règles exactes du simulateur](#9-les-règles-exactes-du-simulateur)
10. [Méthode niveau par niveau](#10-méthode-niveau-par-niveau)
11. [Lire les logs](#11-lire-les-logs)
12. [Checklist de soutenance](#12-checklist-de-soutenance)
13. [Tables à mémoriser](#13-tables-à-mémoriser)

---

## 1. Une IP, c'est 32 bits

`192.168.1.42` n'est pas quatre nombres : c'est **un** nombre de 32 bits, découpé en
4 octets pour être lisible.

```
192      . 168      . 1        . 42
11000000 . 10101000 . 00000001 . 00101010
```

Chaque octet vaut de 0 à 255. Les bits d'un octet valent, de gauche à droite :

```
128  64  32  16  8  4  2  1
```

Apprends cette ligne par cœur, tout le projet en dépend.

Conversion à faire de tête, dans les deux sens :

- `192` = 128 + 64 → `11000000`
- `10101010` = 128 + 32 + 8 + 2 = `170`

Une IP est composée de deux parties : un **préfixe réseau** (à gauche) et un
**identifiant d'hôte** (à droite). C'est le masque qui dit où est la frontière.

---

## 2. Le masque de sous-réseau

Le masque est lui aussi un nombre de 32 bits, mais avec une contrainte forte :
**des 1 à gauche, des 0 à droite, jamais de mélange**.

```
255.255.255.0   = 11111111.11111111.11111111.00000000   → 24 bits à 1 → /24
255.255.255.192 = 11111111.11111111.11111111.11000000   → 26 bits à 1 → /26
255.255.0.0     = 11111111.11111111.00000000.00000000   → 16 bits à 1 → /16
```

Les deux notations sont équivalentes : `255.255.255.0` et `/24` désignent le même
masque. NetPractice accepte les deux dans les champs.

**`255.255.255.240` est valide** (`11110000`). **`255.255.255.10` ne l'est pas**
(`00001010` : des 0 avant des 1) — le simulateur répond `invalid netmask`.

Les seules valeurs possibles pour l'octet "de transition" sont donc :

| Octet | Binaire | Bits à 1 ajoutés |
| --- | --- | --- |
| 0 | `00000000` | 0 |
| 128 | `10000000` | 1 |
| 192 | `11000000` | 2 |
| 224 | `11100000` | 3 |
| 240 | `11110000` | 4 |
| 248 | `11111000` | 5 |
| 252 | `11111100` | 6 |
| 254 | `11111110` | 7 |
| 255 | `11111111` | 8 |

Si tu vois un octet qui n'est pas dans cette liste dans un masque, c'est faux.

### Le test fondamental

**Deux interfaces sont dans le même sous-réseau si `IP AND masque` donne le même
résultat des deux côtés**, avec le même masque.

`AND` bit à bit : 1 si les deux bits valent 1, 0 sinon. En pratique, là où le masque
vaut 255 on recopie l'octet de l'IP, là où il vaut 0 on écrit 0.

```
A : 192.168.1.10  /24  →  réseau 192.168.1.0
B : 192.168.1.77  /24  →  réseau 192.168.1.0    ✅ même réseau
C : 192.168.2.10  /24  →  réseau 192.168.2.0    ❌ réseau différent
```

Attention au piège classique : **les deux côtés doivent avoir le même masque**. Si A
est en `/24` et B en `/25`, ils peuvent calculer des réseaux différents alors qu'ils
sont sur le même câble — le simulateur te le refusera.

---

## 3. Adresse réseau, broadcast, plage utilisable

Dans chaque sous-réseau, **deux adresses sont interdites pour un hôte** :

- l'**adresse réseau** : tous les bits d'hôte à **0** (ex. `192.168.1.0/24`) ;
- l'**adresse de broadcast** : tous les bits d'hôte à **1** (ex. `192.168.1.255/24`).

Tout ce qui est entre les deux est assignable.

```
192.168.1.0/24
  réseau     : 192.168.1.0       ❌ interdit
  premier    : 192.168.1.1       ✅
  ...
  dernier    : 192.168.1.254     ✅
  broadcast  : 192.168.1.255     ❌ interdit
  → 254 hôtes utilisables
```

Formule : un sous-réseau `/n` contient `2^(32-n)` adresses, donc
**`2^(32-n) - 2` hôtes utilisables**.

| CIDR | Masque | Adresses | Hôtes |
| --- | --- | --- | --- |
| /24 | 255.255.255.0 | 256 | 254 |
| /25 | 255.255.255.128 | 128 | 126 |
| /26 | 255.255.255.192 | 64 | 62 |
| /27 | 255.255.255.224 | 32 | 30 |
| /28 | 255.255.255.240 | 16 | 14 |
| /29 | 255.255.255.248 | 8 | 6 |
| /30 | 255.255.255.252 | 4 | 2 |

Le `/30` est le masque typique d'un lien point à point entre deux routeurs : deux
adresses utilisables, exactement ce qu'il faut.

> Cas particulier vu dans les niveaux avancés : le `/31`. Il n'a que 2 adresses, donc
> 0 hôte utilisable au sens classique. Dans NetPractice il apparaît surtout dans des
> **routes** (`x.x.x.0/31`), pas sur des interfaces — une route n'est pas soumise à
> la règle réseau/broadcast.

---

## 4. Calcul mental : la méthode du "bloc"

C'est la technique qui te fait gagner le plus de temps en soutenance.

Pour un masque, l'octet intéressant est le dernier qui n'est ni 255 ni 0. On appelle
**taille de bloc** la valeur `256 - octet_du_masque`.

```
/26 → 255.255.255.192 → bloc = 256 - 192 = 64
```

Les sous-réseaux commencent alors à des multiples de la taille de bloc :

```
/26 (bloc 64) sur 192.168.1.x :
  192.168.1.0    → 0   … 63     hôtes 1   … 62    broadcast 63
  192.168.1.64   → 64  … 127    hôtes 65  … 126   broadcast 127
  192.168.1.128  → 128 … 191    hôtes 129 … 190   broadcast 191
  192.168.1.192  → 192 … 255    hôtes 193 … 254   broadcast 255
```

**Procédure pour une IP quelconque**, par exemple `192.168.1.100/26` :

1. bloc = 256 − 192 = **64** ;
2. le plus grand multiple de 64 sous 100 = **64** → adresse réseau `192.168.1.64` ;
3. broadcast = 64 + 64 − 1 = **127** → `192.168.1.127` ;
4. plage utilisable = `.65` → `.126`.

Trois secondes, sans binaire. Exemple avec `/28` :

```
10.0.0.200/28 → bloc 16 → 200 / 16 = 12,5 → 12 × 16 = 192
  réseau 10.0.0.192, broadcast 10.0.0.207, hôtes .193 → .206
```

Si le masque coupe dans le **3ᵉ** octet (ex. `/20` → `255.255.240.0`), applique
exactement la même méthode sur ce troisième octet : bloc = 256 − 240 = 16, donc les
réseaux sont `x.x.0.0`, `x.x.16.0`, `x.x.32.0`… et le 4ᵉ octet va de 0 à 255 à
l'intérieur.

---

## 5. Switch vs routeur

C'est la distinction qui débloque les niveaux 3 et 4.

**Un switch (niveau 2 OSI)** ne comprend rien aux IP. Il n'a pas d'adresse, pas de
configuration. Il recopie ce qu'il reçoit sur tous ses ports. Conséquence pratique :

> Toutes les machines branchées sur un même switch doivent être dans le **même
> sous-réseau**, avec le **même masque**, et des IP **toutes différentes**.

Dans le simulateur, les logs le disent explicitement : `on switch S: pass to all
connections`.

**Un routeur (niveau 3 OSI)** relie des sous-réseaux **différents**. Chacune de ses
interfaces est un membre à part entière du sous-réseau auquel elle est branchée :

```
        LAN A                    LAN B
   192.168.1.0/24            10.0.0.0/24
        │                          │
   [R1 if1: 192.168.1.254]   [R1 if2: 10.0.0.254]
        └──────── même routeur ────┘
```

Deux règles qui découlent de ça :

- une interface de routeur a une IP **du sous-réseau qu'elle dessert** (une IP en
  `10.x` sur un câble `192.168.1.0/24` ne sert à rien) ;
- **deux interfaces du même routeur ne doivent jamais être dans le même
  sous-réseau** — le simulateur répond `multiple interface match` et échoue.

Par convention dans NetPractice, l'interface du routeur prend souvent la dernière
adresse utilisable (`.254`, `.62`, `.126`…). Ce n'est pas obligatoire, mais ça aide à
s'y retrouver.

---

## 6. La passerelle par défaut

Quand une machine veut envoyer un paquet, elle fait **une seule** chose :

```
destination AND mon_masque == mon_réseau ?
   oui → je l'envoie directement sur le câble
   non → je l'envoie à ma passerelle
```

D'où la règle la plus importante du projet :

> **La passerelle doit être une IP du sous-réseau de la machine elle-même.**

Sinon la machine ne peut pas la joindre : elle devrait passer par sa passerelle pour
joindre sa passerelle. Le simulateur log alors
`route match but no interface for gateway`.

```
Host A : 192.168.1.10/24
  gateway 192.168.1.254  ✅ (dans 192.168.1.0/24)
  gateway 192.168.2.254  ❌ injoignable
  gateway 192.168.1.10   ❌ c'est elle-même
```

Et la passerelle doit être **l'interface du routeur branchée du bon côté** — pas
celle de l'autre côté du routeur.

### Le piège du retour

Un ping, ce sont deux trajets. Beaucoup d'échecs viennent d'un chemin aller correct
et d'un retour impossible.

```
A → B : A a une route vers le réseau de B  ✅
B → A : B n'a aucune route vers le réseau de A  ❌  → l'objectif échoue
```

**Vérifie toujours les deux sens.** À chaque objectif « X doit joindre Y », pose-toi
la question depuis X *et* depuis Y, ainsi que sur chaque routeur traversé.

---

## 7. Les tables de routage

Une route, c'est : « pour atteindre **ce réseau**, passe par **cette passerelle** ».

```
route : 10.0.0.0/8       gateway : 192.168.1.254
route : 0.0.0.0/0        gateway : 192.168.1.254
```

`0.0.0.0/0` — aussi écrit `default` — est la **route par défaut** : masque de 0 bit,
donc elle correspond à *toutes* les destinations.

Trois choses à savoir sur le simulateur, différentes d'un vrai routeur :

1. **Les routes sont lues dans l'ordre du tableau, et seule la première qui
   correspond est utilisée.** Il n'y a pas de « plus long préfixe d'abord » comme sur
   un vrai routeur. Donc **la route par défaut doit être en dernier**, sinon elle
   avale tout.
2. La destination d'une route s'écrit en **CIDR** (`10.0.0.0/8`), pas en masque
   décimal.
3. La passerelle d'une route obéit à la même règle qu'une passerelle par défaut :
   elle doit être joignable directement, donc dans un sous-réseau d'une des
   interfaces de la machine.

### Comment remplir une table de routage

Pour chaque machine (hôte **ou** routeur), demande-toi : « quels réseaux ne sont pas
directement branchés sur moi, et par où j'y vais ? »

```
   LAN1                 R1                  R2                LAN2
192.168.1.0/24  ──  .1 | 10.0.0.1/30  ──  10.0.0.2/30 | .1  ──  192.168.2.0/24
```

- **R1** connaît directement `192.168.1.0/24` et `10.0.0.0/30`. Il lui manque
  `192.168.2.0/24` → route `192.168.2.0/24` via `10.0.0.2`.
- **R2** connaît directement `192.168.2.0/24` et `10.0.0.0/30`. Il lui manque
  `192.168.1.0/24` → route `192.168.1.0/24` via `10.0.0.1`.
- Les hôtes de LAN1 : route par défaut via `192.168.1.1` ; ceux de LAN2 via
  `192.168.2.1`.

Sur un hôte, une route par défaut suffit presque toujours. Sur un routeur qui a
plusieurs voisins, il faut une route par réseau distant (ou une route par défaut vers
le routeur « du haut » plus des routes spécifiques vers le bas).

---

## 8. Adresses privées et Internet

Trois plages sont réservées aux réseaux privés (RFC 1918) :

```
10.0.0.0/8        →  10.0.0.0    → 10.255.255.255
172.16.0.0/12     →  172.16.0.0  → 172.31.255.255
192.168.0.0/16    →  192.168.0.0 → 192.168.255.255
```

Dans NetPractice, le nœud « Internet » **refuse de router ces plages** :
`private subnets not routed over internet`. Donc si un objectif dit « X doit joindre
Somewhere on the Net », l'interface de X côté Internet doit porter une **adresse
publique**.

Autre particularité : l'Internet du simulateur **n'accepte pas de route par défaut**
(`invalid default route on internet`). Les routes que tu ajoutes côté Internet doivent
donc être des routes précises vers ton réseau public, jamais `0.0.0.0/0`.

---

## 9. Les règles exactes du simulateur

Ces règles viennent du moteur de simulation lui-même. Les connaître évite de perdre
du temps sur des erreurs invisibles.

**Une IP est refusée si :**

- le premier octet dépasse **223** (multicast/réservé) ;
- le premier octet vaut **127** (loopback) ;
- un octet est hors de 0–255 ou le format n'a pas 4 champs ;
- c'est l'**adresse réseau** ou l'**adresse de broadcast** de son propre masque ;
- elle est **identique** à celle d'une autre interface sur le trajet
  (`duplicate IP`).

**Un masque est refusé si** les 1 ne sont pas contigus en tête.

**Le routage échoue si :**

- deux interfaces de la même machine correspondent à la destination
  (`multiple interface match`) → deux interfaces dans le même sous-réseau ;
- la route correspond mais aucune interface ne peut joindre la passerelle
  (`route match but no interface for gateway`) ;
- aucune route ne correspond (`destination does not match any route`) ;
- le paquet arrive sur une interface dont l'IP n'est pas celle visée à ce saut
  (`packet not for me`) ;
- le paquet repasse par une machine déjà traversée (`loop detected`).

**Enfin** : seule la **première route correspondante** est essayée. Si elle échoue,
le simulateur n'essaie pas les suivantes.

---

## 10. Méthode niveau par niveau

### Réflexe général (à appliquer à chaque niveau)

1. **Lis les objectifs** en haut. Ils disent exactement quels couples doivent se
   joindre — rien d'autre n'a besoin de marcher.
2. **Repère les champs éditables** (non grisés). Ce sont tes seuls degrés de liberté ;
   tout le reste est une contrainte imposée.
3. **Pars des valeurs fixes.** Une IP non éditable impose le sous-réseau de tout son
   segment. Calcule ce sous-réseau (méthode du bloc) et déduis-en les valeurs
   possibles.
4. **Remplis segment par segment**, du plus contraint au plus libre.
5. **Check again**, et si ça échoue, **lis les logs** avant de retoucher quoi que ce
   soit.

### Niveaux 1–2 — deux machines sur un fil

Deux hôtes reliés directement. Il faut juste qu'ils soient dans le même sous-réseau
avec le même masque, et que chaque IP soit une adresse d'hôte valide.

Méthode : regarde l'IP fixe, applique son masque, prends n'importe quelle adresse
libre de la plage. Évite `.0` et `.255` (ou l'équivalent selon le masque), et ne
reprends pas l'IP du voisin.

### Niveau 3 — le switch

Trois machines sur un switch. Même sous-réseau pour les trois, même masque, trois IP
distinctes. Le switch lui-même n'a rien à configurer.

### Niveau 4 — une interface de routeur dans le LAN

Le routeur est ici juste un membre de plus du réseau local. Son interface doit avoir
une IP du même sous-réseau que les hôtes. Pas encore de routage à écrire.

### Niveau 5 — les premières routes

Apparition de la table de routage sur les hôtes. La passerelle est l'interface du
routeur côté hôte. Écris la destination en CIDR et vérifie que la passerelle est bien
dans le sous-réseau de l'hôte.

Attention : une destination mal orthographiée (`10..0.0.0/8`) est refusée — le champ
peut contenir un piège volontaire.

### Niveau 6 — sortir sur Internet

Objectif : « A doit joindre Somewhere on the Net ». Il faut :

- une route par défaut sur A (`0.0.0.0/0` ou `default`) ;
- une passerelle qui est l'interface du routeur côté A ;
- côté Internet, une route **précise** vers ton réseau public (jamais une route par
  défaut) ;
- des adresses publiques sur le segment qui part vers Internet.

### Niveau 7 — deux routeurs

Quatre tables à remplir : A, C, R1, R2. Applique le schéma de la section 7 : chaque
routeur a besoin d'une route vers le réseau qu'il ne touche pas, chaque hôte d'une
route par défaut vers son routeur local. Vérifie l'aller **et** le retour.

### Niveau 8 — subnetting

Ici les masques ne sont plus des `/24`. Tu vois des `/26`, `/27`… et des routes du
type `192.168.0.0/26`. Sors la méthode du bloc systématiquement : pour chaque
interface, calcule réseau / broadcast / plage avant d'écrire quoi que ce soit.

Piège classique : deux sous-réseaux qui se chevauchent. `192.168.0.0/26` et
`192.168.0.32/27` partagent des adresses — le simulateur renverra des erreurs
d'interfaces multiples.

### Niveaux 9–10 — tout ensemble

Six à sept objectifs, deux routeurs, un switch, Internet. Ne cherche pas à tout voir
d'un coup.

1. Dessine la topologie sur papier : un rectangle par sous-réseau, et dedans son
   réseau/masque calculé à partir des valeurs fixes.
2. Attribue les IP d'interfaces, segment par segment.
3. Remplis les tables de routage machine par machine, en te posant la question
   « quels réseaux ne sont pas chez moi ? ».
4. Mets les routes par défaut **en dernier** dans chaque table.
5. Traite les objectifs un par un, en traçant le chemin aller puis le chemin retour.

À ce stade, la majorité des erreurs sont : un retour manquant sur un routeur, une
route par défaut placée trop haut, ou une passerelle hors sous-réseau.

---

## 11. Lire les logs

Les logs en bas de page donnent le point exact de rupture. Table de traduction :

| Log | Cause | Correction |
| --- | --- | --- |
| `invalid IP address` | IP mal formée, réseau/broadcast, 127.x, 1er octet > 223 | Recalcule la plage utilisable |
| `invalid netmask` | 1 non contigus | Utilise une valeur de la table des masques |
| `duplicate IP` | Deux interfaces avec la même IP | Change l'une des deux |
| `packet not for me` | Le paquet arrive sur une interface dont l'IP ≠ cible du saut | Passerelle ou IP d'interface incorrecte |
| `destination does not match any interface` | Normal : on passe à la table de routage | Informatif |
| `destination does not match any route` | Aucune route ne couvre la destination | Ajoute une route ou une route par défaut |
| `route match but no interface for gateway` | La passerelle n'est pas dans un sous-réseau local | Mets une passerelle du bon sous-réseau |
| `multiple interface match` | Deux interfaces de la machine dans le même sous-réseau | Sépare les sous-réseaux |
| `loop detected` | Deux routeurs se renvoient le paquet | Une route pointe dans le mauvais sens |
| `private subnets not routed over internet` | Adresse RFC 1918 envoyée vers Internet | Utilise une IP publique côté Internet |
| `invalid default route on internet` | `0.0.0.0/0` sur le nœud Internet | Mets une route précise |
| `destination IP reached` | ✅ | — |

La ligne qui précède l'erreur te dit **sur quelle machine** ça casse. C'est là qu'il
faut corriger, pas ailleurs.

---

## 12. Checklist de soutenance

Trois niveaux tirés au hasard, temps limité, pas d'outil. Ordre d'attaque :

1. **Objectifs** : qui doit joindre qui ?
2. **Champs éditables** : qu'est-ce que j'ai le droit de changer ?
3. **Par segment** : même réseau des deux côtés ? même masque ? IP valides et
   distinctes ?
4. **Par machine** : la passerelle est-elle dans mon sous-réseau ?
5. **Par objectif** : l'aller passe ? le retour passe ?
6. **Ordre des routes** : la route par défaut est-elle en dernier ?
7. Check again, puis lire les logs.

Et avant de passer au niveau suivant : **[Get my config]**. Les 10 fichiers doivent
finir à la racine du dépôt, et ton login doit être renseigné dans l'interface.

---

## 13. Tables à mémoriser

**Puissances de 2 / bits d'un octet**

```
128  64  32  16  8  4  2  1
```

**Masques et blocs**

| CIDR | Masque | Bloc | Hôtes |
| --- | --- | --- | --- |
| /8 | 255.0.0.0 | 256 (octet 1) | 16 777 214 |
| /16 | 255.255.0.0 | 256 (octet 2) | 65 534 |
| /24 | 255.255.255.0 | 256 (octet 4) | 254 |
| /25 | 255.255.255.128 | 128 | 126 |
| /26 | 255.255.255.192 | 64 | 62 |
| /27 | 255.255.255.224 | 32 | 30 |
| /28 | 255.255.255.240 | 16 | 14 |
| /29 | 255.255.255.248 | 8 | 6 |
| /30 | 255.255.255.252 | 4 | 2 |
| /32 | 255.255.255.255 | 1 | 1 (hôte unique) |

**Plages privées**

```
10.0.0.0/8        172.16.0.0/12        192.168.0.0/16
```

**Les trois questions**, encore :

1. Même sous-réseau ? → `IP AND masque` identique des deux côtés, même masque.
2. IP valide ? → ni réseau, ni broadcast, ni doublon, 1er octet ≤ 223, pas 127.
3. Sortie ? → passerelle dans mon propre sous-réseau, route par défaut en dernier.
