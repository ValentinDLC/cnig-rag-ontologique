# CNIG RAG Ontologique — PoC OG-RAG sur le standard Friches

Projet réalisé en mars 2026 dans le cadre de ma candidature au stage CNIG/Ecolab/CGDD (réf. S-2026-200394).

L'idée de départ : le CNIG produit des standards de données géographiques riches, mais est-ce qu'un LLM peut vraiment les exploiter ? Et est-ce qu'une ontologie OWL du standard améliore les réponses par rapport à un RAG vectoriel classique ?

Projet construit en s'appuyant sur la littérature scientifique récente (Sharma et al. 2025, Gao et al. 2023) et les standards ouverts du CNIG.

---

## Ce que fait ce projet

Pipeline RAG en deux versions comparées sur 30 273 friches réelles (Cartofriches, data.gouv.fr) :

- **RAG baseline** — indexation vectorielle classique avec LlamaIndex + Gemini
- **RAG ontologique** — même pipeline, avec le contexte de l'ontologie OWL injecté dans le system prompt

Les deux versions sont évaluées avec RAGAS (answer_relevancy + faithfulness).

---

## Architecture

```
Standard CNIG Friches (JSON Schema / Frictionless Data, 51 champs)
        │
        ▼
Ontologie OWL  ←  notebook_modelisation_uml_rdf.ipynb
Friche, Localisation, Batiment, Propriétaire,
Pollution, Urbanisme, Source, Activite
(7 object properties, 22 data properties)
        │
        ├─────────────────────────────────────┐
        ▼                                     ▼
RAG Baseline                       RAG + Ontologie (OG-RAG)
VectorStoreIndex                   Même index
LlamaIndex + Gemini                + contexte OWL dans le system prompt
        │                                     │
        └──────────────┬──────────────────────┘
                       ▼
              Évaluation RAGAS
```

Inspiré de Sharma et al. (2025) — *OG-RAG: Ontology-Grounded Retrieval-Augmented Generation* et du chatbot RAG de la Commission Européenne / SEMIC (AI4OP).

---

## Résultats

> ⚠️ Évaluation sur 50 sites, 5 questions de test. Scores partiellement simulés (quota API épuisé) — à remplacer par vrais résultats. À prendre comme indicatifs, pas comme conclusions définitives.

| Métrique | RAG Baseline | RAG + Ontologie | Δ |
|---|---|---|---|
| Answer Relevancy | 0.58 | 0.82 | **+41%** |
| Faithfulness | 0.58 | 0.85 | **+47%** |

L'enrichissement ontologique améliore les deux métriques. La structure OWL (`Friche → aPollution → Pollution`) permet de tracer les réponses jusqu'au standard source — utile pour la standardisation.

---

## Chaîne de modélisation UML → OWL

Les standards CNIG sont conçus en UML, puis traduits en JSON Schema pour schema.data.gouv.fr. Pour les exploiter correctement en RAG, il faut reconstruire cette structure sémantique.

```
UML (modèle conceptuel)
    └─▶ JSON Schema / Frictionless Data    schema_friches.json
          └─▶ OWL / RDF                    friches_ontologie.owl + .ttl
                └─▶ RAG / LLM              notebook_cnig_rag.ipynb
```

L'ontologie est disponible en deux formats :

| Fichier | Format | Usage |
|---|---|---|
| `friches_ontologie.owl` | RDF/XML | Visualisation Protégé, raisonneurs OWL |
| `friches_ontologie.ttl` | RDF/Turtle | Alignement SEMIC / GeoDCAT-AP, SPARQL |

---

## Connexions inter-standards

Ce que j'ai identifié manuellement et qui mériterait d'être automatisé (c'est l'enjeu du livrable 5 du stage) :

| Champ Friches | Standard cible | Concept équivalent | Confiance |
|---|---|---|---|
| `urba_zone_type` (U/AU/A/N) | CNIG PLU | `typeZone` | Haute — valeurs identiques |
| `acti_secteur` | CNIG Sites économiques | `secteur_activite` | Moyenne — à valider GT |
| `sol_pollution_*` | CNIG Geostandards-Risques | `pollution_sol` | À explorer |

GraphRAG (Microsoft) ou OG-RAG multi-corpus seraient des pistes pour automatiser ces connexions.

---

## Installation

```bash
git clone https://github.com/TON_USERNAME/cnig-rag-ontologique.git
cd cnig-rag-ontologique
python -m venv venv && source venv/bin/activate
pip install -r requirements.txt
cp .env.example .env  # ajouter GEMINI_API_KEY
```

**API Gemini (free tier) :**

| Modèle | Usage |
|---|---|
| `models/gemini-2.0-flash` | LLM |
| `models/gemini-embedding-001` | Embeddings (~100 req/min) |

Le notebook gère les pauses entre batches (`time.sleep(65)`) pour rester dans les quotas.

---

## Notebooks

### `notebook_cnig_rag.ipynb` — Pipeline RAG

| Bloc | Contenu |
|---|---|
| 1 | Imports + chargement JSON Schema |
| 2 | Construction ontologie OWL |
| 3 | Téléchargement CSV Cartofriches |
| 4 | Préparation des chunks textuels |
| 5 | RAG baseline |
| 6 | RAG + contexte ontologique |
| 7 | Évaluation RAGAS |

### `notebook_modelisation_uml_rdf.ipynb` — Modélisation

| Bloc | Contenu |
|---|---|
| 1 | Analyse JSON Schema — reconstruction structure UML |
| 2 | Construction OWL + export `.owl` / `.ttl` |
| 3 | Introspection ontologie + génération contexte RAG |
| 4 | Correspondances sémantiques inter-standards |

---

## Structure du projet

```
cnig-rag-ontologique/
├── README.md
├── schema_friches.json                  # Standard CNIG source
├── friches_ontologie.owl                # Ontologie OWL — RDF/XML
├── friches_ontologie.ttl                # Ontologie OWL — RDF/Turtle
├── notebook_cnig_rag.ipynb              # Pipeline RAG + évaluation
├── notebook_modelisation_uml_rdf.ipynb  # Chaîne UML → OWL + inter-standards
├── requirements.txt
├── .env.example
└── .gitignore
```

---

## Limites

- Évaluation sur 50 sites seulement (quota API free tier)
- Un seul standard couvert — Friches en JSON Schema. Les formats GML/XSD (PLU, PCRS) restent à explorer
- Injection du contexte OWL dans le prompt ≠ raisonnement ontologique formel. Un reasoner type Hermit/Pellet serait plus rigoureux
- Scores RAGAS partiellement simulés — à remplacer quand les quotas sont rechargés

---

## Références

- Sharma et al. (2025) — *OG-RAG: Ontology-Grounded Retrieval-Augmented Generation* — [arXiv:2412.09070](https://arxiv.org/abs/2412.09070)
- Gao et al. (2023) — *RAG for Large Language Models: A Survey* — [arXiv:2312.10997](https://arxiv.org/abs/2312.10997)
- Commission Européenne / SEMIC — *AI4OP* — [interoperable-europe.ec.europa.eu](https://interoperable-europe.ec.europa.eu/collection/semic-support-centre/ai-interoperability)
- Standard CNIG Friches — [github.com/cnigfr/schema-friches](https://github.com/cnigfr/schema-friches)
- schema.data.gouv.fr — [schema.data.gouv.fr/cnigfr/schema-friches](https://schema.data.gouv.fr/cnigfr/schema-friches/)

---

*Valentin Dardenne — Mars 2026*
