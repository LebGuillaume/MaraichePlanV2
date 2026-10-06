# Mémoire de session — MaraîchPlan

Point de reprise pour continuer le projet sur une autre machine ou dans une
nouvelle session Claude Code. À lire en début de session, à mettre à jour en
fin de session. Le détail des décisions est dans [DEVLOG.md](DEVLOG.md).

## État actuel (2026-10-06)

- **Jalon actif : 1 — Modélisation**, encore au stade du papier.
  Pas encore de `schema.prisma`, de Next.js ni de `package.json`.
- Schéma papier v1 : [docs/schema-papier-v1.png](docs/schema-papier-v1.png).
  Il est **dépassé** sur plusieurs points (voir ci-dessous).

## Modèle retenu à ce stade

- User → Exploitation (N-1, à reconfirmer : rôles prévus au jalon 3)
- Exploitation → Parcelles (1-N)
- Parcelle → Planches (N-N sur le papier, **à revoir** : probablement 1-N)
- **Culture** (nouvelle entité, remplace le lien direct Légume → Planche) :
  un légume sur une planche pendant un intervalle de dates.
  - id UUID (v4 ou v7 à trancher)
  - commence à l'implantation (plantation ou semis direct)
  - **bornes `[)`, en jours** (pas d'heure) :
    `datePlancheOccupee` (début, inclus) et `datePlancheDisponible`
    (fin, exclue = premier jour où la planche est libre, saisie et affichée
    sans conversion ±1 jour)
  - stockage : deux colonnes `date` (`@db.Date`), pas de colonne `daterange`
    (non supportée par Prisma)
  - CHECK `datePlancheOccupee < datePlancheDisponible` (migration SQL perso)
  - contrainte d'exclusion : même planche + plages qui se chevauchent
    = interdit (btree_gist + `daterange(datePlancheOccupee,
    datePlancheDisponible, '[)')` construit dans la contrainte)
  - chevauchement (domaine, jalon 5) : `a < d et c < b` ;
    fin le jour J et début le jour J = pas de conflit
  - `datePlancheOccupee` est la **seule** source de la date d'implantation
    (pas de tâche « plantation »)
  - `modeImplantation` : enum obligatoire (plantation, semis direct)
- Culture → Tâches (1-N) : FK `Tache.cultureId` (côté N), index à prévoir ;
  obligatoire ou non : **à trancher**
- Tâches : événements (0 ou N par culture : semis en pépinière, désherbage,
  récolte…). Le semis en pépinière n'est pas dans l'intervalle de la culture.
  - `datePrevue` et `dateRealisation`, toutes deux facultatives
    (`dateRealisation` NULL = pas encore faite ; pas de booléen)
  - CHECK `datePrevue IS NOT NULL OR dateRealisation IS NOT NULL`
    (migration SQL perso)
  - pas de CHECK `dateRealisation >= datePrevue` (avance = cas normal)
  - date de première récolte calculée (`MIN`), pas stockée
- Conventions de nommage : français, camelCase, ASCII sans accents,
  sans abréviations.

## Prochaine étape

Traiter, dans l'ordre, les questions ouvertes du DEVLOG :
1. ~~bornes, nom des champs, condition de chevauchement, stockage~~ (fait)
2. ~~source de vérité de la date d'implantation~~ (fait le 2026-10-06)
3. **à reprendre ici** : `Tache.cultureId` obligatoire ou non ? (désherbage
   d'une planche vide, semis en pépinière sans planche encore choisie)
   Puis : le raisonnement « prévu / réalisé » vaut-il aussi pour
   `Culture.datePlancheOccupee` ? (question réservée, voir DEVLOG 2026-10-06)
4. Parcelle–Planche, Légume/Espèce/famille botanique, propriétaire du catalogue
5. cohérence Tâche ↔ exploitation
6. 403 ou 404 pour une ressource d'une autre exploitation

Une fois le schéma papier v2 justifié : passer à `schema.prisma`, en vérifiant
d'abord la documentation actuelle de Prisma.

## Rappels de méthode

- Mode socratique (voir CLAUDE.md). Code généré seulement avec « GÉNÈRE: ».
- Fichiers à créer : AI_POLICY.md, IDEES.md (prévus par CLAUDE.md).
