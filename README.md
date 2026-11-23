# AutoSense Ai

**Assistant multimodal de diagnostic documentaire pour flottes d'entreprise**

Le conducteur prend en photo un voyant du tableau de bord. L'assistant explique ce qui ne va pas, indique l'urgence et, si c'est possible, comment le régler soi-même. Sinon, il oriente vers un mécanicien, dit quoi lui dire, et peut envoyer une **fiche de panne (PDF)** directement au garage partenaire, **après approbation du conducteur**.

> Projet intégrateur d'IA générative multimodale — réalisé par **Achraf Alla**.
> Aide documentaire uniquement : le système ne pilote aucun véhicule, ne garantit pas un diagnostic et ne remplace ni les procédures de sécurité ni un mécanicien qualifié.

---

## Fonctionnalités

- **Photo annotée** : le modèle vision-langage localise le voyant et une boîte est dessinée sur l'image.
- **Carte de réponse sourcée** : ce que c'est, pourquoi, niveau d'urgence (rouge, orange, vert), actions possibles, avec document, page et figure.
- **Réparer seul ou aller au garage** : étapes de vérification guidées, sinon consignes et phrase à dire au mécanicien.
- **Fiche de panne PDF** : photo annotée, diagnostic, historique, sources, « quoi dire au mécanicien ».
- **Envoi au garage avec validation humaine** : aucun envoi sans approbation explicite du contenu exact.
- **Clarification et abstention** : photo ambiguë, véhicule inconnu ou source absente → l'assistant pose une question ou s'abstient.
- **Contrôle d'accès** : deux rôles (conducteur, gestionnaire de flotte) avec filtrage des documents.

## Parcours

```
Photo du voyant ─► Identification (VLM + RAG) ─► Carte de réponse
                                                      │
                                         Réparable seul ?
                                   oui ◄──────┴──────► non
                           Étapes guidées            Fiche de panne (PDF)
                                 │ pas résolu               │
                                 └──────────────►  Approbation du conducteur
                                                           │
                                         Envoi au garage (MCP) + gestionnaire notifié
```

## Architecture

| Couche | Composants |
|---|---|
| Interface | Gradio (lien public temporaire), connexion par rôle |
| API et sécurité | FastAPI : authentification, autorisation par rôle et groupe, quotas |
| Orchestration | LangChain + LangGraph (état, limites de boucle, pause d'approbation) |
| Modules | RAG multimodal (BGE-M3, CLIP, fusion RRF, reranker adapté) · VLM Qwen2.5-VL · outils MCP |
| Données | Qdrant (mode local), SQLite, Google Drive |

**Outils MCP** (serveur de flotte simulé)

| Outil | Type | Rôle |
|---|---|---|
| `get_vehicle_history(vehicle_id)` | lecture | Interventions passées d'un véhicule |
| `get_open_work_orders(vehicle_id)` | lecture | OT ouverts |
| `get_partner_garages(city)` | lecture | Garages partenaires (seuls destinataires autorisés) |
| `create_work_order(vehicle_id, garage_id, title, description, approval_token, idempotency_key)` | écriture sensible | Crée l'OT et transmet la fiche au garage (simulé) |

## Structure du dépôt

```
autodoc-pro/
├── src/
│   ├── ingestion/      # PyMuPDF, OCR, découpage, métadonnées
│   ├── rag/            # embeddings, Qdrant, fusion, reranking
│   ├── vlm/            # analyse de photo, boîtes annotées
│   ├── agent/          # graphe LangGraph, approbation, fiche PDF
│   ├── mcp_server/     # serveur et client MCP simulés
│   └── app/            # FastAPI + Gradio
├── prompts/            # prompts versionnés
├── tests/              # unitaires, intégration, sécurité, accès
├── notebooks/          # 00_setup … 06_demo_app
├── data/               # corpus, photos, jeux de test (NON versionné)
├── requirements.txt
├── docker-compose.yml  # exécution hors Colab
└── README.md
```

## Démarrage rapide

Le code s'écrit dans **GitHub Codespaces**, se publie sur **GitHub** et s'exécute dans **Google Colab**. Le code passe par GitHub, les données par Google Drive.

1. **Ouvrir un notebook dans Colab** (remplacer `<utilisateur>`) :

   [![Open In Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/<utilisateur>/autodoc-pro/blob/main/notebooks/00_setup.ipynb)

2. **Première cellule de chaque notebook** :

   ```python
   from google.colab import drive, userdata
   drive.mount('/content/drive')
   token = userdata.get('GITHUB_TOKEN')   # secret Colab (dépôt privé uniquement)
   !git clone https://$token@github.com/<utilisateur>/autodoc-pro.git
   %cd autodoc-pro
   !pip install -r requirements.txt
   ```

   Dépôt public : `!git clone https://github.com/<utilisateur>/autodoc-pro.git` suffit.

3. **Données** : placer les PDF, photos et jeux de test dans `MyDrive/autodoc/` (`data/`, `index/`, `models/`, `db/`). Ne jamais les committer.

4. **Mettre à jour** : modifier le code dans Codespaces, `git push`, puis relancer la cellule de clonage (ou `git pull`) et redémarrer l'exécution Colab.

## Notebooks

| Notebook | GPU | Contenu |
|---|---|---|
| `00_setup` | non | Clonage, installation, montage de Drive, `config.py` |
| `01_ingestion_rag` | non | PDF → passages → embeddings → Qdrant, RAG texte |
| `02_multimodal_vlm` | oui | VLM, boîtes annotées, figures, recherche image, fusion |
| `03_reranker_finetune` | oui | Jeu d'adaptation, entraînement, comparaison nDCG/MRR |
| `04_agent_mcp` | non | LangGraph, serveur et client MCP, approbation, fiche PDF |
| `05_evaluation` | oui | ≈ 150 scénarios, tests de sécurité et d'accès |
| `06_demo_app` | oui | Gradio + MCP + modèles dans une seule session |

> Une seule session Colab à la fois : chaque notebook sauvegarde ses sorties sur Drive avant d'être fermé.

## Configuration et secrets

Les secrets sont fournis par les **secrets Colab** (icône clé), jamais dans Git, les prompts ou les journaux.

| Secret | Usage |
|---|---|
| `GITHUB_TOKEN` | Clonage d'un dépôt privé |
| `HF_TOKEN` | Téléchargement de modèles si nécessaire |

## Corpus

30 à 50 documents autorisés sur deux familles de véhicules : manuels du conducteur, carnets d'entretien, fiches « voyant » et « vérifications simples » rédigées pour le projet, documents d'entreprise fictifs et documents de test (injection, contradiction). Chaque document a une source, une licence et un groupe d'accès (`tous`, `gestionnaire`, `test`).

Les PDF constructeurs ne sont **pas** republiés dans ce dépôt.

## Sécurité

- Documents, textes OCR et réponses d'outils traités comme des **données**, jamais comme des instructions.
- Filtrage par utilisateur et groupe avant la récupération, puis à l'affichage.
- Autorisations vérifiées côté serveur MCP à chaque appel ; envoi uniquement vers les garages partenaires enregistrés.
- Approbation liée à l'utilisateur et au contenu exact (token HMAC avec expiration) ; clé d'idempotence contre les doublons.
- Budgets : 4 appels MCP, 6 appels modèle, 60 s de traitement automatique par demande.
- Données fictives ; photos sans visage ni plaque lisible ; aucun contenu brut dans les journaux.

## Évaluation

Environ 150 scénarios (texte, photo, MCP, sources absentes, accès, injections). Indicateurs : Recall@5, nDCG@5, MRR@10, validité des citations, complétude de la fiche, latence p50/p95, fuites et envois non approuvés.

Les objectifs (par exemple ≥ 90 % de citations valides) sont des **objectifs**, pas des résultats acquis. Les résultats observés seront publiés tels quels, y compris sans gain du reranker.

## Avancement

- [x] **S1** Cadrage, corpus, architecture et analyse des risques
- [ ] **S2** Ingestion, recherche textuelle et visuelle, gestion des accès (V1)
- [ ] **S3** VLM, citations et sorties structurées (V2)
- [ ] **S4** LangGraph, serveur MCP et validation humaine (V3)
- [ ] **S5** Reranker adapté, évaluations et tests d'injection
- [ ] **S6** Déploiement, documentation et soutenance (V4)

## Limites

- Le niveau d'urgence et les conseils dépendent des documents du corpus et ne constituent pas un diagnostic garanti.
- Les couleurs et symboles des voyants varient selon le constructeur.
- La précision des boîtes annotées et la couverture des voyants sont mesurées, pas supposées.
- L'envoi au garage est simulé dans le prototype.
- La démonstration Colab n'est pas un hébergement : le lien public est éphémère.

## Auteur

**Achraf Alla** — projet intégrateur d'IA générative multimodale.
