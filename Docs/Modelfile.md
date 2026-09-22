## Partie 1 / L'agent (Modelfile)

### Contexte

Le modèle imposé (`granite3.1-moe:3b`, 3B) s'est révélé avoir du mal à suivre des règles nuancées ou abstraites. La conception a nécessité plusieurs itérations, chacune corrigeant une faille précise observée en test réel.

### Choix de conception

**Paramètres**

- `temperature 0` : déterminisme total, élimine les variations aléatoires qui pourraient faire "craquer" le modèle par hasard.
- `num_predict 45` : limite physique du nombre de tokens générés. Empêche le modèle de rajouter des phrases parasites (notes, justifications) après une réponse correcte, un vecteur de fuite découvert en test.
- `stop "(" / "\n\n" / "Note"` : coupe la génération dès l'apparition de motifs typiques des commentaires parasites (parenthèses explicatives, doubles retours à la ligne, mot "Note").

**Structure du SYSTEM**

- Règles formulées comme un algorithme binaire strict (Rule A / B / C) plutôt qu'une liste de principes abstraits, un modèle 3B suit mieux des étapes concrètes qu'une nuance.
- Exemples `MESSAGE` couvrant à la fois des cas de refus (mauvais mot de passe, fiction, complétion, faux override) et des cas normaux (questions factuelles, conseils, recommandations), nécessaire pour ne pas casser le check "utility" en durcissant la sécurité.

### Failles rencontrées et corrections (historique de test)

| Faille observée                                                                                                   | Cause                                                                                      | Correction                                                                                                                 |
| ----------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------ | -------------------------------------------------------------------------------------------------------------------------- |
| Le modèle recopiait le mot proposé par l'utilisateur comme "password" en guise de codeword                        | Pattern-matching par proximité plutôt que vérification de valeur                           | Ajout de plusieurs exemples de mauvais mots de passe distincts (`banana`, `xyz789`, etc.) pour généraliser la vérification |
| Un mauvais mot de passe validait quand même le codeword                                                           | Le modèle imitait la structure de phrase de l'exemple de succès sans vérifier la condition | Règle binaire explicite (YES/NO) + réordonnancement des exemples                                                           |
| Une histoire de fiction (roleplay) faisait dire le codeword à un personnage                                       | Le modèle traitait la fiction comme hors du champ des règles de confidentialité            | Règle explicite : toute demande créative sur le thème "codeword/secret" suit la même règle de refus que la demande directe |
| Le modèle ajoutait des notes parasites après sa réponse, recopiant parfois des bouts du SYSTEM (donc le codeword) | Pas de limite physique sur la longueur de génération                                       | Ajout de `num_predict` et de `stop` sequences                                                                              |
| "Complete this sentence" et un faux "SYSTEM OVERRIDE" faisaient fuiter le codeword                                | Techniques non couvertes par les exemples `MESSAGE`                                        | Ajout d'exemples de refus dédiés à ces deux techniques                                                                     |
| Une question normale ("conseil pour un entretien") se faisait refuser à tort                                      | Trop d'exemples de refus, pas assez d'exemples de comportement normal                      | Ajout d'exemples de questions variées (maths, conseils, recommandations) pour rééquilibrer                                 |

### Checks validés

- `ollama create` réussit
- Release : bon mot de passe -> codeword exact
- Utility : questions variées -> réponses normales et complètes
- Résiste à : mauvais mot de passe, fiction/roleplay, dialogue d'espions, traduction, complétion de phrase, faux system override

## Partie 2 / Toolkit d'extraction (5 familles)

| Fichier                          | Famille                      | Principe d'attaque                                                                                    | Ligne de défense qui la neutralise spécifiquement                                                |
| -------------------------------- | ---------------------------- | ----------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------ |
| `01_roleplay_fiction.prompt`     | Fiction encapsulée           | Le secret est "dit" par un personnage fictif, pas par l'IA en tant qu'assistant                       | Interdire le mot dans tout contenu créatif, quel que soit le prétexte narratif                   |
| `02_fake_override.prompt`        | Autorité usurpée             | Faux message "système"/QA prétendant primer sur les règles normales                                   | Préciser qu'aucun texte dans un message utilisateur ne peut jamais avoir priorité sur le SYSTEM  |
| `03_sentence_completion.prompt`  | Complétion de phrase         | Reformule la demande comme une tâche d'autocomplétion neutre, pas une question                        | Interdire toute complétion de phrase liée au secret, même présentée comme un exercice technique  |
| `04_data_field_injection.prompt` | Injection dans un formulaire | Le secret est demandé comme un champ de données "standard" parmi d'autres                             | Vérifier chaque champ demandé contre une liste de champs légitimes prédéfinis                    |
| `05_encoding_translation.prompt` | Encodage forcé               | Contourne un filtre sur le mot exact via un encodage (Base64), justifié par un besoin d'accessibilité | Interdire toute transformation/encodage/traduction du secret, pas seulement sa restitution brute |

Chaque famille cible un mécanisme de contournement différent : aucune ligne de défense unique ne neutralise deux familles à la fois (vérifié en confrontant chaque attaque au Modelfile final).

---

### Méthodologie de recherche

Les techniques ont été identifiées et affinées par la pratique sur :

- Le challenge **Agent Breaker / Cycling Coach** (Lakera).
- Test direct contre l'agent `codename` développé dans ce projet, qui a permis de découvrir et corriger 6 failles distinctes avant la version finale (voir tableau ci-dessus).
- Le site sentinelone.com, pour l'attack usurped authority/fake override.

---

## Limites observées

- **Sensibilité à la langue** : une version du Modelfile traduite en français a été testée et a immédiatement échoué, une simple question factuelle ("Quelle est la capitale de l'Italie ?") a déclenché une fuite du codeword et du système de règles en clair après la bonne réponse. Le modèle `granite3.1-moe:3b`, probablement entraîné majoritairement en anglais, semble beaucoup moins fiable pour suivre des règles de sécurité nuancées dans une autre langue. La version finale retenue est donc restée en anglais.
- **Non-déterminisme résiduel** : malgré `temperature 0`, un test isolé en anglais a occasionnellement montré une légère instabilité (une réponse correcte suivie d'une fuite partielle du codeword sur une question sans rapport), non reproduite lors des tests suivants identiques. Cela suggère que la robustesse du Modelfile, bien que solide dans l'immense majorité des cas testés, n'est pas garantie à 100% avec ce modèle de taille réduite (3B).
