# Compte rendu : tester, sécuriser et appeler l'API LogiAI

**Noms :** À COMPLÉTER  **Groupe :** À COMPLÉTER

## 1\. Casser son API (activité 1)

|Cas|Requête (corps ou URL)|Prévu|Obtenu|
|-|-|-|-|
|champ manquant|`{"distance\_km": 120.0, "heure\_depart": 17, "pluie\_mm": 8.0, "trafic\_index": 7.5, "nb\_arrets": 4, "type\_vehicule": "camion"}` (sans `poids\_kg`)|422|422|
|valeur absurde|`{"distance\_km": 120.0, "poids\_kg": 99999, "heure\_depart": 17, "pluie\_mm": 8.0, "trafic\_index": 7.5, "nb\_arrets": 4, "type\_vehicule": "camion"}`|422|422|
|texte au lieu d'un nombre|`{"distance\_km": "beaucoup", "poids\_kg": 450.0, "heure\_depart": 17, "pluie\_mm": 8.0, "trafic\_index": 7.5, "nb\_arrets": 4, "type\_vehicule": "camion"}`|422|422|
|champ en trop|`{"distance\_km": 120.0, "poids\_kg": 450.0, "heure\_depart": 17, "pluie\_mm": 8.0, "trafic\_index": 7.5, "nb\_arrets": 4, "type\_vehicule": "camion", "couleur": "rouge"}`|422|422|
|mauvaise route|`curl.exe -i http://localhost:8000/v1/predictt`|404|404|

* Une requête acceptée alors qu'elle devait être refusée ? Laquelle ? **Aucune dans l'activité 1** : les cinq cas ont été refusés (422, 422, 422, 422, 404) et les cinq tests de `test\_casser.py` sont passés du premier coup. Les requêtes acceptées à tort ont été trouvées à l'activité 2 (voir section 2).

## 2\. Les 2 bugs (activité 2)

|Bug|Test qui l'a révélé|Comportement avant|Correction|
|-|-|-|-|
|1|`heure\_depart=24` : attendu 422, obtenu 200|`le=24` : l'heure 24 était acceptée alors que la description dit « 0 à 23 »|`le=23`|
|2|`nb\_arrets=-1` : attendu 422, obtenu 200|aucune borne minimum : un nombre d'arrêts négatif passait|ajout de `ge=0`|

* Sortie de `python -m pytest` à la fin de l'activité 2 (abrégée) :

```
.venv\\Lib\\site-packages\\fastapi\\testclient.py:1
  StarletteDeprecationWarning: Using `httpx` with `starlette.testclient` is deprecated; install `httpx2` instead.

42 passed, 1 warning in 1.72s
```

Le warning vient de la bibliothèque de test (httpx / Starlette), pas de mon code.

## 3\. La clé API (sprint 1)

* Ma fonction `verify\_api\_key()` :

```python
def verify\_api\_key(x\_api\_key: str | None = Header(default=None)) -> str:
    """Authentification : renvoie le rôle ('user' ou 'admin') ou lève une 401."""
    if x\_api\_key is None:
        raise HTTPException(
            status\_code=401,
            detail="Clé API manquante (en-tête X-API-Key)",
            headers={"WWW-Authenticate": "ApiKey"},
        )
    if \_egal(x\_api\_key, config.ADMIN\_KEY):
        return "admin"
    if \_egal(x\_api\_key, config.API\_KEY):
        return "user"
    raise HTTPException(
        status\_code=401,
        detail="Clé API invalide",
        headers={"WWW-Authenticate": "ApiKey"},
    )
```

|Requête|Code obtenu|
|-|-|
|`/v1/predict` sans clé|**401** Unauthorized : `{"detail":"Clé API manquante (en-tête X-API-Key)"}`|
|`/v1/predict` fausse clé|**401** Unauthorized : `{"detail":"Clé API invalide"}`|
|`/v1/predict` clé utilisateur|**200** OK : `{"probabilite\_retard":0.732,"alerte":true,"seuil":0.4,"model\_version":"1.0"}`|
|`/v1/model-info` clé utilisateur|**403** Forbidden : `{"detail":"Accès réservé à l'administrateur"}`|
|`/v1/model-info` clé administrateur|**200** OK : `{"version":"1.0","seuil":0.4,"variables":\["distance\_km","poids\_kg","heure\_depart","pluie\_mm","trafic\_index","nb\_arrets","type\_vehicule"]}`|
|`/health` sans clé|**200** OK : `{"status":"ok","model\_loaded":true,"auth\_configured":true}`|

* Preuve que `.env` n'est pas dans Git (sortie de `git status`) :

```
PS C:\\logiai-atelier4\\atelier4> git status
On branch tp-securite
Your branch is up to date with 'origin/tp-securite'.

Changes not staged for commit:
        modified:   logiai/api/main.py
        modified:   logiai/api/schemas.py
        modified:   logiai/api/security.py
        modified:   tests/test\_casser.py
        modified:   tests/test\_limites.py

Untracked files:
        livraison.json
```

`.env` n'apparaît nulle part : il est ignoré par Git.

* Résultat de `python -m pytest` après le sprint 1 :

```
53 passed, 1 warning in 1.68s
```

## 4\. Le client web (sprint 2)

* Mon `fetch` :

```javascript
async function appelerApi(livraison, cle) {
  const reponse = await fetch(API\_URL, {
    method: "POST",
    headers: { "Content-Type": "application/json", "X-API-Key": cle },
    body: JSON.stringify(livraison),
  });
  const corps = await reponse.json();
  return { status: reponse.status, corps };
}
```

* Ce qui s'affiche :

  * livraison normale : « Probabilité de retard : 73.2 %. Alerte : livraison à risque. »
  * pluie de 500 mm : À COMPLÉTER (texte affiché par la page)
  * fausse clé : « Clé API absente ou invalide. »
* Expérience CORS :

  * **Avant d'ajouter le CORS** (page sur le port 5500) : la console affiche `Access to fetch at 'http://localhost:8000/v1/predict' from origin 'http://localhost:5500' has been blocked by CORS policy: Response to preflight request doesn't pass access control check: No 'Access-Control-Allow-Origin' header is present on the requested resource.` Côté uvicorn : `OPTIONS /v1/predict HTTP/1.1" 405 Method Not Allowed` (deux fois).
  * **Après avoir ajouté `CORSMiddleware`** (page sur le port 5500) : uvicorn affiche `OPTIONS /v1/predict ... 200 OK` puis `POST /v1/predict ... 200 OK`, et la page affiche « Probabilité de retard : 73.2 %. Alerte : livraison à risque. »
  * **Page servie sur le port 5501** : 
  * Pourquoi `curl` marche-t-il quand même ? Le CORS est une règle appliquée par le navigateur, qui protège l'utilisateur. `curl` n'est pas un navigateur : il ne pose pas la question « puis-je ? », donc l'API reçoit sa requête, et seule la clé la protège.
* Où voit-on la clé dans F12 ? 

## 5\. Questions de compréhension

1. **Quelle différence entre 401, 403 et 422 ?** 401 : l'API ne sait pas qui appelle (clé absente ou fausse). 403 : elle sait qui appelle, mais l'action n'est pas permise (clé utilisateur sur `/v1/model-info`). 422 : l'appelant est reconnu, mais les données sont invalides (champ manquant, valeur hors bornes). Les contrôles passent dans l'ordre route, clé, données, modèle : sans clé, une pluie de 500 mm donne 401, pas 422.
2. **Pourquoi la clé n'est-elle ni dans le code ni dans Git ?** Une clé commitée reste dans l'historique de Git, même si on supprime le fichier ensuite, et elle est lisible par toute personne qui a accès au dépôt. Il faudrait alors la changer. On la met dans `.env` (ignoré par Git, lu au démarrage) et on commite seulement `.env.example` avec de fausses valeurs.
3. **Pourquoi comparer avec `compare\_digest` plutôt qu'avec `==` ?** `==` s'arrête au premier caractère différent : la durée de la réponse révèle combien de caractères de la clé étaient justes, ce qui permet de la deviner petit à petit (attaque par mesure du temps). `compare\_digest` prend un temps constant.
4. **Pourquoi `curl` fonctionne-t-il alors que la page du port 5501 est bloquée ?** Le blocage est fait par le navigateur, qui n'envoie la vraie requête que si l'API autorise l'origine de la page (liste `ALLOWED\_ORIGINS`). `curl` n'applique pas cette règle : l'API répond à toute requête qui porte une clé valide.
5. **Pourquoi la validation faite par le navigateur ne remplace-t-elle pas celle de l'API ?** La page n'est qu'un client parmi d'autres : `curl`, un script ou un programme Java peuvent envoyer n'importe quoi sans passer par elle, et n'importe qui peut modifier le HTML ou le JavaScript depuis F12. Seule l'API contrôle ce qu'elle accepte.

## 6\. Lien de la pull request



