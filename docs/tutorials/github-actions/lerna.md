# Build et publish un monorepo Lerna sur GitHub, avec Actions

## Cas d'usage
Le but est d'automatiser le build ainsi que le publish d'un monorepos Lerna.
Jusque là, rien de bien méchant. C'était sans compter sur ma méconnaissance d'Actions
et des subtilités de Lerna. 

En détails:

1. Récupérer un package privé présent sur un registry npmjs

2. Build et release la librairie avec Lerna

3. Publish le package sur le registry GitHub de mon organisation

4. Générer et lier un CHANGELOG pour chaque nouvelle version du package

Voici ci-dessous la méthode qui m'a permis d'arriver au bout en respectant la demande initiale.

## Actions utilisées
| Action                    | Marketplace                                    |
| ------------------------- | ---------------------------------------------- |
| `actions/checkout@v2`     | :link: [Actions Marketplace](https://github.com/marketplace/actions/checkout){:target="_blank"} |
| `actions/setup-node@v2`   | :link: [Actions Marketplace](https://github.com/marketplace/actions/setup-node-js-environment){:target="_blank"} |

## Création du workflow

A la racine de votre monorepo, ouvrez un terminal et copiez-collez le block ci-dessous :

```yaml linenums="1"
mkdir -p ./.github/workflows && 
echo '
name: Release independent Lerna libs

on:
  push:
    branches:
      - master
  pull_request:
    branches:
      - master

jobs:
  release:
    runs-on: ubuntu-20.04
    steps:

    - uses: actions/checkout@v2
      with:
        fetch-depth: 0

    - name: Setup Node with NPM Registry
      uses: actions/setup-node@v2
      with:
        node-version: '12.x'
        registry-url: 'https://registry.npmjs.org'
        always-auth: false
        scope: '@your-npm-owner'

    - name: On PR, checkout to ${GITHUB_HEAD_REF}
    - if: github.ref != 'refs/heads/master'
      run: git checkout -b ${GITHUB_HEAD_REF}

    - run: npm install
      env:
        NODE_AUTH_TOKEN: ${{secrets.NPM_TOKEN}}

    - name: Setup Node with GITHUB Registry
      uses: actions/setup-node@v2
      with:
        registry-url: 'https://npm.pkg.github.com'
        scope: '@your-github-organization'

    - name: Git Config, required by Lerna
      run: |
        git config --global user.name "${{ github.actor }}"
        git config --global user.email "${{ github.actor }}@users.noreply.github.com"

    - name: On PR, run Lerna BOOTSTRAP and BUILD
      if: github.ref != 'refs/heads/master'
      run: |
        npm run boot:libs
        npm run build:libs
    - name: On master, run Lerna BOOTSTRAP, RELEASE, BUILD and PUBLISH
      if: github.ref == 'refs/heads/master'
      run: |
        npm run boot:libs
        npm run release:libs
        npm run build:libs
        npm run publish:libs
      env:
        NODE_AUTH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
' > ./.github/workflows/cicd.yaml
```

