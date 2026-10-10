# ceffio-console

Image conteneur de la **console d'administration Ceffio**.

    ghcr.io/ceffio-suite/ceffio-console:0.3.0

C'est un **dépôt de publication**, pas un dépôt de code : la console vit dans le
dépôt principal [`ceffio-suite/ceffio`](https://github.com/ceffio-suite/ceffio),
sous `internal/console/`. Ce dépôt-ci existe pour une raison précise — rendre
l'image **adressable et versionnée**, afin qu'un déploiement puisse la tirer sans
rien reconstruire sur la machine cible.

## Pourquoi une image séparée

La console n'est pas un service autonome : elle **embarque le binaire `ceffio`**,
qui porte à la fois la page web et la logique d'administration (lecture de l'état
du cluster, gestion des comptes Keycloak). On ne peut donc pas la construire à
partir d'un simple `Dockerfile` dans son dossier — il faut le binaire complet.

Plutôt que d'extraire la console dans un dépôt de code séparé (ce qui
dupliquerait le projet, ou créerait une dépendance croisée entre deux dépôts), ce
dépôt porte **le workflow** : il va chercher les sources, construit le binaire,
l'emballe et le publie.

## Ce que fait le workflow

```
1. Construire le binaire Ceffio    depuis github.com/ceffio-suite/ceffio
2. L'emballer en image minimale    FROM scratch, binaire statique seul
3. Publier sur GHCR                ghcr.io/ceffio-suite/ceffio-console:<version>
```

L'image part d'un système **vide**. Le binaire est statique (`CGO_ENABLED=0`), il
n'a besoin d'aucune bibliothèque ; seules les autorités de certification sont
ajoutées, car la console appelle l'API Kubernetes et Keycloak en HTTPS. Pour une
console d'administration joignable en réseau, réduire la surface d'attaque à
presque rien n'est pas un détail.

## Publier une version

Par tag — c'est la voie normale, et elle garantit que le tag Git et le tag
d'image correspondent :

```bash
git tag -a v0.3.0 -m "Console 0.3.0"
git push origin v0.3.0
```

Ou manuellement, pour republier une version existante :

```bash
gh workflow run build.yml -f version=0.3.0
```

## Utilisation

L'image est prévue pour tourner **dans le cluster Ceffio**, où elle lit son état
directement de l'API Kubernetes.

```yaml
containers:
- name: console
  image: ghcr.io/ceffio-suite/ceffio-console:0.3.0
  args: ["console", "--domain", "example.com", "--addr", ":8088"]
  env:
  # Clé d'accès à la console (l'interface la demande, puis l'envoie en en-tête).
  - name: CEFFIO_KEY
    valueFrom: {secretKeyRef: {name: ceffio-console-key, key: key}}
  # Identifiants d'administration Keycloak, pour la gestion des comptes SSO.
  - name: KC_ADMIN
    valueFrom: {secretKeyRef: {name: keycloak-admin, key: username}}
  - name: KC_ADMIN_PASSWORD
    valueFrom: {secretKeyRef: {name: keycloak-admin, key: password}}
  ports:
  - {name: http, containerPort: 8088}
```

En pratique on ne l'écrit pas à la main : `ceffio install` (phase 11) déploie la
console et **importe l'image localement** quand elle n'est pas disponible dans un
registre. Cette image publiée sert aux installations qui préfèrent tirer plutôt que
construire.

## Permissions nécessaires dans le cluster

La console a besoin d'un `ServiceAccount` pouvant **lire** les pods, services,
déploiements, ingress et certificats. Elle n'écrit pas dans le cluster : le
déploiement reste l'affaire de la CLI. Une installation Ceffio crée ce rôle
automatiquement.

## Vérifier une image

Un tag qui existe ne prouve rien. Pour vérifier qu'une image est réellement
publiée et téléchargeable :

```bash
docker pull --platform linux/amd64 ghcr.io/ceffio-suite/ceffio-console:0.3.0
```

Le `--platform` est nécessaire depuis un Mac ARM : l'image cible **amd64**, comme
les serveurs Linux qu'elle équipe.

## Licence

À définir (open source — licence en attente de décision).