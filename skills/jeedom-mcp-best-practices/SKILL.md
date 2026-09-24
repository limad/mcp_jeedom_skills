---
name: jeedom-mcp-best-practices
description: >
  Best practices for driving a Jeedom smart home through an mcp_jeedom MCP server:
  minimizing token usage on state/device queries (targeted lookups vs full dumps, ETag reuse,
  bulk operations, choosing summary vs compact format, avoiding full_map) and correctly
  authoring/editing Jeedom scenarios (trigger_tags syntax, common quoting pitfall, deprecated
  old-style triggers). Trigger when: repeatedly polling device or room state, about to fetch
  the full house state or full_map without needing the whole house, controlling several
  devices in sequence, or creating/editing a Jeedom scenario with a trigger condition.
---

# Jeedom MCP — bonnes pratiques d'usage

Complète le comportement déjà imposé par le serveur (intégrité des affirmations, masquage des
IDs internes à l'utilisateur, confirmation avant action sensible). Ce guide couvre ce qui n'y
tient pas : le choix d'options pour économiser des tokens, et les pièges de syntaxe des
scénarios.

## Deux jeux de noms d'outils

Le serveur peut être configuré en `tools_mode=merged` (actions groupées dans peu d'outils,
ex. `state(action=...)`) ou `tools_mode=legacy` (un outil par action, nom dédié). Les deux
existent en usage réel — vérifier la liste d'outils reçue plutôt que supposer laquelle est
active ; ne pas halluciner un outil qui n'y figure pas.

| Concept | Merged | Legacy |
|---|---|---|
| État complet de la maison | `state(action=full)` | `get_full_state` |
| Recherche ciblée (generic_type, pièce...) | `state(action=find)` | `find_command` |
| Lecture groupée de plusieurs commandes | `state(action=bulk)` | `bulk_read_values` |
| Changements récents | `state(action=changes)` | `get_changes` |
| Exécuter une action | `execute(action=single)` | `execute_action` |
| Exécuter plusieurs actions | `execute(action=bulk)` | `bulk_execute` |
| Lister les équipements | `devices(action=list)` | `list_devices` |

`scenario_write` (création/édition de scénarios) garde le même nom dans les deux modes.

## Économie de tokens

Chaque option ci-dessous existe côté serveur mais n'est appliquée QUE si l'appelant la demande —
ne pas se rabattre par défaut sur la lecture la plus lourde.

| Besoin | Utiliser | Éviter |
|---|---|---|
| État d'un équipement précis | recherche ciblée (`state find` / `find_command`) avec `generic_type`/`jeeobject_name` | état complet puis filtrage côté agent |
| Vue d'ensemble narrative de la maison | `format=summary` (~3K tokens) | `format=json` par défaut (~25K tokens) |
| Longue liste d'équipements/commandes | `format=compact` (tabulaire, dense) + pagination `limit`/`offset` + `only_active=true` | tout charger en JSON complet sans filtre |
| Reposer la même requête (poll répété) | renvoyer l'`etag` reçu précédemment → réponse `[UNCHANGED etag=...]` si rien n'a changé | rappeler sans `etag`, qui repaie le payload complet à chaque fois |
| Savoir ce qui a changé récemment | action `changes` / `get_changes` (fenêtre en minutes) | re-tirer l'état complet pour comparer soi-même |
| Lire/agir sur plusieurs commandes déjà identifiées | lecture/exécution `bulk` | boucler un appel par commande |
| Cartographie exhaustive de tous les IDs | seulement si vraiment nécessaire — la resource dédiée pèse à elle seule ~16K tokens | la lire "par précaution" en début de conversation |
| Vérifier qu'une action a bien été appliquée | relire l'état après ~2s | ré-exécuter l'action "pour vérifier" (ne prouve rien, coûte un aller-retour de plus) |
| Contexte maison (habitants, generic_types...) | lire les resources dédiées une fois en début de session | les relire à chaque tour |

## Scénarios : trigger_tags

Avant de créer/modifier un scénario avec condition de déclenchement, lire la resource
`scenario_schema` du serveur (syntaxe complète + exemples) plutôt que deviner — une erreur ici
coûte plus cher en tokens (essai/erreur, relecture de logs) que la lecture du schema.

**Ne pas utiliser l'ancienne syntaxe `#cmdId#`** (ex. `#1234#` retournant 1/0) pour un nouveau
scénario : dépréciée depuis Jeedom ≥ 4.5, encore fonctionnelle en compatibilité descendante
mais pas la syntaxe à générer — c'est pourtant celle que renvoie souvent la connaissance
générale d'un modèle sur Jeedom, antérieure à cette version. Utiliser les trigger_tags.

Piège récurrent des trigger_tags : la VALEUR comparée à `#trigger_name#` doit être entre
guillemets droits `"`, jamais entre `#...#` ni entre backticks.

- Correct : `#trigger_name# == "[Salon][Interrupteur][Appui]"`
- Incorrect : `#trigger_name# == #[Salon][Interrupteur][Appui]#`
- Incorrect : `` #trigger_name# == `[Salon][Interrupteur][Appui]` ``

Alternatives selon le besoin : `#trigger_id#` (comparaison numérique, insensible au renommage),
`#trigger_value#` (payload du déclencheur), `#trigger#` (origine : schedule/cmd/...).

Workflow recommandé : résoudre les notations de commandes → construire/choisir un trigger →
créer le scénario → valider avant de le considérer terminé.
