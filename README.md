Idée globale

Dockerfile : décrit comment construire ton image

Image : contient Nginx + tes fichiers web

Container : exécute l’image et sert le site sur un port

Ici, on va utiliser Nginx comme serveur web.

2) Créer le Dockerfile

Dans ce dossier, crée un fichier nommé Dockerfile (sans extension) avec :

# 1) On part d'une image Nginx légère
FROM nginx:alpine

# 2) On copie ton site (PC) dans le dossier web de Nginx (container)
COPY . /usr/share/nginx/html

# 3) (Optionnel mais propre) Nginx écoute déjà sur 80
EXPOSE 80

Ce que ça veut dire

FROM nginx:alpine : “je veux Nginx déjà prêt”

COPY . ... : “copie tout le dossier dans le container”

EXPOSE 80 : “documente que le container écoute sur 80” (ça n’ouvre pas le port tout seul, c’est juste informatif)

⚠️ Petite règle : COPY . copie tout, donc plus tard on mettra souvent un .dockerignore pour éviter de copier des trucs inutiles (node_modules, etc.). Là, pas besoin.

3) Construire l’image (build)

Toujours dans le dossier docker-html, dans Git Bash :

docker build -t mon-site:1.0 .


Décryptage :

docker build = construire une image

-t mon-site:1.0 = nom + version (tag)

. = “le contexte = le dossier actuel” (Dockerfile + fichiers)

4) Lancer le container (run)
docker run --name site-test -p 8080:80 -d mon-site:1.0


Décryptage :

--name site-test = nom du container

-p 8080:80 = PC:8080 → container:80

-d = arrière-plan

mon-site:1.0 = l’image à exécuter

Puis ouvre :
http://localhost:8080

5) Arrêter / supprimer (proprement)
docker stop site-test
docker rm site-test

6) Point IMPORTANT (qui surprend toujours)

Avec cette méthode COPY, si tu modifies index.html sur ton PC :

le container ne voit pas le changement automatiquement
👉 il faut rebuild puis relancer.

Workflow :

docker stop site-test
docker rm site-test
docker build -t mon-site:1.0 .
docker run --name site-test -p 8080:80 -d mon-site:1.0


*******DOCKERFILE + BUILD + RUN********************
2) Crée le Dockerfile

Dans ce même dossier, crée un fichier Dockerfile :

touch Dockerfile


Puis ouvre VS Code :

code .


Dans Dockerfile, colle :

FROM nginx:alpine
COPY . /usr/share/nginx/html
EXPOSE 80


✅ Ça veut dire : “je pars de Nginx, je copie mon site dedans”.

3) Construis l’image

Toujours dans ce dossier :

docker build -t docker-html:1.0 .


-t = nom de l’image

. = “utilise le Dockerfile et les fichiers du dossier actuel”

4) Lance le container
docker run --name site-test -p 8080:80 -d docker-html:1.0


Puis ouvre :
http://localhost:8080

5) Stop / nettoyage

Quand tu veux arrêter :

docker stop site-test
docker rm site-test

Mini point important (à retenir)

Avec ce Dockerfile (COPY . …), si tu modifies index.html, il faudra rebuild l’image pour voir le changement.

**************************************************************
1️⃣ CMD / ENTRYPOINT : à quoi ça sert et pourquoi on ne l’a pas encore vu
2️⃣ Pourquoi ton image Nginx marche sans CMD
3️⃣ Le “build automatique depuis Git” : la vraie logique (sans Dockerfile magique)

1️⃣ CMD et ENTRYPOINT — enfin expliqués clairement

👉 CMD et ENTRYPOINT servent à dire :

« Quand un container démarre, qu’est-ce qu’il doit lancer ? »

⚠️ Important :

RUN = pendant le build

CMD / ENTRYPOINT = au démarrage du container

CMD (le plus simple)

Exemple basique :

CMD ["nginx", "-g", "daemon off;"]


👉 Traduction humaine :

« Quand le container démarre, lance nginx et reste au premier plan »

📌 CMD :

peut être remplacé au docker run

sert souvent de commande par défaut

ENTRYPOINT (plus strict)

Exemple :

ENTRYPOINT ["nginx", "-g", "daemon off;"]


👉 Là :

impossible de le remplacer facilement

le container a toujours le même rôle

📌 ENTRYPOINT = “ce container sert à faire UNE chose”

CMD + ENTRYPOINT ensemble (niveau un peu plus pro)
ENTRYPOINT ["nginx"]
CMD ["-g", "daemon off;"]


👉 ENTRYPOINT = le programme
👉 CMD = les options par défaut

2️⃣ Question clé : pourquoi on n’a PAS mis de CMD avec Nginx ?

👉 Parce que l’image officielle nginx:alpine a déjà un ENTRYPOINT + CMD.

Quand tu écris :

FROM nginx:alpine


Tu hérites de :

son ENTRYPOINT

son CMD

sa logique de démarrage

👉 C’est pour ça que :

tu n’as rien écrit

et pourtant Nginx démarre

📌 Très important à retenir :

Si tu ne redéfinis pas CMD/ENTRYPOINT, Docker garde ceux de l’image de base.

Quand TOI tu devras écrire un CMD

image Node

image PHP CLI

script custom

container utilitaire

Exemple :

FROM node:18-alpine
CMD ["node", "server.js"]

3️⃣ Maintenant le gros sujet : “build automatique depuis Git”

Et là, je te corrige une idée très importante (sans te casser l’élan).

❌ Ce qu’on NE fait PAS

un Dockerfile qui “clone le repo à chaque run”

un container qui refait un git pull tout seul

👉 Pourquoi ?

pas fiable

pas reproductible

casse le principe d’image figée

✅ Ce qu’on fait EN VRAI (logique DevOps)

👉 Le build se fait quand le code change, pas quand le container démarre.

Le cycle pro, c’est :

Code → Git push → Build image → Image stockée → Déploiement

4️⃣ Concrètement, avec ton repo Git
Étape A — tu crées ton repo (ce que tu veux faire)

GitHub ou GitLab

tu push :

Dockerfile

index.html

style.css

script.js

👉 Jusqu’ici : parfait.

Étape B — build automatique (CI/CD)

À chaque git push :

une machine (GitHub Actions / GitLab CI)

fait :

docker build
docker tag
docker push


👉 L’image est reconstruite automatiquement

📌 C’est là que ton “rebuild automatique” doit vivre, pas dans le Dockerfile.

Étape C — exécution

Sur ton PC ou un serveur :

docker pull mon-image:latest
docker run ...


👉 Tu lances une image déjà construite


*************************************************************
Objectif : à chaque git push sur main, GitHub va builder ton Dockerfile et publier l’image sur GHCR (GitHub Container Registry), sans Docker Hub.

1) Crée le dossier du workflow

Dans Git Bash, à la racine de ton projet (là où il y a Dockerfile) :

mkdir -p .github/workflows

2) Crée le fichier GitHub Actions
touch .github/workflows/docker-image.yml


Ouvre VS Code :

code .


Puis colle exactement ça dans .github/workflows/docker-image.yml :

name: Build and push Docker image (GHCR)

on:
  push:
    branches: ["main"]

permissions:
  contents: read
  packages: write

jobs:
  build-and-push:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Login to GHCR
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Build and push
        uses: docker/build-push-action@v6
        with:
          context: .
          push: true
          tags: |
            ghcr.io/${{ github.repository }}:latest
            ghcr.io/${{ github.repository }}:${{ github.sha }}

Ce que ça fait (ultra simple)

Sur chaque push sur main :

récupère ton code

se connecte à GHCR

build l’image depuis ton Dockerfile

push 2 tags :

latest

le SHA du commit (version unique)

3) Commit + push

Toujours dans Git Bash :

git add .github/workflows/docker-image.yml
git commit -m "CI: build & push Docker image to GHCR"
git push

4) Vérifie sur GitHub

Va sur ton repo : Saliha174/docker-html → onglet Actions
Tu dois voir un workflow qui tourne.
Quand c’est vert ✅ : ton image est publiée.

5) Lancer l’image publiée (sur ton PC)

Une fois le workflow OK :

docker pull ghcr.io/saliha174/docker-html:latest
docker run --name site-ghcr -p 8080:80 -d ghcr.io/saliha174/docker-html:latest


Puis navigateur :

http://localhost:8080

Stop / cleanup :

docker stop site-ghcr
docker rm site-ghcr
***************************************
🔁 Le test que tu veux faire (et qui est le bon)
🎯 Objectif

Modifier le HTML → pousser sur GitHub → GitHub rebuild l’image → tu pulls → le site change

C’est exactement comme ça que ça doit fonctionner.

1️⃣ Modifie ton index.html (localement)

Par exemple, change clairement quelque chose :

<h1>Hello depuis la CI GitHub 🚀</h1>
<p>Build automatique OK</p>


Sauvegarde.

2️⃣ Commit + push (déclenche le rebuild automatique)

Dans ton terminal :

git add index.html
git commit -m "Update HTML for CI test"
git push


👉 À CE MOMENT-LÀ :

GitHub Actions se déclenche

Une nouvelle image Docker est rebuild

Le tag latest est mis à jour

Va jeter un œil dans Actions → tu dois voir un nouveau run (vert à la fin).

3️⃣ Très important : gérer le container local

Si tu as déjà un container lancé avec l’ancienne image, il ne changera pas tout seul.

Stoppe et supprime l’ancien container :
docker ps
docker stop site-ghcr
docker rm site-ghcr

4️⃣ Pull la nouvelle image
docker pull ghcr.io/saliha174/docker-html:latest


👉 Là, tu récupères la nouvelle image rebuild par GitHub.

5️⃣ Relance le container
docker run --name site-ghcr -p 8080:80 -d ghcr.io/saliha174/docker-html:latest

6️⃣ Vérifie dans le navigateur

👉 http://localhost:8080

Tu dois voir :

le nouveau contenu HTML

Si oui :
🎉 CI/CD validé de bout en bout

🧠 Règle d’or à retenir (hyper importante)

🔹 Changer le code ≠ changer un container
🔹 Changer le code → rebuild image → relancer container

Un container = photo figée d’une image à un instant T.

Ce que tu sais faire maintenant (sans exagérer)

Modifier du code

Déclencher un rebuild automatique

Publier une image

Déployer une nouvelle version

👉 C’est exactement le cycle pro.

Fais le test tranquillement.
Si tu veux, dis-moi juste “ça a changé” ou “ça n’a pas changé”, et je t’aide à diagnostiquer en 30 secondes si besoin.