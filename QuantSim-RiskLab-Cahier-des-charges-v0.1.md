# QuantSim RiskLab — Cahier des charges

**Plateforme Java client-serveur de simulation stochastique et d’analyse du risque financier**  
Module : Programmation Objet Avancée en Java  
Version : 0.1 — proposition de spécification, 23 septembre 2026  
Auteurs et échéance : **À COMPLÉTER**

Ce document définit le comportement attendu du prototype et les conditions permettant de vérifier sa conformité. Il ne constitue pas un bilan d’implémentation. Aucun code ni résultat de test n’a été examiné à ce stade : toutes les exigences sont **prévues, non vérifiées**.

## 1. Contexte

La valeur d’un portefeuille d’actions dépend de l’évolution conjointe des prix de ses actifs. La prise en compte de leurs volatilités et de leurs dépendances permet d’étudier différents scénarios de variation de la valeur du portefeuille. La simulation Monte Carlo fournit une distribution de résultats possibles, conditionnelle au modèle et aux paramètres retenus.

QuantSim RiskLab vise à réunir la configuration d’un portefeuille, la simulation d’un marché multivarié, la réévaluation du portefeuille et le calcul d’indicateurs de risque dans une application pédagogique. Les paramètres seront saisis par l’utilisateur ; le prototype ne prétendra ni prévoir les marchés ni fournir une mesure exhaustive du risque réel.

Le projet est réalisé en binôme. L’étudiant 1 porte principalement la modélisation, la corrélation et la simulation ; l’étudiant 2 porte principalement la valorisation et les mesures de risque. L’architecture, les objets partagés, le réseau, l’interface et l’intégration sont des responsabilités communes.

Dans le cadre du module, la séparation entre un client Swing et un serveur de calcul permettra de mobiliser la programmation objet, les interfaces, les exceptions, les collections, les sockets TCP et la concurrence.

## 2. Problématique

Comment concevoir une application Java modulaire permettant à un utilisateur de simuler l’évolution corrélée de plusieurs actions et d’en déduire des indicateurs de risque de portefeuille, tout en garantissant la cohérence des calculs, la reproductibilité des expériences et la réactivité de l’interface ?

La difficulté porte notamment sur la cohérence entre les actifs et la matrice de corrélation, les conventions de calcul du risque, l’exécution parallèle et la circulation des données entre le client et le serveur.

## 3. Objectif général

Développer un prototype Java client-serveur permettant de configurer un portefeuille d’actions, de générer des scénarios de marché par Monte Carlo et de consulter la distribution des profits et pertes ainsi que les VaR et Expected Shortfall à 95 % et 99 % dans une interface Swing.

## 4. Objectifs spécifiques

1. Représenter les instruments, positions et portefeuilles avec des objets encapsulés et des responsabilités distinctes.
2. Simuler un mouvement brownien géométrique multivarié avec paramètres constants et corrélations spécifiées.
3. Réévaluer le portefeuille pour chaque scénario et construire sa distribution de P&L.
4. Calculer des indicateurs de risque selon des conventions explicites et testables.
5. Répartir les scénarios entre plusieurs tâches à l’aide d’un `ExecutorService`.
6. Transmettre les demandes et résultats par sockets TCP.
7. Maintenir une interface Swing utilisable pendant les échanges et calculs.
8. Vérifier les résultats déterministes, les propriétés statistiques et le fonctionnement intégré.

## 5. Périmètre du MVP et conventions

### 5.1 Parcours nominal

L’utilisateur crée un portefeuille, renseigne les paramètres des actifs et leur corrélation, configure Monte Carlo puis lance le calcul. Le client transmet une demande au serveur. Celui-ci valide les entrées, génère les scénarios, calcule les valeurs terminales et les mesures de risque, puis renvoie les résultats au client pour affichage.

Le MVP couvre une session de calcul à la fois par client. Le traitement simultané de plusieurs clients reste une extension ; l’utilisation de TCP ne suffit pas à garantir cette capacité.

### 5.2 Hypothèses proposées pour cette version

Ces choix précisent le périmètre initial ; ils ne décrivent pas une implémentation déjà réalisée.

| Élément | Convention du MVP |
|---|---|
| Instruments | Actions uniquement ; une position par symbole unique. |
| Positions | Quantités strictement positives, portefeuille non vide lors du lancement ; ventes à découvert hors MVP. |
| Monnaie | Une devise commune, affichée ; aucune conversion de change. |
| Détention | Quantités constantes jusqu’à l’horizon ; aucun rééquilibrage. |
| Prix et volatilités | Prix initiaux strictement positifs ; volatilités positives ou nulles ; valeurs numériques finies. |
| Drift | Paramètre annuel réel fini saisi par l’utilisateur, employé sous une mesure physique ; il ne s’agit pas d’un taux de valorisation risque-neutre. |
| Temps | Horizon T strictement positif exprimé en années ; K pas entiers strictement positifs ; Δt = T/K. |
| Simplifications | Pas de dividendes, de coûts de transaction, de financement ni d’apports ou retraits. |
| Corrélations | Corrélations constantes des incréments browniens, et non corrélations imposées aux prix terminaux. |
| Données | Saisie manuelle ; aucune calibration automatique ni connexion à des cours réels. |
| Environnement | Démonstration locale, client et serveur dans des processus Java distincts. |

### 5.3 Simulation

Le modèle retenu est :

\[
dS_i(t)=\mu_i S_i(t)dt+\sigma_i S_i(t)dW_i(t),
\qquad d\langle W_i,W_j\rangle_t=\rho_{ij}dt.
\]

La méthode de référence proposée pour le MVP est la transition exacte du GBM sur chaque pas :

\[
S_i(t+\Delta t)=S_i(t)\exp\left[(\mu_i-\sigma_i^2/2)\Delta t+\sigma_i\sqrt{\Delta t}Z_i\right],
\qquad Z\sim\mathcal N(0,R).
\]

Les vecteurs gaussiens sont indépendants entre pas et entre scénarios. Pour la construction par Cholesky, R = CCᵀ et Z = Cε avec ε de composantes normales standard indépendantes.

Le MVP exige une matrice symétrique, de diagonale unitaire, à coefficients dans [−1,1], et **strictement définie positive**. Une matrice de corrélation mathématiquement valide peut être seulement semi-définie positive ; ces cas singuliers sont exclus de cette première version pour simplifier Cholesky. Les tolérances numériques seront documentées et les matrices rejetées ne seront pas corrigées silencieusement.

La comparaison avec Euler–Maruyama est classée SHOULD. Il s’agit d’une autre méthode de simulation du même modèle GBM, non d’un autre modèle stochastique. Elle peut produire des prix négatifs pour certains pas : cette limite devra être explicitement traitée si la comparaison est réalisée.

### 5.4 Valorisation et risque

Pour un portefeuille de d actions et N scénarios :

\[
V_0=\sum_{i=1}^{d}q_iS_i(0),\qquad
V_T^{(m)}=\sum_{i=1}^{d}q_iS_i^{(m)}(T),\qquad
PnL_m=V_T^{(m)}-V_0,\qquad L_m=-PnL_m.
\]

Les mesures de risque sont calculées sur les pertes L, en unités monétaires, à l’horizon T. Une perte positive correspond à une diminution de valeur. Les résultats ne sont pas tronqués artificiellement à zéro.

Pour éviter les ambiguïtés sur les petits échantillons, la convention empirique suivante est proposée. Soient les pertes triées L₍₁₎ ≤ … ≤ L₍N₎, α ∈ {0,95 ; 0,99} et k = ⌈Nα⌉ :

\[
\operatorname{VaR}_\alpha=L_{(k)},\qquad
\operatorname{ES}_\alpha=
\frac{(k-N\alpha)L_{(k)}+\sum_{j=k+1}^{N}L_{(j)}}{N(1-\alpha)}.
\]

Cette ES correspond à la moyenne de la fraction supérieure 1−α de la distribution empirique, avec pondération de l’observation frontière lorsque nécessaire. Elle évite l’ambiguïté d’une simple moyenne des pertes supérieures ou égales à la VaR en présence d’ex æquo.

La classe envisagée `HistoricalMonteCarloVaR` reste dans l’architecture fournie. Sa documentation devra cependant préciser qu’elle calcule un quantile empirique de pertes **simulées** ; le prototype ne réalise pas une simulation historique à partir de rendements observés. Un éventuel renommage sera discuté lors du diagramme de classes.

## 6. Acteurs

| Acteur | Rôle |
|---|---|
| Utilisateur | Saisir le portefeuille, configurer le modèle et la simulation, lancer le calcul, consulter les résultats et corriger les entrées invalides. |

Le serveur de calcul est un composant interne de QuantSim RiskLab : il n’est pas un acteur externe dans le diagramme de cas d’utilisation du système complet. Les deux étudiants sont des contributeurs au développement ; le professeur est une partie prenante de l’évaluation.

## 7. Besoins fonctionnels et critères d’acceptation

**MUST** : indispensable au MVP. **SHOULD** : objectif complémentaire après stabilisation du MVP. **COULD** : extension facultative. Les critères ci-dessous sont des conditions attendues, non des résultats de tests obtenus.

| ID | Besoin | Priorité | Critère d’acceptation |
|---|---|---|---|
| BF01 | Créer et modifier un portefeuille ; ajouter et supprimer des positions. | MUST | Un portefeuille de deux actions peut être créé ; une suppression met à jour son contenu ; le lancement sur portefeuille vide est refusé. |
| BF02 | Saisir pour chaque action le symbole, S₀, q, μ et σ. | MUST | Les valeurs acceptées sont conservées dans la demande ; prix non positif, quantité non positive, volatilité négative, symbole vide ou dupliqué sont rejetés avec un message précis. |
| BF03 | Configurer la matrice de corrélation. | MUST | Une matrice identité est proposée ; lignes et colonnes portent les symboles ; une modification du portefeuille impose une matrice cohérente avant lancement. |
| BF04 | Valider la matrice avant simulation. | MUST | Une matrice valide est acceptée ; dimensions incorrectes, asymétrie, diagonale incorrecte, coefficients hors bornes et échec de Cholesky sont signalés. |
| BF05 | Saisir T, K, N, seed et nombre de threads. | MUST | Les paramètres valides sont transmis sans modification implicite ; entiers non positifs, valeurs non finies et dépassements des limites documentées sont refusés. |
| BF06 | Lancer une simulation depuis Swing via TCP. | MUST | Un client et un serveur exécutés séparément échangent une demande valide et un résultat associé ; un second lancement du même client est bloqué pendant le calcul. |
| BF07 | Produire N scénarios multivariés selon le GBM retenu. | MUST | Chaque scénario contient un prix terminal par actif dans un ordre identifié ; le cas σ = 0 reproduit S₀ exp(μT) à la tolérance numérique fixée. |
| BF08 | Valoriser le portefeuille et construire les P&L. | MUST | Dix actions à 100 donnent V₀ = 1 000 ; à un prix terminal de 110, Vₜ = 1 100 et P&L = 100. |
| BF09 | Calculer VaR et ES à 95 % et 99 %. | MUST | Les quatre valeurs coïncident avec les résultats de référence de la section 9 selon la convention retenue. |
| BF10 | Présenter les résultats dans Swing. | MUST | V₀, les quatre mesures de risque et un histogramme des P&L sont affichés avec devise, horizon et nombre de scénarios ; les effectifs de l’histogramme totalisent N. |
| BF11 | Afficher l’état de la demande et les erreurs. | MUST | L’interface distingue attente, calcul, succès et échec ; une indisponibilité du serveur ou une entrée invalide produit un message exploitable et permet un nouvel essai. |
| BF12 | Répartir les simulations sur le nombre de workers demandé. | MUST | Une exécution à plusieurs workers traite chaque indice de scénario exactement une fois ; le total produit reste égal à N. |
| BF13 | Comparer transition exacte et Euler–Maruyama. | SHOULD | La méthode est identifiable dans les résultats ; la comparaison utilise des paramètres communs et documente l’effet du pas ainsi que les prix négatifs éventuels. |
| BF14 | Appliquer un stress test déterministe de prix. | SHOULD | Un choc uniforme de −20 % sur un portefeuille long de valeur 1 000 donne une valeur stressée de 800 et un P&L de −200 ; ce résultat est séparé des VaR/ES. |
| BF15 | Traiter plusieurs clients simultanément. | COULD | Deux demandes concurrentes reçoivent leurs propres résultats sans mélange de données et dans les limites de ressources définies. |

## 8. Besoins non fonctionnels et critères d’acceptation

| ID | Exigence | Priorité | Critère d’acceptation |
|---|---|---|---|
| BNF01 | Respecter la stack Java du module. | MUST | Client Swing, `Socket`/`ServerSocket` et calcul Java ; version du JDK et dépendances consignées ; compilation et lancement documentés depuis un dépôt récupéré à neuf. |
| BNF02 | Séparer présentation, communication, simulation, finance et risque. | MUST | Les packages proposés sont conservés ; les calculs métier sont testables sans Swing ni socket ; aucune dépendance métier vers `gui`. |
| BNF03 | Employer les concepts objet de manière justifiée. | MUST | Interfaces de simulation et de mesure de risque effectivement utilisées ; encapsulation des données ; relations d’héritage cohérentes avec le domaine ; absence d’héritage ajouté uniquement pour multiplier les classes. |
| BNF04 | Garantir la reproductibilité dans une configuration fixée. | MUST | Même entrée, seed, version logicielle, environnement Java et configuration de threads : mêmes P&L ordonnés lors de deux exécutions. La seed et la configuration accompagnent les résultats. |
| BNF05 | Garantir aussi la reproductibilité lorsque le nombre de workers change. | SHOULD | À paramètres identiques, les P&L ordonnés sont identiques avec 1, 2 et 4 workers ; cette propriété n’est annoncée qu’après vérification. |
| BNF06 | Préserver la réactivité de Swing. | MUST | Réseau et calcul hors EDT ; mises à jour visuelles sur EDT ; la fenêtre reste déplaçable et se redessine pendant une simulation. |
| BNF07 | Borner la consommation de ressources. | MUST | Plafonds documentés sur actifs, scénarios, pas, workers et taille des messages ; refus explicite des demandes dépassant ces plafonds ; absence de thread créé par scénario. Valeurs : À COMPLÉTER. |
| BNF08 | Mesurer les performances sans promettre un gain non vérifié. | MUST | Mesures de temps moteur et de temps total sur une configuration décrite, comparées pour 1, 2 et 4 workers avec au moins trois répétitions ; résultats et mémoire si mesurée : À COMPLÉTER. |
| BNF09 | Valider les données côté serveur. | MUST | Une demande invalide envoyée sans passer par Swing est rejetée ; NaN, infinis et résultats numériques non finis ne sont pas présentés comme un succès. |
| BNF10 | Gérer les erreurs réseau et libérer les ressources. | MUST | Déconnexion et serveur indisponible sont testés ; absence d’attente indéfinie grâce à des délais documentés ; sockets et pools sont fermés lors de l’arrêt. |
| BNF11 | Assurer la sûreté de l’exécution concurrente. | MUST | Aucune collection de résultats ni générateur mutable n’est partagé sans stratégie explicite ; absence de scénarios manquants ou dupliqués lors d’exécutions répétées. |
| BNF12 | Rendre les résultats interprétables. | MUST | Unités, sens du P&L, horizon et niveaux de confiance apparaissent dans l’interface ; un avertissement indique lorsqu’un faible nombre de scénarios rend la queue à 99 % peu résolue. Seuil d’avertissement : À COMPLÉTER. |
| BNF13 | Limiter l’exposition du prototype. | MUST | Écoute sur l’interface locale par défaut ; entrées et tailles des messages contrôlées ; si une désérialisation Java est retenue, types autorisés et taille/profondeur bornés. L’accès réseau distant nécessite une configuration explicite. |
| BNF14 | Documenter et vérifier les contrats métier. | MUST | Javadoc des interfaces publiques et conventions de calcul ; tests exécutables des règles métier ; résultats réels associés aux exigences avant toute déclaration de validation. |

L’accélération par parallélisation est un objectif à évaluer, et non une garantie : les coûts de coordination et les ressources disponibles peuvent limiter le gain. Aucun objectif arbitraire de temps de réponse n’est fixé sans machine et charge de référence.

## 9. Recette initiale et résultats attendus

Les essais déterministes ci-dessous peuvent être définis avant l’implémentation. Leur statut initial est **non exécuté**.

| Test | Données et opération | Résultat attendu | Exigences |
|---|---|---|---|
| T01 | Une position : q = 10, S₀ = 100. | V₀ = 1 000. | BF01, BF08 |
| T02 | Même position avec Sₜ = 110. | Vₜ = 1 100 ; P&L = 100 ; perte = −100. | BF08 |
| T03 | Échantillon de 100 pertes : 0, 1, …, 99. | VaR₉₅ = 94 ; ES₉₅ = 97 ; VaR₉₉ = 98 ; ES₉₉ = 99. | BF09 |
| T04 | Dix pertes : 0, 1, …, 9. | VaR₉₅ = ES₉₅ = VaR₉₉ = ES₉₉ = 9 ; illustre la faible résolution de queue. | BF09, BNF12 |
| T05 | Cent pertes toutes égales à 5. | Les quatre mesures valent 5, y compris en présence d’ex æquo. | BF09 |
| T06 | S₀ = 100, μ = 0, σ = 0 ; T et K valides. | Tous les prix restent à 100 ; P&L et quatre mesures de risque nuls. | BF07–BF09 |
| T07 | Matrice [[1, 0,5], [0,5, 1]]. | Validation et factorisation réussies. | BF04 |
| T08 | Matrices asymétrique, de diagonale différente de 1, avec coefficient 1,2, puis [[1,1],[1,1]]. | Rejet motivé dans chaque cas ; le dernier cas est singulier et hors périmètre Cholesky du MVP. | BF04 |
| T09 | Réexécuter une demande avec la même configuration et la même seed. | P&L ordonnés identiques. | BNF04 |
| T10 | Répartir 101 scénarios entre 4 workers. | Exactement 101 scénarios, aucun indice absent ou dupliqué. | BF12, BNF11 |
| T11 | Exécuter le parcours Swing → TCP → serveur → résultat. | Résultats associés à la demande ; unités et paramètres cohérents ; fenêtre réactive. | BF06, BF10, BNF06 |
| T12 | Serveur indisponible, puis connexion interrompue. | Message d’échec, retour à un état permettant un nouvel essai, aucune attente indéfinie. | BF11, BNF10 |

Pour les tests numériques, les tolérances absolues et relatives seront fixées dans les tests et documentées avant validation. Les tests statistiques devront comparer les moments du GBM et les corrélations des innovations simulées à leurs valeurs théoriques avec des tolérances tenant compte de N ; une égalité exacte n’est pas attendue. Paramètres, méthode statistique et seuils : **À COMPLÉTER**.

La recette du MVP sera acquise lorsque toutes les exigences MUST seront vérifiées sur la version identifiée du code, avec preuves de tests et anomalies résiduelles documentées. Les fonctions SHOULD et COULD ne conditionnent pas cette recette.

## 10. Fonctionnalités explicitement hors périmètre

- Connexion à un courtier, passage d’ordres et trading réel.
- Récupération de données de marché en temps réel et calibration automatique.
- Optimisation du portefeuille, rééquilibrage dynamique et stratégies de trading.
- Options, obligations, produits dérivés et modèles de taux dans le MVP.
- Portefeuilles multidevises, ventes à découvert, financement et coûts de transaction.
- Simulation historique, backtesting réglementaire et certification des mesures de risque.
- Base de données, comptes utilisateurs, authentification et exploitation sur Internet.
- Architecture cloud, microservices, Spring Boot et interface Web.
- Prise en charge générale des matrices de corrélation singulières.
- Conservation exhaustive et affichage de toutes les trajectoires : seules les données nécessaires aux résultats du MVP sont requises.

Euler–Maruyama, les stress tests et la concurrence entre plusieurs clients constituent les extensions explicitement identifiées en section 7. Ils ne doivent pas retarder la validation du parcours minimal.

## 11. Éléments connus et éléments à compléter

| Déjà défini dans le cadrage | À compléter avec les décisions techniques ou les preuves réelles |
|---|---|
| Projet Java en binôme ; client Swing et serveur TCP. | Noms, échéance, version Java, outil de compilation et dépendances autorisées. |
| Packages `model`, `simulation`, `finance`, `risk`, `network`, `gui`, `exception`. | Signatures, contrats des objets échangés, format et délimitation des messages TCP. |
| GBM corrélé, portefeuille d’actions, P&L, VaR et ES. | Confirmation des conventions proposées : quantile, ES, devise et restriction aux positions longues. |
| Répartition principale simulation / finance-risque. | Responsables précis des tâches communes et journal de bord. |
| Besoin de multithreading et de reproductibilité. | Stratégie des flux aléatoires, plafonds de ressources et délais réseau. |
| Critères de recette du présent document. | Commit testé, résultats des tests, tolérances, machine de référence et mesures de performance. |

L’architecture initiale est conservée. Les interfaces détaillées et les diagrammes UML seront établis dans un livrable ultérieur, à partir de ces exigences. L’état de l’art sera traité séparément après stabilisation du présent cahier des charges.
