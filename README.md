# Luberon & Pyrénées — site refondu

Site statique (HTML/CSS/JS pur, aucune dépendance serveur) reprenant le contenu,
la structure et les photos du site e-monsite d'origine, avec un module de
réservation en plus.

## 1. Avant de mettre en ligne

1. **Récupérer les photos originales.** Elles ne sont pas encore dans ce
   dossier : lancez une fois, sur un ordinateur connecté à Internet :
   ```
   chmod +x download-images.sh
   ./download-images.sh
   ```
   Cela remplit `images/gordes/` et `images/marquixanes/`. Le site fonctionne
   déjà sans cette étape en pointant vers l'ancien site, mais il vaut mieux ne
   plus dépendre de l'hébergement e-monsite une fois que vous l'aurez résilié.

2. **Renseigner votre e-mail de réservation** dans `js/config.js` :
   ```js
   contactEmail: "contact@luberon-et-pyrenees.fr",
   ```
   C'est l'adresse qui recevra les demandes (voir section 3).

3. **Vérifier les tarifs et les disponibilités**, toujours dans
   `js/config.js`, propriété par propriété (`gordes` et `marquixanes`) :
   - `pricing.highSeason` / `pricing.lowSeason` : les deux tarifs affichés.
   - `unavailable` : liste des périodes déjà réservées, au format
     `{ start: "AAAA-MM-JJ", end: "AAAA-MM-JJ" }`. C'est la seule chose à
     mettre à jour manuellement à chaque nouvelle réservation confirmée.

## 2. Structure du site

```
index.html          page d'accueil (les deux maisons)
gordes.html          page maison — Gordes
marquixanes.html      page maison — Marquixanes
contact.html          page de contact
pages/manuel-gordes.html        guide pratique Gordes (non listé dans le menu)
pages/manuel-marquixanes.html   guide pratique Marquixanes (non listé dans le menu)
css/style.css         design du site
js/config.js           TOUT le contenu variable : textes, photos, tarifs, disponibilités
js/main.js             menu mobile, galerie photo, lightbox
js/house-page.js       remplit une page maison à partir de config.js
js/booking.js          calendrier de disponibilité + calcul de prix + formulaire
images/                photos (à peupler avec download-images.sh) + logo
data/availability.json disponibilités Airbnb/Booking, généré automatiquement
scripts/sync-availability.mjs   script de synchronisation iCal
.github/workflows/     Action GitHub qui lance la synchronisation chaque jour
```

Pour ajouter/retirer une photo de galerie, ou changer un texte de présentation,
tout se passe dans `js/config.js` — vous n'avez pas besoin de toucher au HTML.

## 3. Le module de réservation, tel qu'il fonctionne aujourd'hui

- Le calendrier grise les dates passées, les périodes listées dans
  `unavailable` (blocages saisis à la main), **et** les périodes déjà
  réservées sur Airbnb ou Booking une fois la synchronisation automatique
  activée (voir section 4 ci-dessous).
- Le prix est recalculé automatiquement selon vos règles telles que décrites
  sur l'ancien site : en juillet-août, uniquement à la semaine du samedi au
  samedi (tarif fixe) ; le reste de l'année, au tarif à la nuit avec un
  minimum de 3 nuits.
- Quand le visiteur envoie sa demande, **sa messagerie s'ouvre** avec un
  e-mail pré-rempli (dates, prix estimé, coordonnées) adressé à
  `contactEmail`. Il ne reste qu'à cliquer sur "Envoyer". Aucun serveur,
  aucune base de données, aucun compte à créer : ça fonctionne dès la mise en
  ligne.
- Après confirmation d'une réservation, ajoutez la période correspondante
  dans `unavailable` pour qu'elle disparaisse du calendrier.

### Aller plus loin (facultatif)

Le système "mailto" ci-dessus a une limite : il dépend de la messagerie
installée sur l'appareil du visiteur, et vous mettez à jour les
disponibilités à la main. Si vous voulez un jour passer à l'étape supérieure :

- **Formulaire qui arrive dans une vraie boîte mail sans ouvrir Outlook/Mail** :
  un service comme Formspree ou EmailJS s'intègre en remplaçant simplement la
  fonction `onSubmit` de `js/booking.js` par un appel à leur API — quelques
  lignes de JavaScript, aucune limite technique de mon côté.
- **Paiement en ligne (acompte à la réservation)** : nécessite Stripe ou
  équivalent, et donc un minimum de logique serveur.

Dites-moi si l'un de ces points vous intéresse, je peux le construire.

## 4. Synchronisation automatique avec Airbnb et Booking

Le calendrier de réservation peut se mettre à jour tout seul chaque jour à
partir de vos calendriers Airbnb et Booking, grâce à une "Action" GitHub
(gratuite) qui tourne en coulisses. Rien à installer sur votre ordinateur.

**Important : ne mettez jamais les liens iCal directement dans les fichiers du
site.** Ce sont des liens "secrets" — toute personne qui les obtient peut
voir vos réservations. Comme le dépôt GitHub est public, on les enregistre
plutôt dans un espace protégé du dépôt, prévu pour ça :

1. Sur GitHub, ouvrez votre dépôt (`luberon-et-pyrenees`)
2. **Settings** → dans le menu de gauche, **Secrets and variables** → **Actions**
3. Cliquez **"New repository secret"**, et créez ces 6 secrets un par un
   (nom exact à gauche, lien iCal correspondant à droite) :
   - `AIRBNB_GORDES_ICAL`
   - `BOOKING_GORDES_ICAL`
   - `AIRBNB_MARQUIXANES_ICAL`
   - `BOOKING_MARQUIXANES_ICAL`
   - `GREENGO_MARQUIXANES_ICAL`
   - `ABRITEL_MARQUIXANES_ICAL`

   **Attention à ne pas coller deux fois le même lien par erreur** (par
   exemple le lien Airbnb de Gordes dans le secret de Marquixanes) : c'est la
   cause la plus fréquente d'un calendrier qui affiche les mauvaises dates.
   Le script de synchronisation détecte ce cas et affiche un avertissement
   dans les logs de l'Action si deux maisons se retrouvent avec exactement
   les mêmes dates bloquées.
4. Toujours dans **Settings**, allez dans **Actions** → **General**, section
   "Workflow permissions" : cochez **"Read and write permissions"**, puis
   **Save**. (Nécessaire pour que la synchronisation puisse enregistrer les
   nouvelles dates dans le dépôt.)
5. Onglet **Actions** en haut du dépôt → cliquez sur **"Synchroniser les
   disponibilités"** dans la liste de gauche → bouton **"Run workflow"** →
   **"Run workflow"**, pour lancer une première synchronisation tout de suite
   (sinon elle attendra le lendemain 5h).

Une fois lancée, vous verrez sur chaque page maison, sous le calendrier, la
mention "Synchronisé avec Airbnb et Booking — dernière mise à jour le...".
Ensuite, tout est automatique : la synchronisation tourne chaque jour, et les
dates déjà réservées sur Airbnb ou Booking apparaissent grisées sur le site,
en plus de celles que vous ajoutez à la main dans `unavailable`
(`js/config.js`) pour vos propres blocages (travaux, usage personnel...).

Pour vérifier ou déboguer une synchronisation : onglet **Actions** → cliquez
sur une exécution → dépliez l'étape "Récupérer les calendriers..." : le
détail s'affiche maison par maison, avec pour chaque source (Airbnb,
Booking, GreenGo, Abritel) le nombre de caractères reçus, puis le nombre
total de périodes bloquées et les premières dates. C'est le bon endroit pour
vérifier que Gordes et Marquixanes n'affichent pas les mêmes dates.

## 5. Pages "manuel de la maison" (non listées dans le menu)

Deux pages pratiques existent, comme sur l'ancien site, mais ne sont
accessibles que par lien direct — elles n'apparaissent dans aucun menu :

- `pages/manuel-gordes.html`
- `pages/manuel-marquixanes.html`

Elles reprennent les consignes de sécurité et le mode d'emploi de chaque
maison (électricité, chauffage, TV, coffre-fort...). Envoyez le lien
correspondant à vos voyageurs avant leur arrivée (par exemple dans l'e-mail
de confirmation). Une fois le site en ligne, les adresses seront de la forme
`https://votre-domaine/pages/manuel-gordes.html`.

## 7. Référencement (SEO)

J'ai posé les bases techniques du référencement :

- Balises **titre** et **meta description** propres à chaque page
- Balises **Open Graph** et **Twitter Card** (aperçu soigné quand un lien est
  partagé sur WhatsApp, Facebook, etc.)
- Données structurées **Schema.org** (`LodgingBusiness`) sur les pages Gordes
  et Marquixanes, pour aider Google à afficher des informations enrichies
  (prix, équipements)
- `robots.txt` et `sitemap.xml`, pour que Google découvre et indexe les
  bonnes pages (les deux pages "manuel" restent hors indexation, comme
  prévu)

**Une seule chose à faire avant mise en ligne : remplacer le nom de domaine
placeholder.** J'ai utilisé partout `https://luberon-et-pyrenees.fr` en
attendant de connaître votre vrai domaine. Une fois que vous savez lequel
utiliser (domaine personnalisé ou adresse GitHub Pages du type
`https://votre-compte.github.io/luberon-et-pyrenees`), remplacez cette valeur
dans : `sitemap.xml`, `robots.txt`, et l'en-tête de `index.html`,
`gordes.html`, `marquixanes.html`, `contact.html` (cherchez
`luberon-et-pyrenees.fr`).

**Point à connaître : une partie du texte (descriptions, équipements) est
injectée par JavaScript** depuis `js/config.js`, plutôt qu'écrite en dur dans
le HTML. Google sait généralement lire ce type de contenu, mais certains
outils qui n'exécutent pas le JavaScript (certains robots d'aperçu,
notamment) ne le verront pas. Les balises meta/Open Graph couvrent
l'essentiel de ce risque. Si vous voulez que ce texte soit aussi écrit en
dur dans le HTML pour un référencement optimal, dites-le-moi : c'est
faisable, mais cela veut dire modifier le texte à deux endroits (HTML et
config) si vous éditez à nouveau les descriptions plus tard.

**Une fois en ligne**, pour que Google indexe rapidement :
1. Créez un compte sur [Google Search Console](https://search.google.com/search-console)
2. Ajoutez votre site (propriété "Préfixe d'URL")
3. Vérifiez la propriété (Search Console propose plusieurs méthodes, la plus simple ici est d'ajouter un fichier de vérification ou une balise meta qu'il vous fournira)
4. Une fois vérifié, section "Sitemaps" → soumettez `sitemap.xml`

L'indexation prend en général de quelques jours à quelques semaines.

## 8. Mettre le site en ligne

Ce site est 100 % statique : il fonctionne sur n'importe quel hébergeur qui
sert des fichiers HTML (OVH, Netlify, Vercel, GitHub Pages, etc.). Il suffit
de transférer l'ensemble de ce dossier tel quel, `index.html` à la racine.
