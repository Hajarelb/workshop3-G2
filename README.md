# Workshop 3 : Sécurité des dépendances (SCA)

**Dependabot & OWASP Dependency-Check**

- **Module** : DevSecOps, Master DevOps et Cloud Computing
- **Université** : Abdelmalek Essaâdi, FP Larache
- **Groupe** : G2
- **Membres** :
  - Hajar EL BACHIRI
  - Keltouma ACHGUER
  - Mohamed El Amine HOUMADA
- **Projet analysé** : [WebGoat](https://github.com/WebGoat/WebGoat), application Java/Maven volontairement vulnérable (OWASP)

## 1. Objectif

Sécuriser les dépendances d'un projet Maven en utilisant :
- **Dependabot** : détection des dépendances vulnérables et mises à jour automatiques via des Pull Requests.
- **OWASP Dependency-Check** : analyse SCA qui compare les dépendances à la base NVD pour détecter les CVE.

Ces outils s'intègrent dans un pipeline CI/CD (GitHub Actions) pour appliquer le principe du *Shift Left*.

## 2. Définitions

|         Terme          |                                  Définition                                  |
|------------------------|------------------------------------------------------------------------------|
| Dépendance             | Bibliothèque externe utilisée par le projet, déclarée dans `pom.xml`         |
| CVE                    | Identifiant public d'une faille de sécurité connue                           |
| CVSS                   | Score de gravité d'une CVE, de 0 à 10                                        |
| SCA                    | Software Composition Analysis : analyse des composants tiers                 |
| Dependabot             | Outil GitHub qui alerte et propose des mises à jour de dépendances           |
| OWASP Dependency-Check | Outil SCA qui détecte les dépendances ayant des CVE connues                  |
| Pull Request (PR)      | Proposition de modification à relire avant fusion dans la branche principale |

## 3. Environnement de travail

Outils utilisés :
- Git, GitHub (dépôt + GitHub Actions)
- Java 25 (version requise par WebGoat), installé automatiquement par le workflow GitHub Actions
- Éditeur : Visual Studio Code

![Versions des outils](images/01-outils.png)

## 4. Mise en place du dépôt

Le dépôt `workshop3-G2` a été créé vide sur GitHub, puis alimenté avec le code de WebGoat (clone, changement de remote, push).

![Dépôt vide](images/02-depot-vide.png)
![Code de WebGoat poussé](images/03-depot-webgoat.png)
## 5. Pipeline hérité de WebGoat et problème rencontré

WebGoat fournit déjà des workflows GitHub Actions (`.github/workflows/build.yml`, etc.) et un fichier `.github/dependabot.yml`. Ils s'exécutent automatiquement à chaque push.

À notre premier push, le job **Pre-commit check** a échoué : l'outil **Spotless** contrôle aussi le formatage des fichiers Markdown et notre `README.md` n'était pas conforme. Le job **build** a alors été ignoré (*skipped*).

![Pre-commit en échec](images/04-pre-commit-echec.png)

Correction : nous avons lancé `mvn spotless:apply` en local, qui a reformaté le README, puis poussé le résultat. Le pipeline passe au vert : le projet compile sans erreur avec Java 25.

![Pipeline réussi](images/05-pipeline-vert.png)

> Règle retenue : avant chaque push qui modifie le README, exécuter `mvn spotless:apply`.

## 6. Dependabot : configuration d'origine

Le projet WebGoat contenait **déjà** un fichier `.github/dependabot.yml`. Avant de le modifier, nous l'avons analysé pour comprendre son fonctionnement.

![dependabot.yml d'origine](images/06-dependabot-yml-origine.png)

| Paramètre | Rôle |
|---|---|
| `version: 2` | Version du format de configuration |
| `package-ecosystem` | Type de dépendances à surveiller |
| `directory` | Dossier où se trouve le fichier de dépendances (`/` = racine) |
| `schedule.interval` | Fréquence de vérification (ici `weekly`, une fois par semaine) |

Trois écosystèmes sont surveillés :

| Écosystème | Ce qui est surveillé |
|---|---|
| `maven` | Les dépendances Java du `pom.xml` |
| `github-actions` | Les actions utilisées dans les workflows |
| `docker` | L'image de base du `Dockerfile` |

Cette configuration était déjà active : elle a généré les alertes et les Pull Requests présentées dans les sections suivantes.
## 7. Dependabot Alerts : état initial

Nous avons activé **Dependency graph** et **Dependabot alerts** dans *Settings → Advanced Security*. GitHub détecte **36 alertes** (de gravité High et Moderate) dans les dépendances du projet.

![Alertes avant correction](images/07-alertes-avant.png)

## 8. Pull Requests de Dependabot

Grâce au `dependabot.yml` existant, Dependabot a ouvert **7 PRs** de mise à jour. Chacune comporte une description et déclenche les vérifications du pipeline.

![Liste des PRs](images/08-liste-prs.png)

![Détail d'une PR](images/09-pr-detail.png)

### Analyse

| PR | Mise à jour | Vérifications | Décision |
|---|---|---|---|
| #1 | `java-jwt` 4.6.1 → 4.6.2 | OK | Fusionnée |
| #2 | `docker/setup-qemu-action` 4.2.0 → 4.4.0 | OK | Fusionnée |
| #3 | `jjwt` 0.9.1 → 0.13.0 | **Échec** | Non fusionnée |
| #4 | `docker/build-push-action` 7.3.0 → 7.4.0 | OK | Fusionnée |
| #5 | `rest-assured` 6.0.1 → 6.2.0 | OK | Fusionnée |
| #6 | `bootstrap` 5.3.5 → 5.3.8 | OK | Fusionnée |
| #7 | `bcprov-jdk18on` 1.85 → 1.86 | OK | Fusionnée |

La PR #3 est une mise à jour **majeure** (0.9.1 → 0.13.0) : l'API de la bibliothèque a changé et le pipeline échoue. Elle nécessite une adaptation du code et n'a donc pas été fusionnée automatiquement.

![Échec de la PR jjwt](images/10-pr-jjwt-echec.png)

Les autres PRs, validées par le pipeline, ont été fusionnées une par une.

Après la fusion de la PR #6, la PR #7 a présenté un **conflit** sur le `pom.xml` : les deux PRs modifient des lignes voisines (versions de `bootstrap` et de `bouncycastle`).

![Conflit sur la PR #7](images/11b-conflit-pr7.png)

Pour le résoudre, nous avons commenté `@dependabot rebase` dans la PR. Dependabot a recalculé la branche à partir du `main` à jour (`force-push`), ce qui a supprimé le conflit et relancé les vérifications du pipeline. GitHub affiche alors **No conflicts with base branch**.

![Rebase de la PR #7 et vérifications relancées](images/11c-rebase-pr7.png)

Une fois les vérifications au vert, la PR a été fusionnée.

![PR fusionnée](images/11-pr-fusionnee.png)

## 9. Résultat après fusion

Après les fusions, le nombre d'alertes Dependabot **n'a pas changé** : il reste de **36** alertes ouvertes (0 fermée).

![Alertes après fusion](images/12-alertes-apres.png)

Les alertes portent uniquement sur la bibliothèque **XStream** (exécution de code à distance, injection de commandes, déni de service, SSRF), qu'aucune des PRs fusionnées ne mettait à jour.

Les PRs de *version updates* maintiennent les dépendances à jour, mais ne ciblent pas spécifiquement les dépendances vulnérables. Pour cela, il faut activer les **Dependabot security updates** (voir section 10).
