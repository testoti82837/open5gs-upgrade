# Contribuer à open5gs-Production

Ce dépôt est un fork modifié d'Open5GS ; lire d'abord la
[notice de fork](FORK-NOTICE.md).

**Pour contribuer au projet Open5GS amont, ce dépôt n'est pas le bon endroit** :
passer par <https://github.com/open5gs/open5gs> et son
[Contributor License Agreement](https://open5gs.org/open5gs/cla/).

## Stratégie de branches

| Branche | Rôle |
| --- | --- |
| `production/v2.8.x` | **Branche de production.** Base gelée sur le tag amont `v2.8.0` plus les correctifs rétroportés et les développements du fork. Seule branche intégrable. |
| `main` | Miroir en lecture seule de `main` amont (Open5GS). Sert uniquement de source pour le tri et le rétroportage. Ne reçoit aucun développement du fork. |

### Règles non négociables

1. **Toute modification part de `production/v2.8.x`.** Une branche par ticket, créée
   depuis `production/v2.8.x`, dans son propre *worktree*.
2. **Aucun rebase sur `main`.** La branche de production n'est jamais rebasée, ni
   fusionnée, ni réalignée sur `main` amont, ni sur un tag amont ultérieur. La stratégie
   amont retenue est le **gel sur v2.8.x avec rétroportage** ; la remettre en cause est
   une décision de cadrage, pas une opération de branche.
3. **Les correctifs amont arrivent par rétroportage explicite**, un commit amont à la
   fois (`git cherry-pick -x`), regroupés par domaine (NAS/NGAP/S1AP, GTP/PFCP, SBI,
   Diameter) et une demande de fusion par lot. Le message de commit conserve la
   référence du commit amont d'origine.
4. **Un ticket, une demande de fusion.** Une demande de fusion couvre une procédure ou
   un jeu de messages d'une interface, avec ses tests et une trace PCAP de référence.
5. **Aucun nouvel `ogs_assert` sur une entrée externe.** Une donnée venant du réseau qui
   ne respecte pas la spécification est rejetée conformément à celle-ci, et incrémente
   un compteur. Jamais d'abandon de processus.
6. **Les sections 3GPP et GSMA implémentées sont citées** depuis le texte de la
   spécification, jamais de mémoire.
7. **Les messages produits sont décodés sans erreur par Wireshark / tshark**, en plus
   des tests du dépôt.
8. **Jamais de `git add -A` ni `git add .`** : nommer les fichiers explicitement. Les
   *worktrees* des agents vivent sous `.paperclip/worktrees/` à l'intérieur du dépôt ;
   ils sont ignorés par `.gitignore`, mais un ajout global reste à éviter — il
   embarquerait aussi tout artefact de build non ignoré.

## Conditions de fusion

Une demande de fusion n'entre dans `production/v2.8.x` qu'avec :

- l'intégration continue verte (build Debug avec `-Db_sanitize=address,undefined`,
  suite de tests, cibles de fuzzing en mode court, et build Release) ;
- une **relecture par quelqu'un qui n'a pas écrit le code** ;
- pour les **zones critiques**, une **revue humaine en plus** de cette relecture :
  TLS et OAuth2, interception légale, SEPP et itinérance, socket de commande et accès
  SSH, facturation.

## Licence des contributions

En contribuant, vous acceptez que votre contribution soit distribuée sous la
GNU Affero General Public License v3 ou ultérieure, licence de l'œuvre d'origine
conservée par ce fork. Ne pas modifier le fichier `LICENSE` ni retirer une mention de
droit d'auteur amont.
