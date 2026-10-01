# Mémoire de session — MaraîchPlan

Point de reprise pour continuer le projet sur une autre machine ou dans une
nouvelle session Claude Code. À lire en début de session, à mettre à jour en
fin de session. Le détail des décisions est dans [DEVLOG.md](DEVLOG.md).

## État actuel (2026-10-01)

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
  - contrainte d'exclusion : même planche + intervalles qui se chevauchent
    = interdit (btree_gist + daterange)
  - commence à l'implantation (plantation ou semis direct)
  - fin le jour J et début le jour J = pas de conflit
- Tâches : le semis en pépinière est une tâche, pas une partie de l'intervalle
  de la culture.

## Prochaine étape

Répondre à la question en suspens :
> La règle « fin le jour J / début le jour J = pas de conflit » est décidée,
> mais le choix des bornes `[]` / `[)` est encore ouvert. Pourquoi ces deux
> points sont-ils liés, et lequel des deux choix de bornes contredit la règle ?

Puis traiter, dans l'ordre, les questions ouvertes du DEVLOG :
1. bornes de l'intervalle et sens de la date de fin
2. source de vérité de la date d'implantation (tâche ou culture)
3. lien Tâche → Culture (obligatoire ou non)
4. Parcelle–Planche, Légume/Espèce/famille botanique, propriétaire du catalogue
5. cohérence Tâche ↔ exploitation

Une fois le schéma papier v2 justifié : passer à `schema.prisma`, en vérifiant
d'abord la documentation actuelle de Prisma.

## Rappels de méthode

- Mode socratique (voir CLAUDE.md). Code généré seulement avec « GÉNÈRE: ».
- Fichiers à créer : AI_POLICY.md, IDEES.md (prévus par CLAUDE.md).
