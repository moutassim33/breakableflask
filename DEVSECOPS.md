# Guide Docker et DevSecOps

Ce document explique ce qui a été mis en place dans ce dépôt et comment le
vérifier sans se perdre dans les commandes.

## 1. Le projet

`breakableflask` est une application Flask volontairement vulnérable. Elle
sert de support pédagogique pour observer des vulnérabilités applicatives,
notamment l'injection de commandes, l'injection SQL et la désérialisation
Python.

Le durcissement présenté ici concerne surtout **l'image Docker et la chaîne
CI/CD**. Il ne supprime pas les vulnérabilités présentes dans l'application.

## 2. Les fichiers importants

| Fichier | Rôle |
| --- | --- |
| `main.py` | Code de l'application Flask |
| `requirements.txt` | Dépendances Python |
| `Dockerfile` | Instructions de construction de l'image |
| `.github/workflows/devsecops.yml` | Pipeline GitHub Actions |

## 3. Construire et lancer l'application localement

Depuis la racine du dépôt :

```bash
docker build -t breakableflask:1.0 .
docker run --rm --name breakableflask -p 4000:4000 breakableflask:1.0
```

L'option `-p 4000:4000` relie le port 4000 de la machine au port 4000 du
conteneur. L'application est alors accessible uniquement sur cette machine :

<http://localhost:4000>

Cette adresse n'est pas une URL publique. Pour rendre l'application accessible
sur Internet, il faut la déployer chez un fournisseur comme Render, Railway,
Fly.io ou sur une machine virtuelle.

Pour arrêter l'application lancée au premier plan, utiliser `Ctrl+C`.

## 4. Ce que sécurise le Dockerfile

Le [Dockerfile](Dockerfile) :

- utilise une image Python récente et légère ;
- installe les dépendances avant le code afin de profiter du cache Docker ;
- désactive le cache pip dans l'image finale ;
- exécute l'application avec l'utilisateur non-root `app` ;
- vérifie régulièrement que l'application répond avec un `HEALTHCHECK` ;
- expose le port 4000 utilisé par Flask.

L'image est construite avec des privilèges root, ce qui est normal pour
installer les dépendances. En revanche, le processus de l'application est
lancé avec l'utilisateur `app`.

## 5. À quoi sert Trivy ?

Trivy est un outil d'analyse de sécurité. Dans ce projet, il est utilisé de
deux manières :

1. **Scan des dépendances** : recherche de vulnérabilités connues dans les
   dépendances décrites par `requirements.txt`.
2. **Scan de configuration** : vérification des bonnes pratiques du
   `Dockerfile`.

Le pipeline est configuré pour échouer si une vulnérabilité de niveau `HIGH`
ou `CRITICAL` est détectée. `ignore-unfixed: true` est utilisé uniquement pour
le scan des dépendances, afin d'ignorer les vulnérabilités pour lesquelles
aucun correctif n'est disponible.

## 6. Comprendre le pipeline GitHub Actions

Le fichier
[`.github/workflows/devsecops.yml`](.github/workflows/devsecops.yml) est
déclenché lors d'un `push` ou d'une `pull_request`.

Les jobs s'exécutent dans cet ordre :

```text
Dependances -> Dockerfile -> Deploy
```

### Job `Dependances`

- récupère le dépôt ;
- lance Trivy sur le dépôt, ce qui permet d'analyser `requirements.txt` ;
- bloque la suite si une vulnérabilité `HIGH` ou `CRITICAL` est détectée.

### Job `Dockerfile`

Ce job dépend du succès de `Dependances` grâce à :

```yaml
needs: Dependances
```

Il lance Trivy en mode `config` sur le `Dockerfile`.

### Job `Deploy`

Ce job ne déploie pas encore l'application. Il affiche simplement un message
pour montrer que les contrôles sont passés. Un vrai déploiement devra être
ajouté séparément avec les secrets et la configuration du fournisseur choisi.

## 7. Lire le résultat dans GitHub

1. Ouvrir le dépôt GitHub.
2. Aller dans l'onglet **Actions**.
3. Sélectionner le workflow **Automatisation des tests DevSecOps**.
4. Ouvrir le run correspondant au commit.

Un résultat vert signifie que les trois jobs ont réussi. Un résultat rouge
signifie qu'un job a bloqué la chaîne. Les jobs qui utilisent `needs` sont
alors marqués comme ignorés.

## 8. Commandes Git utiles

Vérifier l'état local :

```bash
git status
```

Voir les derniers commits :

```bash
git log --oneline -5
```

Envoyer les modifications sur GitHub :

```bash
git add Dockerfile .github/workflows/devsecops.yml DEVSECOPS.md README.md
git commit -m "Document Docker and DevSecOps workflow"
git push origin master
```

Le `push` déclenche automatiquement un nouveau workflow.

## 9. Dépannage rapide

### Le pipeline échoue avant l'exécution des jobs

Vérifier les références des actions GitHub, par exemple :

```yaml
uses: aquasecurity/trivy-action@v0.36.0
```

La version doit exister comme tag dans le dépôt de l'action.

### Le job `Dockerfile` est ignoré

Cela signifie généralement que le job `Dependances` a échoué. Il faut ouvrir
les logs de `Dependances` en premier.

### Trivy signale une vulnérabilité

Mettre à jour la dépendance concernée dans `requirements.txt`, reconstruire
l'image, puis pousser la modification. Si aucune correction n'existe, le
résultat dépend de la politique choisie pour `ignore-unfixed`.

### Le conteneur démarre mais le navigateur ne répond pas

Vérifier que le port est publié :

```bash
docker ps
docker logs breakableflask
```

Relancer avec :

```bash
docker run --rm -p 4000:4000 breakableflask:1.0
```

## 10. Ce qui reste à faire

Le projet dispose maintenant d'un contrôle automatique, mais pas encore d'un
déploiement réel. Les prochaines améliorations possibles sont :

1. figer précisément les versions de `requirements.txt` ;
2. ajouter un scan de l'image Docker construite ;
3. publier l'image dans un registre ;
4. configurer un vrai déploiement ;
5. corriger séparément les vulnérabilités applicatives volontaires si
   l'objectif devient de produire une application sécurisée.
