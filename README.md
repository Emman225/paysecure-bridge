# PaySecure Bridge

Page HTML statique HTTPS qui sert de **proxy de retour** pour PaySecure pendant le développement local de FNE Connect.

## Pourquoi ?

PaySecure exige que `Url_Retour` (la page sur laquelle l'utilisateur revient après paiement) soit en **HTTPS public**. En développement local (`http://localhost:5173`), c'est impossible sans tunnel.

Ce bridge résout le problème :

```
PaySecure → https://ton-bridge.vercel.app/api/v1/paysecure/return/{code}?frontend=http://localhost:5173
        ↓ (redirection JavaScript automatique)
        → http://localhost:5173/payment/result?code={code}&status=success&...
```

L'utilisateur atterrit sur le bridge, qui le redirige instantanément vers son frontend local.

## Limitation

Ce bridge ne couvre **que la return URL** (redirection navigateur). Il ne peut PAS recevoir les **webhooks** PaySecure (POST serveur-à-serveur). Le statut de la transaction sera donc synchronisé via le **fallback `confirm-from-return`** déjà implémenté côté backend FNE Connect ([PaySecureSyncController](../fne-connect-backend/app/Http/Controllers/Api/V1/Public/PaySecureSyncController.php)).

## Déploiement

### Option A — Vercel (recommandé, 2 minutes)

1. Crée un compte gratuit sur https://vercel.com
2. Installe le CLI : `npm i -g vercel`
3. Dans ce dossier :
   ```bash
   cd paysecure-bridge
   vercel deploy --prod
   ```
4. Vercel te donne une URL HTTPS du genre `https://paysecure-bridge-xxxx.vercel.app`

**Alternative sans CLI** : drag-and-drop le dossier `paysecure-bridge/` sur https://vercel.com/new

### Option B — Netlify

1. Crée un compte sur https://netlify.com
2. Drag-and-drop le dossier `paysecure-bridge/` sur https://app.netlify.com/drop
3. Netlify te donne une URL HTTPS du genre `https://xxxxxx.netlify.app`

### Option C — GitHub Pages

1. Push ce dossier dans un repo GitHub
2. Settings → Pages → Source : ce dossier
3. URL : `https://<user>.github.io/<repo>`

## Configuration FNE Connect

Une fois déployé, mets à jour `.env` du backend :

```env
PAYSECURE_BASE_URL=https://paysecure-bridge-xxxx.vercel.app
```

Puis vide le cache :
```bash
php artisan config:clear
```

C'est tout. Refais un test de paiement : tu seras automatiquement redirigé vers ton localhost après paiement.

## Comment ça marche techniquement

1. **catch-all routing** (vercel.json / netlify.toml / _redirects) : tout chemin est servi par `index.html`
2. **JavaScript** dans `index.html` :
   - Extrait `code_paiement` du `window.location.pathname` (dernier segment)
   - Lit `?frontend=...` du `window.location.search`
   - Construit `{frontend}/payment/result?code=...&status=success&...`
   - Redirige via `window.location.replace()`

## Sécurité

Le bridge ne stocke rien et n'a aucune logique métier. Il fait juste une redirection navigateur. Aucune donnée sensible ne transite par lui (le code paiement est public dans l'URL et n'a de valeur qu'avec une transaction valide en base).
