# Chapitre 6 : Le Protocole HTTP — Travaux Pratiques

> **Auteur :** Hasna Latrach
> 
> **Source :** <https://http.daaif.net/tp.html>
> 
> **Environnement :** Windows (PowerShell / Git Bash)

-----

## TP 1 : Exploration avec les DevTools

### 1.2 Observer une requête simple — `https://httpbin.org/get`

**Réponses aux questions :**

- **Code de statut :** `200 OK`
- **Headers de requête envoyés** (typiques depuis Chrome) :
  - `Host: httpbin.org`
  - `User-Agent: Mozilla/5.0 ...`
  - `Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8`
  - `Accept-Language: fr-FR,fr;q=0.9,en;q=0.8`
  - `Accept-Encoding: gzip, deflate, br`
  - `Connection: keep-alive`
- **Content-Type de la réponse :** `application/json`

### 1.3 Tester différentes méthodes

Le `fetch GET` retourne un JSON avec les champs `args`, `headers`, `origin`, `url`.
Le `fetch POST` retourne en plus `data`, `json` (le body parsé), et `form` (vide pour un POST JSON).

### 1.4 Codes de statut observés

|URL                     |Code observé                |Signification        |
|------------------------|----------------------------|---------------------|
|`httpbin.org/status/200`|`200 OK`                    |Succès               |
|`httpbin.org/status/404`|`404 Not Found`             |Ressource introuvable|
|`httpbin.org/status/500`|`500 Internal Server Error` |Erreur serveur       |
|`httpbin.org/redirect/3`|`302` puis `200` après suivi|Redirection en chaîne|

### Exercice — Tableau récapitulatif

|URL                     |Méthode|Code|Content-Type              |
|------------------------|-------|----|--------------------------|
|`httpbin.org/get`       |GET    |200 |`application/json`        |
|`httpbin.org/post`      |POST   |200 |`application/json`        |
|`httpbin.org/status/201`|GET    |201 |`text/html; charset=utf-8`|

-----

## TP 2 : Maîtrise de cURL

### 2.1 — Différence entre `-i` et `-v`

- **`-i` (`--include`)** : ajoute uniquement les **headers de réponse** au-dessus du body. Sortie propre, utile pour voir la réponse complète du serveur.
- **`-v` (`--verbose`)** : mode debug complet. Affiche **tout le dialogue** : résolution DNS, handshake TLS, headers de **requête** envoyés (préfixe `>`), headers de **réponse** reçus (préfixe `<`), et le body. Idéal pour diagnostiquer un problème.

En résumé : `-i` montre ce que le serveur **renvoie**, `-v` montre tout l’**échange** dans les deux sens.

### 2.2 POST avec données — Observations

- **Form data** (`-d "name=...&email=..."`) → `Content-Type: application/x-www-form-urlencoded`, et la donnée apparaît dans le champ `form` de la réponse.
- **JSON** (`-H "Content-Type: application/json" -d '{...}'`) → la donnée apparaît dans `json` et `data`.

### 2.3 Headers personnalisés

`curl -H "Authorization: Bearer mon-token-secret" https://httpbin.org/headers` renvoie les headers tels que reçus par le serveur, ce qui permet de vérifier qu’ils sont bien transmis.

### 2.4 Redirections

- **Sans `-L`** : cURL affiche la réponse `302 Found` avec le header `Location:` mais s’arrête.
- **Avec `-L`** : cURL suit chaque redirection jusqu’à obtenir une réponse finale `200`.

### 2.5 Téléchargement

- `-o nom.ext` : sauvegarde sous le nom choisi.
- `-O` : conserve le nom du fichier distant (extrait de l’URL).

### Exercice avancé — Commande cURL complète

```bash
curl -i -X POST \
  -H "Content-Type: application/json" \
  -H "X-Custom-Header: MonHeader" \
  -d '{"action": "test", "value": 42}' \
  https://httpbin.org/post
```

> ⚠️ **Sous Windows PowerShell**, `curl` est un alias d’`Invoke-WebRequest`. Pour utiliser le vrai cURL, il faut soit :
> 
> - Utiliser **Git Bash** (recommandé), soit
> - Appeler explicitement `curl.exe` (présent depuis Windows 10 1803), soit
> - Échapper les guillemets : `curl.exe -X POST -H "Content-Type: application/json" -d "{\"action\":\"test\",\"value\":42}" https://httpbin.org/post`

-----

## TP 3 : API REST avec JavaScript

### 3.1 GET — `jsonplaceholder.typicode.com/users`

Retourne un tableau de 10 utilisateurs (id, name, username, email, address, phone, website, company).

### 3.2 / 3.3 / 3.4 — POST / PUT / DELETE

JSONPlaceholder est un faux serveur : il **simule** la création/modification/suppression mais ne persiste rien. POST renvoie `201` avec un `id: 101`, PUT renvoie `200` avec l’objet modifié, DELETE renvoie `200` avec un objet vide.

### Exercice pratique — `fetchWithRetry`

```javascript
async function fetchWithRetry(url, options = {}, maxRetries = 3) {
  for (let attempt = 1; attempt <= maxRetries; attempt++) {
    try {
      const response = await fetch(url, options);

      // Succès ou erreur client (4xx) → on retourne sans réessayer
      if (response.status < 500) {
        return response;
      }

      // Erreur 5xx → on réessaie sauf si dernière tentative
      if (attempt === maxRetries) {
        throw new Error(`Échec après ${maxRetries} tentatives (HTTP ${response.status})`);
      }

      console.warn(`Tentative ${attempt}/${maxRetries} : HTTP ${response.status}, nouvel essai dans 1s...`);
      await new Promise(resolve => setTimeout(resolve, 1000));

    } catch (error) {
      // Erreur réseau : on réessaie aussi
      if (attempt === maxRetries) throw error;
      await new Promise(resolve => setTimeout(resolve, 1000));
    }
  }
}

// Test
fetchWithRetry('https://httpbin.org/status/500', {}, 3)
  .then(r => console.log('Statut final :', r.status))
  .catch(err => console.error('Erreur :', err.message));
```

-----

## TP 4 : Analyse des Headers de Sécurité

### 4.1 Headers observés sur quelques sites

Commande utilisée :

```bash
curl -s -D - https://github.com -o /dev/null | grep -i "strict\|x-frame\|x-content\|content-security\|referrer"
```

> Sous PowerShell (sans `grep`) :
> 
> ```powershell
> (Invoke-WebRequest -Uri https://github.com -Method Head).Headers
> ```

### Exercice — Analyse de 3 sites

|Site           |HSTS                                            |X-Frame-Options|CSP           |Note (estimée via securityheaders.com)|
|---------------|------------------------------------------------|---------------|--------------|--------------------------------------|
|**github.com** |✅ `max-age=31536000; includeSubDomains; preload`|✅ `deny`       |✅ très stricte|**A+**                                |
|**google.com** |❌ absent sur `www` (présent sur sous-domaines)  |✅ `SAMEORIGIN` |⚠️ partielle   |**D**                                 |
|**mozilla.org**|✅ `max-age=31536000`                            |✅ `DENY`       |✅ stricte     |**A**                                 |

### Synthèse des bonnes pratiques

- **HSTS** force le navigateur à utiliser HTTPS pendant la durée `max-age`, empêchant les attaques de downgrade.
- **X-Frame-Options** (ou `frame-ancestors` en CSP) protège contre le **clickjacking**.
- **X-Content-Type-Options: nosniff** empêche le navigateur de deviner le MIME type (anti-XSS).
- **CSP** est la défense la plus puissante contre l’injection de scripts.
- **Referrer-Policy** limite la fuite d’informations vers les sites tiers.

-----

## TP 5 : Cache HTTP

### 5.1 Headers de cache observés sur `httpbin.org/cache/60`

```
Cache-Control: public, max-age=60
ETag: "..."
```

Le navigateur peut servir la ressource depuis son cache pendant 60 secondes sans contacter le serveur.

### 5.2 Requête conditionnelle avec ETag

- 1ʳᵉ requête : `200 OK` + body + header `ETag: "test123"`.
- 2ᵉ requête avec `If-None-Match: test123` : `304 Not Modified`, **sans body** → économie de bande passante.

### 5.3 Cache navigateur

- **F5** : revalidation (le navigateur peut renvoyer un `If-Modified-Since` ou `If-None-Match`).
- **Ctrl+F5 / Ctrl+Shift+R** : *hard reload*, ignore complètement le cache (envoie `Cache-Control: no-cache`).
- L’onglet Network affiche **`(from disk cache)`** ou **`(from memory cache)`** pour les ressources servies localement.

### Exercice — Headers de cache recommandés

```nginx
# Image (versionnée par hash dans le nom)
location ~* \.(png|jpg|webp)$ {
  add_header Cache-Control "public, max-age=31536000, immutable";
}

# CSS / JS versionnés (ex: app.a3f5b2.css)
location ~* \.(css|js)$ {
  add_header Cache-Control "public, max-age=31536000, immutable";
}

# HTML (toujours frais)
location ~* \.html$ {
  add_header Cache-Control "no-cache, must-revalidate";
}
```

**Logique :** les assets versionnés par hash peuvent être cachés “à vie” (`immutable`), tandis que le HTML doit toujours être revalidé pour pouvoir pointer vers les nouvelles versions des assets.

-----

## Exercices Récapitulatifs

### Exercice 1 — Client HTTP minimaliste

```html
<!DOCTYPE html>
<html lang="fr">
<head>
  <meta charset="UTF-8">
  <title>Mini Client HTTP</title>
  <style>
    body { font-family: system-ui, sans-serif; max-width: 800px; margin: 2rem auto; padding: 1rem; }
    label { display: block; margin: 0.5rem 0 0.25rem; font-weight: 600; }
    input, select, textarea, button { width: 100%; padding: 0.5rem; font-family: inherit; box-sizing: border-box; }
    textarea { font-family: monospace; min-height: 100px; }
    button { background: #2563eb; color: white; border: 0; padding: 0.75rem; margin-top: 1rem; cursor: pointer; border-radius: 4px; }
    pre { background: #f3f4f6; padding: 1rem; overflow-x: auto; border-radius: 4px; }
    .status-ok { color: #16a34a; } .status-err { color: #dc2626; }
  </style>
</head>
<body>
  <h1>Mini Client HTTP</h1>

  <label for="url">URL</label>
  <input id="url" type="url" value="https://httpbin.org/get">

  <label for="method">Méthode</label>
  <select id="method">
    <option>GET</option><option>POST</option><option>PUT</option>
    <option>PATCH</option><option>DELETE</option>
  </select>

  <label for="body">Body (JSON, optionnel)</label>
  <textarea id="body" placeholder='{"key": "value"}'></textarea>

  <button id="send">Envoyer</button>

  <h2>Réponse</h2>
  <p>Statut : <span id="status">—</span></p>
  <h3>Headers</h3>
  <pre id="headers">—</pre>
  <h3>Body</h3>
  <pre id="response">—</pre>

  <script>
    document.getElementById('send').addEventListener('click', async () => {
      const url = document.getElementById('url').value;
      const method = document.getElementById('method').value;
      const bodyText = document.getElementById('body').value.trim();

      const opts = { method, headers: {} };
      if (bodyText && method !== 'GET') {
        opts.headers['Content-Type'] = 'application/json';
        opts.body = bodyText;
      }

      const statusEl = document.getElementById('status');
      try {
        const res = await fetch(url, opts);
        statusEl.textContent = `${res.status} ${res.statusText}`;
        statusEl.className = res.ok ? 'status-ok' : 'status-err';

        const headers = {};
        res.headers.forEach((v, k) => headers[k] = v);
        document.getElementById('headers').textContent = JSON.stringify(headers, null, 2);

        const text = await res.text();
        try {
          document.getElementById('response').textContent = JSON.stringify(JSON.parse(text), null, 2);
        } catch {
          document.getElementById('response').textContent = text;
        }
      } catch (err) {
        statusEl.textContent = 'Erreur réseau';
        statusEl.className = 'status-err';
        document.getElementById('response').textContent = err.message;
      }
    });
  </script>
</body>
</html>
```

### Exercice 2 — Questions théoriques

**1. `no-cache` vs `no-store` ?**

- `no-cache` : la ressource **peut être stockée** dans le cache, mais doit être **revalidée** auprès du serveur (via ETag/Last-Modified) avant chaque réutilisation.
- `no-store` : la ressource **ne doit jamais être stockée**, ni dans le cache du navigateur ni dans aucun cache intermédiaire. Utilisé pour les données sensibles (relevés bancaires, etc.).

**2. Pourquoi POST n’est-il pas idempotent ?**
Une méthode est *idempotente* si N appels identiques produisent le même état serveur qu’un seul appel. POST sert généralement à **créer** une ressource : appelé deux fois, il crée **deux ressources différentes** (deux commandes, deux messages…). Le résultat dépend du nombre d’appels, donc POST n’est pas idempotent. À l’inverse, PUT et DELETE le sont.

**3. Que se passe-t-il avec un code 301 ?**
`301 Moved Permanently` indique que la ressource a **définitivement** changé d’URL. Le client (navigateur, moteur de recherche) doit :

- suivre l’URL donnée dans le header `Location:`,
- mettre à jour ses signets / index pour utiliser la nouvelle URL,
- les requêtes suivantes peuvent aller directement à la nouvelle URL (mise en cache de la redirection).

À comparer avec `302 Found` (redirection **temporaire**, on continue d’utiliser l’URL originale).

**4. À quoi sert le header `Origin` ?**
Envoyé par le navigateur lors de requêtes **cross-origin** (CORS) et de toutes les requêtes POST. Il indique l’origine (schéma + domaine + port) de la page qui a déclenché la requête. Le serveur l’utilise pour :

- décider d’autoriser ou non la requête (politique CORS via `Access-Control-Allow-Origin`),
- se protéger contre les attaques **CSRF**.

Contrairement à `Referer`, il ne contient **pas** le chemin ni les paramètres, donc préserve mieux la vie privée.

**5. Pourquoi `HttpOnly` sur les cookies de session ?**
Le flag `HttpOnly` rend le cookie **inaccessible depuis JavaScript** (`document.cookie` ne le voit pas). C’est une défense critique contre les attaques **XSS** : si un attaquant parvient à injecter du JS sur la page, il ne peut pas voler le cookie de session pour usurper l’identité de l’utilisateur. Le cookie reste cependant envoyé automatiquement par le navigateur à chaque requête vers le domaine, donc l’authentification continue de fonctionner normalement.

À combiner avec `Secure` (HTTPS uniquement) et `SameSite=Strict` ou `Lax` (anti-CSRF).

-----

## Conclusion

Ces TPs couvrent les fondamentaux d’HTTP : analyse via DevTools, manipulation en ligne de commande avec cURL, consommation d’API REST en JavaScript, audit de sécurité via les headers, et stratégies de cache. La maîtrise de ces outils est indispensable pour tout développement web moderne et pour le diagnostic en production.
