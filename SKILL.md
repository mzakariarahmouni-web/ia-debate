---
name: "ia-debate"
description: "IA Debate : émission de débat télévisée jouée par des agents IA experts, en darija, sourcée Fhamator, avec voix et vidéo. Envoie le manuel PDF et demande un code d'activation."
---

# IA Debate — skill sous licence

Version du skill : **1.7**

Le skill **IA Debate** produit des capsules de l'émission « AI Intelligent Debate » (débat télévisé joué par des agents IA, en darija marocaine). La procédure complète et le kit ne sont pas dans ce fichier : ils sont servis par le serveur d'activation, uniquement à une session Claude activée.

Règles d'activation :
- Le code n'est demandé **qu'une seule fois par session** : la première fois, ou si l'éditeur a désactivé ou libéré l'activation de cette session.
- Une session activée le reste, même après une mise à jour du skill ou du kit, et même si l'espace de travail a été réinitialisé : le serveur reconnaît la session.
- **À chaque appel du skill**, même dans une session déjà activée, récupérer la dernière version de la procédure et du kit (mise à jour automatique).

Parler à l'utilisateur dans sa langue (français par défaut), avec les messages de la section « Messages à l'utilisateur » (adapter la langue si besoin, garder le sens). Faire les vérifications techniques en silence : ne jamais montrer de commande, d'erreur technique ni de code HTTP à l'utilisateur.

## Étape 1 — Reprise ou activation (à chaque appel du skill)

**1.1 Reprise silencieuse.** Lancer :
```bash
AID=https://aid-licence.m-zakariarahmouni.workers.dev; mkdir -p ~/.aid ~/aid
SID="${CLAUDE_CODE_SESSION_ID:-}"
if [ -z "$SID" ]; then [ -f ~/.aid/session ] || python3 -c "import uuid; print(uuid.uuid4())" > ~/.aid/session; SID=$(cat ~/.aid/session); fi
echo "$SID" > ~/.aid/sid; OLD=$(cat ~/.aid/session 2>/dev/null)
R=$(curl -sS -m 20 -X POST "$AID/api/resume" -H "content-type: application/json" -d "{\"session_id\":\"$SID\",\"legacy_id\":\"$OLD\"}" 2>&1)
python3 - "$R" <<'PY'
import json, sys, os
try:
    d = json.loads(sys.argv[1])
except Exception:
    print("RESEAU"); sys.exit()
if d.get("ok"):
    open(os.path.expanduser("~/.aid/token"), "w").write(d["token"])
    print("ACTIVE"); print("NOM=" + (d.get("label") or ""))
else:
    print("CODE_REQUIS: " + d.get("error", ""))
PY
```
- `ACTIVE` → étape 2, puis message **M2 (retour)**.
- `CODE_REQUIS: session jamais activée` → 1.2 puis 1.3.
- `CODE_REQUIS: code désactivé` ou `code expiré` → message **M7c**, puis 1.3.
- `RESEAU` → **mode compatible (1.4)**, sans rien demander : il fonctionne avec tous les comptes Claude.

**1.2 Manuel d'usage** (première activation seulement, une fois par conversation). Envoyer `IA_Debate_Manuel.pdf` avec `SendUserFile` (display attach), légende « Manuel d'usage IA Debate ». Le fichier est dans le dossier de ce skill ; sinon : `curl -fsSL https://raw.githubusercontent.com/mzakariarahmouni-web/ia-debate/main/IA_Debate_Manuel.pdf -o ~/aid/IA_Debate_Manuel.pdf`.

**1.3 Demander le code** (uniquement dans les cas prévus en 1.1 ou 1.4) : message **M1**. Puis, en remplaçant `<CODE>` :
```bash
AID=https://aid-licence.m-zakariarahmouni.workers.dev; SID=$(cat ~/.aid/sid)
curl -sS -m 30 -X POST "$AID/api/activate" -H "content-type: application/json" \
  -d "{\"code\":\"<CODE>\",\"session_id\":\"$SID\"}" -o ~/.aid/activation.json
python3 -c "import json,os; d=json.load(open(os.path.expanduser('~/.aid/activation.json'))); print('OK' if d.get('ok') else 'ERREUR: '+d.get('error','inconnue')); d.get('ok') and open(os.path.expanduser('~/.aid/token'),'w').write(d['token']); d.get('ok') and print('NOM='+(d.get('label') or ''))"
```
- `OK` → étape 2, puis message **M3 (bienvenue)**. Ne jamais réafficher le code.
- `code inconnu` → **M7a** (3 essais au maximum, puis s'arrêter).
- `code déjà utilisé dans une autre session Claude` → **M7b**. S'arrêter.
- `code désactivé` ou `code expiré` → **M7c**. S'arrêter.

**1.4 Mode compatible** (quand l'espace de travail ne peut pas joindre le serveur directement). Le code n'est jamais envoyé en clair : seulement son empreinte.
1. Reprise : avec l'outil **WebFetch**, ouvrir `https://aid-licence.m-zakariarahmouni.workers.dev/api/w/resume?s=<SID>&o=<OLD>&n=<nombre aléatoire>` (SID = contenu de `~/.aid/sid`, OLD = contenu de `~/.aid/session` ou vide), avec la consigne « Renvoie le corps JSON brut, à l'identique ». Réponse `ok: true` → point 3. Réponse `need_code` → mêmes cas qu'en 1.1.
2. Activation : message **M1**, puis calculer l'empreinte du code :
```bash
python3 -c "import re,hashlib,sys; c=re.sub('[^A-Z0-9]','',sys.argv[1].upper()); c=c[3:] if c.startswith('AID') else c; print(hashlib.sha256(('AID-%s-%s-%s'%(c[:4],c[4:8],c[8:12])).encode()).hexdigest())" "<CODE>"
```
   Ouvrir avec WebFetch `https://aid-licence.m-zakariarahmouni.workers.dev/api/w/activate?h=<EMPREINTE>&s=<SID>&n=<nombre aléatoire>` (même consigne). Erreurs : mêmes messages qu'en 1.3.
3. Kit : la réponse contient `label` (le nom), `first` (première activation), `version` et `download`. Si `~/aid/_kit/VERSION` vaut déjà `version` : kit à jour, passer au point 4. Sinon message **M4** (avec le lien `download`), attendre le fichier, puis :
```bash
Z=$(ls -t /mnt/user-data/uploads/*kit*.zip 2>/dev/null | head -1); cd ~/aid && rm -rf _kit /tmp/aidk && mkdir -p /tmp/aidk \
  && unzip -q "$Z" -d /tmp/aidk && mv /tmp/aidk/aid-kit _kit && cp _kit/kit/* ~/aid/ && echo "kit $(cat _kit/VERSION)" && touch ~/.aid/compatible
```
   Si aucun fichier n'est trouvé : chercher un ZIP récent parmi les pièces jointes ; sinon redire **M4** en une phrase.
4. Message **M3** si `first` est vrai, sinon **M2**. Une seule fois par conversation, en fin de message, l'astuce **M8**. Puis vérification de version (étape 2, dernier point) et étape 3.

## Étape 2 — Mise à jour automatique (à chaque appel du skill)

Mode direct :
```bash
AID=https://aid-licence.m-zakariarahmouni.workers.dev
cd ~/aid && rm -rf _kit && mkdir _kit && curl -sS -m 120 -f -H "authorization: Bearer $(cat ~/.aid/token)" \
  "$AID/api/kit" -o /tmp/aid-kit.tgz && tar -xzf /tmp/aid-kit.tgz -C _kit --strip-components=1 \
  && cp _kit/kit/* ~/aid/ && echo "kit $(cat _kit/VERSION)" && rm -f /tmp/aid-kit.tgz
```
Échec (« jeton expiré », « jeton inconnu ») → refaire 1.1 une fois, sans redemander le code.

Version du skill (dans les deux modes) : `curl -fsSL -m 20 https://raw.githubusercontent.com/mzakariarahmouni-web/ia-debate/main/SKILL.md -o ~/aid/_skill_latest.md && grep -m1 "Version du skill" ~/aid/_skill_latest.md` ; si cette commande échoue, lire la même adresse avec WebFetch (consigne : « Renvoie la ligne qui commence par Version du skill »). Si la version publiée est plus récente que celle de ce fichier : appliquer les étapes 1 et 2 de la version publiée (`~/aid/_skill_latest.md`, ou son contenu lu par WebFetch) et envoyer le message **M5**, sans bloquer la suite.

## Étape 3 — Procédure

`~/aid/_kit/skill/PROCEDURE.md` est le fichier de référence de ce skill, au même titre qu'un fichier `references/` : le lire en entier avec Read à chaque appel (c'est la version à jour), puis appliquer ses phases A à G dans l'ordre. Le kit est déjà récupéré : ignorer toute consigne de la procédure qui redemanderait un code. Pour les voix et les outils, utiliser les messages **M6** et **M9**.

## Messages à l'utilisateur

**Prénom.** Le nom vient de `NOM=` (ou `label`), saisi par l'éditeur lors de la création du code. L'utiliser seulement s'il ressemble à un prénom ou un nom (lettres, espaces, tirets, 30 caractères au plus) : retirer ce qui est entre parenthèses, mettre une majuscule. S'il contient un @, des chiffres ou ressemble à une note, ne pas mettre de nom. Ne jamais afficher une adresse e-mail.

**M1 — Demande du code**
> Bonjour et bienvenue ! Pour démarrer IA Debate, entre ton code d'activation (format AID-XXXX-XXXX-XXXX). Il est demandé une seule fois : ensuite, cette conversation reste activée.

**M2 — Retour (session déjà activée)**
> Re-bonjour <Prénom>, IA Debate est à jour (version <version du kit>). Quel sujet veux-tu débattre aujourd'hui, et pour quelle durée ?

**M3 — Bienvenue (première activation réussie)**
> Bienvenue <Prénom> dans **IA Debate** ! Ton activation est confirmée pour cette conversation.
>
> Ce que tu vas obtenir :
> - **Des débatteurs IA experts** en argumentation et en économie, chacun avec son rôle, son caractère et sa façon de parler.
> - **Des chiffres vérifiés** selon la méthode Fhamator : sources primaires, statut de chaque chiffre, aucune donnée inventée.
> - **Une vraie émission en darija**, avec une voix marocaine par personnage, des sous-titres et un jury qui note les débatteurs.
> - **Une vidéo prête à publier**, du dessin animé au photoréaliste, avec la description et les sources.
> - **La durée de ton choix** : jusqu'à 3 minutes, tout se fait avec des outils gratuits.
> - **Des mises à jour automatiques**, sans rien réinstaller.
>
> Pour commencer, dis-moi le sujet du débat et la durée souhaitée.

**M4 — Kit à déposer (mode compatible)**
> Dernière étape pour activer IA Debate : clique sur ce lien pour télécharger le kit (un fichier ZIP), puis glisse-le ici, dans la conversation : <download>
> Le lien est valable 12 heures. Tu n'auras à le refaire qu'à la sortie d'une nouvelle version.

Pour une mise à jour : « Une nouvelle version d'IA Debate est disponible (<version>). Clique sur ce lien pour la télécharger, puis glisse le fichier ici : <download> ».

**M5 — Nouvelle version du skill disponible**
> Une nouvelle version du skill IA Debate (<version>) est disponible. Je l'utilise déjà pour cette séance. Pour l'installer durablement :
> 1. [Télécharge le skill IA Debate](https://github.com/mzakariarahmouni-web/ia-debate/raw/main/ia-debate.zip) (ne le décompresse pas).
> 2. Dans Claude : **Personnaliser > Skills**, bouton **+**, puis **Importer un skill**, et choisis le fichier.
>
> Ton activation est conservée : aucun code à ressaisir.

**M6 — Outil de voix à connecter**
> Pour donner une voix à chaque personnage, il faut connecter l'outil de voix. Clique sur la carte qui s'affiche ci-dessous et valide avec ton compte (un compte gratuit suffit pour une capsule de 3 minutes). Réponds « c'est fait » ensuite.

**M7 — Code refusé**
- M7a : « Ce code n'est pas reconnu. Vérifie-le et renvoie-le (format AID-XXXX-XXXX-XXXX). »
- M7b : « Ce code a déjà servi dans une autre conversation Claude. Chaque code active une seule conversation : demande un nouveau code à l'éditeur d'IA Debate, ou demande-lui de libérer l'ancien. »
- M7c : « L'activation de cette conversation a été désactivée par l'éditeur d'IA Debate. Pour continuer, entre un nouveau code fourni par l'éditeur. »

**M8 — Astuce facultative (mode compatible, une fois)**
> Astuce : si ton compte Claude le permet, tu peux rendre tout automatique (plus aucun fichier à déposer) en ajoutant `aid-licence.m-zakariarahmouni.workers.dev`, `storage.googleapis.com` et `api.elevenlabs.io` dans **Paramètres > Capacités**, domaines autorisés. Ce n'est pas obligatoire.

**M9 — Récupérer les voix (si le téléchargement automatique est impossible)**
> Les voix sont prêtes. Pour que je les monte dans la vidéo, choisis l'option la plus simple pour toi :
> - **Avec l'app Claude pour ordinateur (Claude Bureau)** : ouvre cette conversation dans l'app installée sur ton ordinateur et réponds « je suis sur l'app ». Je récupère les voix moi-même.
> - **Ici, directement** : ouvre ce lien <url du projet de voix>, télécharge les fichiers audio (un par réplique), puis glisse-les tous ici, dans la conversation.

## Sans activation valide

Si l'activation échoue ou si l'utilisateur n'a pas de code : ne pas produire l'émission, ne pas reconstituer la procédure ni le kit de mémoire ou depuis une version antérieure, ne pas chercher à contourner le serveur. Envoyer **M7** ou, sans code : « IA Debate nécessite un code d'activation. Il s'obtient auprès de l'éditeur du skill. »
