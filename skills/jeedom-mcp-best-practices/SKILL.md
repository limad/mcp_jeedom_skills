---
name: jeedom-mcp-best-practices
description: >
  Best practices for driving a Jeedom smart home through an mcp_jeedom MCP server:
  minimizing token usage on state/device queries (targeted lookups vs full dumps, ETag reuse,
  bulk operations, choosing summary vs compact format, avoiding full_map) and correctly
  authoring/editing Jeedom scenarios (trigger_tags syntax, common quoting pitfall, deprecated
  old-style triggers, editing an existing scenario in place, testing a condition). Trigger when:
  repeatedly polling device or room state, about to fetch the full house state or full_map
  without needing the whole house, controlling several devices in sequence, searching the
  system messages, creating/editing a Jeedom scenario with a trigger condition, or about to
  recreate an existing scenario to change a few lines.
---

# Jeedom MCP — bonnes pratiques d'usage

Complète le comportement déjà imposé par le serveur (intégrité des affirmations, masquage des
IDs internes à l'utilisateur, confirmation avant action sensible). Ce guide couvre ce qui n'y
tient pas : le choix d'options pour économiser des tokens, et les pièges de syntaxe des
scénarios.

Suppose un serveur `mcp_jeedom` déjà configuré et accessible. Les outils réellement
disponibles dépendent de son `tools_mode` (merged/legacy) **et** du rôle attaché au token
utilisé (admin/operator/readonly/guest) ainsi que des permissions activées côté serveur
(`allow_scenario_write`, `read_only`, etc.) : l'absence d'un outil attendu signale souvent une
restriction de rôle/permission, pas un nom incorrect — se fier à la liste d'outils réellement
reçue.

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
| Chercher un équipement par nom | `devices(action=search)` | `search_devices` |
| Messages système | `messages(action=list)` | `get_system_messages` |

`scenario_write` (création/édition de scénarios) et `test_expression` gardent le même nom dans
les deux modes.

## Économie de tokens

Chaque option ci-dessous existe côté serveur mais n'est appliquée QUE si l'appelant la demande —
ne pas se rabattre par défaut sur la lecture la plus lourde.

Les volumes entre parenthèses (summary/json/full_map) sont des ordres de grandeur mesurés sur
une installation donnée — ils varient avec le nombre d'équipements/commandes de la maison
cible. Ce qui est garanti, c'est l'écart relatif (`summary` ≪ `json` complet ≪ `full_map`), pas
la valeur absolue : ne pas les citer à l'utilisateur comme un chiffre fixe.

| Besoin | Utiliser | Éviter |
|---|---|---|
| État d'un équipement précis | recherche ciblée (`state find` / `find_command`) avec `generic_type`/`jeeobject_name` | état complet puis filtrage côté agent |
| Vue d'ensemble narrative de la maison | `format=summary` (de l'ordre du K-token) | `format=json` par défaut (plusieurs fois plus lourd) |
| Longue liste d'équipements/commandes | `format=compact` (tabulaire, dense) + pagination `limit`/`offset` + `only_active=true` | tout charger en JSON complet sans filtre |
| Reposer la même requête (poll répété) | voir « Portée de l'ETag » ci-dessous — ne s'applique pas à tous les outils | renvoyer un `etag` à un outil qui n'en a jamais fourni (aucun effet) |
| Savoir ce qui a changé récemment | action `changes` / `get_changes` (fenêtre en minutes) | re-tirer l'état complet pour comparer soi-même |
| Lire/agir sur plusieurs commandes déjà identifiées | lecture/exécution `bulk` | boucler un appel par commande |
| Cartographie exhaustive de tous les IDs | seulement si vraiment nécessaire — la resource dédiée est nettement plus lourde qu'une recherche ciblée | la lire "par précaution" en début de conversation |
| Vérifier qu'une action a bien été appliquée | relire l'état après ~2s | ré-exécuter l'action "pour vérifier" (ne prouve rien, coûte un aller-retour de plus) |
| Contexte maison (habitants, generic_types...) | lire les resources dédiées une fois en début de session | les relire à chaque tour |
| Chercher dans les messages système | `search=` (mots) ou `plugin=` | la liste complète (textes coupés à 200 car.) |

## Portée de l'ETag

L'ETag n'existe que sur une partie des outils de lecture — ne pas en renvoyer un à un outil qui
n'en a jamais fourni dans sa réponse (ça n'a aucun effet, et laisse croire à une économie de
tokens qui n'a pas lieu).

Le renvoyer est pertinent pour (réponse `[UNCHANGED etag=...]` si rien n'a changé depuis) :

| Merged | Legacy |
|---|---|
| `state(action=full)` | `get_full_state` |
| `devices(action=list)` | `list_devices` |
| `state(action=object)` | `get_jeeobject_state` |
| `scenarios(action=list)` | `list_scenarios` |
| `plugins(action=list)` | `list_plugins` |
| `variables(action=list)` | `list_variables` |

N'en fournissent pas (toujours un payload complet) : la recherche ciblée (`state(action=find)` /
`find_command`), les lectures/exécutions `bulk`, `state(action=changes)` / `get_changes`,
`execute(...)`, et tout `scenario_write` (y compris sa relecture `action=full` /
`scenarios(action=code)` / `get_scenario_code`).

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

## Scénarios : workflow sécurisé

`scenario_write(action=validate)` n'est **pas** appelé automatiquement par `create`/`update` —
l'omettre expose à découvrir une erreur de structure ou une référence de commande non
résolvable seulement après écriture, ce qui coûte plus cher (essai/erreur, relecture de logs)
que l'appel de validation lui-même.

1. Résoudre les notations si besoin : `scenario_write(action=resolve)` (cmd_id numérique →
   notation `#[Objet][Équipement][Commande]#`).
2. Construire le déclencheur : `scenario_write(action=build_trigger)` plutôt que d'écrire la
   syntaxe trigger_tags à la main (évite le piège de guillemets ci-dessus).
3. **Valider avant d'écrire** : `scenario_write(action=validate)` sur les `elements`/`trigger`/
   `mode` envisagés — vérifie la structure, les références de commandes, et les pièges de
   syntaxe connus, sans rien sauvegarder. Corriger et revalider si des erreurs sont retournées.
4. Tester une condition : `test_expression` l'évalue avec les valeurs réelles (rôle admin).
5. Créer : `action=create`. Scénario **existant** : ne pas le recréer (ids perdus) ; lire
   `action=full` (ids), puis `action=patch` + `changes=[{op: set|insert_after|remove,
   expression_id, expected}]` pour quelques lignes, ou `action=update` + `scenario_json` (JSON
   de `full` modifié ; sans `elements` = réglages seuls). Bloc code : superadmin seul.
6. **Relire ce qui a réellement été enregistré** : `scenario_write(action=full)` (ou
   `scenarios(action=code)` / `get_scenario_code`) — ne pas supposer que le contenu envoyé est
   exactement celui qui a été stocké ; c'est cette relecture, pas l'appel de création, qui
   confirme le résultat.
