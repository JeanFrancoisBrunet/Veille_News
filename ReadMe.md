# Veille 🔎

Veille proactive pour l'agent **Agent_Groq** (Raspberry Pi 5) : à intervalles réguliers, le script lit des sources nommées (flux RSS/Atom, versions GitHub ou PyPI, pages web), retient ce qui est nouveau et pertinent pour chaque thème, puis envoie un **digest Telegram** par le bot LLM_Groq, lisible sur le téléphone.

Le projet est composé de deux fichiers :

- **`veille.py`** — le moteur : lecture des sources, dédoublonnage, filtrage, notation par le LLM, digest.
- **`veille.yaml`** — la configuration : thèmes, sources, fréquences, seuils, heures calmes.

> ℹ️ La veille est **proactive et en lecture seule** : le script décide *quand* chercher et *quoi* signaler, mais ne déclenche jamais d'action à partir d'un texte lu sur le web. Le LLM ne fait que noter et résumer.

---

## Principe

```
 cron (toutes les heures)
        │
        ▼
 veille.py ──► thèmes « dus » ? (every_hours, heures calmes)
        │
        ▼  pour chaque thème dû
 lecture des sources ──► dédoublonnage ──► filtre mots-clés ──► 1 appel LLM par thème
 (RSS, Atom, GitHub,        (déjà vu ?)      (exclure,            (note 0-10 + résumé
  PyPI, pages)                                 exiger)               en français)
        │
        ▼
 note ≥ seuil ──► digest Telegram (S23) + copie digests/AAAAMMJJ-HHMM.md
```

**Deux niveaux de fréquence**

1. Le **battement** : le crontab réveille `veille.py` (une seule ligne, réglée une fois).
2. `every_hours`, **par thème** : un thème n'est traité que s'il est « dû ». La fréquence effective d'un thème ne peut pas être plus fine que le battement.

---

## Installation

1. Placer les fichiers dans le dossier `Veille/`, à côté du dossier qui contient `agent_groq_ng.py` (par défaut : le dossier parent de `Veille/`). Autre emplacement : variable d'environnement `VEILLE_AGENT_DIR`.
2. Prérequis : ceux de l'agent, plus `PyYAML`. La clé Groq et la configuration Telegram sont celles de l'agent (`~/.telegram_config`).
3. Tester les sources : `python3 veille.py --check`
4. Simuler un digest sans rien envoyer : `python3 veille.py --dry-run --force`
5. Programmer le battement (`crontab -e`) :

```
0 * * * * cd ~/Projects/Groq_agent/Veille && /usr/bin/python3 veille.py >> veille.log 2>&1
```

Toutes les 30 minutes : remplacer `0 * * * *` par `*/30 * * * *`.

6. Ajouter au `.gitignore` : `veille_state.json`, `veille.lock`, `veille.log`, `digests/`, `veille_feedback.jsonl`, `veille_alert.stamp`.

La première lecture d'une source ne déverse pas son historique : seuls les `first_run_items` éléments les plus récents sont candidats, le reste est marqué comme déjà vu.

---

## Configuration : `veille.yaml`

### Section `global`

| Clé                   | Rôle                                                                                                                                                                                                           |
|---                    |---                                                                                                                                                                                                             |
| `model`               | Modèle de notation et de résumé (par défaut `openai/gpt-oss-20b`)                                                                                                                                              |
| `min_score`           | Note minimale (0-10) pour figurer dans le digest                                                                                                                                                               |
| `max_items_per_theme` | Éléments maximum par thème et par digest (le surplus reste pour le suivant)                                                                                                                                    |
| `llm_batch_max`       | Éléments soumis au LLM par thème et par passage                                                                                                                                                                |
| `first_run_items`     | Éléments retenus à la toute première lecture d'une source                                                                                                                                                      |
| `max_age_days`        | Âge maximal d'un élément (réglable par thème ou par source)                                                                                                                                                    |
| `daily_token_budget`  | Tokens par jour que la veille peut consommer                                                                                                                                                                   |
| `keep_free_tokens`    | Réserve laissée sur le quota Groq du jour : en dessous, plus d'appel LLM                                                                                                                                       |
| `quiet_hours`         | `[début, fin]` en heures locales : aucun passage dans cette plage (`[]` pour désactiver). Avec `[22, 6]` : silence de 22h00 à 5h59, premier passage à 6h                                                       |
| `notify`              | Envoi Telegram (`false` = fichiers `digests/` seulement)                                                                                                                                                       |
| `notify_empty`        | Message « rien de nouveau » quand il n'y a rien                                                                                                                                                                |
| `feedback`            | Boutons 👍/👎 sous chaque digest et ajustement des notes (`false` : désactivé). Réglages fins : `feedback_half_life_days` (60), `feedback_src_step` (0,5), `feedback_kw_step` (0,25), `feedback_max_bias` (2)  |
| `save_digests`        | Copie de chaque digest dans `digests/`                                                                                                                                                                         |

### Section `themes`

Chaque thème est un bloc nommé :

| Clé                                      | Rôle                                                                              |
|---                                       |---                                                                                |
| `label`, `emoji`                         | Titre affiché dans le digest                                                      |
| `enabled`                                | Actif ou non                                                                      |
| `every_hours`                            | Fréquence du thème, en heures (décimales acceptées)                               |
| `focus`                                  | Centre d'intérêt, en une ou deux phrases : c'est ce que le LLM utilise pour noter |
| `keywords`                               | Mots-clés (mots entiers, insensibles à la casse) : servent au tri et au filtre    |
| `exclude`                                | Mots qui écartent un élément                                                      |
| `min_score`, `max_items`, `max_age_days` | Surcharges facultatives pour ce thème                                             |
| `sources`                                | Liste de sources nommées                                                          |

### Types de sources

| Type               | Paramètre                  | Description                                                                                    |
|---                 |---                         |---                                                                                             |
| `rss`              | `url`                      | Flux RSS ou Atom                                                                               |
| `github_releases`  | `repo: propriétaire/dépôt` | Versions publiées d'un dépôt                                                                   |
| `pypi`             | `package`                  | Versions publiées d'un paquet PyPI                                                             |
| `page`             | `url`                      | Page sans flux : une alerte est émise quand son contenu change (sensible aux pages dynamiques) |

Options facultatives par source : `require_keyword: true` (ignore les éléments sans mot-clé du thème) et `max_age_days`.

> ℹ️ Un bot ne lit que les serveurs Discord où il a été invité : Discord ne peut donc pas être une source de veille.

---

## Commandes

| Commande                 | Effet                                                                             |
|---                       |---                                                                                |
| `python3 veille.py`      | Passage normal (thèmes dus uniquement) : celui du cron                            |
| `--check`                | Teste la lecture de chaque source (lecture seule)                                 |
| `--status`               | Thèmes, fréquences, dernier et prochain passage, tokens du jour, sources en échec |
| `--dry-run --force`      | Simulation complète : rien n'est envoyé ni mémorisé                               |
| `--theme NOM`            | Traite ce thème maintenant, même s'il n'est pas dû (répétable)                    |
| `--force`                | Ignore fréquences et heures calmes (thèmes actifs)                                |
| `--no-llm`               | Mots-clés seulement, aucun token consommé                                         |
| `--freq NOM HEURES`      | Change la fréquence d'un thème sans éditer le YAML                                |
| `--on NOM` / `--off NOM` | Active ou désactive un thème                                                      |
| `--reset NOM`            | Retire les surcharges d'un thème (retour au YAML)                                 |
| `--reset-votes`          | Oublie les avis 👍/👎 : les notes ne sont plus ajustées                           |

Les surcharges (`--freq`, `--on`, `--off`) sont conservées dans `veille_state.json`, ce qui laisse les commentaires de `veille.yaml` intacts. Dans `--status`, un astérisque signale un thème surchargé.

---

## Digest Telegram

Un message par passage, regroupé par thème. Chaque élément indique son titre, sa note, un résumé de 25 mots maximum, sa source et son lien. La mention « sans IA » signale une note de repli par mots-clés. Les digests longs sont découpés sous la limite de Telegram (4 096 caractères). Les sources en échec depuis au moins 3 passages sont signalées en bas du digest.

---

## Avis 👍/👎 : la veille apprend de vos choix

Chaque digest est suivi d'un message à boutons : une ligne **👍 N / 👎 N** par article, numérotée comme le digest. Un appui enregistre l'avis ; le bouton choisi se coche (✓) et reste modifiable.

```
 digest  ──►  boutons 👍/👎  ──►  bot Telegram  ──►  veille_feedback.jsonl  ──►  veille.py (passage suivant)
                                  (enregistre)        (une ligne par avis)       note ajustée par source et mot-clé
```

- **Le bot ne fait qu'enregistrer.** Un seul processus peut lire les messages Telegram : les boutons sont donc traités par `telegram_bot_groq_ng.py`, qui doit tourner. Sans lui, les boutons restent sans effet.
- **`veille.py` applique les avis au passage suivant**, avant de noter les articles : chaque article reçoit un ajustement en points de note, ajouté à la note du modèle, puis la note est bornée entre 0 et 10.
- **Poids par source et par mot-clé :** 0,5 point par avis net sur la source (plafond ±1,5), 0,25 point par avis net sur chacun de ses mots-clés (plafond ±1), ajustement total plafonné à ±2 points.
- **Un avis par article :** le dernier l'emporte.
- **Les avis s'estompent :** un avis perd la moitié de son poids en 60 jours, puis disparaît. Une source qui s'améliore retrouve sa note normale.
- **L'ajustement est visible :** dans le digest, une note modifiée de 0,5 point ou plus s'affiche ainsi : `[8/10 (+1)]`. `--status` liste les ajustements actifs.
- **Ce qui est ajusté :** uniquement la note, donc le passage ou non sous `min_score` et l'ordre. Les alertes « page modifiée » ne sont jamais ajustées. Le texte des articles n'intervient pas.

Remise à zéro : `python3 veille.py --reset-votes`. Désactivation : `feedback: false` dans `veille.yaml`.

> ℹ️ Les avis n'étant pris en compte qu'au passage suivant d'un thème, ils ne changent pas un digest déjà envoyé.

---

## Quota Groq et mode dégradé

La veille partage le quota gratuit de l'agent. Elle fait **au plus un appel LLM par thème et par passage**, avec un modèle léger et un effort de raisonnement minimal. Elle s'abstient d'appeler le LLM si :

- le budget quotidien `daily_token_budget` est atteint ;
- le quota Groq du jour est entamé au point de passer sous `keep_free_tokens` (le chat de l'agent reste prioritaire) ;
- la clé Groq est indisponible, ou `--no-llm` est demandé.

Dans ces cas, la notation se fait par mots-clés, sans résumé IA. Un échec d'appel ou une réponse illisible a le même effet.

---

## Sécurité

- Le texte des sources est traité comme une **donnée non fiable** : jamais comme des instructions.
- Le LLM n'a **aucun outil** ; sa réponse ne sert qu'à produire une note et un résumé. Les liens présents dans le résumé sont supprimés, sa longueur est limitée ; les liens affichés viennent uniquement des sources.
- Aucune action n'est exécutée à partir du contenu lu.
- La lecture web passe par les fonctions blindées de l'agent (refus des adresses internes, taille limitée).
- Un verrou (`veille.lock`) empêche deux passages simultanés.
- Si `veille.yaml` est illisible, le passage du cron s'arrête sans rien lire ni envoyer, et un message Telegram prévient (au plus une fois par 12 h, avec la ligne fautive).
- Un digest n'est marqué « déjà vu » qu'après un envoi Telegram réussi : en cas d'échec, les éléments sont reproposés au passage suivant.

---

## Utilisation depuis l'agent

`veille_news` figure dans la liste blanche `LAUNCHABLE_SCRIPTS` de `agent_groq_ng.py` :

- terminal : `/news` (état, sans appel au modèle), `/news check` (test des sources), `/news dry-run force` (simulation) ; équivalent : `/tool run veille_news --status` ;
- en conversation, l'agent peut lancer la veille avec le paramètre `options` de son outil `run` (état, test des sources, simulation).

Le nom `veille_news` évite la confusion avec « la veille » (le jour précédent) ; le script reste `veille.py`, le cron n'est pas concerné. Les demandes du type « mes news », « mes nouvelles », « mon actualité » ou « ma veille » déclenchent l'outil.

Sans option, l'agent lance `--status` (lecture seule). Les options qui envoient un vrai digest (`--force`) ou modifient l'état (`--freq`, `--on`, `--off`, `--reset`) demandent une **confirmation**. Le passage normal reste réservé au cron.

---

## Fichiers générés

| Fichier                 | Contenu                                                                               |
|---                      |---                                                                                    |
| `veille_state.json`     | Éléments déjà vus, derniers passages, tokens du jour, surcharges, sources en échec    |
| `veille.lock`           | Verrou d'instance unique                                                              |
| `veille.log`            | Journal du cron                                                                       |
| `digests/`              | Copie de chaque digest (les 60 derniers)                                              |
| `veille_feedback.jsonl` | Avis 👍/👎 en attente, écrits par le bot, vidés par `veille.py` à chaque lecture      |
| `veille_alert.stamp`    | Date de la dernière alerte « configuration invalide » (anti-répétition, 12 h)         |

---

## Dépannage

| Symptôme                                   | Piste                                                                                                                                                                                                                                     |
|---                                         |---                                                                                                                                                                                                                                        |
| Aucun message reçu                         | `--status` (prochain passage, heures calmes), puis `--dry-run --force` ; vérifier `notify: true` et la configuration Telegram de l'agent                                                                                                  |
| Plus aucun digest depuis des jours         | Regarder `tail veille.log` : un `veille.yaml` invalide (indentation, tabulation) arrête chaque passage. Un message Telegram « ⚠ Veille arrêtée » le signale désormais (au plus une fois par 12 h). Valider : `python3 veille.py --status` |
| Une source ne répond pas                   | `--check` ; elle est signalée dans le digest après 3 échecs consécutifs                                                                                                                                                                   |
| Trop de bruit                              | Monter `min_score`, ajouter `exclude`, ou `require_keyword: true` sur les sources généralistes                                                                                                                                            |
| Rien de nouveau alors que la source publie | Vérifier `max_age_days` (global, thème ou source) et la liste `keywords`                                                                                                                                                                  |
| Pas de résumé IA                           | Voir « Quota Groq et mode dégradé » ; `--status` affiche la consommation du jour                                                                                                                                                          |
| Import impossible de `agent_groq_ng.py`    | Définir `VEILLE_AGENT_DIR` ou replacer `Veille/` à côté de l'agent                                                                                                                                                                        |
| Alertes répétées sur une page surveillée   | Page dynamique (contenu qui change à chaque lecture) : préférer un flux ou un dépôt                                                                                                                                                       |

---

## Limites

- Pas encore de commande Telegram dédiée pour relire les digests ou régler les fréquences depuis le téléphone.
- L'apprentissage ajuste des notes, pas le sens des articles : quelques avis suffisent à décaler une source de 1 à 2 points, il en faut beaucoup plus pour la faire taire.
- Les pages sans flux (`page`) détectent un changement, pas son contenu.
- Les résumés sont produits par un modèle léger : ils orientent la lecture, ils ne la remplacent pas.

---

*Auteur : Jean-François BRUNET – JFBConseils – Novembre 2026*
