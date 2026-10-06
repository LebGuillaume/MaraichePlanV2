# DEVLOG — MaraîchPlan

Journal des décisions, blocages et erreurs. Une entrée datée par session.

---

## 2026-10-01 — Jalon 1 : modélisation (sur papier)

### Décisions

- **Culture est une entité à part entière, pas une table de jointure.**
  Une culture = un légume qui occupe une planche pendant un intervalle de dates.
  Pourquoi : le couple (légume, planche) se répète dans le temps (des carottes
  sur la planche 3 en 2026 puis en 2028). Il ne peut donc pas servir de clé
  primaire.
  Écarté : PK composite `(legumeId, plancheId)`, et le nom « Planification »
  (le cadre et les jalons parlent de « cultures »).

- **Interdire le chevauchement avec une contrainte d'exclusion PostgreSQL.**
  Deux cultures ne peuvent pas être sur la même planche avec des intervalles
  qui se chevauchent.

- **Règle d'occupation : une culture occupe la planche à partir de son
  implantation, par plantation ou par semis direct.**
  Le semis en pépinière n'occupe pas la planche : c'est une tâche, pas une
  partie de l'intervalle.

- **Une culture qui se termine le jour J et une culture qui commence le jour J
  ne sont pas en conflit** (arrachage le matin, plantation l'après-midi).

- **Identifiants : UUID, pas d'entier auto-incrémenté.**
  Pourquoi : un compteur qui se suit permet d'énumérer les ressources et révèle
  la volumétrie de la plateforme (ex. id 1000 puis 1250 une semaine plus tard).
  À retenir : la vraie protection est le filtre `exploitationId` dans chaque
  requête. L'UUID n'est qu'une défense en profondeur.
  À confirmer : v4 (100 % aléatoire, fragmente l'index B-tree) ou v7
  (horodaté, meilleur pour l'index, révèle la date de création).

- **Intervalle d'occupation en `daterange` avec bornes `[)`, en jours.**
  Pourquoi : avec `[]`, les deux bornes sont incluses. Carottes `[1 mars, 15 juin]`
  et poireaux `[15 juin, …]` partagent le 15 juin : la contrainte d'exclusion
  rejetterait la plantation, ce qui contredit la règle du jour J. Avec `[)`, le
  15 juin n'appartient plus aux carottes, il n'y a plus de jour commun.
  Écarté : ajouter une heure (`tstzrange`). Le maraîcher ne connaît pas l'heure,
  il faudrait la saisir au hasard, et avec `[]` deux horodatages égaux se
  chevauchent quand même. Le vrai problème venait des bornes, pas de la précision.

- **La date de fin stockée = premier jour où la planche est libre.**
  C'est aussi la date que le maraîcher saisit et voit : « planche libérée le 15 ».
  Conséquence : aucune conversion ±1 jour n'est nécessaire, ni dans `lib/db` ni
  ailleurs. (Une conversion dans `lib/db` avait été envisagée tant que la date
  saisie était supposée être le dernier jour d'occupation.)
  Risque : un utilisateur qui comprend « fin » comme « dernier jour de récolte »
  saisira le 14. Le libellé du champ doit rendre ce sens impossible à mal lire.

### Appris

- **IDOR** (Insecure Direct Object Reference) : un `findUnique` par id seul
  permet de lire la culture d'une autre exploitation en modifiant l'id dans
  l'URL. D'où la règle : toujours filtrer par `id` ET par `exploitationId`.
- **Deux intervalles se chevauchent dès qu'ils ont au moins un point en commun.**
  Le sens des bornes (incluse / exclue) décide donc à lui seul si une passation
  le même jour est un conflit.
- **PostgreSQL convertit toujours un `daterange` en `[)`.**
  `'[2026-03-01,2026-06-14]'` est stocké `[2026-03-01,2026-06-15)`. À vérifier
  soi-même quand la base tournera.
- **Vérifier les chevauchements uniquement dans le code ne suffit pas** : deux
  requêtes simultanées peuvent chacune vérifier, ne rien voir, puis insérer.
  Seule une contrainte en base est fiable face à ces requêtes concurrentes.

### Questions ouvertes

- [ ] 403 ou 404 quand la ressource appartient à une autre exploitation ?
- [x] Bornes de l'intervalle : `[)`, date de fin = premier jour libre, affichée
      telle quelle (voir Décisions).
- [x] Nom du champ / de la colonne de fin, pour éviter la confusion avec le
      « dernier jour de récolte » (voir entrée du 2026-10-05).
- [x] Condition de chevauchement de `[a, b)` et `[c, d)` en TypeScript (domaine,
      jalon 5), qui doit donner le même verdict que PostgreSQL (voir entrée du
      2026-10-05).
- [x] Source de vérité de la date d'implantation : la tâche « plantation » ou
      le début de l'intervalle de la culture ? (risque de désynchronisation)
      → `Culture.datePlancheOccupee` seule (voir entrée du 2026-10-06).
- [ ] Lien Tâche → Culture : comment rattacher le semis en pépinière à sa
      culture ? Ce lien est-il obligatoire (désherbage sur une planche vide) ?
      → Sens tranché le 2026-10-06 (`Tache.cultureId`), obligatoire ou non :
      encore ouvert.
- [ ] Parcelle–Planche : N-N sur le schéma papier. Une planche peut-elle
      vraiment appartenir à deux parcelles ?
- [ ] Légume / Espèce : sens de la relation, et où placer la famille botanique
      (nécessaire pour les rotations) ?
- [ ] Catalogue de légumes : global, par exploitation, ou les deux ? (le schéma
      papier le relie à User en N-N)
- [ ] Tâche reliée à la fois à l'exploitation et aux planches : qu'est-ce qui
      empêche une incohérence entre exploitations ?

## 2026-10-05 — Jalon 1 : modélisation (sur papier)

### Décisions

- **Bornes de la culture : `datePlancheOccupee` (début, inclus) et
  `datePlancheDisponible` (fin, exclue).**
  Pourquoi : les noms décrivent l'état de la planche, pas la vie de la culture.
  - `dateFin` / `dateAvailable` écartés : sur `Culture`, « culture disponible »
    se lit comme « récoltable », c'est la confusion qu'on voulait éviter.
  - `datePlantation` écarté : faux pour un semis direct (carottes).
  - « Occupée » plutôt que « Indisponible » : c'est le vocabulaire de la
    contrainte d'exclusion (occupation, chevauchement).
  Conventions retenues pour tout le schéma : noms métier en français, champs en
  camelCase, sans accents (ASCII), sans abréviations.

- **Condition de chevauchement (domaine, jalon 5) : `[a, b)` et `[c, d)` se
  chevauchent si `a < d et c < b`.**
  Même verdict que l'opérateur `&&` de PostgreSQL sur des `daterange` `[)`.
  Cas de référence : `[1 mars, 15 juin)` / `[15 juin, 1 sept)` → pas de conflit ;
  `[1 mars, 15 juin)` / `[14 juin, 1 sept)` → conflit.

- **CHECK `datePlancheOccupee < datePlancheDisponible` sur `Culture`.**
  Pourquoi : une culture de zéro jour ou à l'envers n'a pas de sens. `<` strict
  (pas `<=`) : une occupation de zéro jour est une erreur de saisie.
  UNIQUE écarté : il compare entre lignes (deux plantations le même jour sur des
  planches différentes seraient refusées), alors que l'erreur est dans une seule
  ligne. Prisma ne gère pas les CHECK : migration SQL personnalisée.

- **Stockage : deux colonnes `date` (`@db.Date`), pas de colonne `daterange`.**
  Pourquoi : `daterange` n'est pas supporté par Prisma, il deviendrait
  `Unsupported("daterange")`. Un champ `Unsupported` obligatoire retire
  `create`/`update` du client Prisma pour ce modèle : toutes les écritures de
  `Culture` passeraient en SQL brut. Avec deux `date`, Prisma gère le CRUD et les
  filtres du tableau de bord sont de simples comparaisons.
  La plage est construite dans la contrainte d'exclusion :
  `daterange(datePlancheOccupee, datePlancheDisponible, '[)')`.
  Le CHECK reste nécessaire : `daterange(d, d)` donne une plage vide, qui ne
  chevauche rien, donc l'exclusion la laisserait passer sans erreur.
  Coût accepté : les requêtes en `$queryRaw` qui utilisent `&&` doivent
  reconstruire la plage de la même façon.
  (Réponse donnée par l'IA à la demande, pas trouvée seul.)

### Appris

- **Raisonner par l'inverse.** « Pas de chevauchement » est plus simple à écrire :
  `b <= c ou a >= d` (l'un finit avant que l'autre commence). On inverse ensuite.
- **Inverser une comparaison** : le contraire de `<=` est `>` (pas `>=`). Le cas
  d'égalité ne peut appartenir qu'à un seul des deux côtés.
- **De Morgan** : NON (X ou Y) = (NON X) et (NON Y). Erreur commise : garder le
  « ou », ce qui déclarait un conflit dès que `a < d`, donc presque toujours.
  Le test du 15 juin a détecté le bug.
- **Portée des contraintes** : FK (vers une autre table), UNIQUE et EXCLUSION
  (entre lignes), CHECK (une ligne seule).

## 2026-10-06 — Jalon 1 : modélisation (sur papier)

### Décisions

- **Pas de tâche « plantation » : la date d'implantation n'existe que dans
  `Culture.datePlancheOccupee`.**
  Pourquoi : un fait stocké une seule fois ne peut pas diverger. C'est cette
  colonne que la contrainte d'exclusion protège, c'est donc elle qui fait foi.
  Cas de référence : poireaux prévus le 15 mai, plantés le 22. On corrige une
  seule date, rien à synchroniser.
  Écartés :
  - (i) une tâche plantation avec sa propre date, égale à `datePlancheOccupee` :
    deux copies du même fait. Aucune contrainte en base ne peut garantir
    l'égalité (un CHECK ne voit qu'une ligne, et les dates sont dans deux
    tables). Une panne entre les deux `update` laisse la base incohérente.
  - (ii) une tâche plantation sans date (`NULL`), qui lit la date dans la
    culture : la ligne n'apprend rien que la culture ne dise déjà. Elle
    imposerait en plus un CHECK (« plantation ⇒ date nulle ») et un index unique
    partiel (une seule plantation par culture), pour une ligne inutile.
  - Synchronisation par service, transaction ou trigger : elle oblige à
    protéger chaque chemin d'écriture (écran culture, écran tâches…), et un
    futur chemin peut l'oublier.

- **Mode d'implantation : colonne `modeImplantation` sur `Culture`, enum
  obligatoire (plantation, semis direct).**
  Pourquoi une colonne et pas une tâche : une culture a exactement un mode
  d'implantation, c'est un attribut. Une tâche est un événement qui peut se
  produire 0 ou N fois (désherbages, récoltes).
  Pourquoi un enum : un texte libre laisserait passer `"Plantation"`,
  `"plantation "`, `"plant."`, et le tableau de bord les compterait à part.
  Pourquoi obligatoire : toute culture commence par l'une ou l'autre.
  (Justifications de l'enum et du caractère obligatoire formulées par l'IA.)
  À vérifier plus tard : ajouter une valeur (ex. bouturage) demandera une
  migration. Quel SQL sera généré ? À relire dans le `migration.sql`.

- **Relation Culture → Tâche : 1-N, clé étrangère `Tache.cultureId`.**
  Pourquoi : une colonne ne contient qu'une valeur, donc `Culture.tacheId` ne
  pourrait désigner qu'une seule tâche. Une table intermédiaire ne sert que pour
  le N-N, or une tâche appartient à une seule culture.
  À prévoir : PostgreSQL n'indexe pas une FK automatiquement. Il faudra un index
  sur `Tache.cultureId` (requête fréquente : les tâches d'une culture).
  Encore ouvert : `cultureId` obligatoire ou non (désherbage sur une planche
  vide, semis en pépinière sans planche).

- **Une tâche peut être prévue ou réalisée : deux colonnes `datePrevue` et
  `dateRealisation`, toutes deux facultatives.**
  Pourquoi : le tableau de bord « récoltes à 30 jours » doit lire des récoltes
  qui n'ont pas encore eu lieu. La ligne de tâche existe donc à l'avance.
  Une seule colonne `date` perdait une information : récolte prévue le 1er oct,
  faite le 5. Soit on perd la date réelle, soit on perd le retard.
  `dateRealisation` à `NULL` = tâche pas encore faite.
  `datePrevue` facultative : une tâche imprévue (désherbage fait en voyant les
  adventices lever) s'enregistre après coup, sans date prévue.
  Écarté : un booléen `realisee`. Il se déduit de `dateRealisation IS NOT NULL`.
  Le garder ferait deux informations qui peuvent se contredire
  (`realisee = false` avec une date de réalisation remplie).

- **CHECK `"datePrevue" IS NOT NULL OR "dateRealisation" IS NOT NULL` sur
  `Tache`.**
  Pourquoi : une tâche ni prévue ni faite ne représente rien. La règle ne porte
  que sur une ligne, c'est donc un CHECK. `OR` et pas `AND` : il faut pouvoir
  enregistrer une tâche prévue pas encore faite, et une tâche imprévue faite.
  Prisma ne gère pas les CHECK : migration SQL personnalisée.
  (Réponse donnée par l'IA à la demande, pas trouvée seul.)

- **Pas de CHECK `dateRealisation >= datePrevue`.**
  Pourquoi : une récolte faite plus tôt que prévu (poireaux prêts en avance) est
  un cas métier normal. Une contrainte interdit l'impossible, pas l'inhabituel.

- **La date de première récolte n'est pas stockée : elle se calcule.**
  `MIN(dateRealisation)` sur les tâches de type récolte de la culture.
  Pourquoi : la stocker dans `Culture` créerait une copie qui diverge dès qu'on
  supprime ou corrige une récolte (même raisonnement que pour la plantation).

- **Tableau de bord : deux blocs de récoltes, sans recouvrement.**
  - « À venir » : `datePrevue BETWEEN aujourd'hui AND aujourd'hui + 30`
    et `dateRealisation IS NULL`.
  - « En retard » : `datePrevue < aujourd'hui` et `dateRealisation IS NULL`.
  `dateRealisation IS NULL` exclut une récolte faite en avance (prévue le
  20 oct, faite le 3). Une récolte prévue aujourd'hui n'apparaît que dans
  « à venir » : `BETWEEN` inclut ses bornes, `<` exclut aujourd'hui.

- **Filtrer « récoltes à 30 jours » par exploitation : chemin
  `Tache.cultureId` → `Culture.plancheId` → `Planche.parcelleId` →
  `Parcelle.exploitationId`.**
  Sans ce filtre, chaque maraîcher voit les récoltes de tous les autres.
  En Prisma : `where: { culture: { planche: { parcelle: { exploitationId } } } }`.
  Suppose Parcelle → Planche en 1-N (encore N-N sur le papier).
  (Chemin et requête donnés par l'IA à la demande.)

### Appris

- **Normalisation (source unique de vérité)** : stocker un fait à un seul
  endroit rend la divergence impossible par construction, au lieu de
  l'interdire par une règle qu'il faut maintenir.
- **Attribut ou événement** : ce qui existe exactement une fois par entité est
  une colonne de cette entité. Ce qui peut arriver 0 ou N fois est une ligne
  d'une autre table.
- **Dans une 1-N, la FK va du côté N.** À réutiliser pour Exploitation →
  Parcelle et Parcelle → Planche.
- **Une FK pointe vers une ligne (son id), pas vers une valeur.** On lit la
  date de la culture en suivant la FK (jointure), on ne la recopie pas.
- Erreur commise : passer de (iii) à (i), puis à (ii), par élimination plutôt
  que par argument. Le tableau d'exemple (lignes C1, T1, T2) a débloqué le
  raisonnement.
- **Donnée calculée plutôt que stockée** : ce qui se déduit d'autres lignes
  (`MIN`, `COUNT`, `IS NOT NULL`) ne se stocke pas, sinon c'est une copie à
  synchroniser.
- **Prévu / réalisé** : une seule date mélange le plan et le fait. Deux dates
  permettent de mesurer l'écart (retard, avance).
- **`NULL` en SQL** : `x <> NULL` vaut `NULL`, jamais vrai. Un CHECK qui vaut
  `NULL` est considéré comme satisfait : il faut écrire `IS NOT NULL`, sinon la
  contrainte laisse tout passer sans erreur.
- **CHECK ou `WHERE`** : un CHECK refuse une ligne invalide à l'écriture. Une
  condition de `WHERE` choisit, à la lecture, les lignes valides à afficher.
  Une récolte déjà faite est valide : on l'exclut d'un écran, on ne l'interdit
  pas.
- **Une requête sans filtre d'exploitation est une fuite de données.** Quand la
  table ne porte pas `exploitationId`, on remonte les FK jusqu'à elle.
- **Une FK facultative casse le chemin de filtrage** : avec `cultureId` à
  `NULL`, la tâche disparaît du résultat (`JOIN`), et même avec un `LEFT JOIN`
  (`NULL = $1` n'est jamais vrai). Ce n'est pas une fuite, c'est une perte
  silencieuse.
- **FK composite** : `FOREIGN KEY ("cultureId", "exploitationId") REFERENCES
  "Culture" (id, "exploitationId")` garantit que la tâche et sa culture sont
  dans la même exploitation. Elle demande un `UNIQUE (id, "exploitationId")`
  sur `Culture`. Avec `MATCH SIMPLE` (par défaut), la FK n'est pas vérifiée si
  `cultureId` est `NULL`.

### Questions ouvertes

- [ ] Le raisonnement « prévu / réalisé » vaut-il aussi pour
      `Culture.datePlancheOccupee` (et `datePlancheDisponible`) ?
- [ ] Comment retrouver l'exploitation d'une tâche sans culture (désherbage
      d'une planche vide, semis en pépinière) ? Options :
      - `Tache.plancheId` facultatif : ne couvre pas la pépinière ;
      - `Tache.exploitationId` obligatoire + FK composite vers `Culture`
        (demande `exploitationId` sur `Culture`) : couvre tout, propage
        `exploitationId` ;
      - `cultureId` obligatoire : ne couvre aucun des deux cas.
      Lié aux questions ouvertes n°3 (`cultureId` obligatoire) et n°5
      (cohérence Tâche ↔ exploitation).
- [ ] Compréhension : avec la FK composite, une tâche `exploitationId = A`
      pointant vers une culture de B : que répond PostgreSQL, et pourquoi ?

---

[généré par IA] DEVLOG.md (entrée du 2026-10-01) - 2026-10-01
[généré par IA] MEMOIRE.md, .gitignore - 2026-10-01
[généré par IA] DEVLOG.md et MEMOIRE.md (décision sur les bornes `[)`) - 2026-10-01
[généré par IA] DEVLOG.md (nommage des bornes de Culture, condition de chevauchement) - 2026-10-05
[généré par IA] Choix de stockage des dates de Culture (deux `date` vs `daterange`) - 2026-10-05
[généré par IA] MEMOIRE.md (mise à jour après les décisions du 2026-10-05) - 2026-10-06
[généré par IA] DEVLOG.md (source de vérité de la date d'implantation, modeImplantation, Tache.cultureId) - 2026-10-06
[généré par IA] MEMOIRE.md (décisions du 2026-10-06) - 2026-10-06
[généré par IA] CHECK au moins une date sur Tache (datePrevue OR dateRealisation) - 2026-10-06
[généré par IA] DEVLOG.md et MEMOIRE.md (Tache : datePrevue, dateRealisation, CHECK, première récolte calculée) - 2026-10-06
[généré par IA] Chemin Tache → Exploitation et requête « récoltes à 30 jours » - 2026-10-06
[généré par IA] Tâche sans culture : options pour retrouver l'exploitation, FK composite - 2026-10-06
[généré par IA] DEVLOG.md (tableau de bord, filtre par exploitation, FK composite) - 2026-10-06
[généré par IA] MEMOIRE.md (tableau de bord, filtre par exploitation, point de reprise) - 2026-10-06
