---
layout: post
title: "Nettoyer Ubuntu des programmes installés au travers d'un PPA"
taxonomies: 
  tags: 
    - Ubuntu
---
<p>Les <a href="http://doc.ubuntu-fr.org/ppa">PPA Ubuntu</a> sont bien pratiques pour tester ou évaluer des programmes qui ne sont pas encore dans les dépôts officiels d'Ubuntu ou bien dont on veut tester une version plus récente.</p> <br />
<p>Certains programmes fournissent cependant de nouvelles versions de librairies déjà présentes dans les dépôts Ubuntu et, lors de la désinstallation du paquet et la suppression du PPA, celles-ci restent dans la version fournie par le PPA et ne sont plus mises à jour par la suite !</p> <br />
<p><a href="http://www.webupd8.org/2009/12/remove-ppa-repositories-via-command.html">Ce script</a> (en bas de la page) permet de supprimer un PPA, de supprimer les paquets qu'il fournit et de les remplacer par ceux existants dans les dépôts officiels.<br /></p>
