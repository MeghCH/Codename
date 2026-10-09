# CODENAME

Projet Epitech / Prompt Engineering & Sécurité des agents conversationnels.

## Structure du repository

├── Modelfile # Agent final

├── Modelfile_test # Copie de travail pour tests locaux

├── attacks/ # Les 5 prompts d'extraction

│ ├── 01_roleplay_fiction.prompt

│ ├── 02_usurped_authority.prompt

│ ├── 03_sentence_completion.prompt

│ ├── 04_data_field_injection.prompt

│ └── 05_encoding_translation.prompt

└── Docs/ # Documentation détaillée

├── Modelfile.md # Choix de conception de l'agent

├── Attack_1.md # Roleplay / fiction encapsulée

├── Attack_2.md # Autorité usurpée

├── Attack_3.md # Complétion de phrase

├── Attack_4.md # Injection dans un formulaire

└── Attack_5.md # Encodage / traduction forcée

## Partie 1 / L'agent

Voir [`Docs/Modelfile.md`](Docs/Modelfile.md) pour :

- les choix de conception (paramètres, structure des règles),
- l'historique complet des failles détectées et corrigées durant les tests,
- les checks validés (release, utility, résistance aux attaques de base).

## Partie 2 / Toolkit d'extraction

Chaque attaque cible une famille distincte de technique de prompt injection. Détails complets dans les fichiers `Docs/Attack_*.md` :

| #   | Famille                       | Fichier attaque                          | Documentation                   |
| --- | ----------------------------- | ---------------------------------------- | ------------------------------- |
| 1   | Roleplay / fiction encapsulée | `attacks/01_roleplay_fiction.prompt`     | [Attack_1.md](Docs/Attack_1.md) |
| 2   | Autorité usurpée              | `attacks/02_usurped_authority.prompt`    | [Attack_2.md](Docs/Attack_2.md) |
| 3   | Complétion de phrase          | `attacks/03_sentence_completion.prompt`  | [Attack_3.md](Docs/Attack_3.md) |
| 4   | Injection dans un formulaire  | `attacks/04_data_field_injection.prompt` | [Attack_4.md](Docs/Attack_4.md) |
| 5   | Encodage / traduction forcée  | `attacks/05_encoding_translation.prompt` | [Attack_5.md](Docs/Attack_5.md) |

## Limites observées

- **Sensibilité à la langue** : une version française du Modelfile a été testée et a immédiatement échoué (fuite du codeword sur une simple question anodine). Le modèle imposé semble bien moins fiable hors anglais pour suivre des règles de sécurité nuancées, la version finale est donc restée en anglais.
- **Non-déterminisme résiduel** : malgré `temperature 0`, une instabilité ponctuelle a été observée une fois en anglais (fuite partielle non reproduite lors des tests suivants identiques), suggérant que la robustesse n'est pas garantie à 100% avec ce modèle de taille réduite (3B).

## Méthodologie

Les techniques d'attaque ont été identifiées et affinées par la pratique le challenge **Agent Breaker / Cycling Coach** (Lakera), avant d'être testées directement contre l'agent `codename` développé dans ce projet.

---

## Lancement du projet

- Création de l'agent à partir du Modelfile: `ollama create codename -f Modelfile`
- Lancement de l'agent en mode interactif: `ollama run codename` --> Tu tapes tes messages directement. Pour quitter : /bye ou Ctrl+D
- Lancement avec un seul message: `ollama run codename "la question ?"`
- Lancement d'une attaque depuis un fichier: `ollama run codename "$(cat attacks/01_roleplay_fiction.prompt)"`

- Voir le Modelfile d'un modèle déjà créé: `ollama show codename --modelfile`
- Suppression du modèle: `ollama rm codename`

- Lancement du test ./test.sh
- Lancement du test ./release_check.sh Modelfile
- Lancement du test ./release_check2.sh Modelfile