[← Sommaire](../README.md#table-des-matières)

# 11. Estimation de densité par mélanges gaussiens

### Le modèle de mélange gaussien

Imaginez que vous regardez la liste des tailles de tous les élèves d'une grande école, du CP à la terminale, mélangés dans un même tableau. Si vous tracez l'histogramme de ces tailles, vous ne verrez **pas** une seule cloche bien nette: vous verrez plutôt deux ou trois bosses. Une bosse vers $`1{,}20`$ m (les petits), une bosse vers $`1{,}70`$ m (les grands), et peut-être une bosse intermédiaire. Une seule loi normale (loi gaussienne) ne peut pas décrire ça: elle n'a qu'une seule bosse. Mais si on **additionne plusieurs cloches**, chacune avec sa position, sa largeur et son importance, on peut épouser n'importe quelle forme bosselée. C'est exactement l'idée du **mélange gaussien** (Gaussian mixture model, abrégé GMM).

Ce chapitre répond à une question très concrète: *étant donné un nuage de points, comment apprendre la « forme » de la densité de probabilité qui les a engendrés, quand cette forme n'est pas une simple gaussienne ?* La réponse, superposer des gaussiennes et ajuster automatiquement leurs paramètres, est l'une des techniques les plus utilisées en apprentissage automatique (machine learning): segmentation de clientèle, compression d'images, reconnaissance vocale, détection d'anomalies, initialisation de réseaux de neurones génératifs.

#### Rappel express: une seule gaussienne

On suppose connue (chapitre 6) la **densité gaussienne multivariée** sur $`\mathbb{R}^d`$. Pour un vecteur $`\mathbf{x} \in \mathbb{R}^d`$, de moyenne $`\boldsymbol{\mu} \in \mathbb{R}^d`$ et de matrice de covariance $`\boldsymbol{\Sigma}`$ (symétrique définie positive de taille $`d \times d`$), elle vaut:

```math
\mathcal{N}(\mathbf{x} \mid \boldsymbol{\mu}, \boldsymbol{\Sigma})
= \frac{1}{(2\pi)^{d/2}\,\lvert \boldsymbol{\Sigma}\rvert^{1/2}}
\exp\!\left(-\tfrac{1}{2}(\mathbf{x}-\boldsymbol{\mu})^{\top}\boldsymbol{\Sigma}^{-1}(\mathbf{x}-\boldsymbol{\mu})\right).
```

> **Le symbole $`\mid`$ (barre verticale).** Ce symbole représente l'idée de « sachant que » ou « avec les réglages ». Quand on écrit $`\mathcal{N}(\mathbf{x} \mid \boldsymbol{\mu}, \boldsymbol{\Sigma})`$, on lit: « la valeur de la cloche au point $`\mathbf{x}`$, **avec** comme réglages le centre $`\boldsymbol{\mu}`$ et l'étalement $`\boldsymbol{\Sigma}`$ ». C'est comme une machine à dessiner des cloches: à gauche de la barre, l'endroit où l'on regarde; à droite, les boutons de réglage de la machine.

Cette unique cloche a un centre unique et une seule région de forte densité. Insuffisant pour des données multimodales (à plusieurs bosses).

#### Definition du modele de melange

> **Définition (mélange gaussien).** Un mélange gaussien à $`K`$ composantes est la densité de probabilité sur $`\mathbb{R}^d`$ définie par
> ```math
> p(\mathbf{x}) = \sum_{k=1}^{K} \pi_k \, \mathcal{N}(\mathbf{x} \mid \boldsymbol{\mu}_k, \boldsymbol{\Sigma}_k),
> ```
> où les $`\pi_k`$ sont des réels appelés **poids de mélange** (mixing coefficients) vérifiant
> ```math
> \pi_k \ge 0 \quad \text{pour tout } k, \qquad \sum_{k=1}^{K} \pi_k = 1,
> ```
> et où $`(\boldsymbol{\mu}_k, \boldsymbol{\Sigma}_k)`$ sont la moyenne et la covariance de la $`k`$-ième composante (cloche).

Décortiquons chaque symbole nouveau.

> **Le symbole $`K`$.** Ce symbole représente **le nombre de cloches** qu'on empile. Si $`K=3`$, on mélange trois gaussiennes. C'est comme décider combien de groupes on pense qu'il y a dans la classe: les petits, les moyens, les grands $`\rightarrow K=3`$.

> **Le symbole $`\pi_k`$ (les poids de mélange).** Ce symbole représente **l'importance relative de chaque cloche**. Le petit $`k`$ en bas (l'indice) dit « de quelle cloche on parle »: $`\pi_1`$ est le poids de la cloche n°1, $`\pi_2`$ celui de la cloche n°2, etc. Attention, ce $`\pi`$ ici n'a rien à voir avec le nombre $`3{,}1415\dots`$: c'est juste la lettre grecque qu'on réutilise pour nommer une proportion. Imaginez un gros gâteau coupé en parts: $`\pi_k`$ est la taille de la part de la cloche $`k`$. Toutes les parts mises bout à bout font le gâteau entier, donc elles s'additionnent à $`1`$ ($`100\%`$). Et une part ne peut pas être négative, donc $`\pi_k \ge 0`$. Concrètement, si dans l'école il y a $`50\%`$ de petits, $`30\%`$ de moyens et $`20\%`$ de grands, alors $`\pi_1=0{,}5`$, $`\pi_2=0{,}3`$, $`\pi_3=0{,}2`$.

> **Le symbole $`\boldsymbol{\mu}_k`$ et $`\boldsymbol{\Sigma}_k`$.** Le $`\boldsymbol{\mu}`$ (lettre grecque « mu », en gras car c'est un vecteur) représente **le centre** de la cloche $`k`$: l'endroit où elle culmine. Le $`\boldsymbol{\Sigma}`$ (lettre grecque « sigma » majuscule, une matrice) représente **l'étalement et l'inclinaison** de la cloche $`k`$: est-elle large ou serrée, ronde ou ovale, penchée ou droite ? Ce sont les mêmes objets qu'au chapitre 6, mais maintenant on en a un jeu **par cloche**, d'où l'indice $`k`$.

La contrainte $`\sum_k \pi_k = 1`$ avec $`\pi_k \ge 0`$ fait que $`p(\mathbf{x})`$ est bien une densité: elle est positive (somme de termes positifs) et son intégrale vaut $`1`$, car

```math
\int_{\mathbb{R}^d} p(\mathbf{x})\,\mathrm{d}\mathbf{x}
= \sum_{k=1}^{K} \pi_k \underbrace{\int_{\mathbb{R}^d}\mathcal{N}(\mathbf{x}\mid\boldsymbol{\mu}_k,\boldsymbol{\Sigma}_k)\,\mathrm{d}\mathbf{x}}_{=\,1}
= \sum_{k=1}^{K}\pi_k = 1.
```

> **Rappel sur le symbole $`\sum`$.** On le suppose connu: c'est « une boucle qui additionne ». $`\sum_{k=1}^{K} a_k`$ veut dire « fais la somme $`a_1 + a_2 + \dots + a_K`$ ». Ici la boucle additionne les $`K`$ cloches pondérées. On a pu sortir chaque $`\pi_k`$ de l'intégrale car l'intégration porte sur $`\mathbf{x}`$, pas sur $`k`$.

#### Le melange comme densite vraiment universelle

Pourquoi se donner tant de mal ? Parce qu'un mélange gaussien est un **approximateur universel de densités**. Intuitivement: en plaçant beaucoup de petites cloches côte à côte (à la manière des pixels qui reconstituent une image, ou des briques Lego qui épousent une courbe), on approche d'aussi près qu'on veut n'importe quelle densité continue raisonnable.

> **Théorème (densité des mélanges gaussiens, version informelle).** Soit $`p^\star`$ une densité de probabilité continue sur $`\mathbb{R}^d`$. Pour tout $`\varepsilon > 0`$, il existe un mélange gaussien fini $`p`$ tel que $`\int_{\mathbb{R}^d} \lvert p(\mathbf{x}) - p^\star(\mathbf{x})\rvert\,\mathrm{d}\mathbf{x} < \varepsilon`$.

*Idée de preuve.* On approche $`p^\star`$ par une convolution avec un noyau gaussien d'écart-type $`\sigma`$: la fonction $`p_\sigma = p^\star * \mathcal{N}(\cdot \mid \mathbf{0}, \sigma^2 \mathbf{I})`$ converge vers $`p^\star`$ en norme $`L^1`$ quand $`\sigma \to 0`$ (propriété d'approximation de l'identité). Or

```math
p_\sigma(\mathbf{x}) = \int_{\mathbb{R}^d} p^\star(\mathbf{y})\,\mathcal{N}(\mathbf{x}\mid \mathbf{y}, \sigma^2\mathbf{I})\,\mathrm{d}\mathbf{y}
```

est une « somme continue » (intégrale) de gaussiennes centrées en chaque $`\mathbf{y}`$, pondérées par $`p^\star(\mathbf{y})`$. On discrétise cette intégrale par une somme de Riemann finie: on obtient un mélange fini de gaussiennes (toutes de covariance $`\sigma^2 \mathbf{I}`$) aussi proche qu'on veut de $`p_\sigma`$, donc de $`p^\star`$. $`\;\blacksquare`$

> **Le symbole $`L^1`$ et la norme $`\|\cdot\|_{L^1}`$.** « Converger en norme $`L^1`$ » veut simplement dire que **l'aire totale de l'écart** entre les deux courbes, $`\int \lvert p_\sigma - p^\star\rvert`$, devient aussi petite qu'on veut. Image: on superpose les deux dessins de densité et on mesure la surface coloriée qui dépasse; cette surface tend vers $`0`$.

> **Remarque (universel ne veut pas dire facile).** L'universalité est un résultat d'*existence*: elle garantit qu'un bon mélange existe, pas qu'on saura le trouver, ni avec combien de composantes. Trouver les bons $`\pi_k, \boldsymbol{\mu}_k, \boldsymbol{\Sigma}_k`$ à partir de données est précisément le problème d'apprentissage des sections suivantes.

#### Variantes de structure de covariance

En pratique on contraint souvent la forme des $`\boldsymbol{\Sigma}_k`$ pour réduire le nombre de paramètres (et donc le risque de surapprentissage / overfitting). Pour des données en dimension $`d`$:

| Type de covariance | Forme de $`\boldsymbol{\Sigma}_k`$ | Forme des cloches | Nb total de paramètres de covariance |
|---|---|---|---|
| `spherical` | $`\sigma_k^2\,\mathbf{I}`$ | boules de rayon variable | $`K`$ |
| `diag` | $`\mathrm{diag}(\sigma_{k,1}^2,\dots,\sigma_{k,d}^2)`$ | ellipsoïdes alignés sur les axes | $`K d`$ |
| `full` | matrice SDP quelconque | ellipsoïdes penchés quelconques | $`K\,\dfrac{d(d+1)}{2}`$ |
| `tied` | une seule $`\boldsymbol{\Sigma}`$ commune | même forme pour toutes | $`\dfrac{d(d+1)}{2}`$ |

> **Le symbole $`\mathbf{I}`$.** Il représente la **matrice identité**: des $`1`$ sur la diagonale, des $`0`$ ailleurs. Multiplier par $`\sigma^2 \mathbf{I}`$, c'est dire « une cloche parfaitement ronde, de même largeur dans toutes les directions ». C'est le réglage le plus simple.

> **Pourquoi $`\dfrac{d(d+1)}{2}`$ pour une covariance `full` ?** Une matrice $`d\times d`$ a $`d^2`$ cases, mais $`\boldsymbol{\Sigma}`$ est **symétrique** ($`\boldsymbol{\Sigma}=\boldsymbol{\Sigma}^{\top}`$): la moitié au-dessus de la diagonale répète la moitié en dessous. Il reste donc la diagonale ($`d`$ termes) plus le triangle strictement supérieur ($`\tfrac{d(d-1)}{2}`$ termes), soit $`d + \tfrac{d(d-1)}{2} = \tfrac{d(d+1)}{2}`$ nombres libres.

> **Piège (le nombre de paramètres explose).** Une covariance `full` coûte $`\frac{d(d+1)}{2}`$ nombres **par composante**. En dimension $`d=100`$ avec $`K=10`$, cela fait déjà $`10 \times \frac{100\times 101}{2} = 50\,500`$ paramètres rien que pour les covariances. Si on a peu de données, on préfère `diag` ou `spherical`, ou on régularise (voir plus loin).

#### Generer un point: le mode d'emploi

Un mélange n'est pas qu'une formule: c'est une **recette pour fabriquer des données**. Pour tirer un point au hasard selon $`p`$:

```mermaid
flowchart TD
    A["Choisir une cloche k au hasard<br/>avec probabilites pi_1, ..., pi_K"] --> B["Tirer x dans la gaussienne<br/>N(mu_k, Sigma_k) de cette cloche"]
    B --> C["Renvoyer x"]
```

Autrement dit: d'abord on lance un dé truqué (les faces ont les probabilités $`\pi_k`$) pour décider de quel groupe vient le point; ensuite on tire le point dans la cloche de ce groupe. Cette lecture « en deux temps » est la clé de toute la théorie (section sur la variable latente).

> **Exemple chiffré (genèse à la main).** Prenons $`d=1`$, $`K=2`$, avec $`\pi_1=0{,}7`$, $`\pi_2=0{,}3`$, $`\mu_1=0`$, $`\sigma_1=1`$, $`\mu_2=5`$, $`\sigma_2=0{,}5`$.
> 1. Je tire un nombre $`u`$ uniforme dans $`[0,1]`$. Disons $`u=0{,}55`$. Comme $`0{,}55 < 0{,}7 = \pi_1`$, je choisis la cloche 1.
> 2. Je tire $`x`$ dans $`\mathcal{N}(0, 1)`$. Disons $`x = 0{,}82`$. Mon point est $`0{,}82`$.
> Si j'avais obtenu $`u = 0{,}9 > 0{,}7`$, j'aurais choisi la cloche 2 et tiré $`x`$ dans $`\mathcal{N}(5, 0{,}25)`$ (rappel: ici $`\sigma_2=0{,}5`$ donc la variance vaut $`\sigma_2^2=0{,}25`$).

Voici le code correspondant, qui génère un jeu de données et trace la densité théorique.

```python
import numpy as np

rng = np.random.default_rng(0)

# Parametres du melange (1D, K=2)
pis    = np.array([0.7, 0.3])
mus    = np.array([0.0, 5.0])
sigmas = np.array([1.0, 0.5])          # ecarts-types (pas variances)

def echantillonne_melange(n, pis, mus, sigmas, rng):
    # Etape 1 : choisir la cloche de chaque point (le "de truque")
    k = rng.choice(len(pis), size=n, p=pis)
    # Etape 2 : tirer dans la gaussienne choisie
    return rng.normal(loc=mus[k], scale=sigmas[k])

X = echantillonne_melange(10_000, pis, mus, sigmas, rng)

def densite_melange(x, pis, mus, sigmas):
    # somme_k pi_k * N(x | mu_k, sigma_k^2)
    comp = (pis[None, :]
            * np.exp(-0.5 * ((x[:, None] - mus[None, :]) / sigmas[None, :]) ** 2)
            / (np.sqrt(2 * np.pi) * sigmas[None, :]))
    return comp.sum(axis=1)

grille = np.linspace(-4, 8, 400)
print("Integrale numerique de p :",
      np.trapz(densite_melange(grille, pis, mus, sigmas), grille))
# -> proche de 1.0 : c'est bien une densite
```

> **Application en machine learning.** Ce mécanisme « choisir un groupe puis générer » est le squelette de tout *modèle génératif à variable latente*: mélanges gaussiens, mais aussi modèles de Markov cachés, auto-encodeurs variationnels (VAE). Comprendre le GMM, c'est poser la première brique conceptuelle des modèles génératifs profonds modernes.

---

### Apprentissage par maximum de vraisemblance

On dispose maintenant de données $`\mathbf{x}_1, \dots, \mathbf{x}_N`$ (le nuage observé) et on **suppose** qu'elles ont été tirées indépendamment d'un mélange gaussien dont on ignore les paramètres. Le but: **retrouver** les meilleurs $`\pi_k, \boldsymbol{\mu}_k, \boldsymbol{\Sigma}_k`$. La méthode est celle du chapitre 8: le **maximum de vraisemblance** (maximum likelihood).

> **Le symbole $`N`$.** Il représente **le nombre de points de données** qu'on a observés (la taille de l'échantillon). À ne pas confondre avec $`K`$ (le nombre de cloches): $`N`$ se compte en milliers (les élèves mesurés), $`K`$ en unités (les groupes supposés).

> **Le symbole $`\boldsymbol{\theta}`$.** Il représente **le sac contenant TOUS les réglages à apprendre**: $`\boldsymbol{\theta} = \{\pi_k, \boldsymbol{\mu}_k, \boldsymbol{\Sigma}_k\}_{k=1}^{K}`$. Plutôt que d'écrire la longue liste à chaque fois, on l'emballe dans une seule lettre, comme on rangerait tous ses outils dans une seule boîte à outils nommée $`\boldsymbol{\theta}`$ (« thêta »).

#### La fonction de vraisemblance

La **vraisemblance** (likelihood) d'un jeu de paramètres, c'est « la probabilité que ce modèle aurait donnée à nos données ». Comme les points sont indépendants, la probabilité de les voir tous ensemble est le **produit** des probabilités de chacun:

```math
\mathcal{L}(\boldsymbol{\theta}) = p(\mathbf{x}_1,\dots,\mathbf{x}_N \mid \boldsymbol{\theta})
= \prod_{n=1}^{N} p(\mathbf{x}_n \mid \boldsymbol{\theta})
= \prod_{n=1}^{N} \sum_{k=1}^{K} \pi_k\, \mathcal{N}(\mathbf{x}_n \mid \boldsymbol{\mu}_k, \boldsymbol{\Sigma}_k).
```

> **Le symbole $`\prod`$ (produit).** Frère jumeau de $`\sum`$, mais il **multiplie** au lieu d'additionner. $`\prod_{n=1}^{N} a_n`$ veut dire $`a_1 \times a_2 \times \dots \times a_N`$. Pourquoi un produit ? Parce que pour des événements indépendants, les probabilités se multiplient (la proba de « pile puis pile » est $`\tfrac12 \times \tfrac12`$). C'est une boucle, mais qui multiplie.

#### Pourquoi on passe au logarithme

Multiplier des milliers de nombres tous compris entre $`0`$ et $`1`$ donne un résultat astronomiquement petit (sous-débordement numérique / underflow: l'ordinateur l'arrondit à $`0`$). Et un produit est pénible à dériver. On prend donc le **logarithme**, qui transforme les produits en sommes et est strictement croissant (donc il ne change pas l'emplacement du maximum):

```math
\ell(\boldsymbol{\theta}) = \ln \mathcal{L}(\boldsymbol{\theta})
= \sum_{n=1}^{N} \ln\!\left( \sum_{k=1}^{K} \pi_k\, \mathcal{N}(\mathbf{x}_n \mid \boldsymbol{\mu}_k, \boldsymbol{\Sigma}_k) \right).
```

> **Le symbole $`\ln`$.** C'est le **logarithme naturel** (logarithme en base $`e`$). Vu comme une « règle à calcul » magique: il transforme une multiplication en addition ($`\ln(ab) = \ln a + \ln b`$) et écrase les ordres de grandeur. Comme il est strictement croissant, l'endroit où $`\mathcal{L}`$ est maximale est aussi l'endroit où $`\ln\mathcal{L}`$ est maximale: passer au log ne déplace pas le sommet. On l'utilise partout en apprentissage car il rend les calculs stables et additifs.

> **Le symbole $`\ell`$ (« l » cursif).** Il représente la **log-vraisemblance** (log-likelihood): juste le logarithme de la vraisemblance $`\mathcal{L}`$. Maximiser $`\ell`$ revient exactement à maximiser $`\mathcal{L}`$, mais en plus confortable. L'estimateur du maximum de vraisemblance est
> ```math
> \hat{\boldsymbol{\theta}} = \arg\max_{\boldsymbol{\theta}} \ell(\boldsymbol{\theta}).
> ```

> **Piège central (le log d'une somme).** Pour une seule gaussienne, le $`\ln`$ « mange » l'exponentielle et tout se simplifie. Ici, le $`\ln`$ porte sur une **somme** $`\sum_k \pi_k \mathcal{N}(\cdot)`$: impossible de la casser proprement, car $`\ln(a+b) \ne \ln a + \ln b`$. C'est cette somme **à l'intérieur** du logarithme qui rend le problème difficile et interdit une solution en forme close. Tout l'algorithme EM (section suivante) est né pour contourner cet obstacle.

#### Les equations de stationnarite

Cherchons les points où le gradient s'annule. On introduit une quantité qui va devenir centrale.

> **Les responsabilités $`r_{nk}`$.** Ce symbole représente **la part de responsabilité de la cloche $`k`$ dans l'apparition du point $`\mathbf{x}_n`$**. Les deux indices se lisent: $`n`$ = quel point, $`k`$ = quelle cloche. Image: un point de donnée est une « affaire à résoudre », et les $`K`$ cloches sont des suspectes; $`r_{nk}`$ est la probabilité que ce soit la cloche $`k`$ la coupable pour le point $`n`$. Pour un point donné, les responsabilités de toutes les cloches s'additionnent à $`1`$ ($`\sum_k r_{nk}=1`$): on est sûr qu'**une** des cloches l'a produit. Formellement, c'est la probabilité a posteriori (règle de Bayes) que le point $`n`$ vienne de la cloche $`k`$:
> ```math
> r_{nk} = \frac{\pi_k\,\mathcal{N}(\mathbf{x}_n \mid \boldsymbol{\mu}_k, \boldsymbol{\Sigma}_k)}{\displaystyle\sum_{j=1}^{K} \pi_j\,\mathcal{N}(\mathbf{x}_n \mid \boldsymbol{\mu}_j, \boldsymbol{\Sigma}_j)}.
> ```
> Le numérateur, c'est « cloche $`k`$ choisie ET point généré par elle »; le dénominateur, c'est la proba totale du point (toutes cloches confondues). On divise pour normaliser, comme on répartirait $`100\%`$ de soupçons entre les suspectes.

Dérivons $`\ell`$ par rapport à chaque paramètre. Pour $`\boldsymbol{\mu}_k`$, en utilisant
$`\dfrac{\partial}{\partial \boldsymbol{\mu}_k}\mathcal{N}(\mathbf{x}_n\mid\boldsymbol{\mu}_k,\boldsymbol{\Sigma}_k) = \mathcal{N}(\mathbf{x}_n\mid\boldsymbol{\mu}_k,\boldsymbol{\Sigma}_k)\,\boldsymbol{\Sigma}_k^{-1}(\mathbf{x}_n - \boldsymbol{\mu}_k)`$
et la règle de dérivation $`\dfrac{\partial}{\partial \boldsymbol{\mu}_k}\ln(\cdot) = \dfrac{1}{(\cdot)}\dfrac{\partial(\cdot)}{\partial \boldsymbol{\mu}_k}`$:

```math
\frac{\partial \ell}{\partial \boldsymbol{\mu}_k}
= \sum_{n=1}^{N} \underbrace{\frac{\pi_k\,\mathcal{N}(\mathbf{x}_n\mid\boldsymbol{\mu}_k,\boldsymbol{\Sigma}_k)}{\sum_j \pi_j\,\mathcal{N}(\mathbf{x}_n\mid\boldsymbol{\mu}_j,\boldsymbol{\Sigma}_j)}}_{=\,r_{nk}} \boldsymbol{\Sigma}_k^{-1}(\mathbf{x}_n - \boldsymbol{\mu}_k) = \mathbf{0}.
```

Comme par magie, la responsabilité $`r_{nk}`$ **apparaît toute seule** dans le gradient: le $`\pi_k\mathcal{N}(\cdot)`$ venant de la dérivée se combine avec le $`1/p(\mathbf{x}_n)`$ venant du $`\ln`$. En multipliant à gauche par $`\boldsymbol{\Sigma}_k`$ (inversible) et en isolant $`\boldsymbol{\mu}_k`$:

```math
\boxed{\;\boldsymbol{\mu}_k = \frac{\sum_{n=1}^{N} r_{nk}\,\mathbf{x}_n}{\sum_{n=1}^{N} r_{nk}} = \frac{1}{N_k}\sum_{n=1}^{N} r_{nk}\,\mathbf{x}_n\;}, \qquad N_k := \sum_{n=1}^{N} r_{nk}.
```

> **Le symbole $`N_k`$ (effectif mou).** Il représente le **nombre « mou » de points attribués à la cloche $`k`$**: on additionne les responsabilités de tous les points pour cette cloche. Si la cloche $`k`$ « possède » fermement $`300`$ points, $`N_k \approx 300`$. Mais comme les responsabilités sont des fractions, $`N_k`$ peut valoir $`287{,}4`$: c'est un comptage flou, pas entier. Comme $`\sum_k r_{nk}=1`$ pour chaque point, on a toujours $`\sum_k N_k = N`$.

La moyenne optimale est donc la **moyenne des points pondérée par leur responsabilité**: chaque point « vote » pour le centre de la cloche $`k`$ avec un poids égal à sa responsabilité envers elle. De même, en dérivant par rapport à $`\boldsymbol{\Sigma}_k`$ (calcul matriciel sur $`\ln\lvert\boldsymbol{\Sigma}\rvert`$ et $`\boldsymbol{\Sigma}^{-1}`$, chapitre 6) et en annulant le gradient:

```math
\boxed{\;\boldsymbol{\Sigma}_k = \frac{1}{N_k}\sum_{n=1}^{N} r_{nk}\,(\mathbf{x}_n - \boldsymbol{\mu}_k)(\mathbf{x}_n - \boldsymbol{\mu}_k)^{\top}\;}.
```

> **Le symbole $`(\mathbf{x}_n - \boldsymbol{\mu}_k)(\mathbf{x}_n - \boldsymbol{\mu}_k)^{\top}`$ (produit extérieur).** Un vecteur colonne $`\mathbf{v}\in\mathbb{R}^d`$ multiplié par sa propre transposée $`\mathbf{v}^{\top}`$ (ligne) donne une **matrice** $`d\times d`$: la case $`(i,j)`$ vaut $`v_i v_j`$. C'est l'opposé du produit scalaire $`\mathbf{v}^{\top}\mathbf{v}`$ (qui, lui, donne un seul nombre). Ici cette matrice mesure comment l'écart d'un point à son centre s'étale dans chaque paire de directions; en moyennant ces matrices (pondérées par $`r_{nk}`$) on reconstruit la forme de la cloche.

Pour les poids $`\pi_k`$, il faut tenir compte de la contrainte $`\sum_k \pi_k = 1`$. On ajoute un **multiplicateur de Lagrange** $`\lambda`$ (chapitre sur le lagrangien) et on annule la dérivée de $`\ell(\boldsymbol{\theta}) + \lambda\big(\sum_k \pi_k - 1\big)`$ par rapport à $`\pi_k`$:

```math
\frac{\partial}{\partial \pi_k}\left[\ell + \lambda\Big(\textstyle\sum_j \pi_j - 1\Big)\right]
= \sum_{n=1}^{N} \frac{\mathcal{N}(\mathbf{x}_n\mid\boldsymbol{\mu}_k,\boldsymbol{\Sigma}_k)}{\sum_j \pi_j\,\mathcal{N}(\mathbf{x}_n\mid\boldsymbol{\mu}_j,\boldsymbol{\Sigma}_j)} + \lambda = \frac{N_k}{\pi_k} + \lambda = 0.
```

(On a reconnu $`\frac{N_k}{\pi_k}`$: en multipliant et divisant le terme de gauche par $`\pi_k`$, on fait apparaître $`r_{nk}`$ au numérateur, dont la somme sur $`n`$ vaut $`N_k`$.) Donc $`\pi_k = -N_k/\lambda`$. En sommant sur $`k`$ et en utilisant $`\sum_k \pi_k = 1`$ et $`\sum_k N_k = N`$, on trouve $`-N/\lambda = 1`$, soit $`\lambda = -N`$, d'où:

```math
\boxed{\;\pi_k = \frac{N_k}{N}\;}.
```

> **Le symbole $`\lambda`$ (multiplicateur de Lagrange).** C'est une **variable supplémentaire** qu'on s'autorise à introduire pour gérer une contrainte d'égalité (ici « les poids somment à $`1`$ »). Image: un ressort qui rappelle la solution vers la zone autorisée; sa « raideur » $`\lambda`$ s'ajuste toute seule pour que la contrainte soit pile respectée. On le détermine en réinjectant la contrainte, comme on vient de le faire pour trouver $`\lambda=-N`$.

> **Lecture intuitive.** Le poids optimal d'une cloche est simplement **sa part du gâteau**: le nombre mou de points qu'elle possède, divisé par le total. Limpide.

#### Le serpent qui se mord la queue

Regardons bien les trois formules encadrées. Elles donnent $`\boldsymbol{\mu}_k, \boldsymbol{\Sigma}_k, \pi_k`$ **en fonction des responsabilités** $`r_{nk}`$. Mais la définition de $`r_{nk}`$ dépend elle-même de $`\boldsymbol{\mu}_k, \boldsymbol{\Sigma}_k, \pi_k`$ ! C'est un système **implicite**: pour calculer les paramètres il faut les responsabilités, et pour les responsabilités il faut les paramètres.

```mermaid
flowchart LR
    P["Parametres<br/>pi_k, mu_k, Sigma_k"] -->|"definissent"| R["Responsabilites<br/>r_nk"]
    R -->|"definissent"| P
```

> **L'idée qui sauve tout.** Quand un système s'auto-référence comme ça, une stratégie naturelle est le **point fixe**: on devine des paramètres, on en déduit les responsabilités, puis on recalcule les paramètres à partir de ces responsabilités, et on recommence jusqu'à stabilisation. Cette alternance porte un nom: c'est l'algorithme espérance-maximisation, objet de la section suivante. Les équations encadrées ci-dessus ne sont donc pas une solution close, mais les **règles de mise à jour** d'une boucle.

---

### L'algorithme espérance-maximisation (EM)

L'algorithme **espérance-maximisation** (expectation-maximization, EM) résout le problème du serpent qui se mord la queue par une alternance simple et élégante. C'est l'un des algorithmes les plus importants de tout l'apprentissage statistique.

#### Le principe en deux temps

> **Intuition (le jeu du professeur et des groupes).** Vous êtes professeur face à une classe mélangée et vous voulez (a) deviner à quel groupe appartient chaque élève et (b) décrire chaque groupe (sa taille moyenne, sa dispersion). Problème: pour assigner les élèves il faut connaître les groupes, et pour décrire les groupes il faut savoir qui en fait partie. Solution pragmatique:
> - **Étape E:** avec votre description actuelle des groupes, calculez pour chaque élève sa probabilité d'appartenance à chaque groupe (les responsabilités).
> - **Étape M:** en supposant ces appartenances, **re-décrivez** chaque groupe (recalculez moyenne, dispersion, taille).
> Répétez. À chaque tour, votre description s'améliore.

```mermaid
flowchart TD
    Init["Initialisation<br/>pi_k, mu_k, Sigma_k"] --> E
    E["Etape E : calculer les responsabilites<br/>r_nk avec les parametres actuels"] --> M
    M["Etape M : recalculer pi_k, mu_k, Sigma_k<br/>a partir des r_nk"] --> Conv{"Log-vraisemblance<br/>stabilisee ?"}
    Conv -->|non| E
    Conv -->|oui| Fin["Renvoyer les parametres"]
```

#### Algorithme detaille

> **Algorithme (EM pour mélange gaussien).**
> **Entrée:** données $`\{\mathbf{x}_n\}_{n=1}^N`$, nombre de composantes $`K`$.
> **Initialisation:** choisir $`\boldsymbol{\theta}^{(0)} = \{\pi_k, \boldsymbol{\mu}_k, \boldsymbol{\Sigma}_k\}`$ (souvent via $`k`$-moyennes, voir plus bas).
> **Répéter** pour $`t = 0, 1, 2, \dots`$ jusqu'à convergence:
>
> **Étape E (espérance).** Pour tout $`n, k`$, calculer la responsabilité avec les paramètres courants $`\boldsymbol{\theta}^{(t)}`$:
> ```math
> r_{nk}^{(t)} = \frac{\pi_k^{(t)}\,\mathcal{N}(\mathbf{x}_n \mid \boldsymbol{\mu}_k^{(t)}, \boldsymbol{\Sigma}_k^{(t)})}{\sum_{j=1}^{K} \pi_j^{(t)}\,\mathcal{N}(\mathbf{x}_n \mid \boldsymbol{\mu}_j^{(t)}, \boldsymbol{\Sigma}_j^{(t)})}.
> ```
>
> **Étape M (maximisation).** Poser $`N_k = \sum_{n} r_{nk}^{(t)}`$, puis mettre à jour:
> ```math
> \pi_k^{(t+1)} = \frac{N_k}{N}, \quad
> \boldsymbol{\mu}_k^{(t+1)} = \frac{1}{N_k}\sum_{n} r_{nk}^{(t)}\mathbf{x}_n, \quad
> \boldsymbol{\Sigma}_k^{(t+1)} = \frac{1}{N_k}\sum_{n} r_{nk}^{(t)} (\mathbf{x}_n - \boldsymbol{\mu}_k^{(t+1)})(\mathbf{x}_n - \boldsymbol{\mu}_k^{(t+1)})^{\top}.
> ```
>
> **Critère d'arrêt:** s'arrêter quand $`\ell(\boldsymbol{\theta}^{(t+1)}) - \ell(\boldsymbol{\theta}^{(t)}) < \varepsilon`$ (gain de log-vraisemblance négligeable).

> **Le symbole $`t`$ et l'exposant $`^{(t)}`$.** Le $`t`$ représente **le numéro du tour de boucle** (l'itération). L'exposant entre parenthèses, comme dans $`\boldsymbol{\mu}_k^{(t)}`$, veut dire « la valeur de ce paramètre **au tour $`t`$** ». À ne pas confondre avec une puissance: $`\boldsymbol{\mu}_k^{(3)}`$ n'est pas « mu au cube », c'est « mu après 3 tours ». La parenthèse est justement là pour signaler « ce n'est pas un exposant de puissance ».

#### Lien avec k-moyennes

L'algorithme des **$`k`$-moyennes** (k-means) est un cas limite « dur » d'EM. Au lieu de responsabilités floues $`r_{nk} \in [0,1]`$, on assigne chaque point à sa cloche la plus proche ($`r_{nk} \in \{0,1\}`$). On le retrouve en prenant des covariances $`\boldsymbol{\Sigma}_k = \sigma^2 \mathbf{I}`$ communes et en faisant tendre $`\sigma \to 0`$: dans la formule des responsabilités, l'écart le plus petit écrase exponentiellement tous les autres, et la responsabilité se concentre entièrement sur la composante la plus proche.

> **Initialisation en pratique.** On initialise presque toujours EM par **$`k`$-means++** (variante de $`k`$-moyennes à tirage initial intelligent), qui évite les mauvais minima et accélère nettement la convergence. C'est le défaut de `scikit-learn` (`init_params='kmeans'` ou `'k-means++'`). Pour de très grands jeux de données, on utilise des variantes **mini-batch / stochastiques** d'EM, dans l'esprit des optimiseurs par lots du deep learning.

| Aspect | $`k`$-moyennes | EM (GMM) |
|---|---|---|
| Assignation | dure ($`0`$ ou $`1`$) | molle (responsabilités $`\in [0,1]`$) |
| Forme des groupes | sphériques, même taille | ellipsoïdes, tailles variables |
| Sortie | étiquettes | densité de probabilité complète |
| Cas particulier de... |, | EM avec $`\boldsymbol{\Sigma}=\sigma^2\mathbf{I}, \sigma\to 0`$ |

#### Garantie de convergence: la log-vraisemblance ne descend jamais

C'est la propriété fondamentale d'EM, démontrée en détail dans la section suivante via l'ELBO.

> **Théorème (monotonie d'EM).** À chaque itération d'EM, la log-vraisemblance des données ne diminue pas:
> ```math
> \ell(\boldsymbol{\theta}^{(t+1)}) \ge \ell(\boldsymbol{\theta}^{(t)}).
> ```
> Comme $`\ell`$ est majorée (une log-vraisemblance de densité ne tend pas vers $`+\infty`$ sur des configurations raisonnables, hors singularités traitées plus bas), la suite croissante $`\ell(\boldsymbol{\theta}^{(t)})`$ converge.

> **Piège (convergence vers un optimum LOCAL).** EM garantit de **monter**, pas d'atteindre le sommet le plus haut. Selon l'initialisation, on peut rester coincé sur une colline secondaire (optimum local). Remède standard: lancer EM plusieurs fois avec des initialisations différentes et **garder la solution de plus grande log-vraisemblance**.

> **Piège (singularités / divergence vers $`+\infty`$).** Si une composante se concentre sur un **seul** point ($`\boldsymbol{\mu}_k = \mathbf{x}_n`$) et que sa variance tend vers $`0`$, sa densité en ce point explose et $`\ell \to +\infty`$: c'est une singularité pathologique du maximum de vraisemblance, pas une vraie solution. Remèdes: ajouter une petite **régularisation** $`\boldsymbol{\Sigma}_k \leftarrow \boldsymbol{\Sigma}_k + \epsilon \mathbf{I}`$ (covariance floor), borner les variances, ou ré-initialiser une composante qui s'effondre.

#### Exemple chiffre deroule pas a pas

Prenons $`1`$ D, $`N=4`$ points: $`\mathbf{x} = (0,\ 1,\ 8,\ 9)`$, et $`K=2`$. Initialisons grossièrement: $`\pi_1=\pi_2=0{,}5`$, $`\mu_1=1`$, $`\mu_2=8`$, $`\sigma_1^2=\sigma_2^2=4`$.

**Étape E (premier tour).** Calculons $`r_{n1}`$ (responsabilité de la cloche 1). Avec $`\mathcal{N}(x\mid\mu,\sigma^2) = \tfrac{1}{\sqrt{2\pi\cdot 4}}\exp\!\big(-(x-\mu)^2/8\big)`$ ici:

| point $`x_n`$ | $`\mathcal{N}(x_n\mid 1,4)`$ | $`\mathcal{N}(x_n\mid 8,4)`$ | $`r_{n1}`$ | $`r_{n2}`$ |
|---|---|---|---|---|
| $`0`$ | $`0{,}1760`$ | $`\approx 0{,}0000`$ | $`\approx 1{,}000`$ | $`\approx 0{,}000`$ |
| $`1`$ | $`0{,}1995`$ | $`\approx 0{,}0000`$ | $`\approx 1{,}000`$ | $`\approx 0{,}000`$ |
| $`8`$ | $`\approx 0{,}0000`$ | $`0{,}1995`$ | $`\approx 0{,}000`$ | $`\approx 1{,}000`$ |
| $`9`$ | $`\approx 0{,}0000`$ | $`0{,}1760`$ | $`\approx 0{,}000`$ | $`\approx 1{,}000`$ |

Les points $`\{0,1\}`$ sont clairement attribués à la cloche 1, les points $`\{8,9\}`$ à la cloche 2.

**Étape M (premier tour).** $`N_1 = 1+1+0+0 = 2`$, $`N_2 = 2`$.
- $`\pi_1 = \pi_2 = 2/4 = 0{,}5`$.
- $`\mu_1 = (1\cdot 0 + 1\cdot 1)/2 = 0{,}5`$; $`\mu_2 = (8+9)/2 = 8{,}5`$.
- $`\sigma_1^2 = \tfrac12\big[(0-0{,}5)^2 + (1-0{,}5)^2\big] = 0{,}25`$; de même $`\sigma_2^2 = 0{,}25`$.

En **un seul tour**, les centres sont passés de $`(1, 8)`$ à $`(0{,}5,\ 8{,}5)`$, les vraies moyennes des deux paquets, et les variances ont chuté à $`0{,}25`$. Les tours suivants ne bougent quasiment plus: on a convergé. Vérifions par le code.

```python
import numpy as np

def em_gmm_1d(X, K, n_iter=50, eps=1e-6, seed=0):
    rng = np.random.default_rng(seed)
    N = len(X)
    # Initialisation
    mu = rng.choice(X, size=K, replace=False).astype(float)
    var = np.full(K, X.var() + 1e-3)
    pi = np.full(K, 1.0 / K)
    log_vrais = []

    def gauss(x, m, v):
        return np.exp(-0.5 * (x - m) ** 2 / v) / np.sqrt(2 * np.pi * v)

    for _ in range(n_iter):
        # ---- Etape E ----
        comp = pi[None, :] * gauss(X[:, None], mu[None, :], var[None, :])  # (N, K)
        total = comp.sum(axis=1, keepdims=True)                            # (N, 1)
        r = comp / total                                                   # responsabilites
        log_vrais.append(np.log(total).sum())
        # ---- Etape M ----
        Nk = r.sum(axis=0)                       # effectifs mous
        pi = Nk / N
        mu = (r * X[:, None]).sum(axis=0) / Nk
        var = (r * (X[:, None] - mu[None, :]) ** 2).sum(axis=0) / Nk
        var = np.maximum(var, 1e-6)              # garde-fou anti-singularite
        if len(log_vrais) > 1 and log_vrais[-1] - log_vrais[-2] < eps:
            break
    return pi, mu, var, log_vrais

X = np.array([0.0, 1.0, 8.0, 9.0])
pi, mu, var, lv = em_gmm_1d(X, K=2)
print("pi  =", np.round(pi, 3))     # ~ [0.5 0.5]
print("mu  =", np.round(mu, 3))     # ~ [0.5 8.5] (a l'ordre des composantes pres)
print("var =", np.round(var, 3))    # ~ [0.25 0.25]
print("log-vraisemblance croissante :",
      all(lv[i+1] >= lv[i] - 1e-9 for i in range(len(lv)-1)))  # True
```

#### Application concrete en machine learning

> **Segmentation et détection d'anomalies.** Après apprentissage, un GMM donne pour tout nouveau point $`\mathbf{x}`$: (1) une **étiquette de groupe** $`\arg\max_k r_{k}(\mathbf{x})`$ (segmentation de clientèle, regroupement de documents); (2) une **densité** $`p(\mathbf{x})`$, un point où $`p(\mathbf{x})`$ est très faible est une **anomalie** (fraude, défaut industriel). La covariance `full` capture des corrélations entre variables que $`k`$-moyennes ignore totalement.

> **Compression d'image par quantification.** En modélisant les couleurs (RVB, donc $`d=3`$) des pixels d'une image par un GMM à $`K`$ composantes, on remplace chaque pixel par sa composante dominante: on passe de millions de couleurs à $`K`$ couleurs représentatives. Même idée qu'une palette, mais apprise statistiquement.

---

### Perspective par variable latente

Nous avons jusqu'ici manipulé EM comme une recette qui marche. Cette dernière section dévoile **pourquoi** elle marche, grâce à un changement de point de vue d'une grande portée: voir le mélange comme un modèle à **variable latente** (latent variable), puis construire la **borne inférieure de l'évidence** (ELBO). C'est la clé théorique qui justifie la monotonie d'EM et qui sous-tend les modèles génératifs profonds modernes.

#### Le tirage en deux temps, formalise

Reprenons la recette de génération (« choisir une cloche, puis tirer dedans »). Introduisons une variable cachée qui dit **de quelle cloche** vient chaque point.

> **La variable latente $`\mathbf{z}_n`$.** Ce symbole représente **l'étiquette cachée** du point $`n`$: « de quelle cloche venez-vous ? ». On l'encode en *one-hot* (codage où un seul élément vaut $`1`$ et tous les autres valent $`0`$): $`\mathbf{z}_n = (z_{n1}, \dots, z_{nK})`$ est un vecteur de $`0`$ avec un seul $`1`$. Si $`z_{nk}=1`$, le point $`n`$ vient de la cloche $`k`$. C'est « latent » (caché) car dans les vraies données, on voit la taille de l'élève mais **pas** son groupe d'origine: cette information existe mais nous est invisible. Image: chaque point porte une étiquette secrète pliée dans sa poche; EM essaie de deviner ce qui est écrit dessus.

Le modèle génératif s'écrit alors en deux lois:

```math
p(z_{nk}=1) = \pi_k \quad\Longleftrightarrow\quad p(\mathbf{z}_n) = \prod_{k=1}^{K} \pi_k^{\,z_{nk}},
\qquad
p(\mathbf{x}_n \mid z_{nk}=1) = \mathcal{N}(\mathbf{x}_n \mid \boldsymbol{\mu}_k, \boldsymbol{\Sigma}_k).
```

> **Pourquoi l'écriture $`\pi_k^{z_{nk}}`$ ?** Astuce d'écriture très pratique: comme $`\mathbf{z}_n`$ est one-hot, dans le produit $`\prod_k \pi_k^{z_{nk}}`$ tous les exposants valent $`0`$ (et $`\pi^0=1`$) sauf celui de la vraie cloche, qui vaut $`1`$ (et $`\pi^1=\pi`$). Le produit se réduit donc au seul $`\pi_k`$ de la cloche choisie. C'est une façon compacte d'écrire « sélectionne le bon terme ».

On retrouve le mélange en **sommant sur la variable cachée** (marginalisation), c'est-à-dire en envisageant tous les groupes d'origine possibles:

```math
p(\mathbf{x}_n) = \sum_{k=1}^{K} p(z_{nk}=1)\,p(\mathbf{x}_n \mid z_{nk}=1) = \sum_{k=1}^{K} \pi_k\,\mathcal{N}(\mathbf{x}_n \mid \boldsymbol{\mu}_k, \boldsymbol{\Sigma}_k).
```

La densité de mélange n'est donc rien d'autre que la **loi marginale** d'un modèle à variable latente. Et la responsabilité $`r_{nk}`$ est exactement la **loi a posteriori** de la variable cachée, par la règle de Bayes:

```math
r_{nk} = p(z_{nk}=1 \mid \mathbf{x}_n) = \frac{p(z_{nk}=1)\,p(\mathbf{x}_n\mid z_{nk}=1)}{\sum_j p(z_{nj}=1)\,p(\mathbf{x}_n\mid z_{nj}=1)} = \frac{\pi_k\,\mathcal{N}(\mathbf{x}_n\mid\boldsymbol{\mu}_k,\boldsymbol{\Sigma}_k)}{\sum_j \pi_j\,\mathcal{N}(\mathbf{x}_n\mid\boldsymbol{\mu}_j,\boldsymbol{\Sigma}_j)}.
```

> **Le déclic.** L'étape E n'est pas une astuce sortie d'un chapeau: c'est **le calcul de la loi a posteriori de la variable cachée**. « Quelle est la proba que ce point vienne de la cloche $`k`$, maintenant que je l'observe ? » Tout EM découle de cette lecture probabiliste.

#### La vraisemblance complete

Si on connaissait les étiquettes $`\mathbf{z}_n`$ (données complètes), la log-vraisemblance serait **facile**, plus de log d'une somme:

```math
\ln p(\mathbf{X}, \mathbf{Z} \mid \boldsymbol{\theta}) = \sum_{n=1}^{N}\sum_{k=1}^{K} z_{nk}\,\big[\ln \pi_k + \ln \mathcal{N}(\mathbf{x}_n \mid \boldsymbol{\mu}_k, \boldsymbol{\Sigma}_k)\big].
```

> **Le symbole $`\mathbf{X}`$ et $`\mathbf{Z}`$.** En majuscules grasses, ils représentent **l'ensemble de toutes les données**: $`\mathbf{X} = \{\mathbf{x}_1,\dots,\mathbf{x}_N\}`$ (tout ce qu'on voit) et $`\mathbf{Z} = \{\mathbf{z}_1,\dots,\mathbf{z}_N\}`$ (toutes les étiquettes cachées). C'est juste un raccourci pour « le paquet entier ».

Le $`\ln`$ tombe maintenant **directement** sur chaque gaussienne (grâce au $`z_{nk}`$ qui sélectionne un seul terme, et au $`\ln`$ d'un produit qui devient somme): c'est ce que l'étape M sait maximiser en forme close. Mais $`\mathbf{Z}`$ est inconnu... On remplace donc, dans cette expression, $`z_{nk}`$ par son **espérance** sous la loi a posteriori courante. Comme $`z_{nk}`$ ne vaut que $`0`$ ou $`1`$, son espérance est $`\mathbb{E}[z_{nk}\mid\mathbf{x}_n] = p(z_{nk}=1\mid\mathbf{x}_n) = r_{nk}`$: c'est l'**étape E** (on prend l'*espérance* de la vraisemblance complète). D'où les deux noms: **E** comme espérance, **M** comme maximisation.

#### La borne inferieure de l'evidence (ELBO)

Voici l'outil qui unifie et justifie tout. Pour n'importe quelle distribution $`q(\mathbf{Z})`$ sur les étiquettes cachées, on a une décomposition exacte.

> **La borne inférieure de l'évidence (ELBO).** Cette quantité, notée $`\mathcal{F}(q, \boldsymbol{\theta})`$ ou ELBO (*Evidence Lower BOund*), représente **un plancher garanti sous la log-vraisemblance** $`\ell(\boldsymbol{\theta}) = \ln p(\mathbf{X}\mid\boldsymbol{\theta})`$ (l'« évidence »). Image: la vraie log-vraisemblance est un plafond qu'on ne sait pas calculer facilement (à cause du log d'une somme); l'ELBO est un **plancher** facile à calculer et à pousser vers le haut. En soulevant le plancher, on pousse forcément le plafond. Sa définition:
> ```math
> \mathcal{F}(q, \boldsymbol{\theta}) = \sum_{\mathbf{Z}} q(\mathbf{Z})\,\ln \frac{p(\mathbf{X}, \mathbf{Z}\mid\boldsymbol{\theta})}{q(\mathbf{Z})} = \mathbb{E}_{q}\!\big[\ln p(\mathbf{X},\mathbf{Z}\mid\boldsymbol{\theta})\big] + \mathbb{H}(q).
> ```
> La somme $`\sum_{\mathbf{Z}}`$ parcourt toutes les configurations possibles d'étiquettes cachées.

> **Le symbole $`q`$ (la distribution auxiliaire).** Il représente **notre hypothèse provisoire sur les étiquettes cachées**: une distribution de probabilité qu'on choisit librement pour deviner $`\mathbf{Z}`$. C'est un « brouillon » de croyance sur les groupes d'origine, qu'on a le droit d'ajuster. Quand ce brouillon coïncide avec la vérité a posteriori, la borne devient exacte (voir ci-dessous).

> **Le symbole $`\mathbb{H}(q)`$ (entropie).** Il représente **la quantité d'incertitude** contenue dans la distribution $`q`$: $`\mathbb{H}(q) = -\sum_{\mathbf{Z}} q(\mathbf{Z})\ln q(\mathbf{Z})`$. Image: si $`q`$ hésite à parts égales entre toutes les cloches, l'entropie est grande (beaucoup de flou); si $`q`$ est sûre d'elle (tout le poids sur une cloche), l'entropie vaut $`0`$ (aucun flou). L'égalité $`\mathbb{E}_q[\ln p] + \mathbb{H}(q)`$ vient juste de couper le $`\ln`$ du quotient en $`\ln p(\mathbf{X},\mathbf{Z}\mid\boldsymbol{\theta}) - \ln q(\mathbf{Z})`$.

> **Le symbole $`\mathrm{KL}(q\,\|\,p)`$ (divergence de Kullback-Leibler).** Il représente **à quel point deux distributions diffèrent**: un « écart » entre la croyance $`q`$ et la vérité $`p(\cdot\mid\mathbf{X})`$. Définie par $`\mathrm{KL}(q\,\|\,p)=\sum_{\mathbf{Z}} q(\mathbf{Z})\ln\frac{q(\mathbf{Z})}{p(\mathbf{Z}\mid\mathbf{X})}`$, elle vaut $`0`$ si et seulement si $`q=p`$, et est strictement positive sinon. Ce n'est pas une distance symétrique, mais une mesure d'« étonnement »: combien on est surpris en croyant $`q`$ alors que la réalité est $`p`$. Important: $`\mathrm{KL} \ge 0`$ **toujours** (inégalité de Gibbs).

L'identité clé, valable pour tout $`q`$ et tout $`\boldsymbol{\theta}`$, est:

```math
\boxed{\;\ell(\boldsymbol{\theta}) = \underbrace{\mathcal{F}(q, \boldsymbol{\theta})}_{\text{plancher (ELBO)}} + \underbrace{\mathrm{KL}\big(q(\mathbf{Z})\,\|\,p(\mathbf{Z}\mid\mathbf{X}, \boldsymbol{\theta})\big)}_{\ge\, 0}\;}.
```

*Démonstration.* Partons de l'ELBO et injectons la règle du produit $`p(\mathbf{X},\mathbf{Z}\mid\boldsymbol{\theta}) = p(\mathbf{Z}\mid\mathbf{X},\boldsymbol{\theta})\,p(\mathbf{X}\mid\boldsymbol{\theta})`$:

```math
\mathcal{F}(q,\boldsymbol{\theta}) = \sum_{\mathbf{Z}} q(\mathbf{Z})\ln\frac{p(\mathbf{Z}\mid\mathbf{X},\boldsymbol{\theta})\,p(\mathbf{X}\mid\boldsymbol{\theta})}{q(\mathbf{Z})}
= \sum_{\mathbf{Z}} q(\mathbf{Z})\ln p(\mathbf{X}\mid\boldsymbol{\theta}) + \sum_{\mathbf{Z}} q(\mathbf{Z})\ln\frac{p(\mathbf{Z}\mid\mathbf{X},\boldsymbol{\theta})}{q(\mathbf{Z})}.
```

Le premier terme vaut $`\ln p(\mathbf{X}\mid\boldsymbol{\theta})\sum_{\mathbf{Z}}q(\mathbf{Z}) = \ell(\boldsymbol{\theta})`$ (car $`\sum_{\mathbf{Z}}q(\mathbf{Z})=1`$ et $`\ln p(\mathbf{X}\mid\boldsymbol{\theta})`$ ne dépend pas de $`\mathbf{Z}`$). Le second vaut $`-\mathrm{KL}(q\,\|\,p(\cdot\mid\mathbf{X},\boldsymbol{\theta}))`$ (le signe vient du quotient inversé dans la définition de la KL). D'où $`\mathcal{F} = \ell - \mathrm{KL}`$, soit $`\ell = \mathcal{F} + \mathrm{KL}`$. Comme $`\mathrm{KL}\ge 0`$, on a $`\ell(\boldsymbol{\theta}) \ge \mathcal{F}(q,\boldsymbol{\theta})`$: l'ELBO est bien un plancher. $`\;\blacksquare`$

#### EM relu comme une montee de colline a deux pas

Cette décomposition donne la **vraie** définition d'EM: une **maximisation par coordonnées alternées** de l'ELBO $`\mathcal{F}(q,\boldsymbol{\theta})`$, d'abord en $`q`$ (étape E), puis en $`\boldsymbol{\theta}`$ (étape M).

```mermaid
flowchart TD
    subgraph EE["Etape E : on bouge q (theta fige)"]
      E1["Maximiser F en q<br/>=> q = p(Z|X, theta)<br/>=> KL = 0, le plancher touche le plafond"]
    end
    subgraph MM["Etape M : on bouge theta (q fige)"]
      M1["Maximiser F en theta<br/>=> nouveaux pi, mu, Sigma<br/>=> le plancher monte, donc le plafond aussi"]
    end
    E1 --> M1 --> E1
```

> **Étape E = annuler la KL.** À $`\boldsymbol{\theta}`$ fixé, $`\ell(\boldsymbol{\theta})`$ ne dépend pas de $`q`$. Maximiser $`\mathcal{F} = \ell - \mathrm{KL}`$ en $`q`$ revient donc à **minimiser** $`\mathrm{KL}(q\,\|\,p(\cdot\mid\mathbf{X},\boldsymbol{\theta}))`$. Le minimum ($`\mathrm{KL}=0`$) est atteint pour $`q^\star(\mathbf{Z}) = p(\mathbf{Z}\mid\mathbf{X},\boldsymbol{\theta})`$, c'est-à-dire $`q^\star`$ donne exactement les responsabilités $`r_{nk}`$ ! À cet instant le plancher **touche** le plafond: $`\mathcal{F}(q^\star,\boldsymbol{\theta}) = \ell(\boldsymbol{\theta})`$.

> **Étape M = soulever le plancher.** À $`q^\star`$ fixé, on maximise $`\mathcal{F}(q^\star,\boldsymbol{\theta})`$ en $`\boldsymbol{\theta}`$. Comme l'entropie $`\mathbb{H}(q^\star)`$ ne dépend pas de $`\boldsymbol{\theta}`$, cela revient à maximiser $`\mathbb{E}_{q^\star}[\ln p(\mathbf{X},\mathbf{Z}\mid\boldsymbol{\theta})]`$, exactement la **vraisemblance complète espérée** vue plus haut, dont la solution close donne les formules de mise à jour de $`\pi_k, \boldsymbol{\mu}_k, \boldsymbol{\Sigma}_k`$.

#### Demonstration propre de la monotonie d'EM

On peut maintenant prouver le théorème de la section précédente proprement.

> **Théorème (monotonie, version ELBO).** La suite des log-vraisemblances produite par EM est croissante: $`\ell(\boldsymbol{\theta}^{(t+1)}) \ge \ell(\boldsymbol{\theta}^{(t)})`$.

*Démonstration.* Notons $`q^{(t)}`$ le $`q`$ choisi à l'étape E du tour $`t`$, soit $`q^{(t)} = p(\mathbf{Z}\mid\mathbf{X},\boldsymbol{\theta}^{(t)})`$.

1. **Après l'étape E**: $`\mathrm{KL}\big(q^{(t)}\,\|\,p(\cdot\mid\mathbf{X},\boldsymbol{\theta}^{(t)})\big)=0`$, donc $`\mathcal{F}(q^{(t)}, \boldsymbol{\theta}^{(t)}) = \ell(\boldsymbol{\theta}^{(t)})`$.
2. **Après l'étape M**: on choisit $`\boldsymbol{\theta}^{(t+1)} = \arg\max_{\boldsymbol{\theta}} \mathcal{F}(q^{(t)}, \boldsymbol{\theta})`$, donc $`\mathcal{F}(q^{(t)}, \boldsymbol{\theta}^{(t+1)}) \ge \mathcal{F}(q^{(t)}, \boldsymbol{\theta}^{(t)})`$.
3. **Borne**: pour tout $`\boldsymbol{\theta}`$, l'identité donne $`\ell(\boldsymbol{\theta}) = \mathcal{F}(q^{(t)},\boldsymbol{\theta}) + \mathrm{KL}(\dots) \ge \mathcal{F}(q^{(t)},\boldsymbol{\theta})`$. En particulier $`\ell(\boldsymbol{\theta}^{(t+1)}) \ge \mathcal{F}(q^{(t)}, \boldsymbol{\theta}^{(t+1)})`$.

En enchaînant: $`\ell(\boldsymbol{\theta}^{(t+1)}) \overset{(3)}{\ge} \mathcal{F}(q^{(t)},\boldsymbol{\theta}^{(t+1)}) \overset{(2)}{\ge} \mathcal{F}(q^{(t)},\boldsymbol{\theta}^{(t)}) \overset{(1)}{=} \ell(\boldsymbol{\theta}^{(t)})`$. $`\;\blacksquare`$

> **Image finale.** Pensez à deux planches: le plafond ($`\ell`$, qu'on veut monter) et le plancher mobile ($`\mathcal{F}`$). **E** soulève le plancher jusqu'à toucher le plafond (sous eux, plus aucun espace: KL = 0). **M** pousse alors le plancher vers le haut; comme le plafond est toujours au-dessus, il monte aussi. On alterne: on ne redescend jamais.

> **Lien avec l'inférence variationnelle.** Cette décomposition $`\ell = \mathrm{ELBO} + \mathrm{KL}`$ est **exactement** la fondation des auto-encodeurs variationnels (VAE) et de l'inférence variationnelle. La différence: quand la loi a posteriori $`p(\mathbf{Z}\mid\mathbf{X})`$ est trop complexe pour être calculée exactement (réseaux profonds), on ne peut plus annuler la KL; on choisit alors une famille paramétrée $`q_{\boldsymbol\phi}`$ et on **optimise l'ELBO par descente de gradient** (différentiation automatique, optimiseur Adam), avec l'astuce de reparamétrisation. EM sur le GMM est le cas « jouet » exactement soluble de ce cadre général: c'est pourquoi le comprendre à fond est un investissement rentable.

#### Vue d'ensemble synthetique

```mermaid
flowchart TD
    A["Donnees multimodales<br/>(plusieurs bosses)"] --> B["Modele : melange<br/>p = somme pi_k N(mu_k, Sigma_k)"]
    B --> C["Variable latente z :<br/>de quelle cloche vient le point ?"]
    C --> D["Vraisemblance = log d'une somme<br/>=> pas de solution close"]
    D --> E["EM : alterner E et M<br/>= monter l'ELBO par coordonnees"]
    E --> F["E : responsabilites r_nk<br/>= posterior p(z|x)"]
    E --> G["M : pi_k=Nk/N, mu_k, Sigma_k<br/>moyennes ponderees"]
    F --> H["Garantie : ell croit, converge<br/>(optimum LOCAL)"]
    G --> H
```

---

### Exercices

Les corrigés sont détaillés juste après chaque énoncé.

#### Exercice 1: Verifier qu'un melange est bien une densite

Soit le mélange $`1`$ D $`p(x) = 0{,}4\,\mathcal{N}(x\mid -2, 1) + 0{,}6\,\mathcal{N}(x\mid 3, 4)`$ (les seconds arguments sont des **variances**).
**(a)** Vérifier que $`p`$ est une densité. **(b)** Calculer l'espérance $`\mathbb{E}[X]`$. **(c)** La variance $`\mathrm{Var}[X]`$.

> **Corrigé.**
> **(a)** Les poids $`0{,}4`$ et $`0{,}6`$ sont positifs et somment à $`1`$; chaque $`\mathcal{N}`$ intègre à $`1`$, donc $`\int p = 0{,}4 + 0{,}6 = 1`$ et $`p\ge 0`$. C'est une densité.
> **(b)** Par linéarité de l'espérance, $`\mathbb{E}[X] = \sum_k \pi_k \mu_k = 0{,}4\times(-2) + 0{,}6\times 3 = -0{,}8 + 1{,}8 = 1{,}0`$.
> **(c)** On utilise $`\mathrm{Var}[X] = \mathbb{E}[X^2] - (\mathbb{E}[X])^2`$ avec $`\mathbb{E}[X^2] = \sum_k \pi_k(\sigma_k^2 + \mu_k^2)`$ (car pour chaque composante $`\mathbb{E}[X^2\mid k] = \sigma_k^2+\mu_k^2`$). Donc $`\mathbb{E}[X^2] = 0{,}4(1 + 4) + 0{,}6(4 + 9) = 0{,}4\times 5 + 0{,}6\times 13 = 2 + 7{,}8 = 9{,}8`$. D'où $`\mathrm{Var}[X] = 9{,}8 - 1{,}0^2 = 8{,}8`$.
> **Leçon:** la variance d'un mélange ($`8{,}8`$) dépasse la moyenne pondérée des variances ($`0{,}4\times1+0{,}6\times4 = 2{,}8`$): l'écart entre les centres ajoute exactement la variance « inter-composantes » $`\sum_k \pi_k(\mu_k-\mathbb{E}[X])^2 = 0{,}4\times 9 + 0{,}6\times 4 = 6{,}0`$, et $`2{,}8 + 6{,}0 = 8{,}8`$.

#### Exercice 2: Calcul de responsabilites a la main

Mélange $`1`$ D: $`\pi_1=0{,}5, \mu_1=0, \sigma_1^2=1`$ et $`\pi_2=0{,}5, \mu_2=2, \sigma_2^2=1`$. Calculer $`r_{1}`$ et $`r_{2}`$ pour le point $`x=1`$.

> **Corrigé.** Les deux composantes ont même $`\pi`$ et même $`\sigma`$, donc le rapport ne dépend que des exponentielles.
> $`\mathcal{N}(1\mid 0,1) \propto \exp(-\tfrac12(1-0)^2) = e^{-0{,}5}`$.
> $`\mathcal{N}(1\mid 2,1) \propto \exp(-\tfrac12(1-2)^2) = e^{-0{,}5}`$.
> Les deux sont égales ! Donc $`r_1 = \frac{0{,}5\,e^{-0{,}5}}{0{,}5\,e^{-0{,}5}+0{,}5\,e^{-0{,}5}} = 0{,}5`$ et $`r_2 = 0{,}5`$.
> **Leçon:** le point $`x=1`$ est pile au milieu des deux centres; il est partagé équitablement (responsabilités $`50/50`$). C'est la frontière de décision.

#### Exercice 3: Une iteration complete d'EM

Données $`1`$ D: $`x = (-1,\ 0,\ 4,\ 5)`$, $`K=2`$. Initialisation: $`\pi_1=\pi_2=0{,}5`$, $`\mu_1=0, \mu_2=5`$, $`\sigma_1^2=\sigma_2^2=1`$. Effectuer **une** étape E puis **une** étape M (on pourra arrondir les responsabilités à $`0`$ ou $`1`$ vu la séparation).

> **Corrigé.**
> **Étape E.** Comme $`\sigma_1=\sigma_2`$ et $`\pi_1=\pi_2`$, la responsabilité ne dépend que de la distance au carré au centre. Pour $`x=-1`$: $`(x-\mu_1)^2=1`$ contre $`(x-\mu_2)^2=36 \Rightarrow r_{1,1}\approx 1`$. Pour $`x=0`$: $`0`$ contre $`25 \Rightarrow r_{1}\approx 1`$. Pour $`x=4`$: $`16`$ contre $`1 \Rightarrow r_{2}\approx 1`$. Pour $`x=5`$: $`25`$ contre $`0 \Rightarrow r_{2}\approx 1`$. Donc cloche 1 = $`\{-1,0\}`$, cloche 2 = $`\{4,5\}`$.
> **Étape M.** $`N_1 = 2, N_2 = 2`$.
> - $`\pi_1=\pi_2 = 2/4 = 0{,}5`$.
> - $`\mu_1 = (-1+0)/2 = -0{,}5`$; $`\mu_2 = (4+5)/2 = 4{,}5`$.
> - $`\sigma_1^2 = \tfrac12[(-1+0{,}5)^2+(0+0{,}5)^2] = \tfrac12[0{,}25+0{,}25] = 0{,}25`$; idem $`\sigma_2^2 = 0{,}25`$.
> **Leçon:** une seule passe place déjà les centres sur les vraies moyennes des paquets et resserre les variances. EM converge très vite quand les groupes sont bien séparés.

#### Exercice 4: Singularite du maximum de vraisemblance

Montrer que pour un GMM $`1`$ D avec $`K\ge 2`$, on peut rendre la log-vraisemblance arbitrairement grande. En déduire un remède.

> **Corrigé.** Plaçons le centre de la composante 1 exactement sur un point de donnée: $`\mu_1 = x_1`$. Sa contribution à la densité en $`x_1`$ est $`\frac{\pi_1}{\sqrt{2\pi}\,\sigma_1}\exp(0) = \frac{\pi_1}{\sqrt{2\pi}\,\sigma_1}`$ (avec $`\pi_1>0`$). Les autres composantes ne contribuent que positivement, donc $`p(x_1) \ge \frac{\pi_1}{\sqrt{2\pi}\,\sigma_1}`$. Quand $`\sigma_1 \to 0^+`$, ce minorant $`\to +\infty`$, donc $`p(x_1)\to +\infty`$; comme les autres points gardent une densité strictement positive bornée inférieurement, $`\ell = \sum_n \ln p(x_n) \to +\infty`$. La vraisemblance n'a **pas** de maximum global fini: c'est une singularité. **Remède:** régulariser en ajoutant $`\epsilon\mathbf{I}`$ aux covariances (covariance floor $`\sigma_k^2 \ge \epsilon`$), adopter une vue bayésienne avec un a priori sur $`\boldsymbol{\Sigma}_k`$, ou ré-initialiser toute composante qui s'effondre. C'est exactement le garde-fou `var = np.maximum(var, 1e-6)` du code EM.

#### Exercice 5: L'ELBO est bien un plancher

Sans recopier la démonstration du cours, montrer directement que $`\ell(\boldsymbol{\theta}) \ge \mathcal{F}(q,\boldsymbol{\theta})`$ pour tout $`q`$, en utilisant l'inégalité de Jensen.

> **Corrigé.** Par définition $`\ell(\boldsymbol\theta) = \ln p(\mathbf X\mid\boldsymbol\theta) = \ln \sum_{\mathbf Z} p(\mathbf X,\mathbf Z\mid\boldsymbol\theta)`$. Introduisons $`q`$ (supposé strictement positif là où $`p`$ l'est) par multiplication/division:
> ```math
> \ell = \ln \sum_{\mathbf Z} q(\mathbf Z)\,\frac{p(\mathbf X,\mathbf Z\mid\boldsymbol\theta)}{q(\mathbf Z)} = \ln \mathbb{E}_q\!\left[\frac{p(\mathbf X,\mathbf Z\mid\boldsymbol\theta)}{q(\mathbf Z)}\right].
> ```
> Le logarithme est **concave**, donc par l'inégalité de Jensen ($`\ln \mathbb{E}[Y] \ge \mathbb{E}[\ln Y]`$):
> ```math
> \ell \ge \mathbb{E}_q\!\left[\ln \frac{p(\mathbf X,\mathbf Z\mid\boldsymbol\theta)}{q(\mathbf Z)}\right] = \mathcal{F}(q,\boldsymbol\theta).
> ```
> **Leçon:** l'ELBO surgit naturellement de Jensen. L'égalité a lieu quand le rapport $`p(\mathbf X,\mathbf Z\mid\boldsymbol\theta)/q(\mathbf Z)`$ est constant en $`\mathbf Z`$, c'est-à-dire quand $`q(\mathbf Z)\propto p(\mathbf X,\mathbf Z\mid\boldsymbol\theta)`$, soit (après normalisation) $`q(\mathbf Z) = p(\mathbf Z\mid\mathbf X,\boldsymbol\theta)`$: on retombe sur l'étape E.

#### Exercice 6: Choisir $`K`$ par critere d'information

On hésite entre $`K=2`$ et $`K=3`$. Expliquer pourquoi maximiser la log-vraisemblance ne suffit pas, et proposer un critère.

> **Corrigé.** La log-vraisemblance (au maximum) ne fait que **croître** avec $`K`$: plus de composantes = plus de souplesse = meilleur ajustement des données d'entraînement, jusqu'au surapprentissage (à l'extrême, une gaussienne par point, cf. exercice 4). Maximiser $`\ell`$ choisirait toujours le $`K`$ le plus grand. Il faut **pénaliser la complexité**. Deux critères classiques, où $`P`$ est le nombre de paramètres libres du modèle:
> ```math
> \mathrm{AIC} = -2\ell + 2P, \qquad \mathrm{BIC} = -2\ell + P\ln N.
> ```
> On choisit le $`K`$ qui **minimise** le critère (l'AIC pénalise moins; le BIC est plus parcimonieux car il pénalise davantage chaque paramètre, via $`\ln N`$). Pour un GMM `full` en dimension $`d`$: $`P = \underbrace{(K-1)}_{\pi} + \underbrace{Kd}_{\boldsymbol\mu} + \underbrace{K\frac{d(d+1)}{2}}_{\boldsymbol\Sigma}`$ (les poids ne comptent que pour $`K-1`$ à cause de la contrainte $`\sum_k\pi_k=1`$). Autres approches: validation croisée sur la log-vraisemblance de données de test, ou mélange bayésien à processus de Dirichlet (`BayesianGaussianMixture` de scikit-learn) qui **élague** automatiquement les composantes inutiles en mettant leur poids à zéro.

#### Exercice 7: Implementer EM en dimension quelconque

Écrire une fonction NumPy `em_gmm(X, K)` pour $`\mathbf{X}\in\mathbb{R}^{N\times d}`$ (covariances `full`), renvoyant $`\pi, \boldsymbol\mu, \boldsymbol\Sigma`$ et la courbe de log-vraisemblance, avec garde-fou anti-singularité.

> **Corrigé.**
> ```python
> import numpy as np
>
> def log_gauss(X, mu, Sigma):
>     # log N(x | mu, Sigma) pour chaque ligne de X, calcule de facon stable
>     d = X.shape[1]
>     L = np.linalg.cholesky(Sigma)                   # Sigma = L L^T, L triangulaire inferieure
>     diff = (X - mu).T                               # (d, N)
>     sol = np.linalg.solve(L, diff)                  # resout L sol = diff (L triangulaire)
>     maha = np.sum(sol ** 2, axis=0)                 # distance de Mahalanobis au carre
>     logdet = 2.0 * np.sum(np.log(np.diag(L)))       # log|Sigma| via Cholesky
>     return -0.5 * (d * np.log(2 * np.pi) + logdet + maha)
>
> def em_gmm(X, K, n_iter=200, eps=1e-6, reg=1e-6, seed=0):
>     rng = np.random.default_rng(seed)
>     N, d = X.shape
>     mu = X[rng.choice(N, size=K, replace=False)].astype(float)   # init sur des points
>     Sigma = np.stack([np.cov(X.T) + reg * np.eye(d) for _ in range(K)])
>     pi = np.full(K, 1.0 / K)
>     hist = []
>     for _ in range(n_iter):
>         # ---- Etape E : log-responsabilites stables (log-sum-exp) ----
>         logp = np.stack([np.log(pi[k]) + log_gauss(X, mu[k], Sigma[k])
>                          for k in range(K)], axis=1)              # (N, K)
>         m = logp.max(axis=1, keepdims=True)
>         lse = m[:, 0] + np.log(np.exp(logp - m).sum(axis=1))      # ln p(x_n)
>         hist.append(lse.sum())                                    # log-vraisemblance
>         r = np.exp(logp - lse[:, None])                           # responsabilites
>         # ---- Etape M ----
>         Nk = r.sum(axis=0) + 1e-12
>         pi = Nk / N
>         mu = (r.T @ X) / Nk[:, None]
>         for k in range(K):
>             diff = X - mu[k]
>             Sigma[k] = (r[:, k, None] * diff).T @ diff / Nk[k] + reg * np.eye(d)
>         if len(hist) > 1 and hist[-1] - hist[-2] < eps:
>             break
>     return pi, mu, Sigma, hist
>
> # Demo : deux paquets gaussiens en 2D
> rng = np.random.default_rng(1)
> A = rng.normal([0, 0], 0.5, size=(200, 2))
> B = rng.normal([4, 4], 0.8, size=(300, 2))
> X = np.vstack([A, B])
> pi, mu, Sigma, hist = em_gmm(X, K=2)
> print("pi =", np.round(pi, 3))                 # ~ [0.4 0.6] (a l'ordre des composantes pres)
> print("mu =\n", np.round(mu, 2))               # ~ [[0,0],[4,4]]
> print("log-vrais. croissante :",
>       all(hist[i+1] >= hist[i] - 1e-9 for i in range(len(hist)-1)))
> ```
> **Points clés du corrigé:** (1) on travaille en **log** avec l'astuce *log-sum-exp* (soustraire le max avant l'exponentielle) pour éviter les débordements; (2) on utilise la **factorisation de Cholesky** $`\boldsymbol\Sigma=LL^{\top}`$: comme $`L`$ est triangulaire, `np.linalg.solve(L,...)` calcule la distance de Mahalanobis sans inverser $`\boldsymbol\Sigma`$, et $`\ln\lvert\boldsymbol\Sigma\rvert = 2\sum_i \ln L_{ii}`$, ce qui est plus stable qu'une inversion directe; (3) le terme `reg * I` est le garde-fou anti-singularité de l'exercice 4; (4) on vérifie que la log-vraisemblance est bien croissante, illustration de la monotonie d'EM.

---

[← Réduction de dimension par ACP](10-reduction-de-dimension-acp.md) · [↑ Sommaire](../README.md#table-des-matières) · [Classification par machines à vecteurs de support →](12-classification-svm.md)
