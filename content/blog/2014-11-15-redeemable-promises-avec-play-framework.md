---
layout: post
title: "Redeemable promises avec Play Framework"
comments: true
published: true
taxonomies: 
  tags: [async, playframework]
---

J'ai eu à intervenir récemment sur un programme écrit en Play Framework v2.3, dont le rôle est assez simple : faire passe-plat entre un client et un serveur et effectuant notamment des transformations protocolaires (comme REST vers TCP par exemple). Ce qui m'a donné l'occasion d'utiliser les [RedeemablePromise](https://www.playframework.com/documentation/2.4.x/api/java/play/libs/F.RedeemablePromise.html) de Play Framework, pas du tout documentées à ce jour.

## Client asynchrone

Les différents échanges de messages entre les systèmes peuvent être représentés à l'aide du diagramme de séquence suivant :

<!-- 
http://www.websequencediagrams.com/?lz=dGl0bGUgQ2xpZW50IGFzeW5jaHJvbmUKCgANBi0-UGFzc2VQbGF0OlBPU1QgbWVzc2FnZQoADgktPlNlcnZldXI6c2VuZAAXCQAOBy0ANQwgYWNrAC4LLT4AcQY6IDIwMAAXBQpub3RlIG92ZXIAgQsHLAB1CSwAXgggUGx1cyB0YXJkLCB1bgCBCAggZGUgcmV0b3VyCgByCQBuDHJlc3BvbnMAgSsNAHYIAIFTBQAYCQCBcQcAgSsNAIEYCACBMAwAgQcJYWNr&s=modern-blue
-->

![Diagram](http://www.websequencediagrams.com/cgi-bin/cdraw?lz=dGl0bGUgQ2xpZW50IGFzeW5jaHJvbmUKCgANBi0-UGFzc2VQbGF0OlBPU1QgbWVzc2FnZQoADgktPlNlcnZldXI6c2VuZAAXCQAOBy0ANQwgYWNrAC4LLT4AcQY6IDIwMAAXBQpub3RlIG92ZXIAgQsHLAB1CSwAXgggUGx1cyB0YXJkLCB1bgCBCAggZGUgcmV0b3VyCgByCQBuDHJlc3BvbnMAgSsNAHYIAIFTBQAYCQCBcQcAgSsNAIEYCACBMAwAgQcJYWNr&s=modern-blue)

Le programme en Play Framework est nommé "PassePlat" dans ce diagramme.

Les messages montants sont implémentés sous forme d'une API RESTful ; tandis que le retour du "Serveur", asynchrone, est implémentée avec des *callbacks* et une requête REST vers le client d'origine.

Voici un exemple de pseudo-code Java sur le traitement du message de retour :


```java
public class ServerReceiver {

	public void onServerResponse(Response response) {
		// Utilisation de la librairie play.libs.ws.WS
		// pour effectuer des appels REST
		WS.url("http://client_url/api/response")
			.setContentType("application/json")
			.post(response.asJson());
	}

	public void onServerError(Throwable error) {
		WS.url("http://client_url/api/response")
			.setContentType("application/json")
			// Serialize exception as JSON
			.post(Helper.asJson(error);
	}

}
```

Le système complet fonctionne bien, mais n'est pas très satisfaisant :

- le Client et le PassePlat sont très couplés. En effet, le Client doit connaître l'adresse du PassePlat pour lui envoyer les messages montants et le PassePlat doit connaitre l'adresse du Client pour lui renvoyer le message de retour. Bref, une dépendance cyclique ;

- Le fonctionnement du Serveur est asynchrone (messages montants et de retours sont décorrélés) et cette implémentation a déporté l'asynchronisme jusqu'au Client, alors que le client (l'humain cette fois-ci :-) voulait plutôt un fonctionnement synchrone du Client, ce qui était plus facile à appréhender pour lui.

## Client synchrone

Nous avons donc travaillés sur une implémentation du système plutôt comme ceci :

<!--
http://www.websequencediagrams.com/?lz=dGl0bGUgQ2xpZW50IHN5bmNocm9uZQoKAAwGLT5QYXNzZVBsYXQ6UE9TVCBtZXNzYWdlCgAOCS0-U2VydmV1cjpzZW5kABcJAA4HLQA1DCBhY2sKCm5vdGUgb3ZlcgBuBywAWQksAEIIIFBsdXMgdGFyZCwgdW4AbAggZGUgcmV0b3VyCgBWCQBSDHJlc3BvbnMAgRAMLT4AgVMGOiAyMDAAGAkgKwCBBQUAgTATADsJQWNr&s=modern-blue
-->


![Diagram](http://www.websequencediagrams.com/cgi-bin/cdraw?lz=dGl0bGUgQ2xpZW50IHN5bmNocm9uZQoKAAwGLT5QYXNzZVBsYXQ6UE9TVCBtZXNzYWdlCgAOCS0-U2VydmV1cjpzZW5kABcJAA4HLQA1DCBhY2sKCm5vdGUgb3ZlcgBuBywAWQksAEIIIFBsdXMgdGFyZCwgdW4AbAggZGUgcmV0b3VyCgBWCQBSDHJlc3BvbnMAgRAMLT4AgVMGOiAyMDAAGAkgKwCBBQUAgTATADsJQWNr&s=modern-blue)

L'appel du Client vers le PassePlat est donc bloqué tant que l'acquittement **et** la réponse du Serveur ne sont pas parvenus au PassePlat.

### Comment ?

On peut faire cela très simplement avec les [RedeemablePromise](https://www.playframework.com/documentation/2.4.x/api/java/play/libs/F.RedeemablePromise.html) de Play Framework.

Les [RedeemablePromise](https://www.playframework.com/documentation/2.4.x/api/java/play/libs/F.RedeemablePromise.html) sont une implémentation du design pattern *promise*, désormais répandu dans l'informatique pour résoudre le [*callback hell*](http://callbackhell.com/) -l'enfer des callbacks-, problème très fréquent avec la programmation asynchrone. En effet, votre code est exécuté au sein de *callbacks* en réponse à des évènements : fin de traitement, lecture d'un fichier, arrivée d'un message par Web Socket, ...

On trouve des implémentations de ce pattern naturellement en [Javascript](https://www.promisejs.org/) ([plusieurs même](https://docs.angularjs.org/api/ng/service/$q)), mais aussi en [Scala](http://docs.scala-lang.org/overviews/core/futures.html), en [Java](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/CompletableFuture.html), ...

Voyons comment nous pouvons les utiliser pour répondre à notre nouveau besoin.

Adaptons la classe qui réceptionne les évènements venant du Serveur :

``` java
public class ServerReceiver {

	// Initialisation de la promesse, à vide
	private RedeemablePromise<Response> promise = RedeemablePromise.empty();

	// get/set

	public void onServerResponse(Response response) {
		// La promesse est résolue ou complétée avec succès ici
		promise.success(response);
	}

	public void onServerError(Throwable error) {
		// La promesse est résolue ou complétée en erreur avec l'exception
		promise.failure(error);
	}

}
```

L'élément clé ici est le fait de "résoudre" ou "compléter" la promesse en succès ou en erreur selon le cas. Attention à n'oublier aucun cas de fin de traitement du Serveur, sinon le Client restera bloqué dans ces cas non prévus.

Jetons maintenant un coup d'oeil au contrôleur REST sur le PassePlat qui reçoit les requêtes HTTP depuis le Client et qui "bloque" tant que la réponse du Serveur n'est pas parvenue :

``` java
public static Promise<Result> sendMessage(...) {

	// Envoi du message entrant vers le serveur
	// server.send(message)

	ServerReceiver serverReceiver = new ServerReceiver();

	// Ecoute des messages retour
	server.listen(serverReceiver);

	return
		// Promesse de type Promise<Response>
		serverReceiver.getPromise()
		// Conversion 'success' en Promise<Result>
		.map(response ->
			ok(JSON.toJson(response));
		);
}
...
```

Dans le cas nominal, la `Response` sera convertie en `Result` au sein de la méthode `map` (programmation fonctionnelle).

Et dans le cas où la promesse a été résolue en erreur ?
Par défaut, Play va générer une réponse HTTP avec un code retour 500 et une sérialisation de l'exception renvoyée. Si vous souhaitez définir vous-même votre propre retour, il faut que le traitement de l'erreur génère un `Result` standard.

Voici comment adapter la transformation de la promesse pour renvoyer une erreur 500 et une sérialisation de l'exception en cas de promesse résolue en erreur :

``` java
return
	// Promesse de type Promise<Response>
	promise
	// Conversion 'success' en Promise<Result>
	.map(response ->
		ok(Json.toJson(response));
	)
	// Conversion 'failure' en Promise<Result>
	.recover(error ->
		// Serialize exception as JSON
		// and send HTTP 500 status
		internalServerError(Json.toJson(error));
	);
```

Au final, nous avons pu rendre l'appel du Client synchrone en à peine quelques lignes de code grâce à l'API riche de Play Framework.
Enfin, vous remarquerez que Java 8 améliore significativement la lisibilité du code. Cependant, on reste loin de Scala, de Groovy ou tout simplement de Javascript.
