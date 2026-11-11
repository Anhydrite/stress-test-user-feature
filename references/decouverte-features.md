# Phase 1 renforcée — découvrir TOUTES les features, et toutes leurs manières d'être utilisées

La version courte de l'ancienne phase 1 — « lis le code de la feature » — trouve ce que le nom
de la feature désigne, et rien de plus. **Mesuré sur AnhyWhere le 2026-11-11** : en partant du seul
nom, on trouve les écrans et deux ou trois routes ; le balayage ci-dessous en trouve **129 routes**
dont **49 sans appelant dans l'interface**, **36 pages** client, **17 pages** d'administration,
**40 tables** et **4 497** tests Go. Un tiers de la surface produit n'apparaît dans aucune des
sources prise seule.

## Règle 0 — Une source unique MENT

Chaque source rate ce que les autres voient :

| Source | Ce qu'elle trouve | Ce qu'elle rate |
|---|---|---|
| Nom de la feature | les écrans et routes évidents | les chemins CLI, les tâches de fond, les routes internes |
| Registre de routes (par nom de variable) | `r.Get(...)` | les sous-routeurs (`api.Post`, `sub.Get`) — **mesuré : 2 routes trouvées au lieu de 12** |
| Interface (SPA) | ce que l'utilisateur voit | tout ce qui n'a pas d'écran (CLI, API publique, admin) |
| Tests | les intentions déjà écrites | les chemins jamais testés — donc les plus dangereux |
| Migrations | l'état persisté | ce que rien n'écrit (tables mortes) |

**Donc : on balaie huit sources, puis on joint.** La jointure est le livrable, pas les listes.

## Règle 1 — Les huit extractions (commandes robustes)

Les commandes ci-dessous cherchent des **chaînes de route**, jamais le nom d'une variable : c'est la
seule forme qui ne dépend pas de la façon dont le routeur est monté.

```bash
# 1. Routes HTTP servies
grep -rhoE '"(/api/[^"]+)"' --include=*.go cmd/ internal/ | sed 's/"//g' | sort -u

# 2. Commandes CLI et sous-commandes
grep -rhoE '\bname:\s*"[a-z][a-z-]+"|case "[a-z][a-z-]+"' --include=*.go cmd/ | sort -u

# 3. Écrans client et administration
ls web/src/pages/*.jsx web/src/pages/admin/*.jsx

# 4. Routes déclarées par le routeur de la SPA
grep -rhoE 'path="[^"]+"' web/src/*.jsx web/src/App.jsx | sort -u

# 5. Tables et leur état
grep -rhoE 'CREATE TABLE (IF NOT EXISTS )?[a-z_.]+' internal/db/migrations/*.up.sql | sort -u

# 6. Tâches de fond, minuteurs, files
grep -rnE 'time\.NewTicker|go func\(|StartProxyConvergence|cron|worker' --include=*.go internal/ cmd/

# 7. Drapeaux de configuration et d'environnement
grep -rhoE '"ANHYWHERE_[A-Z_]+"' --include=*.go . | sort -u

# 8. Intentions déjà écrites (tests) — elles nomment les règles et les pièges
grep -rhoE '^func (Test|TestGuard|TestUserFound)[A-Za-z0-9_]+' --include=*_test.go .
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
à 17 — dont l'import par URL, l'import par morceaux, la reprise après coupure, le refus de type, le
quota, le renommage, la suppression, l'auto-publication, la conversion, les métadonnées, la lecture
des ressources, et les cinq refus de droit correspondants.

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

**Mesure de référence (AnhyWhere, 2026-11-11)** : 119 routes serveur, 110 chemins appelés par
l'interface, **70 en commun**, **49 routes sans appelant front**, **40 chemins front sans route
correspondante** (dont des préfixes construits dynamiquement — un écart se vérifie avant d'être
cru). Ces 49 routes sont exactement les features que le nom de la feature ne trouve jamais.

## Règle 5 — Le benchmark de la découverte (à refaire sur chaque dépôt)

Avant de conclure la phase 1, on mesure **le rendement de chaque source**, pour savoir laquelle
pousse à creuser :

**Banc d'essai du 2026-11-11 sur AnhyWhere** (3 zones, 409 entrées distinctes au total) :

| Méthode | templates | console | auth | Ce qu'elle rate |
|---|---|---|---|---|
| Nom seul (écrans qui portent le mot) | **4** | 3 | 5 | tout le reste |
| Routes serveur seules | 15 | 1 | 10 | les écrans, les tâches de fond, les appels sans route |
| Interface seule (chemins appelés) | 14 | 1 | 9 | les 50 routes sans appelant |
| **Balayage joint des huit sources** | **33** | **7** | **25** | rien de connu |

Le balayage trouve **2,2× à 5×** ce que la meilleure source seule trouve, et jusqu'à **18 entrées
de plus** sur une seule zone. Les 50 routes sans appelant dans l'interface sont exactement la
surface que « lire le code de la feature » ne voit jamais.

Une source dont la colonne « uniques » est vide peut être abrégée la fois suivante ; une source qui
porte beaucoup d'uniques est **obligatoire**. Cette mesure se refait par dépôt, pas par feature.
