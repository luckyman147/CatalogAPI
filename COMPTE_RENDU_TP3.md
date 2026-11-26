# TP n°3 – DevOps : Compte Rendu

## Q1. Stratégie de Test (Unitaires vs Intégration)

| Critère | Tests Unitaires (`InMemoryDatabase`) | Tests d'Intégration (`SQL Server` Docker) |
| :--- | :--- | :--- |
| **Temps d'exécution** | Très rapide (quelques millisecondes/secondes) | Plus lent (démarrage du conteneur SQL Server + requêtes réseau) |
| **Isolation** | Totale : chaque test utilise sa propre base en mémoire (`Guid.NewGuid()`) | Partagée avec une instance réelle de SGBD |
| **Dépendances externes** | Aucune (pas de moteur SQL Server requis) | Dépend d'un conteneur SQL Server actif sur le port `1433` |

**Pourquoi ne peut-on pas se contenter uniquement de tests unitaires ?**  
Le fournisseur `InMemoryDatabase` d'EF Core ne simule pas fidèlement le comportement d'un vrai SGBD relationnel (contraintes SQL, précision décimale `HasPrecision(10, 2)`, longueur maximale `HasMaxLength(30)`, transactions, chaînes de connexion, ou génération d'IDs par SQL Server). Les tests d'intégration sont donc indispensables pour valider le comportement réel de bout en bout avant la mise en production.

---

## Q2. Service Containers dans GitHub Actions

Le bloc `services:` permet à GitHub Actions de démarrer un conteneur Docker compagnon (ici `mcr.microsoft.com/mssql/server:2022-latest`) sur le même réseau que le runner du job, de manière analogue à `compose.yaml` en local.

La directive de **healthcheck** (`--health-cmd "/opt/mssql-tools18/bin/sqlcmd ... -Q 'SELECT 1'"`, `--health-interval 15s`, `--health-timeout 5s`, `--health-retries 6`) garantit que GitHub Actions attend que le moteur SQL Server soit réellement prêt à accepter des connexions avant de lancer les étapes (`steps`) du job. Cela évite les échecs intermittents dus à une API qui tenterait de se connecter pendant l'initialisation de la base.

---

## Q3. Justification de l'Architecture Multi-Jobs

Séparer `unit-tests` et `integration-test` en deux jobs reliés par `needs: [unit-tests]` offre deux avantages majeurs :
1. **Fail-Fast & Économie de ressources** : si une erreur de compilation ou de logique métier est détectée par les tests unitaires, le pipeline s'arrête immédiatement sans gaspiller de temps ni de ressources à démarrer le conteneur SQL Server.
2. **Lisibilité & Modularité** : les logs, rapports TRX et durées d'exécution sont clairement isolés par niveau de test dans l'interface GitHub Actions.

**En cas d'échec du Job 1 (`unit-tests`) :**  
Le Job 2 (`integration-test`) — ainsi que le Job 3 (`build-and-push-docker`) — est **automatiquement ignoré (`skipped`)** par GitHub Actions grâce à la dépendance `needs: [unit-tests]`.

---

## Q4. Allocation des Ressources & Docker

Le conteneur Docker SQL Server est instancié **uniquement dans le Job 2 (`integration-test`)**.  
L'éviter au niveau global ou dans le Job 1 (`unit-tests`) permet de ne pas alourdir ni ralentir l'exécution des tests unitaires (qui utilisent `InMemoryDatabase` et n'ont besoin d'aucune base externe) et de ne consommer de la mémoire/CPU pour SQL Server que lorsque les tests unitaires ont déjà été validés.

---

## Q5. Directives de flux (`if: always()`)

L'étape `actions/upload-artifact@v4` (tout comme `dorny/test-reporter@v1`) doit comporter `if: always()` car, par défaut dans GitHub Actions, lorsqu'une étape précédente échoue (ici `dotnet test` qui détecte une régression), toutes les étapes suivantes du job sont ignorées.

Si `if: always()` était omise et qu'un test échouait, **aucun rapport `.trx` ne serait uploadé ni publié**, privant le développeur des détails de l'échec pour diagnostiquer la régression.

---

## Q6. Directive « `if` » dans le Job 3

```yaml
if: github.ref == 'refs/heads/main' && github.event_name == 'push'
```

Cette condition garantit que la construction et la publication de l'image Docker sur Docker Hub ne s'exécutent **que lorsqu'un `push` (ou une fusion de PR) a lieu sur la branche principale `main`**. Ainsi, les Pull Requests ou les commits sur d'autres branches sont bien vérifiés par les tests (Jobs 1 et 2), mais ne publient jamais d'image non validée sur le registre de production.

---

## Q7. Piliers de CALMS implémentés dans ce TP

- **Automation (Automatisation)** : exécution automatique de la compilation, des tests unitaires, des tests d'intégration et du build/push Docker via GitHub Actions.
- **Measurement (Mesure)** : génération de rapports standardisés `.trx`, publication de résumés visuels (`dorny/test-reporter`) et métriques d'exécution des pipelines.
- **Culture & Sharing (Culture & Partage)** : validation systématique sur chaque Pull Request et partage de résultats transparents avec l'équipe.
- **Lean** : architecture multi-jobs *fail-fast* évitant de lancer des conteneurs lourds si les tests rapides échouent, et séparation stricte des secrets via GitHub Encrypted Secrets.
