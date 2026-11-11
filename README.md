# stress-test-user-feature

Un skill pour prouver qu'une feature fonctionne réellement, au lieu de conclure parce qu'elle apparaît dans l'UI ou renvoie `200 OK`.

## Principe

**La présence n'est pas la preuve.**

- Une VM dans la liste ne prouve pas qu'elle démarre : il faut s'y connecter, exécuter une commande, redémarrer, vérifier la persistance, puis supprimer et vérifier que les ressources sont libérées.
- Un export CSV en `200` ne prouve rien : il faut télécharger le fichier, le parser et comparer son contenu aux données réelles.
- Un formulaire qui affiche « succès » ne prouve rien : il faut recevoir l'email, cliquer le lien, se connecter et vérifier la session.

Le skill impose donc des critères de preuve « last mile », un protocole reproductible, une exécution réelle et un rapport fondé sur des preuves.

## Ce que fait le skill

1. **Cartographie** la feature : points d'entrée, préconditions, états, effets de bord, erreurs, permissions, limites, intégrations, nettoyage.
2. **Définit les preuves** : pour chaque capacité, le résultat observable qui ne peut exister que si la feature marche.
3. **Construit une matrice de tests** : happy path, limites, erreurs, état, concurrence, sécurité, performance, cleanup, régression.
4. **Écrit un protocole** : environnement jetable, setup, ordre d'exécution, captures de preuves, teardown.
5. **Exécute réellement** et capture les preuves brutes.
6. **Rapporte** un verdict `PROUVÉ` / `PARTIEL` / `CASSE` / `BLOQUÉ`, avec les écarts et ce qui n'a pas été testé.
7. **Propose** des tests permanents et des correctifs, puis attend un accord explicite avant de modifier le code.

## Structure

```
.
├── SKILL.md                     # instructions principales
├── references/
│   ├── playbooks.md             # UI web, API, CLI, infra/VM, jobs async
│   └── templates.md             # feature map, matrice, protocole, rapport, bug
├── evals/
│   └── evals.json               # 3 scénarios de validation
├── commands/
│   └── stress-test-user-feature.md  # commande slash Maki (optionnelle)
└── LICENSE
```

## Installation

### Maki

Copier le dossier dans le répertoire des skills :

```bash
cp -r stress-test-user-feature ~/.config/maki/skills/
```

Pour la commande slash, copier aussi :

```bash
cp commands/stress-test-user-feature.md ~/.config/maki/commands/
```

Redémarrer Maki : les commandes sont découvertes au lancement.

### Claude Code / autres agents

Copier le dossier dans le répertoire de skills de l'agent, par exemple :

```bash
cp -r stress-test-user-feature ~/.claude/skills/
```

Le skill est autonome : `SKILL.md` et `references/` suffisent.

## Utilisation

Demander simplement :

- « Teste la feature de création de VM, je veux être sûr qu'elle marche. »
- « Stress-test le nouvel endpoint d'export CSV. »
- « Le flow d'inscription est terminé, vérifie-le. »

Ou avec la commande Maki :

```
/user:stress-test-user-feature création de VM
```

## Format du rapport

Le rapport est volontairement compact : verdict d'abord, tableau de preuves, écarts, non-testés, prochaine étape. Il ne contient pas de « ça devrait marcher » : chaque affirmation est soit observée, soit marquée `NON TESTÉ` ou `BLOQUÉ`.

## Validation

Les 3 scénarios de `evals/evals.json` couvrent :

1. création de VM (preuve par SSH, reboot, persistance, suppression) ;
2. export CSV (téléchargement, parsing, exactitude, sécurité) ;
3. flow d'inscription (email réel, session, doublons, cleanup).

Chaque scénario vérifie que le skill produit une cartographie, des critères de preuve, des cas négatifs, un protocole avec teardown, un verdict fondé sur des preuves, la liste des non-testés et une proposition en attente d'accord.

## Licence

MIT.
