# Spec : orchestrateur d'outils pour le chat de l'appli de logs

## Objectif
Un seul agent dans la chatbox, avec plusieurs outils. Selon la question, le LLM choisit : interroger la grille (SQL), chercher dans la doc, consulter la base de connaissances de logs, ou déclencher une action d'interface (filtre, etc.).

## Stack
- Back : Java, JAX-RS classique (pas de Spring). Front : React.
- LLM : Qwen interne via LiteLLM (API compatible OpenAI, on change juste l'URL de base).
- Lib d'orchestration : **LangChain4j** (`@Tool`, `@P`). Vérifier l'API de la version utilisée : elle évolue vite, ne pas inventer de méthodes.
- Hors périmètre V1 : serveur MCP, multi-agents.

## Principe
1. Le front envoie `POST /chat` avec la question, l'id de la vue courante et son contexte (filtres actifs, etc.).
2. Le back construit la liste d'outils **selon la vue**. Un outil non déclaré n'existe pas pour le LLM.
3. Le LLM choisit un outil d'après les descriptions, la lib exécute la méthode Java annotée et renvoie le résultat au LLM, qui continue ou conclut.
4. Le back répond au front : texte du LLM + sources + liste d'actions.
5. Le **front** exécute les actions. Le back ne fait que valider et préparer.

## Outils V1
| Outil | Type | Rôle | Confirmation |
|---|---|---|---|
| `recherche_doc` | serveur | appelle la route RAG existante, renvoie les extraits bruts | non |
| `interroger_logs` | serveur | agent SQL en lecture seule, renvoie résultats + requête utilisée | non |
| `consulter_base_connaissances` | serveur | API par nom de log, renvoie la fiche et sa description | non |
| `filtrer_grille` | interface | valide les paramètres, range une action | non (V1) |

Plus tard : `mettre_en_valeur` (sans confirmation), `supprimer_lignes` (avec confirmation).

## Écriture d'un outil
- Description de l'outil dans `@Tool` : quand l'utiliser, 3 à 4 exemples variés, 1 ou 2 contre-exemples.
- Description de chaque paramètre dans `@P`. Colonnes et opérateurs en **enum Java** (liste fermée).
- La méthode renvoie toujours une `String` au LLM : un court en-tête de consignes + le résultat.

```java
@Tool("Filtre la grille de logs. À utiliser quand l'utilisateur demande de filtrer ou restreindre les données affichées.")
String filtrerGrille(
    @P("Colonne à filtrer") Colonne colonne,
    @P("Opérateur de comparaison") Operateur operateur,
    @P("Valeur à comparer") String valeur) {
  // 1. valider (colonne/valeur cohérentes) ; si invalide : renvoyer un message d'erreur au LLM
  // 2. actions.add(new Action("filtre_grille", statut, params))
  // 3. return "Filtre préparé : ... Confirme en une phrase ce qui a été filtré, sans rien ajouter.";
}
```

- Les actions sont collectées dans un `ActionCollector` **créé par requête** (instancier les outils par requête, pas de singleton partagé).
- La logique métier vit dans des services Java ordinaires. Les outils sont de fines enveloppes.
- `recherche_doc` ne fait **aucun appel LLM** : elle renvoie les extraits, c'est l'orchestrateur qui rédige.
- Résultat vide : message explicite ("aucun extrait pertinent, dis-le à l'utilisateur") pour éviter les inventions.

## Réponse de `/chat`
```json
{
  "version": 1,
  "message": "J'ai filtré les appels de plus d'une minute.",
  "sources": [{ "titre": "...", "lien": "..." }],
  "actions": [{
    "id": "a1",
    "type": "filtre_grille",
    "statut": "a_appliquer",
    "parametres": { "colonne": "duree", "operateur": "sup", "valeur": "60" },
    "message_confirmation": null
  }]
}
```
Statuts : `a_appliquer` (le front exécute), `en_attente_confirmation` (le front affiche oui/non ou l'arbre de recherche pré-rempli), `refusee`, `erreur`.

## Confirmation
- C'est le **code Java** qui décide du statut, selon le niveau de risque de l'outil. Jamais le LLM.
- Sur `en_attente_confirmation`, le front affiche sa carte. Au clic sur oui, il appelle la vraie route (même code que l'action manuelle) avec les droits de l'utilisateur. Sur non, il jette l'action.
- Le texte renvoyé au LLM le dit ("proposé, pas encore appliqué") pour qu'il rédige une phrase cohérente.
- Pour une suppression : toujours afficher le nombre de lignes touchées.

## Front (React)
- Type d'action en union discriminée TypeScript sur `type`, avec un `switch` exhaustif.
- Un registre : chaque type d'action appelle une fonction existante (état de la grille, arbre de recherche, etc.).

## Sécurité
- SQL : compte en lecture seule (idéalement vues ou réplica), timeout, limite de lignes.
- Droits et périmètre de l'utilisateur imposés **côté serveur** dans la requête, jamais confiés au LLM.
- Valider le fond dans chaque outil (colonne existante, SELECT uniquement). La lib ne vérifie que la forme.
- Plafonner le nombre d'itérations de la boucle d'outils.

## Format des réponses
- Règles générales (langue, ton) dans le prompt système, avec le contexte de la vue.
- Consignes propres à un outil dans le texte renvoyé par l'outil (court, quelques lignes).
- Si un message doit être exact au mot près, le générer en Java.

## Plan
1. Prototype : SQL + doc sur LiteLLM. Tester tôt le tool calling du Qwen interne.
2. `filtrer_grille`, format de réponse, switch front.
3. Base de connaissances, outils par vue.
4. Confirmation, puis streaming (SSE : événements `texte`, `action`, `final`).
5. Jeu de questions types avec l'outil attendu, rejoué à chaque changement. Traces des appels (Langfuse).
