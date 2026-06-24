# A Symfony bundle to add headers to your HTTP response.

A Symfony bundle to easily send headers in your HTTP response

For Symfony 6.4, 7.x, 8.x

### Usage
You define one or more headers response in your yaml configuration, for exemple:

```
---
#config/packages/response_header.yml
response_headers:

  headers:
    X-XSS-Protection: value: 1; mode=block

    Referrer-Policy: strict-origin

    Content-Security-Policy:
      - default-src 'none';
      - script-src 'self' data: 'unsafe-inline' 'unsafe-hashes' 'unsafe-eval';
      - script-src-elem 'self' data: 'unsafe-inline' 'unsafe-hashes' 'unsafe-eval';
      - img-src 'self' data: localhost *.mydite.com;

    X-Frame-Options:
      value: SAMEORIGIN
      condition: "'%env(APP_SERVER_TYPE)%' == 'local'"
      replace: false

    Expires:
      value: 0
      condition: request.getPathInfo() matches '^/admin'
...
```



#### Conditonal header
The conditional is made with symfony expression language, the available variables are:


```
response_headers:
  headers:
    X-Frame-Options:
      value: SAMEORIGIN
      condition: "'%env(APP_SERVER_TYPE)%' == 'local'"
```
The 'X-Frame-Option' header will be inclued in the HTTP response if the 'APP_SERVER_TYPE' environment variable is equal to 'local'.

```
  %env(name)%  : a value from environement
  request: An instance of the class Symfony\Component\HttpFoundation\Request class
  response: An instance of the class Symfony\Component\HttpFoundation\Response class
```

  Example:
```
   condition: request.getPathInfo()  matches '^/admin'
   condition: response.getStatusCode() == 200
```

#### Header values ​​in array or scalar format
For very long headers, it is possible to use a table format. The header value will be reduced to a single line.

##### line format
```
response_headers:

  headers:
    Content-Security-Policy:
      - default-src 'none';
      - script-src 'self' data: 'unsafe-inline' 'unsafe-hashes' 'unsafe-eval';
      - img-src 'self' data: localhost *.mysite.com;
      format: string
```
This is the default format
##### Result:
```
Content-Security-Policy: default-src 'none';script-src 'self' data: 'unsafe-inline';img-src 'self' data: localhost *.mysite.com
```

##### mutliple format

But it's possible to have one more than one HTTP header with the same name

```
response_headers:
  headers:
    Accept:
      - application/json
      - application/xml
      format: array
```

##### Result:
```
Accept: application/json
Accept: application/xml
```



## Installation
The bundle should be automatically enabled by Symfony Flex. If you don't use Flex, you'll need to enable it manually as explained in the docs.


```
composer config extra.symfony.allow-contrib true
composer require republique-et-canton-de-geneve/response-headers-bundle
```



License
Released under the Apache-2.0 license

## Quality and test code
Code coverage :
![coverage line](https://raw.githubusercontent.com/republique-et-canton-de-geneve/response-headers-bundle/refs/heads/main/coverage_line.svg)
![coverage branche](https://raw.githubusercontent.com/republique-et-canton-de-geneve/response-headers-bundle/refs/heads/main/coverage_branch.svg)

[![phpunit](https://github.com/republique-et-canton-de-geneve/response-headers-bundle/actions/workflows/phpunit.yml/badge.svg)](https://github.com/republique-et-canton-de-geneve/response-headers-bundle/actions/workflows/phpunit.yml)
[![phpstan](https://github.com/republique-et-canton-de-geneve/response-headers-bundle/actions/workflows/phpstan.yml/badge.svg)](https://github.com/republique-et-canton-de-geneve/response-headers-bundle/actions/workflows/phpstan.yml)
[![rector](https://github.com/republique-et-canton-de-geneve/response-headers-bundle/actions/workflows/rector.yml/badge.svg)](https://github.com/republique-et-canton-de-geneve/response-headers-bundle/actions/workflows/rector.yml)
[![php-cs-fixer](https://github.com/republique-et-canton-de-geneve/response-headers-bundle/actions/workflows/php-cs-fixer.yml/badge.svg)](https://github.com/republique-et-canton-de-geneve/response-headers-bundle/actions/workflows/php-cs-fixer.yml)



# Analyse du Bundle par Z.ai


Puisque nous partons du postulat que le code est déjà de très bonne qualité (niveau Senior, architecture classique et robuste Symfony), les améliorations possibles ne seront pas des corrections de bugs, mais des **optimisations fines, des évolutions architecturales ou des améliorations de l'expérience développeur (DX)**.

Voici les axes d'amélioration que l'on pourrait appliquer à ce bundle pour le faire passer du statut de "très bon bundle institutionnel" à "bundle de référence absolue" :

### 1. Gestion fine de la stratégie de fusion (Merge Strategy)
C'est un point critique souvent négligé dans la gestion des headers. Si le bundle ajoute un header qui existe déjà (par exemple, mis par un autre bundle ou par le contrôleur), que fait-il ?
*   **Amélioration :** Ajouter une option de configuration par header pour définir le comportement :
    *   `replace` (écrase le header existant - comportement par défaut de Symfony)
    *   `append` (ajoute une valeur supplémentaire au header existant, par exemple pour `Vary` ou `Cache-Control`)
    *   `skip_if_exists` (ne fait rien si le header est déjà présent, très utile pour ne pas écraser une politique de sécurité spécifique définie dans un contrôleur).

### 2. Protection contre les sous-requêtes et optimisation
Dans Symfony, une requête principale génère souvent des sous-requêtes (fragments, ESI, etc.). Injecter des headers de sécurité (ex: `X-Frame-Options`) dans une sous-requête est inutile et coûteux.
*   **Amélioration :** S'assurer que le code vérifie strictement `$event->isMainRequest()` (anciennement `isMasterRequest()` dans les vieilles versions).
*   **Optimisation :** Ajouter un *early return* basé sur le `Content-Type`. Si la réponse finale est une redirection (301/302), un fichier binaire (PDF, image) ou une réponse vide (204), injecter certains headers (comme le CSP lié au HTML) est inutile.

### 3. Ajouter des "Profils" ou Guards de sécurité intégrés
Plutôt que de laisser le développeur configurer chaque header manuellement, le bundle pourrait offrir des raccourcis pour les cas d'usage les plus courants de l'État.
*   **Amélioration :** Créer un système de "profils" (ex: `security_profile: "strict"`) qui applique un bloc de headers d'un coup.
*   **Exemple concret (Le piège du HSTS) :** Le header `Strict-Transport-Security` (HSTS) **ne doit absolument pas** être envoyé en HTTP, sous peine de bloquer l'utilisateur. Le bundle pourrait intégrer une intelligence qui dit : *"Si le profil sécurité est actif, j'envoie HSTS uniquement si la requête initiale est en HTTPS"*. Cela évite une erreur de configuration fatale côté métier.

### 4. Évolution vers les attributs PHP 8 (Developer Experience)
Actuellement, la configuration est probablement 100% dans le `yaml`. Pour des headers très spécifiques à un seul contrôleur, cela alourdit le fichier de configuration.
*   **Amélioration :** Proposer un attribut PHP 8 pour une déclaration locale.
    ```php
    #[AddResponseHeader(name: 'X-Custom-Header', value: 'SpecificValue')]
    public function myAction(): Response { ... }
    ```
    Le bundle combinerait alors la configuration globale (YAML) et les configurations locales (Attributs), le local prenant la priorité.

### 5. Modernisation du typage (PHP 8.2 / 8.3)
Même si le code utilise les type hints, on peut pousser la modernité :
*   **Utilisation des Enums :** Si le bundle a des options (comme la stratégie de fusion mentionnée au point 1), utiliser des PHP Enums (ex: `enum MergeStrategy: string { case Replace = 'replace'; ... }`) au lieu de simples chaînes de caractères. Cela élimine les erreurs de frappe à la configuration.
*   **Readonly Classes :** Si des DTOs ou objets de configuration sont créés en interne, les déclarer en `readonly class` pour garantir l'immuabilité.

### 6. Préparation à l'avenir : Compatibilité PSR-15 (Middleware)
L'Event Subscriber sur `kernel.response` fonctionne parfaitement avec le HttpKernel classique de Symfony. Cependant, l'écosystème PHP tend de plus en plus vers les Middlewares PSR-15 (notamment quand on utilise des runtimes comme Swoole, RoadRunner ou FrankenPHP).
*   **Amélioration (Long terme) :** Proposer une implémentation alternative sous forme de Middleware PSR-15 pur, en plus du Event Subscriber, pour garantir que le bundle fonctionne dans des environnements "sans HttpKernel" (bien que cela soit rare dans l'écosystème Symfony pur, c'est un gage de modernité architecturale).

### En résumé
Le bundle actuel fait très bien son **métier de base**. Les améliorations listées ci-dessus relèvent de l'**ingénierie de précision** (gestion des cas limites comme HTTP vs HTTPS, sous-requêtes) et de **l'ergonomie avancée** (Enums, Attributs). Ce sont des chantiers typiques de "Version 2.0" pour un bundle qui a fait ses preuves en "Version 1.x".