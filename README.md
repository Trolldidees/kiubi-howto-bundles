# Créer des bundles de produits avec l'API de son site Kiubi


## Introduction

Ce dépôt est un tutoriel qui explique comment utiliser l'API de son site [Kiubi](http://www.kiubi.com) pour créer des bundles de produits.

L'objectif étant de pouvoir ajouter dans son panier, en un seul clic, un produit et un de ses produits associés. Avec l'aide de l'API front-office, cela prendra la forme d'un encart "bundles de produits", affiché dans les pages de détail de produit. Un bouton permettra d'ajouter directement un bundle au panier, puis de rediriger l'internaute vers ce dernier.

![image](./docs/bundles.png?raw=true)

## Prérequis

Ce tutoriel suppose que vous avez un site Kiubi et qu'il est bien configuré : 

 - l'API est activée
 - l'e-commerce est activé
 - le site est en thème personnalisé, basé sur Shiroi

Il est préférable d'être à l'aise avec la manipulation des thèmes personnalisés. En cas de besoin, le [guide du designer](http://doc.kiubi.com) est là.

Ce tutoriel est applicable à tout thème graphique mais les exemples de codes donnés sont optimisés pour un rendu basé sur le thème Shiroi.


## Ajout d'une liste de bundles dans une page de détail produit

L'objectif est d'intégrer à une page de détail de produit une liste de composition (un produit + un produit associé), avec le tarif correspondant à la somme des prix unitaires de ces 2 produits.

Pour y arriver, on utilise ici plusieurs composants :

 - le framework jQuery pour les manipulations javascript de base
 - le client Javascript API Front-office de Kiubi (qui est un plugin jQuery) pour récupérer les produits

Nous allons créer un fragment de template spécifique pour faciliter sa mise en place.

### Mise en place 

Copier le fichier `/theme/fr/includes/bundles.html` de ce dépôt dans le dossier `fr/includes/` du thème personnalisé. 

Dans le Back-office du site, éditer une mise en page de type "détail de produit". 

![image](./docs/mise_en_page.png?raw=true)

Placer le widget "Fragment de template" dans la page. Editer la configuration du widget et choisir le template "bundles". 

![image](./docs/fragment.png?raw=true)

La page de détail d'un produit affiche désormais un encart "bundles produits". Si l'encart ne s'affiche pas, c'est que le produit ne dispose pas de produit associé disponible et en stock. Il suffit alors d'associer quelques produits ou de rendre disponible les produits éventuellement déjà associés pour remédier à la situation.


### Explications

Examinons en détail le code HTML du fragment de template `bundles.html` :

<pre lang="html">
&lt;style&gt;
ul.bundles li {
	clear: both;
	display: block;
	height: 130px;
	list-style: none;
	margin-bottom: 10px;
}
ul.bundles li div {	margin: 20px; width: 200px; }
ul.bundles li a img {
	border: 1px solid #EEE;
	height: 120px;
	vertical-align: middle;
	width: 120px;
}
ul.bundles li span.plus {
	color: #AAA;
	font-size: 2.5pc;
	line-height: 120px;
	margin: 0 10px;
	vertical-align: middle;
}
ul.bundles li a,
ul.bundles li div, 
ul.bundles li span.plus {
	float: left;
	display: block;
}
&lt;/style&gt;

&lt;article id="produit_bundles" style="display:none"&gt;
  &lt;h2&gt;Bundles produits&lt;/h2&gt;
  &lt;ul class="bundles"&gt;
	  &lt;li style="display:none" id="bundle_template"&gt;
		  &lt;a class="product_1" href="#"&gt;&lt;img src="{racine}/{theme}/{lg}/images/produit_mini.gif"/&gt;&lt;/a&gt;
		  &lt;span class="plus"&gt;+&lt;/span&gt;
		  &lt;a class="product_2" href="#"&gt;&lt;img src="{racine}/{theme}/{lg}/images/produit_mini.gif"/&gt;&lt;/a&gt;
		  &lt;div&gt;
			  &lt;strong&gt;Les 2 produits pour &lt;span class="price"&gt;&lt;/span&gt;&lt;/strong&gt;
			  &lt;br/&gt;&lt;input type="submit" value="Ajouter au panier" /&gt;
		  &lt;/div&gt;
	  &lt;/li&gt;
  &lt;/ul&gt;
&lt;/article&gt;
</pre>
	
Cette première portion de code met en place le bloc destiné à afficher les bundles de produits. L'élement `<li>` servira ici de template pour générer les différentes compositions de produit.

On inclut ensuite le client Javascript API Front-office de Kiubi :

<pre lang="html">
&lt;script type="text/javascript" src="{cdn}/js/kiubi.api.pfo.jquery-1.0.min.js"&gt;&lt;/script&gt;
</pre>

On détermine l'identifiant unique du produit en cours d'affichage, appelé `product_id`, et on récupère ensuite les informations liées au produit ainsi que ses produits associés :

<pre lang="javascript">
&lt;script&gt; 
jQuery(function($){
	
	// On cherche le produit id courant dans la page.
	var product_id = parseInt($('.produit_detail input[name=pid]').val());
	
	// le produit_id n'a pas été trouvée, le script se termine ici
	if(!product_id) return;
	
	var query = $.when(
		kiubi.catalog.getProduct(product_id, {extra_fields : "price_label,variants"}),
		kiubi.catalog.getLinkedProducts(product_id, {extra_fields : "price_label,variants", limit : 3, sort:'rand' })
	);
</pre>
		
Les compositions vont être créés à partir du résultat de deux requêtes vers l'API. La première va récupérer toutes les informations du produit de la page en cours et la seconde va chercher tous les produits associés à ce produit. La méthode [jQuery.when()](http://api.jquery.com/jQuery.when/) permet ici d'encapsuler les 2 requêtes asynchrones au sein d'un nouvel objet. Cet objet disposera des méthodes `done()` et `fail()` correspondant à la résolution des 2 requêtes ou à l'échec d'au moins l'une d'entre elles.

On attache un callback au succès des deux requêtes. Ce callback prend en argument autant de variables que de requêtes qui ont été encapsulées par la méthode `when()`. 

Concrètement, `request_1` et `request_2` contiendront respectivement un tableau `[meta, data]` des requêtes `kiubi.catalog.getProduct` et `kiubi.catalog.getLinkedProducts`. On notera que la deuxième requête est paramétrée pour retourner 3 produits associés triés aléatoirement. Libre à vous de modifier ces critères selon vos besoins.

On récupère au finale dans le callback `done` les "données" de ces deux requêtes dans la clé `[1]` de chacun des arguments :

<pre lang="javascript">
	query.done(function(request_1, request_2){

		var product = request_1[1];
		var linked_products = request_2[1];
</pre>
	
On itère ensuite sur chacun des produits associés pour créer toutes les compositions dans la page :

<pre lang="javascript">
		$.each(linked_products, function(num, linked_product){
</pre>
	
A ce stade, `product` contient un objet correspondant au produit de la page en cours d'affichage, et `linked_product` (sans `s`) contient un objet correspondant à UN produit associé à celui de la page en cours d'affichage. On créé un bundle avec ces 2 objets : *La méthode `createBundle` est décrite plus loin*

<pre lang="javascript">
			createBundle(product, linked_product);
</pre>

Enfin, si au moins un bundle a été ajouté à la liste des bundles, on affiche tout le bloc `#produit_bundle` :

<pre lang="javascript">
		}); // fin du each
		
		$('#produit_bundles ul.bundles li').length > 1 && $('#produit_bundles').show();
			
	}); // fin du query.done
</pre>



&nbsp;

On définit à présent une fonction `createBundle` acceptant en argument 2 produits. Cette fonction aura pour but de calculer le prix du bundle et de l'afficher dans le listing de bundle présenté plus haut.

<pre lang="javascript">
	function createBundle(a, b) {
		
		var variant_a = getVariant(a);
		var variant_b = getVariant(b);
</pre>
			
La méthode `getVariant` est décrite plus loin, elle permet de récupérer les informations de la première variante disponible et en stock d'un produit.

Si aucune variante disponible et en stock n'a été trouvée pour chacun des 2 produits, on abandonne le rendu du bundle :

<pre lang="javascript">
		if(!variant_a || !variant_b) return;
</pre>

On créé une copie du template, en prenant soin de supprimer l'attribut `id` de l'élément cloné :

<pre lang="javascript">
		var bundle = $('li#bundle_template').clone().attr("id", "");
</pre>

On met à jour les images d'illustration avec les données des variables `variant_a` et `variant_b` :
			
<pre lang="javascript">
		if(variant_a.thumb) $('.product_a img', bundle).attr('src', variant_a.thumb.url_miniature);
		if(variant_b.thumb) $('.product_b img', bundle).attr('src', variant_b.thumb.url_miniature);
</pre>
			
On met à jour titre et URL des 2 liens de produits :

<pre lang="javascript">
		$('.product_a', bundle).attr({href: a.url, title: a.name});
		$('.product_b', bundle).attr({href: b.url, title: b.name});
</pre>



<pre lang="javascript">
		var price = (variant_a.price_inc_vat + variant_b.price_inc_vat).toFixed(2);
		$('.price', bundle).html(price.replace(/\./,',') + ' &euro;');
</pre>
	
On calcul le prix arrondi à 2 chiffres après la virgule.

<pre lang="javascript">
		$('input[type=submit]', bundle).click(function(){
			var items = {};
			items[variant_a.id] = "+1";
			items[variant_b.id] = "+1";
			kiubi.cart.addItems(items).done(function(){
				document.location = "/ecommerce/panier.html"
			});
		});
</pre>

On attache un callback au clic sur le bouton "Ajouter au panier", qui aura pour objet d'ajouter les 2 produits au panier et de rediriger l'internaute vers ce dernier.

<pre lang="javascript">
		bundle.appendTo('#produit_bundles ul.bundles').show();
		
	} // fin de la fonction createBundle
</pre>
		
On ajoute enfin le bundle créé à la liste des bundles.

<pre lang="javascript">
	function getVariant(product) {
			
		if(!product.variants) return false;
		
		var variant = false;
		$.each(product.variants, function() {
			if(!this.is_available || variant) return;
			variant = this;
		});
		
		return variant;
	}

}); // fin de jQuery(function($){
&lt;/script&gt;
</pre>

La page affiche désormais au maximum 3 bundles aléatoires composés avec les produits associés au produit en cours d'affichage.
