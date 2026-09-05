# app-kit — CI partagé des applications Capacitor

Workflows GitHub Actions réutilisables (`workflow_call`) appelés par les dépôts
d'applications. Objectif : une seule chaîne de build à maintenir au lieu d'une
copie divergente par app (elles allaient de 79 à 134 lignes, toutes légèrement
différentes).

## Utilisation depuis une app

`.github/workflows/build-apk.yml` du dépôt de l'app :

```yaml
name: Build Android APK
on:
  push:
    branches: [ main, master ]
  workflow_dispatch:

jobs:
  build:
    # Obligatoire : un workflow appelé ne peut pas s'octroyer plus de droits
    # que son appelant. Sans ce bloc, le run échoue au parsing avec
    # « nested job is requesting contents: write, but is only allowed
    # contents: read » — la publication de la release a besoin de l'écriture.
    permissions:
      contents: write
    uses: laurentsar/app-kit/.github/workflows/build-apk.yml@v1
    with:
      apk_name: bornes-ve
      release_name: Bornes VE
    secrets: inherit
```

Entrées disponibles : `apk_name`, `release_name`, `regen_android`,
`ci_scripts` (scripts `ci/*.py` à jouer dans l'ordre), `pillow`, `ndk`,
`node_version`, `java_version`.

Les noms d'entrées sont en `snake_case` volontairement : dans une expression,
`${{ inputs.apk-name }}` est lu par GitHub comme une soustraction et fait
échouer le workflow au parsing, avant tout job (`startup_failure`).

`secrets: inherit` transmet `ANDROID_KEYSTORE_B64` et
`ANDROID_KEYSTORE_PASSWORD`, qui doivent exister dans le dépôt appelant.

## Ce dépôt ne contient aucune clé

Les keystores et leurs mots de passe vivent uniquement dans `~/app-kit/keys`
(poste local) et dans les secrets GitHub de chaque dépôt. C'est précisément
pourquoi ce dépôt-ci est séparé de `~/app-kit` : ce dernier contient les clés
et ne doit jamais être poussé.

Ce dépôt est public parce qu'un workflow réutilisable hébergé dans un dépôt
privé ne peut pas être appelé par un dépôt public.

## Versionnement

Les apps épinglent un tag (`@v1`). Un changement de comportement se publie en
`v2` : les apps migrent une par une, et un build déjà vert ne casse pas parce
qu'on a touché ici. Après chaque migration, vérifier le certificat de l'APK
produit (`~/app-kit/tools/verify_apk_cert.py`) — un build vert signé de la
mauvaise clé donne une mise à jour qu'Android refuse d'installer.
