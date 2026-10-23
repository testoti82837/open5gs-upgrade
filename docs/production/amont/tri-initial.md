# Tri initial des commits de `main` postérieurs à `v2.8.0`

Livrable du ticket **TES-8 (F1.2)**. Classe tout l'historique d'Open5GS `main` postérieur au tag de gel
`v2.8.0` en trois catégories, pour décider de ce qui est rétroporté avant F3.

Stratégie amont de référence : gel sur `v2.8.x` + rétroportage, sans rebase
(voir [decisions.md](../agents/decisions.md)).

## 1. Référence figée

Le tri porte sur un intervalle **figé par SHA**, pas sur un nombre de commits : l'amont bouge tous les
jours et un décompte serait faux dès le lendemain.

| Borne | SHA | Date |
| --- | --- | --- |
| Début (exclu) | `f2808a2fe0accd438008ff0564fe9ff4bba3d6ae` (tag `v2.8.0`) | — |
| Fin (incluse) | `f8aa307cdac2f1fd53e32fa96a1730d93bf8dc6d` (`origin/main` au 22/10/2026) | 2026-10-22 |

```
git fetch origin --tags
git rev-list --count v2.8.0..f8aa307cdac2f1fd53e32fa96a1730d93bf8dc6d   # 289
```

**Périmètre : `v2.8.0..f8aa307cd`, soit 289 commits.** Les commits amont arrivés après `f8aa307cd`
relèvent du tri hebdomadaire (F1.7, [TES-16](/TES/issues/TES-16)), pas de ce ticket.

Le document de cadrage annonçait 286 commits au 21/10/2026 et la description du ticket 288 au
22/10/2026 : l'écart est de la dérive amont, et c'est précisément pourquoi la référence est un SHA.

Décomposition des 289 : **280 commits ordinaires et 9 commits de fusion**.

## 2. Méthode

### Catégories (une et une seule par commit)

- **À rétroporter** — le message amont établit au moins un de : `abort`/`ogs_assert` déclenché par une
  entrée de pair, crash, déréférencement NULL, lecture ou écriture hors limites, dépassement de tampon,
  use-after-free ou pointeur pendant, lecture non initialisée, comportement indéfini, découverte par un
  fuzzer (OSS-Fuzz, CIFuzz), croissance non bornée déclenchable par un pair, ou défaut de sécurité
  d'authentification, d'identité, de rejeu, de TLS ou de SEPP.
- **Utile plus tard** — fonctionnel, rattaché au chantier concerné quand un des 43 chantiers le couvre.
- **Ignoré** — cosmétique, refactoring lourd, doublon, commit annulé, CI amont, documentation.

La catégorie est tirée du **message de commit amont**, pas de la seule ligne de sujet : les 289 messages
ont été dépouillés, et les 102 commits dont le message ne contenait aucun mot-clé de sûreté mémoire ont
été relus un par un.

### Domaine

Le domaine désigne **le lot de rétroportage qui portera le commit**, donc le propriétaire du code modifié,
dans cet ordre de priorité :

| Domaine | Chemins |
| --- | --- |
| `Diameter` | `lib/diameter`, `src/hss`, `src/pcrf` |
| `NAS-NGAP-S1AP` | `lib/nas`, `src/amf`, `src/mme` |
| `GTP-PFCP` | `lib/gtp`, `lib/pfcp`, `src/smf`, `src/upf`, `src/sgwc`, `src/sgwu` |
| `SBI` | `lib/sbi`, `src/nrf`, `src/scp`, `src/sepp`, `src/ausf`, `src/udm`, `src/udr`, `src/pcf`, `src/bsf`, `src/nssf`, `src/eir` |
| `core` | `lib/core`, `lib/crypt`, `lib/proto`, `lib/ipfw`, `lib/dbi`, `lib/tun`, `lib/sctp`, `lib/app` |
| `autre` | `tests`, `docs`, `configs`, `debian`, `docker`, `misc`, `webui`, `.github` |

### Sections 3GPP

Aucune section 3GPP n'est citée de mémoire dans ce document. Les quelques références de spécification
qui apparaissent dans la colonne « justification » sont **reprises telles quelles du message de commit
amont et signalées comme non vérifiées** : le corpus 3GPP n'est pas encore fourni
(voir `docs/production/specs/3gpp/MANQUANT.md`). Elles devront être revérifiées au moment du
rétroportage effectif, pas au moment du tri.

## 3. Résultat

| Catégorie | Commits | Part |
| --- | --- | --- |
| À rétroporter | 116 | 40 % |
| Utile plus tard | 94 | 33 % |
| Ignoré | 79 | 27 % |
| **Total** | **289** | **100 %** |

**0 commit non classé** : la couverture est vérifiée mécaniquement (chaque SHA de
`git rev-list v2.8.0..f8aa307cd` apparaît exactement une fois dans le tableau, et aucune ligne du
tableau ne désigne un commit hors de l'intervalle).

### Les 116 commits « à rétroporter », par lot

| Domaine | Commits | Lot |
| --- | --- | --- |
| GTP-PFCP | 37 | F1.6–F1.9, lot GTP/PFCP |
| NAS-NGAP-S1AP | 34 | F1.6–F1.9, lot NAS/NGAP/S1AP |
| SBI | 21 | F1.6–F1.9, lot SBI |
| Diameter | 10 | F1.6–F1.9, lot Diameter |
| core | 10 | **aucun lot prévu — voir §4.1** |
| autre | 4 | accompagnent leur correctif ou partent avec F4 — voir §4.1 |

## 4. Points à arbitrer par l'Orchestrateur

### 4.1 Dix correctifs « core » n'ont pas de lot

Les quatre lots de rétroportage sont découpés par interface (NAS/NGAP/S1AP, GTP/PFCP, SBI, Diameter).
Dix correctifs de sûreté mémoire sont dans `lib/core`, `lib/crypt`, `lib/proto` et `lib/ipfw`, donc
sous aucun des quatre :

`d13daddc0` `e7f9c5001` `ad7456b75` `6f87f203d` `8d7dc8007` `cf8d809af` `3c258a041` `bb810d384`
`7b8b250d1` `db5935b33`

Ce sont les plus transverses du lot (bornes TLV, `ogs_buffer_to_bcd()`, conversion hex, bornes de clé
HMAC, analyse de préfixes IP) et plusieurs sont des prérequis des autres lots.
**Proposition : un cinquième lot « core », à passer en premier.** Décision Orchestrateur.

Les quatre commits de domaine `autre` ne posent pas de question de lot : `8d5154e54`, `dc68815bb` et
`2449ff80f` sont les tests livrés avec `e7f9c5001`, `d43dc8011` et `db5935b33` et voyagent avec eux ;
`1a461d639` (permissions du `.env` de la WebUI) relève de F4, pas d'un lot de rétroportage.

### 4.2 Trois rétroportages en zone critique : revue humaine obligatoire

| Commit | Objet | Zone |
| --- | --- | --- |
| `f12556844` | Certificats client expirés acceptés dans `verify_callback` | TLS |
| `552413f49` | Un client de confiance pouvait écraser l'état de négociation N32-c d'un autre pair | SEPP / roaming |
| `1a461d639` | Secrets générés de la WebUI lisibles par tous | WebUI / secrets |

Ces trois-là ne peuvent pas être mergés sur la seule relecture agent.

### 4.3 L'audit compte une NF qui n'existe pas dans `v2.8.0`

L'audit a été écrit au commit `aafde931b` de `main`, pas à `v2.8.0`. Conséquence :

- il recense « 18 NF présentes dans `src/` » dont l'**EIR**, et rattache la commande `imei check <IMEI>`
  au chantier C3 ;
- or `src/eir` n'existe pas dans `v2.8.0` : la NF 5G-EIR est arrivée en amont le 24/07/2026
  (`42c277e8b`), après le gel.

`v2.8.0` compte **17 NF**. Le bloc EIR/S13 représente **30 commits** classés « utile plus tard » et
rattachés à aucun chantier (N5g-eir, S13 vers le MME, WebUI, tests, paquets). Deux issues :
rétroporter ce bloc comme une fonctionnalité à part entière, ou corriger l'audit en 17 NF et sortir
`imei check` de C3. **Ce n'est pas ma décision** : elle touche le périmètre, pas le tri.

### 4.4 Le harnais OSS-Fuzz conditionne le critère de F3

Dix-sept commits amont construisent ou corrigent les cibles et corpus de fuzzing. Je les ai classés
« utile plus tard » parce que ce ne sont pas des correctifs — mais le critère d'acceptation de F3 est
« 72 h de fuzzing sans abort », qui n'est pas mesurable sans eux.

**Proposition : les traiter en un lot précoce, coordonné avec le testeur**, qui possède le contenu des
tests (F2). À défaut, F3 se terminera sans pouvoir être validé.

### 4.5 Un commit amont va contre F3, explicitement non rétroporté

`dde841fa0` « amf: add missing asserts on NGAP allocations » ajoute six `ogs_assert` sur des retours de
`CALLOC()` dans `src/amf/ngap-build.c`. Ce sont des asserts sur échec d'allocation interne, donc pas sur
une entrée externe : la règle de F3 n'est pas violée à la lettre. Mais la direction est l'inverse de la
nôtre — à 99,99 %, un échec d'allocation ne doit pas abattre l'AMF. Classé **ignoré**, à ne pas reprendre.

### 4.6 Le refactoring des tables de hachage va coûter cher en conflits

`d6e4125e5`, `8f4d0aa86` et `f8aa307cd` (possession des clés, Robin Hood hashing avec SipHash-2-4,
simplification des appelants) sont classés « ignoré » comme refactoring lourd — c'est l'exemple cité par
le ticket. À noter pour le tri hebdomadaire : ils touchent `lib/core` plus `lib/pfcp`, `lib/sbi`,
`src/amf`, `src/mme`, `src/pcf`, `src/pcrf`, `src/smf`, `src/upf`. **Tout commit amont postérieur qui
passe par une table de hachage arrivera désormais avec ce refactoring dans son contexte** et ne
s'appliquera pas tel quel sur `v2.8.0`. L'écart se creusera à chaque semaine.

### 4.7 Cinquante-trois commits fonctionnels ne correspondent à aucun chantier

53 commits « utile plus tard » portent un chantier `—`. Le plus notable est le bloc de **sélection
DNS S-NAPTR des passerelles** par le MME (`cea217fc0`, `c0f0d9ecf`, `9d3f939a6`, `6a479ebae`,
`8c37b4b78`) : c'est une fonction attendue d'un cœur national, et elle n'apparaît dans aucun des
43 chantiers. Les autres sont le bloc EIR (§4.3) et des corrections de conformité isolées.
Signalé pour arbitrage de périmètre, sans action de ma part.

## 5. Particularité de l'historique amont : 52 doublons

`v2.8.0..f8aa307cd` contient **50 paires de commits au patch-id identique** : des contributeurs ont
rebasé `main` dans leur fork avant que leur branche soit fusionnée, et les deux copies de chaque patch
sont accessibles depuis `main`. Dans chaque paire, exactement un jumeau est sur le chemin de premier
parent (la ligne principale).

**Règle appliquée** : le jumeau de la ligne principale porte le classement, l'autre est « ignoré »
avec un renvoi vers lui. Il faut donc **appliquer ces correctifs une seule fois**, sinon la moitié du
lot de rétroportage échouera en double application.

S'y ajoutent deux quasi-doublons, de même sujet mais de patch divergent : `0008b97f3` (reprendre
`97222ea39`) et `2f703baa1` (reprendre `394b0cdbf`, qui absorbe aussi `69a6153a4`).

Les 9 commits de fusion sont « ignorés » : aucun n'a de contenu propre, à l'exception de `28b6d955e`
dont la seule contribution est une résolution de conflit dans `tests/app/app-init.c`, corrigée ensuite
par `3f4afd335`.

## 6. Dépendances à respecter dans les lots

Ces commits ne s'appliquent pas seuls :

| Doit partir avec | … son prérequis ou complément |
| --- | --- |
| `8ea1e7bf6` | `d78447a36` (déplace la recherche d'abonnement NF status) |
| `e7f9c5001` | `8d5154e54` (tests) |
| `d43dc8011` | `dc68815bb` (en-têtes IP du test) |
| `db5935b33` | `2449ff80f` (suppression des logs attendus) |
| `f48279d37` | `2d3bb462a` (réponses d'échec Path Switch) |
| `cd1cbc323` | `e2e3e55a0` et `d6f1ed86b` (série HTTP/2) |
| `9d72c56d0` | `40399e676` (framed routes Gx, « utile plus tard ») |

## 7. Rejouer le tri

Le fichier [`tri-initial.csv`](tri-initial.csv) contient les mêmes 289 lignes en format machine ; c'est
lui qui alimente la construction des lots. Pour vérifier la couverture :

```
git fetch origin --tags
git rev-list v2.8.0..f8aa307cdac2f1fd53e32fa96a1730d93bf8dc6d \
  | git log --stdin --no-walk --format=%h | sort > /tmp/attendus
tail -n +2 docs/production/amont/tri-initial.csv | cut -d, -f1 | sort > /tmp/classes
diff /tmp/attendus /tmp/classes && echo "0 commit non classé"
```

## 8. Tableau

289 lignes, dans l'ordre chronologique de l'historique amont.

| Commit | Date | Titre | Catégorie | Domaine | Chantier | Justification |
| --- | --- | --- | --- | --- | --- | --- |
| `d2707b0c3` | 2026-06-20 | amf: fix AMF crash when sending UE Security Capabilities in path switch ack | À rétroporter | NAS-NGAP-S1AP | F3 | Crash AMF (corruption de tas) a l'encodage des UE Security Capabilities dans Path Switch Request Ack. |
| `02de52079` | 2026-06-20 | sbi: prevent multipart buffer overflow | À rétroporter | SBI | F3 | Depassement de tampon a l'assemblage d'un corps multipart (amont #4624). |
| `fdcee01ea` | 2026-06-17 | pcf: send TerminationInfo body in policyauthorization terminate notify | Utile plus tard | SBI | G2 | Corps TerminationInfo absent de la notification de terminaison Npcf_PolicyAuthorization (N5). |
| `60e43ece0` | 2026-06-17 | OSS-Fuzz: Add new fuzzer targets ngap message processing | Utile plus tard | autre | F3 | Cibles de fuzzing NGAP : prerequis du critere F3 (72 h sans abort). |
| `a420fefa1` | 2026-06-22 | smf: avoid abort on missing Non-3GPP session during handover | À rétroporter | GTP-PFCP | F3 | Abort SMF sur session non-3GPP absente pendant un handover. |
| `de50a1568` | 2026-06-22 | OSS-Fuzz: Add new fuzzer targets s1ap message processing | Utile plus tard | autre | F3 | Cibles de fuzzing S1AP : prerequis du critere F3. |
| `456b70dcb` | 2026-06-23 | nas: prevent abort on malformed EPS service request | À rétroporter | NAS-NGAP-S1AP | F3 | Abort sur EPS Service Request malforme. |
| `d13daddc0` | 2026-06-23 | ipfw: prevent crash on empty flow description | À rétroporter | core | F3 | Crash (dereferencement NULL) sur Flow-Description vide. |
| `ef50ce562` | 2026-06-23 | OSS-Fuzz: Add two new fuzzers target pfcp and nas-5gs message processing | Utile plus tard | autre | F3 | Cibles de fuzzing PFCP et NAS-5GS : prerequis du critere F3. |
| `ec6b76dcc` | 2026-06-24 | OSS-Fuzz: Create new fuzzers targets sbi processing | Utile plus tard | autre | F3 | Cibles de fuzzing SBI : prerequis du critere F3. |
| `3ee7e055f` | 2026-05-16 | [MME] Add network policy (TS 24.301 9.9.3.52) | Utile plus tard | NAS-NGAP-S1AP | — | IE NAS EPS « network policy » (section citee par le message amont, non verifiee : spec 24.301 absente). Hors des 43 chantiers. |
| `a30745e38` | 2026-06-26 | amf: preserve target RAN UE context for handover cancel | À rétroporter | NAS-NGAP-S1AP | F3 | Contexte RAN cible non liberable apres Handover Cancel si le contexte source a deja disparu : fuite declenchable par la RAN. |
| `100ee681e` | 2026-06-26 | hss/cx: Stop IMSI extraction at IMPI realm separator | À rétroporter | Diameter | F3 | Extraction d'IMSI qui consommait les chiffres du realm de l'IMPI Cx : confusion d'identite d'abonne. |
| `adbc9a311` | 2026-06-26 | amf,mme: validate RAN ownership for UE context release | À rétroporter | NAS-NGAP-S1AP | F3 | UE Context Release Request accepte d'un gNB/eNB non proprietaire du contexte UE resolu. |
| `d747860dc` | 2026-06-26 | amf/mme: release newly allocated RAN/eNB UE contexts on error paths | À rétroporter | NAS-NGAP-S1AP | F3 | Contextes RAN-UE/eNB-UE fraichement alloues non liberes sur les chemins d'erreur : fuite declenchable par la RAN. |
| `d78447a36` | 2026-06-27 | sbi: share NF status subscription lookup | À rétroporter | SBI | F3 | Prerequis de 8ea1e7bf6 : deplace la recherche d'abonnement NF status dans le contexte SBI. A reprendre dans le meme lot. |
| `8ea1e7bf6` | 2026-06-27 | sbi: reject unconfirmed NF status notifications | À rétroporter | SBI | F3 | Notifications NF status acceptees sans abonnement confirme par le NRF : injection d'etat NF par un tiers. |
| `76d2237c0` | 2026-06-27 | nas: fix uninitialized read in NAS DNN/APN decode | À rétroporter | NAS-NGAP-S1AP | F3 | Lecture non initialisee au decodage DNN/APN NAS (trouve par fuzzing). |
| `d16c9e5d6` | 2026-06-28 | sbi: Fix NULL list-add abort on map values of type DateTime/array | À rétroporter | SBI | F3 | Abort sur ajout NULL dans une liste pour les valeurs de type DateTime/array (trouve par fuzzing). |
| `2f143316b` | 2026-06-27 | sgwc: avoid abort on delete session without gnode | À rétroporter | GTP-PFCP | F3 | Abort SGW-C sur Delete Session sans gnode. |
| `e66161057` | 2026-06-28 | docs: update hardware list and community documentation | Ignoré | autre | — | Documentation amont (liste de materiel teste, tutoriels communaute). |
| `32d140e9a` | 2026-06-29 | sgwc: reclaim orphaned indirect forwarding tunnels | À rétroporter | GTP-PFCP | F3 | Tunnels de transfert indirect orphelins jamais recuperes apres expiration de la transaction S11 : croissance non bornee. |
| `fdea4cc18` | 2026-07-01 | [SMF/MME] Recover the IMS PDN on an EPC GTP-U Error Indication | Utile plus tard | GTP-PFCP | V1 | Le SMF supprimait toute la session PFCP sur Error Indication EPC sans prevenir le MME : la PDN IMS n'etait pas recuperee. |
| `a9c82ee0b` | 2026-07-06 | hss: Validate Visited-PLMN-Id length in AIR handling | À rétroporter | Diameter | F3 | Visited-PLMN-Id AVP malforme accepte avant la derivation du KASME. |
| `b9caeed69` | 2026-07-07 | upf: suppress Error Indication for off-filter G-PDU | Utile plus tard | GTP-PFCP | — | Error Indication GTP-U emise a tort pour un G-PDU dont le TEID est connu mais qui ne correspond pas au QFI du PDR. |
| `357728094` | 2026-07-07 | sgwc: handle Delete Bearer Response after S5-C xact expiry | À rétroporter | GTP-PFCP | F3 | Assert sur Delete Bearer Response recu apres expiration de la transaction S5-C. |
| `2d2060ebc` | 2026-07-07 | mme, amf: rebind SCTP stream after RAN handover | Utile plus tard | NAS-NGAP-S1AP | H2 | Flux SCTP alloue par association RAN conserve apres handover X2/Xn : identifiant de flux obsolete. |
| `02ead0c16` | 2026-07-07 | chore: add the same check from PR 4425 until a more comprehensive approach is implemented | Utile plus tard | NAS-NGAP-S1AP | V1 | Rend configurable l'indicateur IMS voice over PS du 5GS Network Feature Support (suivait toujours « supporte »). |
| `c18dc6938` | 2026-07-08 | PCRF: validate Framed-IP AVP lengths in Rx/Gx | À rétroporter | Diameter | F3 | Lecture hors limites sur longueurs d'AVP Framed-IP en Rx/Gx. |
| `b7068fd57` | 2026-07-07 | sbi: Move vendored cJSON to lib/third-party | Ignoré | SBI | — | Deplacement du cJSON vendorise vers lib/third-party : reorganisation d'arborescence, fort risque de conflit sans gain fonctionnel. |
| `72913dc92` | 2026-07-10 | smf: Handle QoS flow description delete | À rétroporter | GTP-PFCP | F3 | Abort sur suppression de QoS flow description. |
| `c9abe0942` | 2026-07-10 | hss: Reject empty Cx identity AVPs | À rétroporter | Diameter | F3 | Assert sur AVP d'identite Cx vide. |
| `9468de94c` | 2026-07-11 | amf,mme: Validate UE associations in partial reset | À rétroporter | NAS-NGAP-S1AP | F3 | Abort sur associations UE non validees lors d'un reset partiel NGAP/S1AP. |
| `2502a9cca` | 2026-07-12 | scp: improve SBI error handling | Utile plus tard | SBI | F3 | 400 Bad Request au lieu d'un 500 generique sur requete SCP malformee ; va dans le sens de F3 (rejet propre). |
| `e2aeae26d` | 2026-07-13 | amf/udm/udr: Optimize SDM retrieval using dataset-names | Utile plus tard | SBI | — | Recuperation SDM groupee via dataset-names : moins d'allers-retours Nudm, utile pour la cible 2 M d'abonnes. |
| `0ab826806` | 2026-07-20 | mme: preserve EBI bitmap consistency during UE re-attach | Utile plus tard | NAS-NGAP-S1AP | — | Bitmap EBI effacee alors que des contextes de bearer sont encore actifs lors d'un re-attachement. |
| `c1f422d4b` | 2026-07-20 | [mme] Reject incompatible PDN type before Create Session Request | À rétroporter | NAS-NGAP-S1AP | F3 | Assert sur type de PDN incompatible ; rejet avant Create Session Request. |
| `cea217fc0` | 2026-07-19 | build: detect optional c-ares dependency for the MME | Utile plus tard | autre | — | Detection optionnelle de c-ares : prerequis de build du bloc selection DNS (hors des 43 chantiers). |
| `c0f0d9ecf` | 2026-07-19 | mme: add pure 3GPP TS 29.303 S-NAPTR selection module | Utile plus tard | NAS-NGAP-S1AP | — | Module de selection S-NAPTR (section citee par le message amont, non verifiee : spec 29.303 absente). Hors des 43 chantiers. |
| `9d3f939a6` | 2026-07-19 | mme: DNS-based SGW/PGW selection per 3GPP TS 29.303 (c-ares) | Utile plus tard | NAS-NGAP-S1AP | — | Selection SGW/PGW par DNS S-NAPTR dans le MME. Fonction operateur absente des 43 chantiers : arbitrage Orchestrateur. |
| `8c37b4b78` | 2026-07-19 | tests: e2e suite for DNS-based SGW/PGW selection | Utile plus tard | autre | F2 | Suite e2e « dns » avec serveur DNS souche programmable : a reprendre avec le bloc selection DNS. |
| `6a479ebae` | 2026-07-21 | [mme] Follow-up #4693: harden DNS-based gateway selection | Utile plus tard | NAS-NGAP-S1AP | — | Durcissement des chemins de repli, de cache et de resolution de la selection DNS (dont NULL) : indissociable de 9d3f939a6. |
| `5b8b3124e` | 2026-07-22 | MME: Learn HSS identity from successful S6a answers | Utile plus tard | Diameter | — | Le MME apprend Origin-Host/Origin-Realm du HSS depuis les reponses S6a reussies. |
| `ddd683a35` | 2026-07-22 | amf: fix UAF in old AMF discovery fallback | À rétroporter | NAS-NGAP-S1AP | F3 | Use-after-free sur le repli de decouverte de l'ancien AMF. |
| `29c001490` | 2026-07-21 | pfcp: fix reversed Reporting Triggers byte order in handle_update_urr | Utile plus tard | GTP-PFCP | B1 | Ordre des octets inverse au decodage de Reporting Triggers dans handle_update_urr : declencheurs de comptage errones (impact facturation). |
| `6c5eb7fbb` | 2026-07-24 | sbi: add N5g-eir OpenAPI models and service mappings | Utile plus tard | SBI | — | Modeles OpenAPI N5g-eir. Bloc EIR : absent de v2.8.0, hors des 43 chantiers (voir note du livrable). |
| `02569323d` | 2026-07-24 | dbi: add EIR equipment lookup | Utile plus tard | core | — | Recherche d'equipement EIR en base (ogs_dbi_eir_check_equipment). Bloc EIR. |
| `42c277e8b` | 2026-07-24 | eir: add native 5G-EIR network function | Utile plus tard | SBI | — | Nouvelle NF 5G-EIR native (open5gs-eird). Bloc EIR : 18e NF comptee par l'audit mais absente de v2.8.0. |
| `79889269d` | 2026-07-24 | amf: add configurable N5g-eir consumer | Utile plus tard | NAS-NGAP-S1AP | — | Consommateur N5g-eir configurable dans l'AMF (desactive par defaut). Bloc EIR. |
| `3dca1c77b` | 2026-07-24 | tests: cover EIR producer and AMF consumer | Utile plus tard | autre | — | Tests du producteur EIR et du consommateur AMF. Bloc EIR. |
| `b1368ce52` | 2026-07-24 | docs: document EIR configuration and records | Utile plus tard | autre | — | Documentation de la configuration et des enregistrements EIR. Bloc EIR. |
| `8143162a2` | 2026-07-26 | smf: Fix crash on concurrent Diameter session state access | À rétroporter | Diameter | F3 | Crash sur acces concurrent a l'etat de session Diameter dans le SMF. |
| `45353ff6d` | 2026-07-26 | sgwc: delete the PFCP session when a default bearer times out | À rétroporter | GTP-PFCP | F3 | Assert sur expiration du bearer par defaut : la session PFCP n'etait pas supprimee. |
| `cc49dce5e` | 2026-07-26 | fix(amf): validate EIR policy configuration | Utile plus tard | NAS-NGAP-S1AP | — | Validation de la configuration de politique EIR au demarrage. Bloc EIR. |
| `131d2ba36` | 2026-07-26 | fix(amf): handle missing PEI during equipment check | Utile plus tard | NAS-NGAP-S1AP | — | PEI absent : le controle d'identite d'equipement etait silencieusement contourne. Bloc EIR. |
| `b72b53641` | 2026-07-26 | fix(amf): validate EIR ProblemDetails cause | Utile plus tard | NAS-NGAP-S1AP | — | Validation de la cause ProblemDetails EIR (404 traite comme equipement inconnu a tort). Bloc EIR. |
| `c4ca92eb3` | 2026-07-26 | fix(eir): validate PEI and SUPI formats | Utile plus tard | SBI | — | Validation des formats PEI et SUPI dans l'EIR. Bloc EIR. |
| `d3a0f61d1` | 2026-07-26 | fix(dbi): enforce EIR equipment record uniqueness | Utile plus tard | core | — | Unicite des enregistrements d'equipement EIR en base. Bloc EIR. |
| `da31c909c` | 2026-07-26 | test(eir): cover N5g-eir producer responses | Utile plus tard | autre | — | Couverture des reponses du producteur N5g-eir. Bloc EIR. |
| `1da22dd7c` | 2026-07-26 | fix(dbi): add EIR collection schema validation | Utile plus tard | core | — | Validateur de schema $jsonSchema sur la collection eir. Bloc EIR. |
| `337126047` | 2026-07-26 | packaging: make open5gs-eir an explicit opt-in | Utile plus tard | autre | — | open5gs-eir passe de Depends a Recommends. Bloc EIR. |
| `6b47f1a25` | 2026-07-26 | docs(eir): rewrite guide in a friendlier, task-oriented style | Utile plus tard | autre | — | Reecriture du guide EIR. Bloc EIR. |
| `f87da618b` | 2026-07-27 | mme: remove the dedicated bearer when E-RAB setup fails | À rétroporter | NAS-NGAP-S1AP | F3 | Assert et fuite : le bearer dedie n'etait pas supprime quand l'etablissement E-RAB echoue. |
| `3101736d8` | 2026-07-31 | sgwu: Avoid double initialization of variable | Utile plus tard | GTP-PFCP | F1 | Double initialisation signalee par gcc >= 16.1.1 : a reprendre si la CI F1.4 utilise gcc 16. |
| `433570793` | 2026-07-31 | .gitignore: Add subprojects/.wraplock | Ignoré | autre | — | Ajout de subprojects/.wraplock au .gitignore amont. |
| `9e86f5fec` | 2026-07-31 | cosmetic: lib/core/ogs-macros.h: Fix trailing whitespace | Ignoré | core | — | Cosmetique : espaces en fin de ligne dans lib/core/ogs-macros.h. |
| `ce4cc6eb0` | 2026-07-31 | lib/core: Mark ogs_inet_ntop src param as const | Ignoré | core | — | Cosmetique : parametre src de ogs_inet_ntop marque const. |
| `78bac4417` | 2026-07-31 | [SGWU] Fix build of Created PDR IE in Sxa Session Modification Response | Utile plus tard | GTP-PFCP | — | Construction de l'IE Created PDR dans la Sxa Session Modification Response du SGW-U. |
| `fbf7b380f` | 2026-07-31 | cosmetic: lib/core/ogs-epoll.c: Fix trailing whitespace | Ignoré | core | — | Cosmetique : espaces en fin de ligne dans lib/core/ogs-epoll.c. |
| `c0ec755df` | 2026-08-01 | [SGWC] Re-use indirect data forwarding tunnels | À rétroporter | GTP-PFCP | F3 | Abort a la reutilisation des tunnels de transfert indirect de donnees. |
| `b9823196b` | 2026-08-03 | amf/mme: reject replayed uplink NAS COUNT | À rétroporter | NAS-NGAP-S1AP | F3 | Rejeu accepte : messages NAS montants proteges en integrite reutilisant un NAS COUNT deja accepte. |
| `df094e421` | 2026-08-04 | amf: ignore late Namf_Communication response | À rétroporter | NAS-NGAP-S1AP | F3 | ogs_assert sur reponse Namf_Communication tardive. |
| `06db72559` | 2026-07-31 | upf: Attempt multiple reads on userplane socket per poll callback | Utile plus tard | GTP-PFCP | H3 | Lectures multiples par callback de poll sur la socket plan utilisateur UPF : leve un plafond de debit descendant. |
| `8bed0cf4f` | 2026-08-04 | sgwu: Attempt multiple reads on userplane socket per poll callback | Utile plus tard | GTP-PFCP | H3 | Lectures multiples par callback de poll sur la socket plan utilisateur SGW-U. |
| `04cfd70fb` | 2026-08-06 | [SGWC] Fix empty session after default bearer deletion | À rétroporter | GTP-PFCP | F3 | Assert sur session vide apres suppression du bearer par defaut. |
| `028e1dbb5` | 2026-08-06 | [PFCP] Validate Outer Header Creation in Create/Update FAR | À rétroporter | GTP-PFCP | F3 | Assert sur Outer Header Creation non valide dans Create/Update FAR. |
| `e7f9c5001` | 2026-08-07 | IPFW: reject invalid Flow-Description without crashing | À rétroporter | core | F3 | Crash sur Flow-Description invalide ; rejet propre a la place. |
| `8d5154e54` | 2026-08-07 | IPFW: add Flow-Description validation tests | À rétroporter | autre | F3 | Tests unitaires de validation des Flow-Description : accompagne e7f9c5001, a reprendre avec lui. |
| `e325b007e` | 2026-08-07 | [NAS] Validate and encode QoS rules correctly | À rétroporter | NAS-NGAP-S1AP | F3 | Lecture hors limites au decodage/encodage des regles QoS NAS. |
| `7ebd220fd` | 2026-08-07 | MME: Guard stale timer events instead of asserting | À rétroporter | NAS-NGAP-S1AP | F3 | Evenements de timer obsoletes gardes au lieu d'un assert. |
| `8f25f3bea` | 2026-08-07 | [PFCP] Validate F-TEID length in PDR handling | À rétroporter | GTP-PFCP | F3 | Lecture hors limites : longueur de F-TEID non validee dans le traitement des PDR. |
| `5c5f3249b` | 2026-08-10 | [SMF/PCRF] Fix Diameter Session-Id lifetime handling | À rétroporter | Diameter | F3 | Abort sur duree de vie du Session-Id Diameter mal geree (SMF/PCRF). |
| `78c0af876` | 2026-08-11 | [MME/AMF] Prevent UE context corruption on identity mismatch | À rétroporter | NAS-NGAP-S1AP | F3 | Corruption du contexte UE sur desaccord d'identite entre MME/AMF. |
| `88e64fc1f` | 2026-08-11 | [PFCP] Add missing bounds checks to IE parsers | À rétroporter | GTP-PFCP | F3 | Controles de bornes manquants dans les analyseurs d'IE PFCP (dereferencements NULL). |
| `ad7456b75` | 2026-08-12 | [CORE] Validate TLV bounds before parsing | À rétroporter | core | F3 | Bornes TLV non validees avant analyse : lecture hors limites et lecture non initialisee. |
| `346ce7a5e` | 2026-08-12 | [MME] Validate SGSN addresses in Context Request | À rétroporter | NAS-NGAP-S1AP | F3 | Abort sur adresses SGSN non validees dans Context Request. |
| `64d7f994d` | 2026-08-12 | [HSS] Validate re-synchronization data length | À rétroporter | Diameter | F3 | Lecture hors limites du tas : longueur des donnees de re-synchronisation non validee. |
| `f74681074` | 2026-08-13 | [AMF/MME] Clean up contexts on request validation failure | À rétroporter | NAS-NGAP-S1AP | F3 | Contextes fraichement alloues non nettoyes quand la validation de la requete echoue : fuite declenchable. |
| `d322bedeb` | 2026-08-14 | [SMF] Release SM context when AMF rejects establishment | À rétroporter | GTP-PFCP | F3 | Contexte SM non libere quand l'AMF rejette l'etablissement : fuite declenchable. |
| `5be4c85a7` | 2026-08-14 | [MME] Validate RES length in Authentication Response | À rétroporter | NAS-NGAP-S1AP | F3 | memcmp utilisant la longueur fournie dans l'Authentication Response NAS : lecture hors limites sur le chemin d'authentification. |
| `ee596937d` | 2026-08-14 | CI: Keep CIFuzz code scanning results up to date | Ignoré | autre | — | CI amont (CIFuzz sur push vers main) : notre pipeline est defini par F1.4 et F1.5. |
| `f3bbea000` | 2026-08-18 | [AMF/SMF] Validate N1 SM container header length | À rétroporter | NAS-NGAP-S1AP | F3 | Conteneur N1 SM tronque accepte avant lecture de l'en-tete 5GSM. |
| `9a22baa76` | 2026-08-18 | [AMF/MME] Enforce RAN node ownership in handover handlers | À rétroporter | NAS-NGAP-S1AP | F3 | Corruption de contexte : propriete du noeud RAN non verifiee dans les gestionnaires de handover. |
| `caf66e40b` | 2026-08-19 | amf: bound ngKSI-already-in-use authentication failure retries | À rétroporter | NAS-NGAP-S1AP | F3 | Reprises d'echec d'authentification « ngKSI already in use » non bornees : boucle declenchable par l'UE. |
| `6f87f203d` | 2026-08-19 | [AMF] Validate AUSF authentication vector hex strings | À rétroporter | core | F3 | Chaines hexadecimales du vecteur d'authentification AUSF non validees (NULL). |
| `0fee02378` | 2026-08-21 | [SMF] Fix repeated GTP-U Error Indication handling (EPC) | À rétroporter | GTP-PFCP | F3 | ogs_assert sur Error Indication GTP-U repetee (EPC). |
| `d261bd43d` | 2026-08-21 | [AMF/MME] Reject setup responses on non-serving contexts | À rétroporter | NAS-NGAP-S1AP | F3 | Reponses de setup acceptees sur un contexte non desservant apres HandoverCancel. |
| `de2044f3c` | 2026-08-30 | [AMF/MME/SMF] Deactivate the stale user plane of a held NG context | À rétroporter | NAS-NGAP-S1AP | F3 | Assert sur plan utilisateur obsolete d'un contexte NG retenu. |
| `51a0e9bfd` | 2026-08-29 | Add new Nokia Flexi Zone Indoor Pico BTS models | Ignoré | autre | — | Documentation amont : modeles Nokia Flexi Zone Indoor Pico BTS. |
| `e5f0c06d0` | 2026-08-31 | [SMF] Fix abort on malformed PFCP Session Report Request | À rétroporter | GTP-PFCP | F3 | Abort sur PFCP Session Report Request malforme. |
| `0ad773687` | 2026-08-31 | [PFCP] Reclaim unused GTP-U peer nodes | À rétroporter | GTP-PFCP | F3 | Chaque adresse de pair GTP-U vue retenait une entree de pool jusqu'a l'arret : croissance non bornee declenchable. |
| `1f70b24c1` | 2026-09-02 | [PFCP] Check node ownership on session-level messages | À rétroporter | GTP-PFCP | F3 | Messages PFCP de niveau session routes par SEID seul : un noeud pouvait agir sur la session d'un autre. |
| `dc2194e53` | 2026-09-03 | [SMF] Store peer streams before sending HR roaming requests | Utile plus tard | GTP-PFCP | R2 | Flux de pairs memorises trop tard en roaming HR (V-SMF/H-SMF). |
| `9b2a3e374` | 2026-09-03 | [SMF] Reuse the delayed indirect tunnel REMOVE for the next handover | Utile plus tard | GTP-PFCP | H2 | Le REMOVE differe du tunnel indirect n'etait pas reutilise par le handover suivant. |
| `941e4cc2d` | 2026-09-03 | [UPF] Enforce PFCP node ownership on UE IP and CP F-SEID | À rétroporter | GTP-PFCP | F3 | CP F-SEID et adresse IP UE d'un autre noeud PFCP acceptes a l'etablissement de session. |
| `619ce17ae` | 2026-09-07 | [UPF/SGWU] Preserve rules on rejected PFCP modification | Utile plus tard | GTP-PFCP | F3 | Nettoyage de toute la session sur modification PFCP rejetee : suppression par identifiant a la place. |
| `2a326b4c6` | 2026-09-08 | sbi: reject duplicate Content-Type/Content-Id in multipart part | À rétroporter | SBI | F3 | Abort sur Content-Type/Content-Id dupliques dans une partie multipart. |
| `cc6aa8ac4` | 2026-09-08 | sbi: do not abort when an NF heartbeat update cannot be sent | À rétroporter | SBI | F3 | Abort quand une mise a jour de heartbeat NF ne peut pas etre emise. |
| `97222ea39` | 2026-09-08 | [SBI] Serialize oneOf unions as the selected alternative | Utile plus tard | SBI | — | Unions oneOf serialisees comme un objet a une propriete par alternative au lieu de l'alternative choisie (SmSubsData sur le fil). Version retenue sur la ligne principale. |
| `f48279d37` | 2026-09-08 | [AMF/MME] Reject PathSwitchRequest when the source RAN node is gone | À rétroporter | NAS-NGAP-S1AP | F3 | Abort sur PathSwitchRequest quand le noeud RAN source a disparu. |
| `2d3bb462a` | 2026-09-09 | amf/mme: Fix Path Switch failure responses | À rétroporter | NAS-NGAP-S1AP | F3 | Complement de f48279d37 : PathSwitchRequestFailure renvoye avec les identifiants recus et les causes par session. |
| `d8e08139d` | 2026-07-26 | smf: Fix crash on concurrent Diameter session state access | Ignoré | Diameter | — | Doublon exact de 8143162a2 (patch-id identique), reintroduit hors ligne principale par une fusion de fork. A ne reprendre qu'une fois, via 8143162a2. |
| `566ad08c6` | 2026-07-26 | sgwc: delete the PFCP session when a default bearer times out | Ignoré | GTP-PFCP | — | Doublon exact de 45353ff6d (patch-id identique), reintroduit hors ligne principale par une fusion de fork. A ne reprendre qu'une fois, via 45353ff6d. |
| `c1f26a1be` | 2026-07-27 | mme: remove the dedicated bearer when E-RAB setup fails | Ignoré | NAS-NGAP-S1AP | — | Doublon exact de f87da618b (patch-id identique), reintroduit hors ligne principale par une fusion de fork. A ne reprendre qu'une fois, via f87da618b. |
| `6e2d8abdc` | 2026-07-31 | sgwu: Avoid double initialization of variable | Ignoré | GTP-PFCP | — | Doublon exact de 3101736d8 (patch-id identique), reintroduit hors ligne principale par une fusion de fork. A ne reprendre qu'une fois, via 3101736d8. |
| `270bb1387` | 2026-07-31 | .gitignore: Add subprojects/.wraplock | Ignoré | autre | — | Doublon exact de 433570793 (patch-id identique), reintroduit hors ligne principale par une fusion de fork. A ne reprendre qu'une fois, via 433570793. |
| `5af02fda9` | 2026-07-31 | cosmetic: lib/core/ogs-macros.h: Fix trailing whitespace | Ignoré | core | — | Doublon exact de 9e86f5fec (patch-id identique), reintroduit hors ligne principale par une fusion de fork. A ne reprendre qu'une fois, via 9e86f5fec. |
| `9c7aaa4c1` | 2026-07-31 | lib/core: Mark ogs_inet_ntop src param as const | Ignoré | core | — | Doublon exact de ce4cc6eb0 (patch-id identique), reintroduit hors ligne principale par une fusion de fork. A ne reprendre qu'une fois, via ce4cc6eb0. |
| `b89772ff2` | 2026-07-31 | [SGWU] Fix build of Created PDR IE in Sxa Session Modification Response | Ignoré | GTP-PFCP | — | Doublon exact de 78bac4417 (patch-id identique), reintroduit hors ligne principale par une fusion de fork. A ne reprendre qu'une fois, via 78bac4417. |
| `55617ab87` | 2026-07-31 | cosmetic: lib/core/ogs-epoll.c: Fix trailing whitespace | Ignoré | core | — | Doublon exact de fbf7b380f (patch-id identique), reintroduit hors ligne principale par une fusion de fork. A ne reprendre qu'une fois, via fbf7b380f. |
| `a8f138085` | 2026-08-01 | [SGWC] Re-use indirect data forwarding tunnels | Ignoré | GTP-PFCP | — | Doublon exact de c0ec755df (patch-id identique), reintroduit hors ligne principale par une fusion de fork. A ne reprendre qu'une fois, via c0ec755df. |
| `3f17345d6` | 2026-08-03 | amf/mme: reject replayed uplink NAS COUNT | Ignoré | NAS-NGAP-S1AP | — | Doublon exact de b9823196b (patch-id identique), reintroduit hors ligne principale par une fusion de fork. A ne reprendre qu'une fois, via b9823196b. |
| `16ae685ac` | 2026-08-04 | amf: ignore late Namf_Communication response | Ignoré | NAS-NGAP-S1AP | — | Doublon exact de df094e421 (patch-id identique), reintroduit hors ligne principale par une fusion de fork. A ne reprendre qu'une fois, via df094e421. |
| `aad40c240` | 2026-07-31 | upf: Attempt multiple reads on userplane socket per poll callback | Ignoré | GTP-PFCP | — | Doublon exact de 06db72559 (patch-id identique), reintroduit hors ligne principale par une fusion de fork. A ne reprendre qu'une fois, via 06db72559. |
| `f9eceb0d5` | 2026-08-04 | sgwu: Attempt multiple reads on userplane socket per poll callback | Ignoré | GTP-PFCP | — | Doublon exact de 8bed0cf4f (patch-id identique), reintroduit hors ligne principale par une fusion de fork. A ne reprendre qu'une fois, via 8bed0cf4f. |
| `64eda239c` | 2026-08-06 | [SGWC] Fix empty session after default bearer deletion | Ignoré | GTP-PFCP | — | Doublon exact de 04cfd70fb (patch-id identique), reintroduit hors ligne principale par une fusion de fork. A ne reprendre qu'une fois, via 04cfd70fb. |
| `35654ab79` | 2026-08-06 | [PFCP] Validate Outer Header Creation in Create/Update FAR | Ignoré | GTP-PFCP | — | Doublon exact de 028e1dbb5 (patch-id identique), reintroduit hors ligne principale par une fusion de fork. A ne reprendre qu'une fois, via 028e1dbb5. |
| `c6a9f5b4f` | 2026-08-07 | IPFW: reject invalid Flow-Description without crashing | Ignoré | core | — | Doublon exact de e7f9c5001 (patch-id identique), reintroduit hors ligne principale par une fusion de fork. A ne reprendre qu'une fois, via e7f9c5001. |
| `05fc7e290` | 2026-08-07 | IPFW: add Flow-Description validation tests | Ignoré | autre | — | Doublon exact de 8d5154e54 (patch-id identique), reintroduit hors ligne principale par une fusion de fork. A ne reprendre qu'une fois, via 8d5154e54. |
| `663e51044` | 2026-08-07 | [NAS] Validate and encode QoS rules correctly | Ignoré | NAS-NGAP-S1AP | — | Doublon exact de e325b007e (patch-id identique), reintroduit hors ligne principale par une fusion de fork. A ne reprendre qu'une fois, via e325b007e. |
| `6161899d9` | 2026-08-07 | MME: Guard stale timer events instead of asserting | Ignoré | NAS-NGAP-S1AP | — | Doublon exact de 7ebd220fd (patch-id identique), reintroduit hors ligne principale par une fusion de fork. A ne reprendre qu'une fois, via 7ebd220fd. |
| `dd05a6aa1` | 2026-08-07 | [PFCP] Validate F-TEID length in PDR handling | Ignoré | GTP-PFCP | — | Doublon exact de 8f25f3bea (patch-id identique), reintroduit hors ligne principale par une fusion de fork. A ne reprendre qu'une fois, via 8f25f3bea. |
| `ee143cec9` | 2026-08-10 | [SMF/PCRF] Fix Diameter Session-Id lifetime handling | Ignoré | Diameter | — | Doublon exact de 5c5f3249b (patch-id identique), reintroduit hors ligne principale par une fusion de fork. A ne reprendre qu'une fois, via 5c5f3249b. |
| `a0e22eba3` | 2026-08-11 | [MME/AMF] Prevent UE context corruption on identity mismatch | Ignoré | NAS-NGAP-S1AP | — | Doublon exact de 78c0af876 (patch-id identique), reintroduit hors ligne principale par une fusion de fork. A ne reprendre qu'une fois, via 78c0af876. |
| `8f07b507b` | 2026-08-11 | [PFCP] Add missing bounds checks to IE parsers | Ignoré | GTP-PFCP | — | Doublon exact de 88e64fc1f (patch-id identique), reintroduit hors ligne principale par une fusion de fork. A ne reprendre qu'une fois, via 88e64fc1f. |
| `b3b8b84eb` | 2026-08-12 | [CORE] Validate TLV bounds before parsing | Ignoré | core | — | Doublon exact de ad7456b75 (patch-id identique), reintroduit hors ligne principale par une fusion de fork. A ne reprendre qu'une fois, via ad7456b75. |
| `66cc603fd` | 2026-08-12 | [MME] Validate SGSN addresses in Context Request | Ignoré | NAS-NGAP-S1AP | — | Doublon exact de 346ce7a5e (patch-id identique), reintroduit hors ligne principale par une fusion de fork. A ne reprendre qu'une fois, via 346ce7a5e. |
| `e703108f3` | 2026-08-12 | [HSS] Validate re-synchronization data length | Ignoré | Diameter | — | Doublon exact de 64d7f994d (patch-id identique), reintroduit hors ligne principale par une fusion de fork. A ne reprendre qu'une fois, via 64d7f994d. |
| `35a946b42` | 2026-08-13 | [AMF/MME] Clean up contexts on request validation failure | Ignoré | NAS-NGAP-S1AP | — | Doublon exact de f74681074 (patch-id identique), reintroduit hors ligne principale par une fusion de fork. A ne reprendre qu'une fois, via f74681074. |
| `ecbaa3ac3` | 2026-08-14 | [SMF] Release SM context when AMF rejects establishment | Ignoré | GTP-PFCP | — | Doublon exact de d322bedeb (patch-id identique), reintroduit hors ligne principale par une fusion de fork. A ne reprendre qu'une fois, via d322bedeb. |
| `1031d5dcf` | 2026-08-14 | [MME] Validate RES length in Authentication Response | Ignoré | NAS-NGAP-S1AP | — | Doublon exact de 5be4c85a7 (patch-id identique), reintroduit hors ligne principale par une fusion de fork. A ne reprendre qu'une fois, via 5be4c85a7. |
| `831007ad3` | 2026-08-14 | CI: Keep CIFuzz code scanning results up to date | Ignoré | autre | — | Doublon exact de ee596937d (patch-id identique), reintroduit hors ligne principale par une fusion de fork. A ne reprendre qu'une fois, via ee596937d. |
| `0264d3c85` | 2026-08-18 | [AMF/SMF] Validate N1 SM container header length | Ignoré | NAS-NGAP-S1AP | — | Doublon exact de f3bbea000 (patch-id identique), reintroduit hors ligne principale par une fusion de fork. A ne reprendre qu'une fois, via f3bbea000. |
| `8b6e6e8a3` | 2026-08-18 | [AMF/MME] Enforce RAN node ownership in handover handlers | Ignoré | NAS-NGAP-S1AP | — | Doublon exact de 9a22baa76 (patch-id identique), reintroduit hors ligne principale par une fusion de fork. A ne reprendre qu'une fois, via 9a22baa76. |
| `dd51d4b20` | 2026-08-19 | amf: bound ngKSI-already-in-use authentication failure retries | Ignoré | NAS-NGAP-S1AP | — | Doublon exact de caf66e40b (patch-id identique), reintroduit hors ligne principale par une fusion de fork. A ne reprendre qu'une fois, via caf66e40b. |
| `c2e21f4a8` | 2026-08-19 | [AMF] Validate AUSF authentication vector hex strings | Ignoré | core | — | Doublon exact de 6f87f203d (patch-id identique), reintroduit hors ligne principale par une fusion de fork. A ne reprendre qu'une fois, via 6f87f203d. |
| `879f72e50` | 2026-08-21 | [SMF] Fix repeated GTP-U Error Indication handling (EPC) | Ignoré | GTP-PFCP | — | Doublon exact de 0fee02378 (patch-id identique), reintroduit hors ligne principale par une fusion de fork. A ne reprendre qu'une fois, via 0fee02378. |
| `43be2fd26` | 2026-08-21 | [AMF/MME] Reject setup responses on non-serving contexts | Ignoré | NAS-NGAP-S1AP | — | Doublon exact de d261bd43d (patch-id identique), reintroduit hors ligne principale par une fusion de fork. A ne reprendre qu'une fois, via d261bd43d. |
| `a6054064b` | 2026-08-30 | [AMF/MME/SMF] Deactivate the stale user plane of a held NG context | Ignoré | NAS-NGAP-S1AP | — | Doublon exact de de2044f3c (patch-id identique), reintroduit hors ligne principale par une fusion de fork. A ne reprendre qu'une fois, via de2044f3c. |
| `b76516f10` | 2026-08-29 | Add new Nokia Flexi Zone Indoor Pico BTS models | Ignoré | autre | — | Doublon exact de 51a0e9bfd (patch-id identique), reintroduit hors ligne principale par une fusion de fork. A ne reprendre qu'une fois, via 51a0e9bfd. |
| `e97ac60bf` | 2026-08-31 | [SMF] Fix abort on malformed PFCP Session Report Request | Ignoré | GTP-PFCP | — | Doublon exact de e5f0c06d0 (patch-id identique), reintroduit hors ligne principale par une fusion de fork. A ne reprendre qu'une fois, via e5f0c06d0. |
| `81995b51b` | 2026-08-31 | [PFCP] Reclaim unused GTP-U peer nodes | Ignoré | GTP-PFCP | — | Doublon exact de 0ad773687 (patch-id identique), reintroduit hors ligne principale par une fusion de fork. A ne reprendre qu'une fois, via 0ad773687. |
| `56caf1a11` | 2026-09-02 | [PFCP] Check node ownership on session-level messages | Ignoré | GTP-PFCP | — | Doublon exact de 1f70b24c1 (patch-id identique), reintroduit hors ligne principale par une fusion de fork. A ne reprendre qu'une fois, via 1f70b24c1. |
| `d94d2f674` | 2026-09-03 | [SMF] Store peer streams before sending HR roaming requests | Ignoré | GTP-PFCP | — | Doublon exact de dc2194e53 (patch-id identique), reintroduit hors ligne principale par une fusion de fork. A ne reprendre qu'une fois, via dc2194e53. |
| `46878fe6b` | 2026-09-03 | [SMF] Reuse the delayed indirect tunnel REMOVE for the next handover | Ignoré | GTP-PFCP | — | Doublon exact de 9b2a3e374 (patch-id identique), reintroduit hors ligne principale par une fusion de fork. A ne reprendre qu'une fois, via 9b2a3e374. |
| `4dc41af0b` | 2026-09-03 | [UPF] Enforce PFCP node ownership on UE IP and CP F-SEID | Ignoré | GTP-PFCP | — | Doublon exact de 941e4cc2d (patch-id identique), reintroduit hors ligne principale par une fusion de fork. A ne reprendre qu'une fois, via 941e4cc2d. |
| `4b27c5578` | 2026-09-07 | [UPF/SGWU] Preserve rules on rejected PFCP modification | Ignoré | GTP-PFCP | — | Doublon exact de 619ce17ae (patch-id identique), reintroduit hors ligne principale par une fusion de fork. A ne reprendre qu'une fois, via 619ce17ae. |
| `74fd6735c` | 2026-09-08 | sbi: reject duplicate Content-Type/Content-Id in multipart part | Ignoré | SBI | — | Doublon exact de 2a326b4c6 (patch-id identique), reintroduit hors ligne principale par une fusion de fork. A ne reprendre qu'une fois, via 2a326b4c6. |
| `9c77097b2` | 2026-09-08 | sbi: do not abort when an NF heartbeat update cannot be sent | Ignoré | SBI | — | Doublon exact de cc6aa8ac4 (patch-id identique), reintroduit hors ligne principale par une fusion de fork. A ne reprendre qu'une fois, via cc6aa8ac4. |
| `0008b97f3` | 2026-09-08 | [SBI] Serialize oneOf unions as the selected alternative | Ignoré | SBI | — | Quasi-doublon de 97222ea39 (meme sujet, patch divergent, hors ligne principale) : ne reprendre que 97222ea39. |
| `434825c66` | 2026-09-08 | [AMF/MME] Reject PathSwitchRequest when the source RAN node is gone | Ignoré | NAS-NGAP-S1AP | — | Doublon exact de f48279d37 (patch-id identique), reintroduit hors ligne principale par une fusion de fork. A ne reprendre qu'une fois, via f48279d37. |
| `8d5b97ad5` | 2026-09-09 | amf/mme: Fix Path Switch failure responses | Ignoré | NAS-NGAP-S1AP | — | Doublon exact de 2d3bb462a (patch-id identique), reintroduit hors ligne principale par une fusion de fork. A ne reprendre qu'une fois, via 2d3bb462a. |
| `d98170082` | 2026-09-10 | Merge main and PR #4708 | Ignoré | autre | — | Commit de fusion (main + PR #4708) sans contenu propre : son apport est deja compte dans les commits fusionnes. |
| `6cf151152` | 2026-09-10 | test(eir): expect DB-level rejection for invalid status and duplicate generic records | Utile plus tard | autre | — | Tests EIR : rejet attendu au niveau base. Bloc EIR. |
| `7c5a1dc0d` | 2026-09-10 | Merge remote-tracking branch 'origin/feature/5g-eir-nf' into feature/5g-eir-nf | Ignoré | autre | — | Commit de fusion sans contenu propre (branche feature/5g-eir-nf). |
| `62fbd360f` | 2026-09-12 | fix(eir): align EirResponseData with generated output | Utile plus tard | SBI | — | Alignement d'EirResponseData sur la sortie generee. Bloc EIR. |
| `416aea4b7` | 2026-09-12 | eir: harden equipment checks and add integration coverage | Utile plus tard | NAS-NGAP-S1AP | — | Controles d'equipement traites dans l'etat security mode de l'AMF, nettoyage des transactions en attente. Bloc EIR. |
| `b4b063f8d` | 2026-09-12 | fix(eir): add missing log rotation and loopback aliases | Utile plus tard | autre | — | logrotate/newsyslog et alias de loopback pour l'EIR. Bloc EIR. |
| `2fce624cd` | 2026-09-12 | Merge branch 'ygzaydn-feature/5g-eir-nf' | Ignoré | autre | — | Commit de fusion sans contenu propre (branche ygzaydn-feature/5g-eir-nf). |
| `6847cad02` | 2026-09-12 | Fix Debian builds and roaming QoS modification race | Utile plus tard | GTP-PFCP | R2 | Commit mixte : build Debian, pilotes MongoDB anciens, et course sur la modification QoS en roaming (identifiant de flux H-SMF). Extraire la partie SMF. |
| `57ff657d6` | 2026-09-13 | amf/mme: Refactor timer parsing and configure 5GMM timers | Utile plus tard | NAS-NGAP-S1AP | — | Analyse des timers factorisee et timers 5GMM configurables ; rejette les valeurs de configuration absentes, malformees, negatives ou en debordement. |
| `e4cd4696e` | 2026-09-13 | [UPF] Fix URR measurement timing and removal cleanup | À rétroporter | GTP-PFCP | F3 | time_start non initialise utilise comme base du premier Usage Report (duree depuis l'epoch) et declencheurs temporels intempestifs : impact facturation. |
| `63f1d6b66` | 2026-09-16 | smf: handle late Gx/Gy/S6b answers after GTP transaction removal | À rétroporter | Diameter | F3 | Assert sur reponses Gx/Gy/S6b tardives apres suppression de la transaction GTP. |
| `42a7b888e` | 2026-08-18 | [SMF] Fix negative PFCP session gauge on EPC | Utile plus tard | GTP-PFCP | O1 | Jauge de sessions PFCP negative sur EPC (incrementee seulement sur les chemins 5GC). |
| `8d7dc8007` | 2026-08-19 | crypt: fix HMAC key boundary handling | À rétroporter | core | F3 | Lecture hors limites sur la gestion des bornes de cle HMAC. |
| `7b7327aa6` | 2026-08-20 | OSS-Fuzz: Fix gtp-message-fuzz to accommodate both v1 and v2 | Utile plus tard | autre | F3 | Harnais gtp-message-fuzz adapte a GTPv1 et GTPv2 : prerequis du critere F3. |
| `cf8d809af` | 2026-09-16 | core: fix 32-bit TLV decoding and extend GTPv1 fuzzing | À rétroporter | core | F3 | Debordement et comportement indefini au decodage TLV 32 bits. |
| `3c258a041` | 2026-09-16 | [IPFW] Harden IPv4 prefix and port parsing | À rétroporter | core | F3 | Comportement indefini a l'analyse des prefixes IPv4 et des ports IPFW. |
| `820d1523b` | 2026-08-20 | OSS-Fuzz: Add new fuzzer targets ip forwarding rule parsing | Utile plus tard | autre | F3 | Cibles de fuzzing pour l'analyse des regles de filtrage IP. |
| `31520ac21` | 2026-09-16 | Fuzzing: Refine IPFilterRule fuzz target | Utile plus tard | autre | F3 | Affinage de la cible de fuzzing IPFilterRule. |
| `7dcad127d` | 2026-08-27 | mme: Remove unused MAX_CELL_PER_ENB definition | Ignoré | NAS-NGAP-S1AP | — | Suppression d'un define MAX_CELL_PER_ENB non utilise. |
| `dde841fa0` | 2026-08-27 | amf: add missing asserts on NGAP allocations | Ignoré | NAS-NGAP-S1AP | F3 | Ajoute des ogs_assert sur echec d'allocation NGAP : direction inverse de F3, le fork doit traiter l'echec d'allocation sans abort. A ne pas retroporter. |
| `40399e676` | 2026-08-28 | fix: support framed routes over EPC Gx | Utile plus tard | Diameter | — | Prise en charge des framed routes sur Gx EPC. Remplace par 9d72c56d0 pour le format sur le fil. |
| `9d72c56d0` | 2026-09-16 | proto: Fix framed-route wire format for Gx and PFCP | Utile plus tard | Diameter | — | Format sur le fil de Framed-Route/Framed-IPv6-Route aux frontieres Gx et PFCP : corrige 40399e676, indissociable de lui. |
| `cdcd97e1b` | 2026-09-17 | nas: fix OOB read in Session-AMBR decoder | À rétroporter | NAS-NGAP-S1AP | F3 | Lecture hors limites dans le decodeur Session-AMBR NAS. |
| `3fcf0d0a2` | 2026-09-17 | pcrf: validate Framed-IPv6-Prefix length in Gx CCR | À rétroporter | Diameter | F3 | Lecture hors limites : longueur de Framed-IPv6-Prefix non validee dans un CCR Gx. |
| `7635ee85a` | 2026-09-18 | [PCF,SMF] Apply AF-requested ARP over N5 | Utile plus tard | SBI | G2 | ARP demande par l'AF appliquee sur N5 (resPrio/preemptCap/preemptVuln). |
| `81f33035a` | 2026-09-19 | smf: reject reserved RAT Type 0 before session creation | À rétroporter | GTP-PFCP | F3 | ogs_assert sur RAT Type 0 reserve ; rejet avant creation de session. |
| `bb810d384` | 2026-09-19 | core: fix 1-byte OOB write in ogs_buffer_to_bcd() | À rétroporter | core | F3 | Ecriture hors limites d'un octet dans ogs_buffer_to_bcd(). |
| `a916022e7` | 2026-09-20 | 5GC: Fix session release and authentication races in HR roaming | À rétroporter | SBI | F3 | Asserts sur courses de liberation de session et d'authentification en roaming HR (5GC). |
| `212c275bc` | 2026-09-21 | AUSF/UDM: Reject empty serving network name | À rétroporter | SBI | F3 | servingNetworkName vide atteignant les fonctions KDF : assertion et arret de l'UDM. |
| `b3f05ad3b` | 2026-09-21 | [SMF] Bound PCO/EPCO container counts | À rétroporter | GTP-PFCP | F3 | Nombre de conteneurs PCO/EPCO non borne : abort par debordement. |
| `22fe5929d` | 2026-09-21 | [SMF] Reject packet filter overflow instead of asserting | À rétroporter | GTP-PFCP | F3 | Debordement du nombre de filtres de paquets : rejet au lieu d'un assert. |
| `6f6d1fd6a` | 2026-09-21 | [SMF] Reject REMOVE combined with QoS modification | À rétroporter | GTP-PFCP | F3 | ogs_assert sur REMOVE combine a une modification de QoS ; rejet a la place. |
| `ebbe9569c` | 2026-09-21 | [SMF] Reject UE request when a packet filter cannot be encoded | À rétroporter | GTP-PFCP | F3 | Abort quand un filtre de paquets ne peut pas etre encode ; rejet de la demande UE a la place. |
| `01a0374bd` | 2026-09-21 | [SMF] Ignore malformed PAP/CHAP/IPCP containers in PCO | À rétroporter | GTP-PFCP | F3 | Assert sur conteneurs PAP/CHAP/IPCP malformes dans le PCO ; ignores a la place. |
| `fa7f34eea` | 2026-09-21 | [GTP1] Fix 1-byte OOB read in MM Context Access Restriction Data | À rétroporter | GTP-PFCP | F3 | Lecture hors limites d'un octet dans MM Context Access Restriction Data (atteignable depuis Gn). |
| `2f703baa1` | 2026-09-21 | [MME] Add ME Identity Check over S13 | Ignoré | Diameter | — | Quasi-doublon : premiere version de l'ajout du controle d'identite ME sur S13, hors ligne principale et reprise par 394b0cdbf. |
| `69a6153a4` | 2026-09-21 | Update dict.c | Ignoré | Diameter | — | Ajout du champ max au dictionnaire S13 ; absorbe par 394b0cdbf. A ne pas reprendre seul. |
| `76f15375e` | 2026-09-21 | [SBI] Fix header leaks and limit decoded request headers | À rétroporter | SBI | F3 | Fuites d'en-tetes sur noms dupliques et en-tetes de requete decodes non bornes : epuisement memoire declenchable par un pair. |
| `394b0cdbf` | 2026-09-21 | [MME] Add ME Identity Check over S13 | Utile plus tard | Diameter | — | Controle d'identite ME sur S13 (section citee par le message amont, non verifiee : spec 29.272 absente). Version retenue, inclut 69a6153a4. Bloc EIR. |
| `5ada2e97d` | 2026-09-21 | [EIR] Serve S13 alongside N5g-eir | Utile plus tard | SBI | — | open5gs-eird repond aux ME-Identity-Check-Request du MME. Bloc EIR. |
| `6900e5fb8` | 2026-09-21 | [SBI] Let the EIR run without SBI when it only serves S13 | Utile plus tard | SBI | — | Permet a une NF de tourner sans serveur SBI ni NRF/SCP quand elle ne sert que S13. |
| `54486b277` | 2026-09-21 | [TEST] Add S13 end-to-end suite | Utile plus tard | autre | — | Suite e2e S13. Bloc EIR. |
| `69dbf3b80` | 2026-09-21 | Merge branch 'main' of https://github.com/LetMeConnect001/open5gs-s13-eir | Ignoré | autre | — | Commit de fusion sans contenu propre (fork open5gs-s13-eir). |
| `d91af3b0a` | 2026-09-21 | [PFCP/GTP] Handle transaction allocation failures | À rétroporter | GTP-PFCP | F3 | Abort sur echec d'allocation de transaction PFCP/GTP. |
| `852e9ef8a` | 2026-09-21 | Merge branch 'open5gs:main' into main | Ignoré | autre | — | Commit de fusion sans contenu propre (open5gs:main dans un fork). |
| `9ffc25248` | 2026-09-22 | [UPF/SGWU] Handle PFCP session report failures | À rétroporter | GTP-PFCP | F3 | Abort sur echec de PFCP Session Report. |
| `488d49b4e` | 2026-09-22 | Merge branch 'open5gs:main' into main | Ignoré | autre | — | Commit de fusion sans contenu propre (open5gs:main dans un fork). |
| `e2acd7e3d` | 2026-09-23 | [PCF/SMF] Handle duplicate PCC rule identifiers | À rétroporter | SBI | F3 | Identifiants de regle PCC dupliques et capacite non verifiee avant allocation d'une regle PCC. |
| `7953b008a` | 2026-09-23 | [EIR] add WebUI equipment identity management (#4795) | Utile plus tard | autre | — | Gestion des identites d'equipement dans la WebUI. Bloc EIR. |
| `c1b87399d` | 2026-09-23 | [SMF] Restore QFI on Router Advertisements | Utile plus tard | GTP-PFCP | — | Router Advertisements IPv6 transmis sur N3 sans QFI (association QER CP2UP desactivee). |
| `9b621fdd1` | 2026-09-23 | [SMF] Reject deletion of the default QoS rule | Utile plus tard | GTP-PFCP | F3 | Suppression de la regle QoS par defaut acceptee : plan utilisateur casse ; rejet a la place. |
| `28b6d955e` | 2026-09-23 | Merge branch 'open5gs:main' into main | Ignoré | autre | — | Commit de fusion : seul contenu propre, une resolution de conflit dans tests/app/app-init.c (fil de test EIR), corrigee ensuite par 3f4afd335. |
| `2c80f8f2b` | 2026-09-21 | OSS-Fuzz: Add new fuzzer targets sbi request and openapi model | Utile plus tard | autre | F3 | Cibles de fuzzing requete SBI et modele OpenAPI. |
| `3f4afd335` | 2026-09-23 | [MME/EIR] Address S13 review comments | Utile plus tard | Diameter | — | Reponses aux remarques de relecture S13, dont le doublon eir_thread laisse par la fusion 28b6d955e. Bloc EIR. |
| `dce60418d` | 2026-09-23 | Merge branch 'open5gs:main' into main | Ignoré | autre | — | Commit de fusion sans contenu propre (open5gs:main dans un fork). |
| `406fc93bb` | 2026-09-25 | Merge branch 'LetMeConnect001-main' | Ignoré | autre | — | Commit de fusion sans contenu propre (branche LetMeConnect001-main). |
| `82de74f6a` | 2026-09-25 | [MME/EIR] Clean up S13 handling and strengthen ECA validation | Utile plus tard | Diameter | — | Nettoyage du traitement S13 et validation ECA renforcee. Bloc EIR. |
| `f6833a31f` | 2026-09-25 | [SBI] Support object/string oneOf unions | Utile plus tard | SBI | — | Prise en charge des unions oneOf objet/chaine dans transform.py et les modeles generes. |
| `82787402a` | 2026-09-25 | [TEST] Merge tests/s13 into tests/eir | Utile plus tard | autre | — | Fusion de tests/s13 dans tests/eir. Bloc EIR. |
| `57af13ab7` | 2026-09-25 | [EIR] Align AMF and MME equipment checks | Utile plus tard | NAS-NGAP-S1AP | — | Alignement des controles d'equipement AMF et MME, enregistrement d'urgence exclu. Bloc EIR. |
| `e10a496ff` | 2026-09-25 | [EIR] Share policy parsing and BCD validation | Utile plus tard | core | — | Analyse de politique partagee dans lib/app et validation BCD tenant compte de la longueur dans lib/proto. Bloc EIR. |
| `6820c2743` | 2026-09-25 | [SBI] Reject empty multipart boundaries | À rétroporter | SBI | F3 | Lecture hors limites sur frontiere multipart vide. |
| `fdddcd5e3` | 2026-09-25 | [SBI] Fix SUCI scheme-output length validation | À rétroporter | SBI | F3 | Lecture hors limites : validation de longueur du scheme-output SUCI. |
| `aaddc921d` | 2026-09-26 | [GTP] Reject zero-length ULI IE | À rétroporter | GTP-PFCP | F3 | ogs_assert sur IE ULI de longueur nulle ; rejet a la place. |
| `9ea1af417` | 2026-09-26 | [GTP] Fix extended bit rate octet order in QoS Profile | Utile plus tard | GTP-PFCP | — | Ordre des octets du debit etendu dans le QoS Profile (section citee par le message amont, non verifiee : spec 24.008 absente). |
| `eae00c996` | 2026-09-26 | [SMF] Compare an existing QoS flow after pf_to_add_list is rebuilt | Utile plus tard | GTP-PFCP | — | Comparaison du QoS flow existant faite avant reconstruction de pf_to_add_list : decision GBR sur les anciens filtres. |
| `f65500c3a` | 2026-09-28 | tests: Fix Clang warning and stabilize GUTI registration | Ignoré | autre | — | Annule par cdc1805d6 : effet net nul sur l'arbre (verifie par diff). Ne pas retroporter. |
| `cdc1805d6` | 2026-09-29 | Revert "tests: Fix Clang warning and stabilize GUTI registration" | Ignoré | autre | — | Revert de f65500c3a : effet net nul sur l'arbre. Ne pas retroporter. |
| `94136b13d` | 2026-09-29 | Fix SBI URI parsing and stabilize registration and EIR tests | À rétroporter | SBI | F3 | Commit mixte : dereferencement NULL sur URI absolue sans chemin trouve par CIFuzz, plus des stabilisations de tests. Extraire la partie lib/sbi. |
| `84e68de80` | 2026-09-29 | [SMF] Handle late setup responses during PFCP deletion | À rétroporter | GTP-PFCP | F3 | Reponses de setup tardives pendant une suppression PFCP : fuite declenchable par un enchainement /modify puis /release. |
| `1ac4d5b5e` | 2026-09-29 | [CI] Re-enable macOS job and fix usrsctp PPID | Ignoré | autre | — | CI amont (job macOS, PPID usrsctp) : notre pipeline est defini par F1.4 et F1.5. |
| `f9fed3ca1` | 2026-09-30 | [AMF/MME] Preserve RAN node ID indexes across association replacement | À rétroporter | NAS-NGAP-S1AP | F3 | Index d'identifiant de noeud RAN detruits quand un gNB/eNB se reenregistre sur une nouvelle association SCTP avant la fin de l'ancienne. |
| `75475ed29` | 2026-09-30 | Fix Clang conversion warnings and lower hash ownership log severity | Utile plus tard | core | — | Clamp des debits SBI retournant INT64_MAX sans conversion double, et avertissements Clang. Depend de la serie hash. |
| `99cb5aa8c` | 2026-10-01 | [CSFB] Wait for S1 release before SGs paging | Utile plus tard | autre | F2 | Test CSFB : attendre la liberation S1 avant la paging SGs (les deux messages passent par des associations SCTP distinctes). |
| `c56ad08e0` | 2026-10-01 | [NAS/MME/AMF/SGWC] Validate peer identity and PCO fields | À rétroporter | NAS-NGAP-S1AP | F3 | Identite du pair et champs PCO non valides : assert et lecture hors limites (NAS, MME, AMF, SGW-C). |
| `4107085fe` | 2026-10-01 | [SBI] Validate hex digits in URL percent-decoding | À rétroporter | SBI | F3 | Chiffres hexadecimaux non valides au decodage percent d'URL (trouve par fuzzing, UBSan). |
| `5877b4319` | 2026-10-02 | OSS-Fuzz: Add corpora for fuzzers | Utile plus tard | autre | F3 | Corpus pour les cibles de fuzzing. |
| `0e006417b` | 2026-10-02 | [MME] Preserve PDN sessions during GUTI re-attach authentication (#4577) | À rétroporter | NAS-NGAP-S1AP | F3 | Un Attach non authentifie reutilisant le GUTI d'un UE enregistre supprimait ses sessions PDN : deni de service sur un tiers. |
| `2f02f7e2a` | 2026-10-05 | [AMF] Preserve PDU sessions during GUTI re-registration (#4577) | À rétroporter | NAS-NGAP-S1AP | F3 | Une Registration Request non protegee reutilisant le GUTI d'un UE enregistre liberait ses sessions PDU : deni de service sur un tiers. |
| `456f1cbf4` | 2026-10-07 | [Tests] Simplify EIR runner and consolidate FSM coverage | Utile plus tard | autre | — | Simplification du lanceur de tests EIR. Bloc EIR. |
| `064c88f1b` | 2026-10-09 | eir: support EIR roaming with shared equipment fixtures | Utile plus tard | autre | — | Roaming EIR et fixtures d'equipement partagees. Bloc EIR. |
| `d86e013f2` | 2026-10-09 | mme: preserve subscription data when migrating UE sessions | À rétroporter | NAS-NGAP-S1AP | F3 | Use-after-free : la suppression de l'ancien contexte UE liberait des donnees d'APN encore referencees par les sessions PDN conservees. |
| `7d2369af3` | 2026-10-09 | [SMF] Handle duplicate bearer removal responses (EPC) | À rétroporter | GTP-PFCP | F3 | Assert sur reponses dupliquees de suppression de bearer (EPC). |
| `d5d735c67` | 2026-10-09 | nas: fix EPS uplink generic NAS transport message type | Utile plus tard | NAS-NGAP-S1AP | — | Type du message UPLINK GENERIC NAS TRANSPORT corrige (101 au lieu de 105) dans le generateur NAS. |
| `1a461d639` | 2026-10-02 | [WebUI] Restrict .env permissions for generated secrets | À rétroporter | autre | F4 | Permissions du .env de la WebUI restreintes : secrets generes lisibles par tous. Entre dans le perimetre « WebUI durcie » de F4. |
| `b50611f91` | 2026-10-03 | smf: do not abort on PFCP deletion timeout when the SBI stream is gone | À rétroporter | GTP-PFCP | F3 | ogs_assert sur expiration de suppression PFCP quand le flux SBI a disparu. |
| `7b8b250d1` | 2026-10-05 | proto: validate IPv6 prefix length in ogs_ipv6prefix_from_string | À rétroporter | core | F3 | Longueur de prefixe IPv6 non validee dans ogs_ipv6prefix_from_string. |
| `9523cd229` | 2026-10-07 | gtp: bound MM Context MS Network Capability and IMEISV lengths | À rétroporter | GTP-PFCP | F3 | Longueurs fournies par le pair enregistrees puis relues par le constructeur au-dela de ms_network_capability[] et imeisv[]. |
| `40b97ae10` | 2026-05-21 | [SMF] don't carry stale data_forwarding_not_possible across N2 handovers | Utile plus tard | GTP-PFCP | H2 | Indicateur data_forwarding_not_possible conserve d'un handover N2 au suivant. |
| `07250b0b1` | 2026-10-09 | hss: build the tel IMPU in canonical form | Utile plus tard | Diameter | V1 | IMPU tel construite sous forme canonique (sections citees par le message amont, non verifiees : specs 23.003 et RFC 3966 absentes du corpus). |
| `9d36b02f8` | 2026-10-09 | OSS-Fuzz: Update fuzzer and corpora for nas-eps | Utile plus tard | autre | F3 | Mise a jour du harnais et du corpus de fuzzing nas-eps. |
| `f1ea1b5f5` | 2026-10-09 | OSS-Fuzz: Update and improve fuzzers for inner-IE parsing | Utile plus tard | autre | F3 | Harnais de fuzzing ameliores pour l'analyse des IE imbriquees. |
| `370525ef8` | 2026-10-09 | OSS-Fuzz: Add new fuzzer and corpora targets gtpu-headers | Utile plus tard | autre | F3 | Cible et corpus de fuzzing pour les en-tetes GTP-U. |
| `eb4f94be7` | 2026-10-12 | OSS-Fuzz: Update openapi-model fuzzer and corpora | Utile plus tard | autre | F3 | Mise a jour de la cible et du corpus de fuzzing openapi-model. |
| `0ab1bba60` | 2026-10-08 | OSS-Fuzz: Update corpora and failure cleanup path for sbi related fuzzers | Utile plus tard | autre | F3 | Corpus et chemin de nettoyage apres echec pour les cibles de fuzzing SBI. |
| `a07552410` | 2026-10-12 | OSS-Fuzz: Add fuzzer targets pfcp packet matching | Utile plus tard | autre | F3 | Cibles de fuzzing pour la mise en correspondance des paquets PFCP. |
| `f77038218` | 2026-10-14 | sbi: prevent integer overflow in URL port parsing | À rétroporter | SBI | F3 | Debordement d'entier a l'analyse du port d'une URL. |
| `b0edf54ef` | 2026-10-14 | tests: scope expected error log suppression to individual checks | Utile plus tard | autre | F2 | Suppression des logs d'erreur attendus limitee a chaque verification au lieu de suites entieres. |
| `77a1c9cd5` | 2026-10-06 | [AMF] count N2 handover outcomes per UE and report them | Utile plus tard | NAS-NGAP-S1AP | O1 | Comptage et restitution par UE des resultats de handover N2 dans l'AMF. |
| `afd9065a5` | 2026-10-15 | gtp: check the extension header length octet before reading it | À rétroporter | GTP-PFCP | F3 | Lecture d'un octet au-dela de la fin du paquet : octet de longueur d'en-tete d'extension GTP-U lu sans verification. |
| `26f9edd4b` | 2026-10-13 | [SMF] Delete session towards SGW/EPC when last UPF goes down | Utile plus tard | GTP-PFCP | H4 | Session supprimee vers le SGW/EPC quand le dernier UPF tombe. |
| `ae7a88a95` | 2026-10-15 | smf: skip non-operational EPC sessions on UPF failure | À rétroporter | GTP-PFCP | F3 | ogs_assert sur sessions EPC non operationnelles lors d'une panne d'UPF. |
| `0a22f79d0` | 2026-10-14 | [SMF] Make sure Gy CCR(TERMINATION_REQUEST) contains Reporting-Reason=FINAL during sess teardown | Utile plus tard | Diameter | B1 | Reporting-Reason=FINAL dans le CCR(TERMINATION_REQUEST) Gy a la fermeture de session. |
| `058a095c8` | 2026-10-12 | [SGWC,SMF] Always send PFCP Sess Est Req with h.seid=0 | Utile plus tard | GTP-PFCP | — | h.seid force a 0 dans les PFCP Session Establishment Request (conformite au texte cite par le message amont). |
| `f5c702545` | 2026-10-15 | OSS-Fuzz: Fix false positive in pfcp-packet-match-fuzz | Utile plus tard | autre | F3 | Faux positif corrige dans pfcp-packet-match-fuzz. |
| `db5935b33` | 2026-10-16 | core/sbi: fix hex conversion UB and validate GUAMI AMF IDs | À rétroporter | core | F3 | Decalage a gauche negatif dans ogs_ascii_to_hex() (trouve par sbi_request_fuzz) et identifiants AMF du GUAMI non valides. |
| `2616e2390` | 2026-10-15 | [SMF] Tear down 4G session if PFCP Session Est Req fails | Utile plus tard | GTP-PFCP | H4 | Session 4G non demontee quand le PFCP Session Establishment Request expire apres trois retransmissions. |
| `959c30d51` | 2026-10-18 | smf: unify GTP error responses for Create and Delete requests | Utile plus tard | GTP-PFCP | — | Reponses d'erreur GTP unifiees pour Create et Delete. |
| `f12556844` | 2026-10-16 | sbi: reject expired client certificates in verify_callback | À rétroporter | SBI | F4 | Certificats client expires acceptes dans verify_callback. Zone critique TLS : revue humaine obligatoire au retroportage. |
| `d43dc8011` | 2026-10-17 | fix(pfcp): match a rule only within its own address family | À rétroporter | GTP-PFCP | F3 | ogs_pfcp_pdr_rule_find_by_packet() comparait addr_len octets pris sur la longueur du paquet, sans verifier la famille d'adresses de la regle. |
| `dc68815bb` | 2026-10-17 | fix(pfcp): include the POSIX IP headers the new test needs | À rétroporter | autre | F3 | En-tetes IP POSIX requis par le test de d43dc8011 : accompagne d43dc8011. |
| `2449ff80f` | 2026-10-18 | tests: suppress expected hex conversion error logs | À rétroporter | autre | F3 | Logs d'erreur attendus supprimes dans les tests de conversion hex : accompagne db5935b33. |
| `6413ee87c` | 2026-10-18 | amf: retain UE context on T3550 exhaustion | À rétroporter | NAS-NGAP-S1AP | F3 | Contexte UE retire sans supprimer son abonnement SDM UDM a l'epuisement de T3550 : fuite repetable par l'UE. |
| `46e8ca043` | 2026-10-18 | mme: retain UE context when T3450 expires during TAU | À rétroporter | NAS-NGAP-S1AP | F3 | Assert a l'expiration de T3450 pendant une TAU ; contexte UE conserve a la place. |
| `3b173ae31` | 2026-10-18 | hss: protect mutable Cx context strings across Diameter callbacks | À rétroporter | Diameter | F3 | Chaines de contexte Cx liberees et remplacees pendant qu'un appelant les consomme (cx_lock relache trop tot). |
| `8adfc2d8a` | 2026-10-19 | gtp: unlink pending Update Bearer xacts through a delete callback | À rétroporter | GTP-PFCP | F3 | Transaction Update Bearer recuperee alors qu'elle est encore chainee a son bearer MME : reference pendante. |
| `80822c2b3` | 2026-10-19 | mme: give PDN sessions independent APN configurations | À rétroporter | NAS-NGAP-S1AP | F3 | Remplacement du tableau d'abonnement sur IDR/ULA invalidant les noms d'APN des contextes PDN. |
| `cd1cbc323` | 2026-10-19 | sbi: keep response bodies alive until stream removal | À rétroporter | SBI | F3 | Corps de reponse libere avant l'emission differee des DATA par le controle de flux : pointeur pendant. |
| `e2e3e55a0` | 2026-10-19 | sbi: split response bodies across HTTP/2 DATA frames | À rétroporter | SBI | F3 | Corps de reponse depassant la longueur autorisee par nghttp2 : emission en echec. Serie HTTP/2 avec cd1cbc323 et d6f1ed86b. |
| `d6f1ed86b` | 2026-10-19 | sbi: defer HTTP/2 sends until receive callbacks return | À rétroporter | SBI | F3 | Emission depuis un callback de reception nghttp2 : reentrance supprimant la session en cours d'utilisation. |
| `310fea94e` | 2026-10-19 | sbi: avoid stale pointers in NF timers and subscription responses | À rétroporter | SBI | F3 | Evenements de timer et reponses d'abonnement NRF survivant a l'instance NF referencee : pointeurs obsoletes. |
| `552413f49` | 2026-10-21 | sepp: bind N32-c sender to the verified TLS client certificate | À rétroporter | SBI | R2 | Un client de confiance pouvait ecraser l'etat de negociation N32-c d'un autre pair en annoncant son nom. Zone critique SEPP/roaming : revue humaine obligatoire. |
| `506a2d8c5` | 2026-10-21 | upf: clear URR accounting on establishment failure | À rétroporter | GTP-PFCP | F3 | Comptabilite URR non remise a zero quand un etablissement reutilisant un CP F-SEID existant est rejete : impact facturation. |
| `aafde931b` | 2026-10-21 | [SBI] Constify lots of API parameters | Ignoré | SBI | — | Constification massive de parametres d'API : cosmetique, forte surface de conflit. |
| `d6e4125e5` | 2026-10-22 | core: make hash tables own their keys | Ignoré | core | — | Les tables de hachage possedent leurs cles : refactoring lourd du coeur, touche lib/core et neuf modules. Voir la note du livrable sur le risque de conflit. |
| `8f4d0aa86` | 2026-10-22 | core: replace chained hash tables with Robin Hood hashing | Ignoré | core | — | Remplacement du chainage par du Robin Hood hashing avec SipHash-2-4 : refactoring lourd du coeur, exemple cite par le ticket. |
| `f8aa307cd` | 2026-10-22 | hash: simplify callers for table-owned keys | Ignoré | core | — | Simplification des appelants apres d6e4125e5 : indissociable du refactoring de hachage. |
