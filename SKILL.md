---
name: discord-bot-panel
description: Bot Discord (discord.js v14) avec base SQLite (better-sqlite3) et panel web d'administration (Express, thème neko, responsive) pour que des non-développeurs modifient les textes, le statut et les délais du bot sans toucher au code. À utiliser dès qu'on parle du bot du serveur, du panel/dashboard du bot, de config SQLite, d'ajouter ou modifier une commande slash (/kick, /ban, /espace, /say, /botinfo, /detect), du règlement non lu (warn/ban automatique), de textes éditables depuis le site, de KarmaCloud, ou de reprendre/étendre ce projet, même si la personne ne cite pas le nom du skill.
---

# Bot Discord + panel web (SQLite)

Projet livré à une cliente non-développeuse : elle personnalise le bot depuis un site, sans toucher au code. Les fichiers prêts à l'emploi sont dans `assets/bot-panel/` : copie-les comme point de départ (`cp -r assets/bot-panel ./`), puis `npm install`.

## Architecture

```
bot.js        bot Discord + lance le panel (panel.start(client)) ; un seul process
db.js         SQLite : schéma, validation, getConfig(), updateSection(), membres avertis
server.js     Express : login par mot de passe, API /api/config, /api/stats, fichiers statiques
schema.sql    tables (une ligne id=1 par section de config) + valeurs par défaut
public/       index.html (CSS) + app.js (formulaires générés depuis la constante UI)
```

Flux : le panel écrit en base via `PUT /api/config/:section` → `db.updateSection` valide puis émet `config-updated` → le bot réagit (statut, minuteur). Le bot relit `db.getConfig()` à chaque commande : aucun redémarrage nécessaire.

## Règles à respecter (le projet repose dessus)

1. **Aucun texte écrit en dur dans `bot.js`** : tout message envoyé par une commande vient de `db.getConfig()` et passe par `fmt(texte, {variables})`.
2. `fmt` remplace `{variable}` et transforme les `\n` littéraux en vrais retours à la ligne. En base les textes par défaut contiennent `\n` littéral (écrit à la main dans le SQL) ; le serveur les convertit en vrais sauts de ligne pour les champs du panel.
3. `db.getConfig()` garde la forme de l'ancien `config.json` (`command_kick.responses.success`, etc.). Ne pas casser cette forme.
4. Les colonnes éditables sont une **liste blanche** (`SECTIONS` dans `db.js`) avec libellé et limites (texte 2000 caractères, boutons 80, entiers bornés). Jamais de nom de colonne venant de la requête.
5. Limites Discord : message 2000 caractères, libellé de bouton 80.
6. `ban_hours` doit rester supérieur à `warn_hours` (validé dans `updateSection('settings')`).

## Recette : ajouter une commande avec textes modifiables

Quatre endroits, dans cet ordre :

1. `schema.sql` : nouvelle table `config_command_<nom>` (`id INTEGER PRIMARY KEY CHECK (id = 1)` + une colonne `TEXT NOT NULL DEFAULT '...'` par message) et `INSERT OR IGNORE INTO ... (id) VALUES (1);`. Les bases existantes se mettent à jour toutes seules (`CREATE TABLE IF NOT EXISTS`).
2. `db.js` : ajouter la section dans `SECTIONS` (avec `T('Libellé lisible')` / `N(...)`) et l'exposer dans `getConfig()`.
3. `bot.js` : définir le `SlashCommandBuilder` dans `commands`, puis un bloc `if (interaction.commandName === '...')` dans `interactionCreate`. Toujours `const config = db.getConfig()` déjà fait en haut du gestionnaire ; passer chaque texte par `fmt`.
4. `public/app.js` : ajouter un groupe `{ title, section, fields: [...] }` dans `UI.textes` ; `vars('user', 'reason')` affiche les variables insérables d'un clic (les ajouter dans `V` si nouvelles).

Ensuite : `node --check bot.js db.js server.js public/app.js`.

## Recette : ajouter un réglage (nombre, ID, texte)

Colonne dans la table existante (`ALTER TABLE` ou nouvelle table si la base est déjà en production : `CREATE TABLE IF NOT EXISTS` n'ajoute pas de colonne), règle dans `SECTIONS`, champ dans `UI`. Types de champs du panel : `text`, `area`, `int`, `id`.

## Commandes existantes

`/kick`, `/espace` (clics de confirmation + phrases éditables, propriétaire uniquement), `/say`, `/botinfo` (créateur uniquement), `/ban`, `/warn` (avertissement par MP, enregistré dans `member_warnings`), `/detect` (seconds comptes probables, infos publiques uniquement, jamais d'IP : les seuils de score sont dans le code, seuls les messages sont éditables). Boucle automatique « règlement non lu » : avertissement puis bannissement selon les délais du panel.

## Mise en route

`.env` : `TOKEN` (bot), `PANEL_PASSWORD` (mot de passe long, sans lui le panel est désactivé), `PORT` (fourni par l'hébergeur), `DB_PATH` (optionnel). Intents requis : `Guilds` et `GuildMembers` (à activer dans le portail développeur Discord). `node bot.js`.

## Pièges connus

- `database.db` doit être sur un stockage persistant (sinon la config est perdue à chaque redéploiement) ; ne jamais le versionner, ni `.env`.
- better-sqlite3 : sur Node 24 il faut la version 12 ou plus (la 11 n'a pas de binaire précompilé et l'installation échoue sans compilateur, d'où « Cannot find module 'better-sqlite3' »).
- Utiliser le panel en HTTPS (le mot de passe y transite).
- Sessions en mémoire : reconnexion après chaque redémarrage.
- Intervalle de vérification long (1440 min par défaut) = avertissements et bans tardifs ; conseiller 60 min pour des délais précis.
- Le bot ne peut pas kick/ban un membre dont le rôle est au-dessus du sien.
- Aucun test automatisé n'existe : vérifier la syntaxe (`node --check`) puis tester en vrai sur un serveur de test avant la production.

## Charte du panel (design)

Thème « neko », pensé pour une cliente exigeante : sobre, aligné, sans fioritures.

- Clair par défaut, sombre automatique (`prefers-color-scheme`) ; couleurs en variables CSS dans `:root` : fond blanc cassé `#faf6f8`, texte aubergine `#2b2430`, accent rose sakura `#c8386d` (clair) / `#ff7aa8` (sombre).
- Police arrondie « Zen Maru Gothic » (Google Fonts) avec repli système.
- Une seule touche mémorable : la mascotte chat dessinée en SVG (`NEKO('awake' | 'sleep')` dans `app.js`) sur la connexion, le menu et l'état vide. Ne pas multiplier les décorations.
- Mise en page : blocs « titre à gauche, champs à droite » séparés par un filet ; une colonne sur mobile avec navigation en bas.
- Textes d'interface en français, phrase simple, verbes d'action (« Enregistrer », pas « Valider »), erreurs qui disent quoi corriger.

## Documentation (GitHub Pages)

`assets/docs/` contient un petit site statique en français, sans JavaScript : `index.html` (accueil), `membres.html` (membres ordinaires : règlement à valider, messages reçus, vie privée, FAQ), `admins.html` (premiers pas, panel, commandes, contrôle du règlement, textes et variables, dépannage, sécurité, maintenance) et `style.css` (même charte que le panel). Publication : copier le dossier `docs/` à la racine d'un dépôt GitHub, puis Settings → Pages → « Deploy from a branch » → branche `main`, dossier `/docs`. Le fichier `.nojekyll` désactive Jekyll.

À chaque évolution du bot, **mettre à jour la doc dans la même modification** :

- nouvelle commande → ligne dans le tableau « Les commandes » de `admins.html` + variables dans le tableau « Modifier les textes » + mention dans la liste de `index.html` ;
- nouveau message reçu par les membres (MP, sanction) → tableau « Les messages que vous pouvez recevoir » de `membres.html` ;
- nouvelle donnée lue ou conservée par le bot → section « Ce que le bot voit » de `membres.html` ;
- nouvelle limite, nouveau réglage ou nouvelle erreur courante → `admins.html` ;
- changer la date en pied de page.

Ne jamais y mettre de vraie URL de panel, de jeton ni de mot de passe. Les délais (12 h / 24 h) sont présentés comme des valeurs par défaut.

## Travailler avec une IA sur ce projet

Lire d'abord `db.js` (modèle de données et validation), puis `bot.js`, puis `public/app.js`. Modifier peu de fichiers à la fois, garder les noms de variables et la structure de `getConfig()`, et ne jamais demander ni coller le `TOKEN` ou `PANEL_PASSWORD` dans une conversation.
