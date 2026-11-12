# Analyse de stock avec Docker

TP 01 – R5.A.09 Virtualisation avancée (2026-2027).
Analyseur CSV de stock livré dans une image Ubuntu 24.04 avec Python 3. Il calcule la valeur du stock et repère les articles à réapprovisionner. Toutes les données du dépôt sont fictives.

## Auteurs

- Bertrand Pradier

## Construction

Depuis la racine du dépôt :

```
docker build -t analyse-stock:v1 .
```

Le `Dockerfile` :

- `FROM ubuntu:24.04` : image de base imposée.
- `RUN apt-get update && apt-get install -y --no-install-recommends python3 && rm -rf /var/lib/apt/lists/*` : `RUN` exécute la commande pendant le *build*. Tout est dans une seule couche, et la suppression du cache apt dans cette même couche allège l'image.
- `WORKDIR /app` : dossier de travail.
- `COPY stock.py .` : seul le programme est copié. `entrees` et les rapports ne sont pas dans l'image (le CSV arrive par montage), et `.git` en est exclu par `.dockerignore`.
- `CMD ["python3", "stock.py"]` : programme exécuté par défaut. `CMD` (et non `ENTRYPOINT`) permet de lancer une autre commande dans le conteneur pour les tests (`sh`, `cat`…).

## Execution avec montages

Le programme lit `/entrees/stock.csv` et écrit `/resultats/bilan-stock.txt` (chemins par défaut dans le conteneur). Deux montages relient ces chemins au poste hôte :

| Hôte | Conteneur | Mode | Rôle |
|---|---|---|---|
| `./entrees` | `/entrees` | lecture seule (`:ro`) | fournir le CSV sans le copier dans l'image |
| `./resultats` | `/resultats` | lecture/écriture | récupérer le bilan sur l'hôte |

PowerShell (Windows) :

```
mkdir resultats -Force
docker run --rm -v "${PWD}/entrees:/entrees:ro" -v "${PWD}/resultats:/resultats" analyse-stock:v1
```

Bash (Linux / macOS / Git Bash) :

```
mkdir -p resultats
docker run --rm -v "$(pwd)/entrees:/entrees:ro" -v "$(pwd)/resultats:/resultats" analyse-stock:v1
```

Le bilan s'affiche et est aussi écrit dans `resultats/bilan-stock.txt` sur l'hôte. `--rm` supprime le conteneur à la fin ; les données restent sur l'hôte grâce au montage. Le CSV peut être modifié sans reconstruire l'image.

## Calculs et vérifications

- `references` : nombre d'articles.
- `unites` : somme des quantités.
- `valeur` : somme de `quantite × prix_unitaire`, en `Decimal` (pas d'erreur d'arrondi des flottants), affichée avec 2 décimales.
- `faibles` : articles dont la quantité est `<= 5` (les ruptures sont comprises).
- `ruptures` : articles dont la quantité est `0`.

Le lecteur CSV valide le fichier avant tout calcul ; en cas de problème, il affiche `Erreur : …` sur la sortie d'erreur et le programme se termine avec le code `2`, sans écrire de rapport :

- **Colonnes incorrectes** : l'en-tête doit être exactement `article,quantite,prix_unitaire`, pour ne pas interpréter des données décalées.
- **Valeurs négatives** : une quantité ou un prix négatif est absurde pour un stock et fausserait la valeur totale.
- **Nom vide, quantité non entière, prix non décimal ou non fini** : refusés (nom requis, `int` et `Decimal` valides, `NaN`/`Infinity` exclus).
- **Fichier sans article** : un bilan sur zéro article serait trompeur.
- **Fichier absent** : erreur système signalée proprement.

## Tests et resultats

Les cas testés (jeu initial, CSV modifié sans reconstruction, entrée en lecture seule, volume nommé relu depuis un autre conteneur, entrée absente / vide / invalide) sont décrits avec leurs commandes et sorties dans [`preuves/verification.md`](preuves/verification.md).

Résultat attendu avec `entrees/stock.csv` :

```
References : 4
Unites : 16
Valeur totale : 670.00 EUR
References en stock faible : 3
References en rupture : 1
```

## Reproduction apres git clone

```
git clone <url-du-depot> tp-analyse-stock
cd tp-analyse-stock
docker build -t analyse-stock:v1 .
```

Le dossier `resultats/` existe après le clone grâce à `resultats/.gitkeep` (son contenu est ignoré par Git). S'il manque, le créer : `mkdir resultats` (PowerShell : `mkdir resultats -Force`). Puis lancer l'une des commandes de la section « Execution avec montages » et lire `resultats/bilan-stock.txt`.
