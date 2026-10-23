# Notice de fork — open5gs-Production

Ce dépôt est un **fork modifié d'Open5GS**. Cette notice satisfait l'article 5 de la
GNU Affero General Public License v3 (voir [LICENSE](LICENSE)) :

> a) The work must carry prominent notices stating that you modified it, and giving a
> relevant date.
>
> b) The work must carry prominent notices stating that it is released under this
> License and any conditions added under section 7.

## Œuvre d'origine

| | |
| --- | --- |
| Projet amont | Open5GS |
| Dépôt amont | <https://github.com/open5gs/open5gs> |
| Site amont | <https://open5gs.org> |
| Auteur principal amont | Sukchan Lee <acetcom@gmail.com>, et les contributeurs d'Open5GS |

Les mentions de droit d'auteur amont sont conservées intactes dans tous les fichiers
sources, conformément à l'article 4 de l'AGPL v3.

## Base du fork

| | |
| --- | --- |
| Tag de base | `v2.8.0` (tag annoté) |
| Commit de base | `157f611a530e292e40ec50f9d23f0ef5d4fcd6a6` |
| Date du commit de base | 2026-06-20 |
| Branche de production | `production/v2.8.x` |
| Date de création du fork | 2026-10-22 |

La branche `production/v2.8.x` part exactement du tag `v2.8.0`. Elle ne contient aucun
commit de `main` postérieur à ce tag : les correctifs amont jugés nécessaires y sont
rétroportés commit par commit (voir [CONTRIBUTING.md](CONTRIBUTING.md)).

## Nature des modifications

open5gs-Production est un fork de durcissement destiné à un déploiement d'opérateur.
Les modifications apportées à l'œuvre d'origine portent sur :

- la robustesse face aux entrées externes (suppression des `ogs_assert` sur les données
  reçues du réseau, au profit d'un rejet conforme à la spécification et d'un compteur) ;
- la sécurité des interfaces de service (TLS, OAuth2, SEPP et itinérance) ;
- la haute disponibilité, la montée en charge et l'exploitabilité ;
- la conformité 3GPP et GSMA des procédures et des interfaces ;
- l'outillage de test, de fuzzing et d'intégration continue.

Chaque modification est portée par un commit ou une demande de fusion distincte et
reste traçable dans l'historique du dépôt.

## Licence

La licence de l'œuvre d'origine est **conservée sans changement** :
GNU Affero General Public License, version 3 ou ultérieure
(`AGPL-3.0-or-later`), telle que publiée dans le fichier [LICENSE](LICENSE) à la racine
du dépôt. Ce fichier est identique à celui du tag `v2.8.0` amont.

Aucune condition supplémentaire au titre de l'article 7 de l'AGPL v3 n'est ajoutée.
L'intégralité de l'œuvre, modifications comprises, est distribuée sous cette licence,
conformément à l'article 5 c).

## Absence d'affiliation

Ce fork n'est publié ni par le projet Open5GS amont, ni par NewPlane. Il n'est couvert
par aucune de leurs offres de support ou de licence commerciale. Les demandes
concernant ce fork ne doivent pas être adressées au projet amont.

Les correctifs de portée générale produits par ce fork sont proposés en amont lorsque
cela est pertinent, afin de réduire l'écart avec Open5GS.
