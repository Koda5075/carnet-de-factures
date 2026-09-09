# Carnet de Factures

Générateur gratuit de devis/factures pour indépendants et micro-entrepreneurs
français. Actuellement un **site statique à page unique**, sans backend, sans
build step.

> Le fichier applicatif est `index_1.html` (c'est lui qui est déployé). Les
> instructions parlent parfois de « index.html » — c'est le même fichier.

## Contexte produit

- Cible : freelances/auto-entrepreneurs qui n'ont pas d'outil de facturation.
  Zéro friction voulue : pas de compte, pas de connexion, ouverture directe.
- Le produit doit rester utilisable sans backend le plus longtemps possible —
  c'est un choix délibéré, pas un oubli. Ne pas ajouter de compte utilisateur ou
  de backend sans une vraie raison validée par l'usage (voir « Prochaines
  étapes » en bas).
- **Le plus important reste l'ergonomie** : il faut que l'utilisateur se dise
  que c'est pratique et qu'il le réutilisera. Produit encore non validé, sans
  marketing ; la rétention vient uniquement de l'impression « ça m'a fait gagner
  du temps, j'y reviens ». Pour toute évolution, privilégier ce qui réduit la
  friction de ré-usage avant les features.
- Direction artistique : inspirée du carnet de factures autocopiant papier
  (carbone) utilisé par les artisans/commerçants français. Palette : fond papier
  (`--papier`), encre bleu marine (`--encre`), rose « duplicata » (carbone) en
  accent secondaire, rouge « tampon » en accent d'action. Typo : Archivo
  (titres/labels) + IBM Plex Mono (tout ce qui est chiffré : montants, dates,
  numéros). Respecter ce système de design dans toute évolution — ne pas dériver
  vers un style générique.

## Stack technique

- Un seul fichier `index_1.html` : HTML + CSS + JS vanilla, pas de framework,
  pas de bundler.
- Dépendances externes chargées en CDN (`html2canvas` 1.4.1 et `jspdf` 2.5.1,
  UMD, depuis cdnjs) — export PDF fait en rendant le nœud `#docPage` en canvas
  puis en image JPEG insérée dans un PDF A4 via jsPDF.
- Persistance : `localStorage` uniquement (clés préfixées `cdf_`) — infos
  société, clients enregistrés, historique des documents, compteurs de
  numérotation, brouillon en cours, clé de licence, quota PDF. Rien n'est envoyé
  à un serveur.
- Police : Google Fonts (`Archivo`, `IBM Plex Mono`) via `<link>`.
- Déployé en statique sur Vercel. Le fichier étant servi seul (hors wrapper
  d'artifact), il lui faut son propre `<head>` complet (`<meta charset>` /
  `<meta viewport>`) — sans ça les accents et le « € » cassent.

## Ce qui existe déjà (fonctionnel, testé)

- Formulaire société / client / lignes de prestation avec calcul HT / TVA
  (multi-taux) / TTC en direct.
- Bascule Facture / Devis avec numérotation automatique (préfixe FA-/DV-, année,
  compteur local qui s'incrémente réellement à la 1ʳᵉ sauvegarde).
- Mode micro-entrepreneur : désactive la TVA sur les lignes et ajoute la mention
  légale automatiquement.
- Upload de logo (converti en data URL, affiché dans l'aperçu et le PDF).
- Aperçu en direct qui reproduit exactement le document exporté.
- Export PDF (bouton « Télécharger le PDF »), avec quota plan gratuit
  (3 PDF/jour) et filigrane retiré en Premium.
- Historique des documents (localStorage) avec statut cyclable Brouillon /
  Envoyée / Payée, ouverture au clic ou au clavier, « Dupliquer », et export CSV.
- Clients enregistrés, réutilisables via un menu déroulant.
- Brouillon auto-enregistré (`cdf_draft_v1`) restauré au chargement.
- Toasts de confirmation ; modale de confirmation intégrée pour les actions
  destructives (pas de `confirm()`/`alert()` natifs).
- Passe d'accessibilité : `<label for>` sur tous les champs (y compris les
  lignes de prestation), focus clavier visible partout, ordre de tabulation
  logique.
- Thème clair/sombre automatique (le document PDF/aperçu reste toujours sur fond
  papier clair, volontairement, même en mode sombre — c'est un vrai document,
  pas une UI).
- Premium : Stripe Payment Links + clé de licence ECDSA P-256 vérifiée
  hors-ligne. Clé privée dans `outils-cles.html` (local, jamais déployé, exclu
  par `.gitignore`).

## Bugs corrigés récemment (ne pas régresser)

1. **Perte de focus en tapant un prix/quantité** : les champs qté/prix ne
   doivent JAMAIS déclencher un `renderItems()` complet (qui recrée tous les
   inputs et fait perdre le focus). Ils mettent à jour uniquement leur propre
   montant + les totaux (`refreshRow()`), jamais toute la liste.
2. **Dates absurdes** : les `<input type=date>` natifs peuvent transformer une
   saisie manuelle à 2 chiffres dans le champ année en année improbable (ex.
   « 08 » → 2008 au lieu de 2026). Il y a un contrôle de plausibilité
   (`checkDateSanity`) qui affiche un avertissement visible sans bloquer la
   saisie. Ne pas le retirer.
3. **Logo cassé dans le PDF** : `html2canvas` peut capturer une image pas encore
   décodée. L'export attend `img.decode()` sur toutes les images du nœud capturé
   avant de lancer la capture (`waitForImages()`), et masque une image qui
   échoue plutôt que de laisser un glyphe cassé.
4. **PDF à 8 Mo pour une facture d'une page** : `jsPDF.addImage` avec du PNG
   issu d'un canvas stocke les pixels quasiment sans compression. Export en JPEG
   qualité 0.95 à la place → ~200 Ko, qualité visuelle inchangée pour du texte.
   Ne pas repasser en PNG sans vérifier la taille du fichier généré.
5. **Molette de souris qui change un nombre/une date sans le vouloir** : les
   inputs number/date font `blur()` au `wheel` pour éviter qu'un défilement de
   page modifie une valeur par accident.

## Comment vérifier une modification avant de la considérer terminée

Ce projet est vérifié avec Playwright headless avant publication (capture
d'écran + interactions clavier réelles + inspection du PDF/CSV généré, pas
seulement « ça compile »). Pour toute modification touchant à la saisie ou à
l'export :

- Vérifier qu'un champ garde le focus après plusieurs frappes successives (le
  bug n°1 ci-dessus est facile à réintroduire par erreur).
- Générer un vrai fichier de test (PDF, CSV) et vérifier sa taille ET son
  contenu (texte, logo, montants, entêtes) avant de dire qu'une fonctionnalité
  est prête.
- Tester en thème clair ET sombre, et à 375 px de large (mobile) : pas de scroll
  horizontal, cibles tactiles ≥ 40 px.
- Naviguer la page entière au clavier : ordre de Tab logique, anneau de focus
  visible partout, modales pilotables (focus initial, Tab piégé, Échap = fermer).

## Déploiement

- Déployé en statique sur Vercel. Un `git push` sur le dépôt lié redéploie
  automatiquement sur la même URL (intégration Git de Vercel). À défaut de
  dépôt lié : drag & drop du fichier sur vercel.com/drop.
- Pas de variables d'environnement, pas de build command (site 100 % statique).
- `outils-cles.html`, `*.pem`, `*.der`, `*CLE-PRIVEE*` sont exclus par
  `.gitignore` — ils contiennent la clé privée de signature des licences et ne
  doivent JAMAIS être committés ni déployés.

## Prochaines étapes envisagées (backlog, non urgent)

- Factures récurrentes.
- Renseigner les `STRIPE_LINK_*` une fois les Payment Links créés.
- Comptes utilisateur + synchronisation multi-appareils : seulement si la
  demande est confirmée. Si ça devient nécessaire, **Supabase** (Postgres + auth
  managés, s'intègre facilement à Vercel) est le choix raisonnable plutôt que de
  construire un backend maison — mais ne pas l'introduire avant d'avoir une
  vraie raison (aujourd'hui tout fonctionne en `localStorage`, sans compte, ce
  qui est un avantage produit, pas une limitation à corriger prématurément).
