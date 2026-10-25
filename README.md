# MMW Gestion — Dahira Miftaahul Munnah Wassoul

Application web de gestion du dahira, sur ordinateur, tablette et téléphone (installable comme une application) :
adhérents (inscription automatique par le formulaire Google), commissions et bureau, cartes de membre,
mensualités (janvier → décembre, à verser avant le 10), cotisation annuelle par campagne (jusqu'au Magal),
journal des encaissements et reçus, relances WhatsApp, dépenses, état d'un membre, rapports Excel/PDF,
import Excel, sauvegardes automatiques.

```
index.html, manifest.json, sw.js, logo-256.png, icon-*.png  → l'application
MMW_modele_import.xlsx         → modèle à remplir (cartes, mensualités, cotisations, dépenses), pré-rempli avec les 84 adhérents
supabase/01_schema.sql         → base de données : tables, règles, sécurité, import, sauvegarde
supabase/02_import_excel.sql   → reprise des ADHÉRENTS de « MMW .xlsx » (aucun montant)
supabase/03_tests.sql          → 48 tests automatiques des règles (projet de TEST ; base non modifiée)
supabase/04_nettoyage_tests.sql → retire les restes d'une ancienne exécution interrompue des tests
google-apps-script/Code.gs     → formulaire → base, activité quotidienne, sauvegarde hebdomadaire dans Google Drive
tools/restaurer.py             → transforme une sauvegarde en script de restauration
tools/gen_import.py, gen_modele.py → régénèrent la reprise et le modèle depuis l'Excel
```

---

## 1. Architecture

| Brique | Choix | Pourquoi |
|---|---|---|
| Interface | Un seul fichier `index.html` (sans framework) | Rien à compiler ; mise à jour = remplacer un fichier |
| Hébergement | Cloudflare Pages (via GitHub) | Gratuit, HTTPS ; un aperçu par branche sert d'environnement de test |
| Base | Supabase (PostgreSQL) | Contrôles et calculs faits **dans la base** ; connexion sécurisée ; droits par rôle (RLS) |
| Formulaire et sauvegardes | Google Forms + Apps Script | Formulaire actuel conservé ; tâches automatiques gratuites |

### Supabase gratuit : éviter la mise en pause et garder des sauvegardes
L'offre gratuite met le projet en pause après 7 jours d'activité trop faible ; Supabase indique que quelques requêtes
par jour suffisent à l'éviter, et qu'un projet en pause se relance depuis le tableau de bord (données intactes, jusqu'à
1 an) ([documentation Supabase](https://supabase.com/docs/guides/platform/free-project-pausing.md)). L'offre gratuite
n'a pas de sauvegarde automatique ; l'offre Pro (25 $/mois) en garde 7 jours
([Jet Admin, juillet 2026](https://www.jetadmin.io/blog/supabase-pricing-2026-guide-to-plans-limits-and-real-world-costs/)).

Ce qui est en place, sans frais, dans le script Apps Script du formulaire :
- **chaque jour vers 7 h** : une lecture de la base (`etat_base`) → activité régulière, le projet reste éveillé ;
- **chaque dimanche vers 2 h** : sauvegarde complète en JSON dans Google Drive, dossier « MMW Sauvegardes »
  (les 12 dernières sont gardées, soit environ 3 mois) ;
- si une tâche échoue, Google envoie un e-mail récapitulatif au propriétaire du formulaire.
Si Supabase vous envoie quand même un avertissement de pause, ouvrez simplement l'application ou le tableau de bord Supabase.

**Test et production** : deux projets Supabase (l'offre gratuite en permet deux). `CONFIG.production` et `CONFIG.test`
dans `index.html` ; l'adresse contenant « test » ou « preview » (ou `?env=test`) utilise le projet de test, avec un bandeau orange.

---

## 2. Installation (≈ 30 minutes)

### Base de données (projet de test d'abord, puis production)
1. supabase.com → New project (région Europe).
2. **SQL Editor** → exécuter `supabase/01_schema.sql`.
3. **SQL Editor** → exécuter `supabase/02_import_excel.sql` : les 84 adhérents, les commissions et le bureau. Aucun montant.
4. Projet de **test** seulement : coller tout `supabase/03_tests.sql` et cliquer Run → la 1re ligne affiche « BILAN : 48 réussis, 0 échecs » (tout est annulé à la fin, la base n'est pas modifiée).
5. **Authentication › Users › Add user** : votre compte (« Auto confirm »). **Le premier compte devient administrateur.**
6. Désactiver l'inscription libre (« Allow new users to sign up ») dans les réglages d'Authentication.
7. **Project Settings › API** : noter l'URL et la clé **anon public**.

### Application
1. Renseigner `CONFIG.production` (et `CONFIG.test`) en haut du `<script>` de `index.html`.
2. Pousser sur GitHub → Cloudflare Pages (pas de commande de build). Ouvrir en **https**, se connecter.

### Charger les montants avec le modèle Excel
1. Ouvrir `MMW_modele_import.xlsx` (ou application › **Import Excel** › Télécharger le modèle, toujours à jour des adhérents).
2. Remplir les cases jaunes : feuilles Mensualités (une colonne par mois), Cotisation annuelle (montant annuel + versé par tranche),
   Cartes, Dépenses. La colonne « Mensualité » est déjà remplie d'après le formulaire : corrigez-la si besoin.
3. Application › Import Excel › choisir l'année (Mensualités 2026) et la campagne (Magal 2026) › charger le fichier.
4. Vérifier l'aperçu (totaux par type, lignes écartées) › **Importer**. Un rapport liste les lignes ignorées (exportable en Excel).
   Ré-importer le même fichier ne crée aucun doublon : ce qui est déjà payé est ignoré.
5. Une fois la campagne Magal 2026 complète : Cotisations annuelles › **Remettre à zéro** pour ouvrir Magal 2027.

### Formulaire Google et tâches automatiques
1. Ouvrir le formulaire en modification : https://docs.google.com/forms/d/1otq-JxQVPCe0Tu8PkMGxbdV29DLsEBwrA7ImFV2HN0A/edit
2. ⋮ › **Éditeur de scripts** → remplacer le contenu par `google-apps-script/Code.gs`.
3. Remplir `CONFIG` : URL Supabase, clé anon, et les deux jetons (application › Paramètres › « Jeton du formulaire » et
   « Jeton de la sauvegarde automatique »).
4. Exécuter **`installer`** (accepter les autorisations Google : formulaire, Drive, accès internet) → 3 déclencheurs.
5. Exécuter **`rattraperReponsesExistantes`** une fois, puis **`sauvegardeHebdomadaire`** une fois pour vérifier le fichier dans Drive.

---

## 3. Règles de gestion

### Vos règles (appliquées telles quelles)
- **Mensualités de janvier à décembre**, à verser **avant le 10** : « à temps » avant, « en retard » après.
- **Cotisation annuelle par campagne, qui se termine au Magal**, en 3 tranches de montants libres ; la 3e est due au Magal.
  Bouton **Remettre à zéro** : clôture la campagne et en ouvre une nouvelle ; chaque membre actif repart avec ses 3 tranches.
  Rien n'est effacé (l'ancienne campagne reste consultable) ; les restes impayés de l'ancienne campagne sont, par défaut, annulés.
- Carte de membre à 1 000 FCFA ; « payée » seulement si un encaissement existe.
- Responsables désignés uniquement par un administrateur, avec historique des mandats.
- Aucun encaissement supprimé : annulation avec motif, date et nom de l'agent. Numéro unique REC-AAAA-NNNNN, reçu imprimable.
- Échéances générées automatiquement, jamais en double ; un paiement ne peut ni être compté deux fois ni dépasser le dû.
- Montants historisés : un changement ne touche que les échéances pas encore réglées.
- La synchro du formulaire n'écrase jamais une fiche existante.

### Hypothèses (paramétrables)
| Sujet | Ce que fait l'application |
|---|---|
| Début des mensualités | Dues à partir du mois d'adhésion |
| Montant mensuel | Celui choisi au formulaire, modifiable par membre (ou par le modèle Excel) |
| Cotisation 2026 | Pas de montant commun fixé : montants individuels via le modèle, ou montant commun dans Paramètres |
| Dépenses | Rattachées à l'année civile |
| Paiements partiels | Autorisés (Paramètres) |
| Responsables de commission | Voient leurs membres ; les paiements seulement si autorisé (Paramètres) |
| Rôle « Secrétaire » | Ajouté (adhérents, commissions, formulaire) |
| Santé, groupe sanguin, contact d'urgence | Visibles de l'administrateur et du secrétariat seulement |
| Relances | Messages WhatsApp pré-remplis envoyés depuis le téléphone de l'agent, avec historique (pas d'envoi automatique) |

### Rôles
| Rôle | Peut faire |
|---|---|
| Administrateur | Tout : périodes, montants, remise à zéro, import, responsables, utilisateurs, sauvegarde |
| Trésorier | Encaisser, annuler, reçus, dépenses, montants individuels de cotisation, relances |
| Secrétaire | Adhérents, commissions, formulaire, relances |
| Membre du bureau | Consultation des tableaux de bord, listes et états |
| Responsable de commission | Les membres de sa commission (et leurs paiements si autorisé) |

---

## 4. Audit du fichier Excel

| Feuille | Contenu | Reprise |
|---|---|---|
| Base de données | 84 réponses du formulaire (oct. 2024 → mai 2026) | `membres` (matricules conservés : 1 → MMW-0001) ; santé à part |
| Commissions | 7 commissions, 6 instances, têtes et adjoints | `commissions` + `responsables` |
| Cotisation membres, Mensualités, Dépenses | Montants | **non repris** (à votre demande) → modèle Excel |
| Tableau de bord, RECAP ANNUEL | Formules | recalculés par l'application |

Défauts du classeur que la nouvelle structure supprime : feuilles liées par numéro de ligne, septembre hors de la formule
des mensualités, membres 59 à 84 sans total, ajustements manuels non justifiés dans les formules, `#REF!` et `COUNTUNIQUE`
cassés, « Hygiene » ≠ « Hygiène », matricules du bureau sans rapport avec ceux des adhérents.

### Points à vérifier sur les adhérents
**Doublon potentiel**

- Matricule 70 : Même téléphone (777327580) que le matricule 57
- Matricule 70 : Mêmes nom et prénom que le matricule 57 (Serigne Adiouma FAYE)
- Matricule 77 : Même téléphone (777484899) que le matricule 50
- Matricule 77 : Mêmes nom et prénom que le matricule 50 (Cheikh Ndong)

**Mensualité**

- Matricule 2 : A coché « 3000 ou plus » mais 2000 retenu dans l'Excel
- Matricule 5 : A coché « 3000 ou plus » mais 2000 retenu dans l'Excel
- Matricule 7 : A coché « 3000 ou plus » mais 2000 retenu dans l'Excel
- Matricule 58 : Montant précisé au formulaire 3000 mais 2000 retenu dans l'Excel
- Matricule 63 : « 3000 ou plus » sans montant valable précisé (Oui) : 3000 retenu
- Matricule 67 : Montant précisé au formulaire 3000 mais 2000 retenu dans l'Excel
- Matricule 69 : « 3000 ou plus » sans montant valable précisé (vide) : 3000 retenu
- Matricule 70 : « 3000 ou plus » sans montant valable précisé (vide) : 3000 retenu

**Bureau**

- Responsable « Cheikh Diarra NDIAYE » (Dieuwrigne) : pas de fiche correspondante sans ambiguïté — enregistré par son nom, à lier à une fiche
- Responsable « Yacine Pene NDONG » (Dieuwrigne Soxna yi) : pas de fiche correspondante sans ambiguïté — enregistré par son nom, à lier à une fiche
- Responsable « Ousmane NDIAYE PEPE » (Communication / Digital) : pas de fiche correspondante sans ambiguïté — enregistré par son nom, à lier à une fiche
- Adjoint « Adiouma faye » (Communication / Digital) : pas de fiche correspondante sans ambiguïté — enregistré par son nom, à lier à une fiche
- Adjoint « Muhammadu LO » (Finance) : pas de fiche correspondante sans ambiguïté — enregistré par son nom, à lier à une fiche
- Responsable « Muhammadu lo » (Trésorerie) : pas de fiche correspondante sans ambiguïté — enregistré par son nom, à lier à une fiche
- Les numéros de « Matricule » de la feuille Commissions (1 à 12) ne correspondent pas aux matricules des adhérents : non repris
- Commission Sociale sans responsable désigné ; commission Administration absente de la feuille Commissions

### À garder en tête en remplissant le modèle (constats dans les anciennes feuilles de montants)
**Carte**

- Matricule 54 : Carte « Cotisé » sans montant saisi : 1000 FCFA enregistrés (à confirmer)
- Matricule 56 : Carte « Cotisé » sans montant saisi : 1000 FCFA enregistrés (à confirmer)

**Cotisation annuelle**

- Matricule 33 : Tranche 3 marquée « Cotisé » sans montant exploitable ('Total') : non importée
- Matricule 37 : Tranche 1 marquée « Cotisé » sans montant exploitable (None) : non importée
- Matricule 52 : Versé 20000 FCFA pour un montant annuel de 15000 : montant annuel porté à 20000 pour ne perdre aucun encaissement (à confirmer)

**Mensualités**

- Matricule 5 : Total Excel 3000 FCFA ≠ 2 mois « Cotisé » × 2000 = 4000 (formule : =('Base de données'!P6*F6)-1000)
- Matricule 6 : Total Excel 11000 FCFA ≠ 5 mois « Cotisé » × 2000 = 10000 (formule : =('Base de données'!P7*F7)+1000)
- Matricule 7 : Total Excel 3000 FCFA ≠ 2 mois « Cotisé » × 2000 = 4000 (formule : =('Base de données'!P8*F8)-1000)
- Matricule 37 : Total Excel 40000 FCFA ≠ 9 mois « Cotisé » × 5000 = 45000 (formule : ='Base de données'!P38*F38) — le mois de septembre est hors de la plage G:P de la formule
- Matricule 57 : Total Excel 19000 FCFA ≠ 5 mois « Cotisé » × 5000 = 25000 (formule : =('Base de données'!P58*F58)-6000)
- Matricule 58 : Total Excel 8000 FCFA ≠ 5 mois « Cotisé » × 2000 = 10000 (formule : =('Base de données'!P59*F59)-2000)
- Matricule 63 : 2 mois marqués « Cotisé » mais aucune formule de total : non comptés dans le tableau de bord Excel
- Matricule 64 : 4 mois marqués « Cotisé » mais aucune formule de total : non comptés dans le tableau de bord Excel
- Matricule 68 : 5 mois marqués « Cotisé » mais aucune formule de total : non comptés dans le tableau de bord Excel

**Dépenses**

- Magal décembre : valeur texte « 1.600000 » non comptée dans l'Excel ni importée (montant à préciser)

---

## 5. Sauvegarde et restauration
- **Automatique** : chaque dimanche dans Google Drive (section 1).
- **À la demande** : Paramètres › Sauvegarde (JSON), ou « Copie Excel des tables ».
- **Restaurer** sur un projet Supabase neuf : exécuter `01_schema.sql` ; puis
  `python3 tools/restaurer.py MMW_sauvegarde_xxx.json > restauration.sql` et exécuter `restauration.sql` ; recréer les
  comptes (mêmes e-mails) et exécuter les lignes `update public.profils …` affichées en fin de fichier.
  Testé : la base restaurée est identique (adhérents, échéances, paiements, matricules, numéros de reçus).

---

## 6. Guide utilisateur
- **État d'un membre** : champ « État d'un membre… » en haut du menu (loupe sur téléphone). Tapez un nom, un matricule ou
  un téléphone : carte, mois en cours, mois en retard, cotisation, arriérés cumulés ; boutons Fiche et Encaisser ; export Excel.
- **Encaisser** : case rouge dans Mensualités, bouton dans Cotisations, ou depuis la fiche. Cochez les mois ou tranches :
  le montant se calcule ; un montant plus petit = paiement partiel. Choisissez le mode et la référence, puis « Encaisser » :
  le reçu s'ouvre (« Imprimer / enregistrer en PDF »).
- **Corriger** : Encaissements › la ligne › « Annuler cet encaissement » (motif), puis ressaisir.
- **Relancer** : Relances › « Rappel avant le 10 » en début de mois, « Retards après le 10 » ensuite, « Tranches » avant chaque échéance.
- **Nouvelle année** (en décembre) : Paramètres › Nouvelle année → les 12 mensualités de chaque membre sont créées ; « Rendre courante » en janvier.
- **Après le Magal** : Cotisations annuelles › Remettre à zéro (date du prochain Magal et montants des tranches).
- **Exporter** : chaque écran a ses boutons Excel / CSV / PDF ; la fiche d'un membre s'exporte en Excel (fiche, échéances, paiements) ;
  Rapports regroupe tous les états (liste, par commission, bureau, cartes, cotisations, mensualités, impayés, journal, recettes, rapport annuel).

---

## 7. À valider
1. Montant de la cotisation de la campagne **Magal 2027** et de ses tranches (à saisir au moment de « Remettre à zéro »),
   et la date exacte du Magal 2027.
2. Les deux doublons d'adhérents (matricules 57/70 et 50/77).
