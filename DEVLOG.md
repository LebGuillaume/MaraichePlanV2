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
- [ ] Nom du champ / de la colonne de fin, pour éviter la confusion avec le
      « dernier jour de récolte ».
- [ ] Condition de chevauchement de `[a, b)` et `[c, d)` en TypeScript (domaine,
      jalon 5), qui doit donner le même verdict que PostgreSQL.
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
[généré par IA] DEVLOG.md et MEMOIRE.md (décision sur les bornes `[)`) - 2026-10-01
