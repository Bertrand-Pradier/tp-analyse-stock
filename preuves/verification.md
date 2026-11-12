# Verification du projet

Pour chaque cas : commande, sortie, explication. Les sorties sont à coller après exécution (blocs « Sortie »). Commandes en PowerShell, depuis la racine du dépôt ; image construite avec `docker build -t analyse-stock:v1 .`.

## 1. Jeu initial de quatre articles

Commande :

```
mkdir resultats -Force
docker run --rm -v "${PWD}/entrees:/entrees:ro" -v "${PWD}/resultats:/resultats" analyse-stock:v1
type resultats\bilan-stock.txt
```

Sortie attendue :

```
References : 4
Unites : 16
Valeur totale : 670.00 EUR
References en stock faible : 3
References en rupture : 1
```

Sortie obtenue : _à coller_

```
References : 4
Unites : 16
Valeur totale : 670.00 EUR
References en stock faible : 3
References en rupture : 1
```

Explication : 10×25 + 4×15 + 2×180 + 0×8 = 670.00 ; 16 unités ; 3 références avec quantité ≤ 5 (Souris, Ecran, Cable) dont 1 en rupture (Cable). Le fichier `resultats/bilan-stock.txt` est présent sur l'hôte.

## 2. CSV modifie sans reconstruction

Commande : modifier `entrees/stock.csv` (par ex. passer les Cable à 20), puis relancer la même commande `docker run` **sans** `docker build`.

Sortie : _à coller_ (références en rupture : 0, valeur et unités changées)

```
References : 4
Unites : 8
Valeur totale : 205.00 EUR
References en stock faible : 4
References en rupture : 0
```

Explication : le CSV n'est pas dans l'image mais monté depuis l'hôte, donc la nouvelle version est lue sans reconstruire.

## 3. Tentative d'ecriture dans entrees en lecture seule

Commande :

```
docker run --rm -v "${PWD}/entrees:/entrees:ro" analyse-stock:v1 sh -c "touch /entrees/test.txt"
```

Sortie attendue : `touch: cannot touch '/entrees/test.txt': Read-only file system`

Sortie obtenue : _à coller_

```
touch: cannot touch '/entrees/test.txt': Read-only file system
```

Explication : le montage `:ro` interdit toute écriture dans `/entrees` depuis le conteneur ; les données d'entrée sont protégées.

## 4. Rapport relu depuis un autre conteneur avec le meme volume nomme

Commandes :

```
docker volume create stock-resultats
docker run --rm -v "${PWD}/entrees:/entrees:ro" -v stock-resultats:/resultats analyse-stock:v1
docker run --rm -v stock-resultats:/resultats:ro analyse-stock:v1 cat /resultats/bilan-stock.txt
docker volume rm stock-resultats
```

Sortie : _à coller_ (le second conteneur affiche le même bilan)

```
References : 4
Unites : 8
Valeur totale : 205.00 EUR
References en stock faible : 4
References en rupture : 0
```

Explication : un volume nommé survit à la suppression du premier conteneur (`--rm`) et peut être monté dans un autre conteneur.

## 5. Entree absente, vide ou invalide

Commandes (PowerShell) :

```
# absente : dossier sans stock.csv
mkdir vide -Force
docker run --rm -v "${PWD}/vide:/entrees:ro" -v "${PWD}/resultats:/resultats" analyse-stock:v1
echo $LASTEXITCODE

# vide : en-tête seul
mkdir test-vide -Force
"article,quantite,prix_unitaire" | Out-File -Encoding utf8 test-vide/stock.csv
docker run --rm -v "${PWD}/test-vide:/entrees:ro" -v "${PWD}/resultats:/resultats" analyse-stock:v1

# invalide : quantité négative
mkdir test-neg -Force
"article,quantite,prix_unitaire`nClavier,-1,25.00" | Out-File -Encoding utf8 test-neg/stock.csv
docker run --rm -v "${PWD}/test-neg:/entrees:ro" -v "${PWD}/resultats:/resultats" analyse-stock:v1
```

Sorties attendues : `Erreur : [Errno 2] No such file or directory: '/entrees/stock.csv'`, `Erreur : Le fichier ne contient aucun article.`, `Erreur : Nom requis, quantite et prix non negatifs.` ; code de sortie 2 à chaque fois.

Sorties obtenues : _à coller_

```
Erreur : [Errno 2] No such file or directory: '/entrees/stock.csv'
Erreur : Le fichier ne contient aucun article.
Erreur : Nom requis, quantite et prix non negatifs.
```

Explication : le lecteur CSV refuse ces entrées avant tout calcul et n'écrit pas de rapport. Supprimer ensuite les dossiers `vide`, `test-vide` et `test-neg`.
