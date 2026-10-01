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

### Appris

- **IDOR** (Insecure Direct Object Reference) : un `findUnique` par id seul
  permet de lire la culture d'une autre exploitation en modifiant l'id dans
  l'URL. D'où la règle : toujours filtrer par `id` ET par `exploitationId`.

### Questions ouvertes

- [ ] 403 ou 404 quand la ressource appartient à une autre exploitation ?
- [ ] Bornes de l'intervalle : `[]` ou `[)` ? Que représente la date de fin
      stockée (dernier jour de récolte ou premier jour libre) ? Quelle date
      afficher dans l'UI ?
- [ ] Source de vérité de la date d'implantation : la tâche « plantation » ou
      le début de l'intervalle de la culture ? (risque de désynchronisation)
- [ ] Lien Tâche → Culture : comment rattacher le semis en pépinière à sa
      culture ? Ce lien est-il obligatoire (désherbage sur une planche vide) ?
- [ ] Parcelle–Planche : N-N sur le schéma papier. Une planche peut-elle
      vraiment appartenir à deux parcelles ?
- [ ] Légume / Espèce : sens de la relation, et où placer la famille botanique
      (nécessaire pour les rotations) ?
- [ ] Catalogue de légumes : global, par exploitation, ou les deux ? (le schéma
      papier le relie à User en N-N)
- [ ] Tâche reliée à la fois à l'exploitation et aux planches : qu'est-ce qui
      empêche une incohérence entre exploitations ?

---

[généré par IA] DEVLOG.md (entrée du 2026-10-01) - 2026-10-01
[généré par IA] MEMOIRE.md, .gitignore - 2026-10-01
