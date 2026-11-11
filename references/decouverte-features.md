# Phase 1 renforcée — découvrir TOUTES les features, et toutes leurs manières d'être utilisées

La version courte de l'ancienne phase 1 — « lis le code de la feature » — trouve ce que le nom
de la feature désigne, et rien de plus. **Mesuré sur AnhyWhere le 2026-11-11** (commit `012fb41`) :
le balayage qui suit nomme **531 entrées** du produit — **150 routes HTTP**, **56 routes de la
SPA**, **54 écrans**, **23 commandes CLI**, **27 drapeaux CLI**, **38 tables**, **10 tâches de
fond**, **136 drapeaux d'environnement**, **17 codes d'erreur**, **18 variantes de message** et
**2 gardes d'autorisation**. La meilleure source seule en rate **150** (71,8 % de couverture), et
la source « nom de la feature » rate la moitié de la surface.

## Règle 0 — Une source unique MENT

| Source | Ce qu'elle trouve | Ce qu'elle rate (mesuré) |
|---|---|---|
| Nom de la feature | les écrans et routes évidents | les chemins CLI, les tâches de fond, les routes internes |
| Chaînes `"/api/..."` | 131 routes | **19 routes hors `/api/`** : console `/ws/serial`, événements, santé, OAuth, CLI |
| Registre de routes par enregistrement | `.Get/.Post/...` | les routes construites par préfixe dynamique |
| Interface (SPA) | ce que l'utilisateur voit | tout ce qui n'a pas d'écran (CLI, API publique, admin) |
| Tests | les intentions déjà écrites | les chemins jamais testés — donc les plus dangereux |
| Migrations | l'état persisté | ce que rien n'écrit (tables mortes) |

**Donc : on balaie onze sources, puis on joint.** La jointure est le livrable, pas les listes.

## Règle 1 — Les onze extractions (commandes robustes, chacune avec son piège)

Les commandes ci-dessous cherchent des **formes** (enregistrement, `Use:`, `Start*`), jamais le
nom d'une variable : c'est la seule forme qui ne dépend pas de la façon dont le routeur est monté.
Chaque piège indiqué a été **mesuré** le 2026-11-11 sur AnhyWhere.

```bash
# 1. Routes HTTP SERVIES — l'enregistrement, pas la chaîne.
#    Piège : `grep '"/api/'` trouve 250 chaînes pour 131 routes ; les 84 autres sont
#    des lectures d'en-tête (`.Get("X-JWT-Sub")`) et des chemins du client dans les tests.
grep -rhoE '\.(Get|Post|Put|Patch|Delete|Route|Handle|HandleFunc|Mount)\("(/[^"]*)"' \
  --include=*.go --exclude=*_test.go internal/ cmd/
#    Et regarde AUSSI les routes hors /api/ : console, événements, santé, OAuth, CLI.

# 2. Routes de la SPA — IMBRICATION RÉSOLUE.
#    App.jsx : un parent `/app` porte des enfants RELATIFS (`dashboard`, `vms`).
#    admin-app.jsx : un parent `/admin` idem, ET des routes absolues (`/login`, `/dashboard`).
#    Piège : aplatir invente `/dashboard` et fusionne deux écrans distincts (34 routes fausses).
#    On suit la dernière route absolue ouverte, on joint les enfants, on ignore `*`.

# 3. Écrans client et administration
ls web/src/pages/*.jsx web/src/pages/admin/*.jsx

# 4. CLI : cobra `Use:` (pas `name:`), et les drapeaux
#    Piège : `name: "..."` ne trouve que 5 commandes là où `Use:` en déclare 23.
grep -rhoE '\bUse:\s*"[^"]+"' --include=*.go cmd/ | sed -E 's/.*"([^" ]+).*/\1/'
grep -rhoE 'Flags\(\)\.[A-Za-z]+\("[^"]+"' --include=*.go cmd/

# 5. Tables — commentaires SQL retirés, schéma normalisé, nom suivi d'une parenthèse.
#    Piège : un commentaire « CREATE TABLE IF NOT EXISTS. » et un DDL dynamique
#    `CREATE TABLE IF NOT EXISTS %I` produisent deux tables fantômes (`IF`, `public`).
grep -rhoE 'CREATE[[:space:]]+TABLE[[:space:]]+(IF[[:space:]]+NOT[[:space:]]+EXISTS[[:space:]]+)?(public\.)?[a-z_][a-z0-9_]*[[:space:]]*\(' \
  internal/db/migrations/*.up.sql

# 6. Tâches de fond — le REGISTRE, pas le motif.
#    Piège : `go func|ticker|cron|worker` rend 205 lignes et zéro nom. Les tâches
#    réelles sont des fonctions `Start*` (10 mesurées).
grep -rhoE '^func (Start[A-Z][A-Za-z0-9_]*)' --include=*.go internal/ cmd/ --exclude=*_test.go

# 7. Drapeaux de configuration et d'environnement
grep -rhoE '"ANHYWHERE_[A-Z0-9_]+"' --include=*.go internal/ cmd/

# 8. Échecs : codes ET variantes de message (la dimension « échec » de la matrice)
grep -rhoE 'Code:\s*"[A-Z][A-Z0-9_.]+"' --include=*.go internal/errors/ --exclude=*_test.go
jq -r 'keys[]' internal/i18n/messages/fr/errors.json   # ou le catalogue du dépôt

# 9. Acteurs : les gardes d'autorisation de production
#    Piège : `requireAdmin(` apparaît 68 fois, dont 63 appels de harnais de test.
#    On extrait les DÉFINITIONS de méthodes, pas les appels.
grep -rhoE 'func \([^)]*\) (require[A-Z][A-Za-z0-9_]*)' --include=*.go internal/ cmd/ --exclude=*_test.go

# 10. Intentions déjà écrites (tests) — elles nomment les règles et les pièges
grep -rhoE '^func (Test|TestGuard|TestUserFound)[A-Za-z0-9_]+' --include=*_test.go .

# 11. Usage réel : les chemins que le client appelle (ce que l'écran utilise vraiment)
grep -rhoE "['\"\`]/api/v1/[^'\"\`]*" web/src/api*.js
#    Piège : les préfixes construits (`/api/v1/admin/nodes/${nodeId}/drain`) ne tombent
#    sur aucune route littérale. Ce sont des usages, pas des entrées nouvelles.
```

## Règle 2 — La fiche de feature (une par capacité, jamais par fichier)

Une feature n'est pas un fichier : c'est **un état que l'utilisateur veut obtenir**. Une fiche par
capacité, avec son identifiant (`F01`, `F02`…), et six lignes obligatoires :

```
F<nn> <capacité, à l'infinitif>
  Entrées     : écran | CLI | API | tâche de fond | webhook     (chemins exacts)
  Acteurs     : rôle, offre, drapeau, propriétaire              (qui a le droit)
  États       : vide | chargé | erreur | en cours | final       (ce que l'écran montre)
  Effets      : base (tables), fichiers, réseau, facturation, notifications
  Échecs      : entrée invalide, conflit, refus de droit, dépendance morte, quota
  Preuve      : l'action dont le résultat ne peut exister QUE si ça marche
```

## Règle 3 — La matrice des usages (le cœur de la découverte)

Pour chaque fiche, on énumère le **produit cartésien** des manières de l'utiliser, puis on élague
par ce qui est réellement atteignable :

```
entrée (écran, CLI, API, fond)  ×  acteur (anonyme, client, admin, lecture seule, ouvrier)
                               ×  état (vide, chargé, erreur, en cours)
                               ×  chemin (nominal, limite, erreur, reprise, concurrence)
```

Ce produit est volontairement trop grand : il sert à **trouver les cases**, pas à toutes les tester.
Chaque case retenue devient une ligne de la matrice de tests avec sa preuve.

**Le gain mesuré** : la même feature « gabarit » passe de 3 usages (importer, lister, créer une VM)
à **17** — dont l'import par URL, l'import par morceaux, la reprise après coupure, le refus de type,
le quota, le renommage, la suppression, l'auto-publication, la conversion, les métadonnées, la
lecture des ressources, et les cinq refus de droit correspondants.

## Règle 4 — Les contrôles d'orphelins (ce que les listes ne disent pas)

Une source seule produit des listes ; la jointure produit des **écarts**, et ce sont eux qui
révèlent les features manquées :

| Écart | Ce qu'il révèle |
|---|---|
| Route sans appelant dans l'interface | une feature CLI/API seulement, ou une route morte |
| Écran qui n'appelle aucune route | une page qui lit autre chose (état local, autre module), ou un écran factice |
| Table sans écrivain dans le code | un état mort, ou une écriture qui vit ailleurs (migration, trigger) |
| Drapeau d'environnement sans lecteur | une configuration fantôme |
| Test qui nomme une règle sans route | une règle déplacée ou supprimée sous le test |

**Mesure de référence (AnhyWhere, 2026-11-11)** : 150 routes servies, 86 chemins appelés par
l'interface, **12 appels du client qui ne tombent sur aucune route littérale** — tous des préfixes
construits dynamiquement. Un écart se vérifie avant d'être cru.

## Règle 5 — Le benchmark de la découverte (le cœur de ce fichier)

Avant de conclure la phase 1, on mesure **le rendement de chaque source** sur CE dépôt. Le banc
d'AnhyWhere (`tests/skill-decouverte/bench.py`) compare chaque passe à un oracle de 531 entrées ;
une passe qui n'apporte plus rien est la dernière.

**Banc d'essai du 2026-11-11 sur AnhyWhere (531 entrées, 10 passes)** :

| Passe | Source ajoutée | Couvertes | Nouvelles | Bruit | Couverture |
|---|---|---:|---:|---:|---:|
| P01 | les huit extractions de l'ancienne version | 381/531 | 381 | 1672 | 71,8 % |
| P02 | routes par enregistrement | 400/531 | 19 | 0 | 75,3 % |
| P03 | SPA normalisée, routeur admin inclus | 434/531 | 34 | 0 | 81,7 % |
| P04 | CLI par `Use:` | 457/531 | 23 | 0 | 86,1 % |
| P05 | drapeaux CLI | 484/531 | 27 | 0 | 91,1 % |
| P06 | registre `Start*` | 494/531 | 10 | 0 | 93,0 % |
| P07 | codes d'erreur | 511/531 | 17 | 0 | 96,2 % |
| P08 | gardes (acteurs) | 513/531 | 2 | 0 | 96,6 % |
| P09 | variantes de message | 531/531 | 18 | 0 | 100,0 % |
| P10 | appels du client (arrêt) | 531/531 | 0 | 12 | 100,0 % |

Enseignements :

- L'ancienne phase 1 couvrait **71,8 %** de la surface et annonçait **1 672** valeurs bruyantes.
- Les trois plus gros gains sont les **routes hors `/api/`** (19), les **routes SPA imbriquées**
  (34) et les **drapeaux CLI** (27) : trois familles qu'aucune source ne voyait.
- La passe d'arrêt est réelle : une source candidate (les appels du client) n'apporte **aucune
  entrée nouvelle**. On s'arrête là, pas quand « on est fatigué ».
- Ce banc se refait **par dépôt** (`python3 tests/skill-decouverte/bench.py`), jamais par feature.

## Règle 6 — Les six pièges mesurés (à vérifier sur chaque dépôt)

| Piège | Mesure | Correctif |
|---|---|---|
| `grep '"/api/'` | 250 chaînes, 131 routes, 84 faux positifs | extraire l'enregistrement, filtrer les en-têtes |
| `path="..."` dans un `.jsx` | 1 672 valeurs, dont des tracés SVG (`M12 4h8…`) | restreindre au routeur, ancrer `path=` d'une balise `Route` |
| Routes SPA relatives | `/app/dashboard` lu `/dashboard`, collision avec l'admin | suivre la dernière route absolue ouverte |
| `CREATE TABLE` en commentaire | tables fantômes `IF`, `public` | retirer les commentaires, exiger `(` après le nom |
| `name: "..."` pour la CLI | 5 commandes sur 23 | cobra `Use:` |
| `go func\|ticker\|cron` | 205 lignes, 0 tâche nommée | fonctions `Start*` |

## Règle 7 — La dimension hostile (ce qu'un chemin nominal ne voit jamais)

Pour chaque entrée qui reçoit un fichier, une URL, un nom ou un identifiant, on énumère **au moins
une** tentative hostile, avec le résultat attendu. Cas mesurés sur AnhyWhere (feature « gabarit ») :

| Contenu hostile | Où il est jugé | Résultat attendu |
|---|---|---|
| fichier troll (zip, vide, aléatoire) | reconnaissance par les premiers octets | format non reconnu, travail `echec`, aucun gabarit |
| `.qcow2` tronqué | `qemu-img convert` | échec nommé, fichier partiel retiré |
| `.ext4` sans superbloc | contrôle de signature | refus nommé, pas de démarrage |
| nom `../evasion` | validation AVANT tout téléchargement | refus |
| nom > 128 octets / caractères hors liste | `ValidateName` | refus nommé |
| description > 500 runes | validation du corps | refus nommé |
| `file:///etc/passwd` | analyse d'URL | schéma refusé |
| `http://user:pass@hote/x` | analyse d'URL | identifiants refusés |
| `http://localhost/x`, `10.0.0.1` | garde d'adresse résolue | refus avant téléchargement |
| redirection vers une adresse privée | re-jugement de chaque saut (≤ 5) | refus nommé |
| URL lente / infinie | délai du travail | `echec`, serveur non bloqué |
| deux imports du même nom | unicité en base | un seul `pret`, l'autre `echec` nommé |

Sans cette table, la matrice de tests couvre le chemin heureux et laisse passer exactement ce que
le propriétaire d'un produit craint : un fichier qui casse l'interface ou une URL qui atteint le
réseau interne.
