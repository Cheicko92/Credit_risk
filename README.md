# CreditRisk-Test

Stratégie de test complète d'un système d'estimation du risque de défaut de remboursement (microfinance) :
API FastAPI, modèle scikit-learn de test versionné, tests pytest, GitHub Actions.

> ⚠️ **Données synthétiques.** Le jeu de données et le modèle servent uniquement à **tester le système**. Ils ne
> permettent pas de démontrer la performance réelle d'un modèle de crédit, ni son équité ou sa conformité légale.

## Structure

```
CreditRisk-Test/
├── app/                 # validation, prétraitement, prédiction, API, outils de non-régression
├── models/              # modèle de référence versionné + métadonnées + métriques de référence
├── scripts/             # entraînement, prédictions golden, mesures de performance, génération de rapports
├── tests/               # validation, API, intégration, robustesse, non-régression, performance, canari CI
│   ├── data/            # holdout synthétique, prédictions golden
│   └── performance/     # scénario Locust
├── reports/             # plan de test, anomalies, non-régression, performance, qualification
├── .github/workflows/tests.yml
├── pyproject.toml       # configuration pytest, Ruff, coverage
└── requirements.txt     # versions figées
```

## Installation (environnement reproductible)

```bash
python -m venv .venv
source .venv/bin/activate          # Windows : .venv\Scripts\activate
pip install -r requirements.txt
```

Python 3.13 recommandé (version utilisée par la CI). Les versions des bibliothèques sont figées, car le modèle
`.joblib` dépend de la version de scikit-learn.

## Commandes

| Action | Commande |
|---|---|
| Lancer l'API | `uvicorn app.main:app --reload` (documentation interactive : http://127.0.0.1:8000/docs) |
| Tous les tests | `pytest` |
| Tests + couverture | `pytest --cov=app --cov-branch --cov-report=term-missing` |
| Rapport JUnit + table des résultats | `pytest --junitxml=reports/junit.xml && python scripts/generer_resultats.py` |
| Qualité du code | `ruff check .` |
| Famille de tests | `pytest tests/test_api.py` (ou `test_integration.py`, `test_robustness.py`, `test_regression.py`) |
| Mesures de latence / P95 / débit | `python scripts/measure_performance.py --n 1000` |
| Test de charge progressive | voir ci-dessous |
| Rapport de non-régression | `python scripts/rapport_non_regression.py` |
| Régénérer modèle + holdout (décision volontaire) | `python scripts/train_model.py` puis `python scripts/generate_golden.py` |

Test de charge Locust :

```bash
uvicorn app.main:app --port 8000 &
locust -f tests/performance/locustfile.py --headless --host http://127.0.0.1:8000 \
       --csv reports/locust --run-time 45s --only-summary
```

## Contrat de l'API

| Endpoint | Rôle | Codes |
|---|---|---|
| `GET /health` | disponibilité du service | 200 ; 503 si modèle non chargé |
| `GET /model-info` | métadonnées du modèle (version, variables, SHA-256, plages d'entraînement) | 200 ; 503 |
| `POST /predict` | estimation du risque | 200 ; 400 JSON mal formé ; 422 données invalides ou incomplètes ; 503 modèle indisponible ; 500 erreur interne (message générique) |

Exemple :

```bash
curl -X POST http://127.0.0.1:8000/predict -H "Content-Type: application/json" \
  -d '{"age":35,"revenu_mensuel":250000,"montant_credit":1000000,"duree_mois":12,"anciennete_activite":4}'
```

Réponse : `{"probabilite_defaut": 0.4469, "niveau_risque": "modere", "alertes": [], "version_modele": "v1"}`.

Niveaux de risque : `faible` (< 0,30), `modere` (0,30 à 0,60), `eleve` (≥ 0,60). Ces seuils sont des valeurs de
démonstration et non une politique de crédit.

## Intégration continue

Le workflow `.github/workflows/tests.yml` s'exécute à chaque Pull Request, à chaque push sur `main` et à la demande :
installation des dépendances, **Ruff**, **pytest** avec **pytest-cov** (seuil 90 %), publication des rapports JUnit et
de couverture.

**Vérifier qu'un échec fait échouer le workflow** : onglet *Actions* → *Tests* → *Run workflow* → cocher
`force_failure`. Le test canari `tests/test_ci_canary.py` échoue volontairement et le job est marqué en échec.
En local : `FORCE_CI_FAILURE=1 pytest` (code de sortie 1).

## Rapports

- `reports/plan_de_test.md` : exigences, matrice de traçabilité, scénarios, priorités
- `reports/resultats_tests.md` : résultats d'exécution
- `reports/registre_anomalies.md` : anomalies, gravité, corrections
- `reports/rapport_non_regression.md` : référence contre versions candidates
- `reports/rapport_performance.md` : latence, P95, débit, charge progressive
- `reports/rapport_qualification.md` : décision argumentée
