# mcp_jeedom_skills

Skill agent pour bien piloter une maison [Jeedom](https://www.jeedom.com/) via le serveur MCP
[mcp_jeedom](https://github.com/limad/jeedom_mcp) — pour Claude Code, Codex, et tout autre
agent qui lit `AGENTS.md`/les Claude Skills.

## Ce que ça couvre

- **Requêtes économes en tokens** — recherche ciblée plutôt que dump complet de la maison,
  réutilisation de l'`etag` pour les lectures répétées, formats `summary`/`compact`,
  lecture/exécution en masse plutôt que des appels un par un, éviter la carte complète des IDs
  (~16K tokens) sauf besoin réel.
- **Rédaction de scénarios** — syntaxe `trigger_tags` de Jeedom (`#trigger_name#`, `#trigger_id#`,
  `#trigger_value#`, `#trigger#`), le piège de guillemets qui casse une condition silencieusement,
  et pourquoi éviter l'ancienne syntaxe `#cmdId#` (pré-4.5) que la connaissance générale d'un
  modèle sur Jeedom suggère souvent.
- **La distinction `merged`/`legacy`** — `mcp_jeedom` peut exposer soit des outils groupés par
  action (`state(action=find)`), soit un outil par action (`find_command`), selon la
  configuration du serveur ; le skill donne les deux noms pour rester correct dans les deux cas.

## Installation

Comme plugin Claude Code (ajout marketplace), ou en clonant ce repo et en pointant l'agent vers
`skills/jeedom-mcp-best-practices/SKILL.md` / `AGENTS.md` directement.

## Périmètre

Ce skill porte sur *l'usage* d'un serveur mcp_jeedom — pas sur le développement de plugins
Jeedom. Il complète ce que le serveur `mcp_jeedom` impose déjà au niveau protocole (intégrité
des réponses, masquage des IDs internes à l'utilisateur, confirmation avant action sensible)
avec ce qui ne tient pas dans ce champ d'instructions toujours chargé.

## Licence

MIT — voir [LICENSE](LICENSE).
