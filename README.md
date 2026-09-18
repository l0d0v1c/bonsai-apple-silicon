# Bonsai 2 27B — binaires llama.cpp pour Mac Apple Silicon

Ce dossier contient `llama-server`, `llama-cli` et les outils associés, compilés avec le
backend **Metal**, pour exécuter le modèle **Ternary-Bonsai-2-27B** de
[Prism ML](https://prismml.com) en local sur un Mac.

---

## Qu'est-ce que Bonsai 2 27B

Un modèle de langage de **27 milliards de paramètres**, capable de raisonnement, de code et
d'appel d'outils, qui tient dans **6,7 Gio** et tourne sur un portable.

Il dérive de Qwen3.8-27B, dont il conserve l'architecture. Ce qui change, c'est la façon
dont les poids sont stockés : au lieu des 16 bits habituels par poids, chaque poids ne
prend plus que trois valeurs possibles — **−1, 0 ou +1** — avec un facteur d'échelle
partagé par groupe de 128. C'est ce qu'on appelle une représentation **ternaire**. Le coût
réel revient à **1,72 bit par poids**, soit environ **9 fois moins** que le modèle
d'origine en pleine précision (~54 Go).

L'intérêt n'est pas seulement la taille. Les méthodes de compression classiques s'effondrent
sous les 4 bits : elles gardent l'air compétent sur les questions de culture générale mais
perdent la capacité à tenir une chaîne de raisonnement longue. Bonsai est construit pour
éviter précisément cet effondrement.

**En pratique**, ce modèle :

- **raisonne avant de répondre** — il produit une trace de réflexion, séparée de la réponse
  finale dans l'API (voir §5) ;
- accepte un contexte de **262 144 tokens** (~200 000 mots : un dépôt de code entier, un
  livre) ;
- **lit les images** si vous chargez le fichier `mmproj` optionnel ;
- sait **appeler des outils** (fonctions), ce qui permet de l'utiliser comme agent ;
- est publié sous licence **Apache 2.0**, utilisable commercialement ;
- tourne **entièrement hors ligne** : aucune donnée ne quitte la machine.

### Ce qu'il vaut

Chiffres **annoncés par Prism ML** dans la carte modèle, mesurés sur 14 jeux d'évaluation
en mode raisonnement. Ils ne sont pas vérifiés indépendamment ici.

| Modèle | Bits/poids | Taille | Score moyen | % du FP16 |
| :----- | ---------: | -----: | ----------: | --------: |
| Qwen3.8-27B FP16 (référence) | 16,0 | 54 Go | 86,32 | 100 % |
| Qwen3.8-27B UD-Q4_K_XL (« 4 bits ») | 5,2 | 17,6 Go | 85,18 | 98,7 % |
| Qwen3.8-27B IQ2_XXS (« 2 bits ») | 2,8 | 9,4 Go | 72,59 | 84,1 % |
| **Bonsai 2 27B** | **1,72** | **5,9 Go** | **84,78** | **98,2 %** |

Lecture : Bonsai conserve **98,2 %** du niveau du modèle en pleine précision, alors qu'il
est **neuf fois plus petit**. Face à la compression 2 bits conventionnelle, il gagne plus
de **12 points** tout en occupant **un tiers de place en moins**. Et il arrive à 0,4 point
d'une compression 4 bits qui pèse trois fois plus lourd.

Par domaine :

| Domaine | FP16 | Bonsai 2 27B |
| :------ | ---: | -----------: |
| Mathématiques | 97,06 | **96,57** |
| Code | 89,07 | **89,42** |
| Suivi d'instructions | 81,25 | **82,66** |
| Appel d'outils (agent) | 76,74 | 74,92 |
| Connaissances & raisonnement | 85,55 | 79,86 |
| Vision | 71,36 | 66,19 |

Le cœur de raisonnement passe intact : les mathématiques perdent un demi-point, le code et
le suivi d'instructions sont **au niveau ou au-dessus** de la pleine précision. L'écart se
concentre sur les connaissances encyclopédiques et la vision — c'est là qu'il faut
l'attendre au tournant si votre usage en dépend.

### Vitesse réelle

Mesuré **par mes soins** avec ces binaires, sur un **Apple M2 Max** :

| | tok/s |
| :--- | ----: |
| Lecture du prompt (`pp512`) | 131,2 |
| Génération de la réponse (`tg128`) | 16,4 |

Environ **16 tokens par seconde** en génération : c'est un rythme de lecture confortable,
utilisable en conversation. Le modèle raisonnant avant de répondre, comptez tout de même
plusieurs dizaines de secondes pour une réponse complète.

La vitesse dépend surtout de la bande passante mémoire de la puce. Ordres de grandeur
annoncés par Prism ML sur d'autres machines : ~18 tok/s sur M4 Pro, ~29 sur M5 Pro,
~47 sur M5 Max. Un M2 Max est deux générations en arrière, d'où le résultat ci-dessus.

Pour reproduire la mesure sur votre machine, voir §7.

---

## Ces binaires

Ils viennent du fork [`PrismML-Eng/llama.cpp`](https://github.com/PrismML-Eng/llama.cpp)
(commit `1a07bfa`). **Le llama.cpp officiel ne peut pas charger ce modèle**, ni Ollama, ni
LM Studio — voir §8 pour les trois raisons techniques.

---

## 1. Prérequis

| Élément | Exigence | Vérifier |
| :------ | :------- | :------- |
| Processeur | **Apple Silicon** (M1 → M5). Pas d'Intel. | `uname -m` doit dire `arm64` |
| Système | **macOS 26 (Tahoe) ou plus récent** | `sw_vers -productVersion` |
| OpenSSL | **Homebrew `openssl@3`** | `ls /opt/homebrew/opt/openssl@3` |
| Mémoire | 16 Go minimum, 24 Go+ confortable | |
| Disque | ~7 Go pour le modèle | |

### macOS 26 est une exigence stricte

Les binaires sont compilés avec `minos 26.0`. Sur macOS 15 (Sequoia) ou antérieur, ils ne
démarreront pas du tout. Il faut alors recompiler le fork depuis les sources.

### Installer OpenSSL

`llama-server` et `llama-cli` sont liés à l'OpenSSL de Homebrew par chemin absolu.

```sh
# si Homebrew n'est pas installé :
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"

brew install openssl@3
```

Sans cette bibliothèque, le lancement échoue avec :

```
dyld: Library not loaded: /opt/homebrew/opt/openssl@3/lib/libssl.3.dylib
```

---

## 2. Débloquer les binaires (Gatekeeper)

**À faire une fois, obligatoirement**, si vous avez reçu ce dossier par téléchargement,
AirDrop, e-mail ou clé USB. macOS met en quarantaine les exécutables venus de l'extérieur
et les signatures ici sont ad-hoc (pas de certificat développeur payant).

```sh
xattr -dr com.apple.quarantine /chemin/vers/bin
chmod +x /chemin/vers/bin/llama-*
```

Sans cela, macOS refuse le lancement avec « *impossible d'ouvrir, développeur non
vérifié* », ou tue le processus sans message.

Vérification :

```sh
./llama-cli --version
# version: 0.2.0-dev (build 1, commit 1a07bfa)
# built with AppleClang 21.0.0.21000101 for Darwin arm64
```

Si cette commande répond, tout est en place.

---

## 3. Télécharger le modèle

Deux fichiers, depuis
[huggingface.co/prism-ml/Ternary-Bonsai-2-27B-gguf](https://huggingface.co/prism-ml/Ternary-Bonsai-2-27B-gguf) :

| Fichier | Taille | Nécessaire ? |
| :------ | -----: | :----------- |
| `Ternary-Bonsai-2-27B-PQ2_0.gguf` | 6,7 Gio | **Oui** — le modèle de langue |
| `Ternary-Bonsai-2-27B-mmproj-BF16.gguf` | 888 Mio | Non — uniquement pour l'entrée image |

Placez-les dans le dossier **parent** de `bin/` (c'est ce qu'attendent les commandes qui
suivent), ou n'importe où en adaptant le chemin `-m`.

### Avec `curl` (aucune installation)

```sh
cd /chemin/vers/le/dossier-parent

curl -L -o Ternary-Bonsai-2-27B-PQ2_0.gguf \
  "https://huggingface.co/prism-ml/Ternary-Bonsai-2-27B-gguf/resolve/main/Ternary-Bonsai-2-27B-PQ2_0.gguf"
```

Le téléchargement est long (6,7 Gio). `curl -L -C -` reprend un transfert interrompu.

### Avec le client Hugging Face

```sh
pip install -U huggingface_hub
hf download prism-ml/Ternary-Bonsai-2-27B-gguf \
    Ternary-Bonsai-2-27B-PQ2_0.gguf --local-dir .
```

`--local-dir .` est important : sans lui, le fichier atterrit dans le cache global
`~/.cache/huggingface` au lieu du dossier du projet.

### Vérifier le téléchargement

```sh
ls -lh Ternary-Bonsai-2-27B-PQ2_0.gguf    # doit faire ~6,7G
./bin/llama-tokenize -m Ternary-Bonsai-2-27B-PQ2_0.gguf -p "test" >/dev/null && echo "fichier valide"
```

Un fichier tronqué est la cause d'erreur la plus fréquente.

---

## 4. Lancer `llama-server`

Depuis le dossier qui contient le `.gguf` :

```sh
./bin/llama-server \
    --model Ternary-Bonsai-2-27B-PQ2_0.gguf \
    --alias bonsai-2-27b \
    --host 127.0.0.1 --port 8080 \
    --ctx-size 32768 \
    --n-gpu-layers 99 \
    --flash-attn on \
    --jinja \
    --reasoning on --reasoning-format deepseek \
    --temp 1.0 --top-p 0.95 --top-k 20 --min-p 0.0 \
    --no-mmproj
```

Le serveur est prêt quand il affiche :

```
srv  llama_server: model loaded
srv  llama_server: listening on http://127.0.0.1:8080
```

Le premier chargement prend une vingtaine de secondes ; les suivants sont quasi instantanés
(cache disque du système).

**Interface web** : ouvrez `http://127.0.0.1:8080` dans un navigateur.

### Ce que font les options

| Option | Rôle |
| :----- | :--- |
| `--n-gpu-layers 99` | Met toutes les couches sur le GPU. `0` = CPU seul (beaucoup plus lent). |
| `--ctx-size 32768` | Fenêtre de contexte. Le modèle accepte jusqu'à `262144`, mais le cache mémoire grandit d'autant. Sans cette option, le serveur alloue directement 262 144. |
| `--flash-attn on` | Attention optimisée, active sur Metal. |
| `--jinja` | Utilise le gabarit de conversation embarqué dans le modèle. **Indispensable** : sans lui les réponses sont incohérentes. |
| `--reasoning-format deepseek` | Sépare la réflexion du modèle dans `message.reasoning_content` au lieu de la mélanger à la réponse. |
| `--temp / --top-p / --top-k` | Valeurs recommandées par Prism ML en mode raisonnement. |
| `--no-mmproj` | N'alloue pas la tour de vision. À retirer pour l'entrée image (voir §6). |
| `--parallel N` | Nombre de conversations simultanées. **Attention** : le contexte est divisé entre elles — `--parallel 4 --ctx-size 32768` ne laisse que 8 192 tokens par conversation. |

### Arrêter le serveur

`Ctrl+C` dans le terminal, ou :

```sh
pkill -f llama-server
```

---

## 5. Interroger le modèle

L'API est compatible OpenAI.

```sh
curl -s http://127.0.0.1:8080/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
        "model": "bonsai-2-27b",
        "messages": [{"role": "user", "content": "Explique la photosynthese en deux phrases."}],
        "max_tokens": 2000
      }' | python3 -m json.tool
```

Depuis Python, avec n'importe quel client OpenAI :

```python
from openai import OpenAI

client = OpenAI(base_url="http://127.0.0.1:8080/v1", api_key="rien")
r = client.chat.completions.create(
    model="bonsai-2-27b",
    messages=[{"role": "user", "content": "Bonjour"}],
    max_tokens=2000,
)
print(r.choices[0].message.content)
```

### Prévoyez un `max_tokens` généreux

Ce modèle **raisonne avant de répondre**, en effort `xhigh` par défaut. Sa réflexion
consomme le budget de tokens en premier. Avec `max_tokens: 600`, une question simple est
revenue avec 600 tokens consommés et un `content` **vide** : tout était parti dans le
raisonnement.

- Mettez `max_tokens` à **2000 minimum**.
- Ou réduisez l'effort : ajoutez `--reasoning-effort medium` au lancement du serveur.
- L'effort `low` n'est pas pris en charge par ce modèle : il se comporte alors comme `xhigh`.

---

## 6. En ligne de commande, sans serveur

Chat interactif :

```sh
./bin/llama-cli -m Ternary-Bonsai-2-27B-PQ2_0.gguf \
    -ngl 99 -fa on -c 8192 --temp 1.0 --top-p 0.95 --top-k 20
```

Une seule question puis sortie (`-st` = *single turn*) :

```sh
./bin/llama-cli -m Ternary-Bonsai-2-27B-PQ2_0.gguf \
    -ngl 99 -fa on -c 8192 -st -p "Combien font 17 x 23 ?" -n 500
```

L'option `-no-cnv` mentionnée dans certaines documentations **n'existe pas** dans cette
version : la CLI a été remaniée. Utilisez `-st`.

### Entrée image

Téléchargez aussi le fichier `mmproj`, puis remplacez `--no-mmproj` par :

```sh
--mmproj Ternary-Bonsai-2-27B-mmproj-BF16.gguf
```

---

## 7. Mesurer les performances

```sh
./bin/llama-bench -m Ternary-Bonsai-2-27B-PQ2_0.gguf -p 512 -n 128 -ngl 99 -fa 1
```

Mesuré sur **Apple M2 Max** avec ces binaires :

| Test | Signification | tok/s |
| :--- | :------------ | ----: |
| `pp512` | Lecture du prompt | 131,2 ± 7,4 |
| `tg128` | Génération de la réponse | 16,4 ± 0,2 |

La mesure de référence et sa mise en perspective sont commentées dans l'introduction
(« Vitesse réelle »).

---

## 8. Pourquoi ces binaires et pas llama.cpp officiel

Avec un `llama-cli` standard (Homebrew, Ollama, LM Studio), le chargement échoue :

```
gguf_init_from_file_impl: tensor 'output.weight' has invalid ggml type 142 (NONE)
llama_model_load: error loading model: failed to load model
```

Trois raisons, toutes inscrites dans le fichier modèle :

1. **Type `142` = `PQ2_0`**, le format ternaire de Prism ML (2,13 bits par poids). Inconnu
   du llama.cpp officiel, qui s'arrête là.
2. **Architecture `qwen35`** : attention hybride avec blocs à état récurrent. Un bloc sur
   quatre seulement fait de l'attention complète — c'est ce qui rend le contexte de
   262 144 tokens praticable sur un portable.
3. **Rotation de Hadamard** (`prism.hadamard.transform`) : les poids sont stockés dans une
   base tournée. Un runtime qui n'applique pas la transformation inverse aux activations
   produit du charabia, **même s'il parvient à charger le fichier**.

Ollama et LM Studio embarquent le llama.cpp officiel : ils échouent de la même façon.

---

## 9. En cas de problème

| Symptôme | Cause | Solution |
| :------- | :---- | :------- |
| « développeur non vérifié », ou processus tué sans message | Quarantaine macOS | `xattr -dr com.apple.quarantine bin` (§2) |
| `Library not loaded: .../libssl.3.dylib` | OpenSSL absent | `brew install openssl@3` (§1) |
| `Bad CPU type` / rien ne se lance | Mac Intel, ou macOS < 26 | Recompiler le fork depuis les sources |
| `invalid ggml type 142` | Vous utilisez un **autre** `llama-cli` que celui-ci | Préfixer par `./bin/` |
| `failed to load model` | Fichier `.gguf` incomplet | Comparer la taille : ~6,7 Gio |
| `content` vide dans la réponse | `max_tokens` trop bas | Passer à 2000+ (§5) |
| Réponses incohérentes | `--jinja` oublié | Ajouter `--jinja` |
| Très lent | Modèle sur CPU | Vérifier `--n-gpu-layers 99` |
| Mémoire saturée au démarrage | `--ctx-size` absent → 262 144 alloués | Fixer `--ctx-size 32768` |

Erreur de dylib après avoir **déplacé** un binaire hors de ce dossier : les exécutables
cherchent leurs bibliothèques à côté d'eux (`@loader_path`). Gardez le dossier `bin/`
entier, ou appelez-le par son chemin (`/chemin/vers/bin/llama-server`) plutôt que de copier
un fichier seul.

---

## 10. Contenu du dossier

| Outil | Rôle |
| :---- | :--- |
| `llama-server` | Serveur HTTP + interface web (API compatible OpenAI) |
| `llama-cli` | Chat en ligne de commande |
| `llama-bench` | Mesure de débit |
| `llama-mtmd-cli` | Ligne de commande multimodale (texte + image) |
| `llama-tokenize` | Découpe un texte en tokens |
| `llama-embedding` | Produit des vecteurs d'embedding |
| `llama-perplexity` | Évaluation de qualité |
| `llama-quantize` | Conversion entre formats GGUF |
| `llama-imatrix`, `llama-gguf-split`, `llama-speculative`, `llama-batched-bench`, `llama-completion`, `llama` | Outils avancés |

Les fichiers `.dylib` sont les bibliothèques partagées : **ne les supprimez pas**, aucun
outil ne démarre sans elles.

Environ 21 Mo au total.

---

## Licences

Modèle Bonsai 2 27B : Apache 2.0 — [Prism ML](https://prismml.com).
llama.cpp et son fork : MIT.
