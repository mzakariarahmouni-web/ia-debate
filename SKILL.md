---
name: "ia-debate"
description: "IA Debate : émission de débat télévisée jouée par des agents IA experts, en darija, sourcée Fhamator, avec voix et vidéo. Envoie le manuel PDF et demande un code d'activation."
---

# IA Debate — skill sous licence

Version du skill : **1.8**

Le skill **IA Debate** produit des capsules de l'émission « AI Intelligent Debate » (débat télévisé joué par des agents IA, en darija marocaine). La procédure complète et le kit ne sont pas dans ce fichier : ils sont servis après activation.

**Le skill fonctionne avec TOUS les comptes Claude, gratuit comme payant.** Aucun abonnement Claude, aucune autorisation de domaine n'est exigée : si Claude ne peut pas joindre le serveur directement, l'activation se fait dans le navigateur de l'utilisateur (étape 1.4). Ne jamais demander à l'utilisateur de passer à un forfait payant, ni d'autoriser des domaines (c'est seulement une astuce facultative, M8).

Règles d'activation :
- Le code n'est demandé **qu'une seule fois par conversation** : la première fois, ou si l'éditeur a désactivé/libéré l'activation.
- Une conversation activée le reste, même après une mise à jour ou une réinitialisation de l'espace de travail.
- **À chaque appel**, récupérer la dernière version de la procédure et du kit (mise à jour automatique).

Parler à l'utilisateur dans sa langue (français par défaut), avec les messages de la section « Messages à l'utilisateur ». Faire les vérifications techniques en silence : ne jamais montrer de commande, d'erreur technique ni de code HTTP.

## Étape 1 — Reprise ou activation (à chaque appel)

**1.1 Reprise silencieuse (mode direct).** Lancer :
```bash
AID=https://aid-licence.m-zakariarahmouni.workers.dev; mkdir -p ~/.aid ~/aid
SID="${CLAUDE_CODE_SESSION_ID:-}"
if [ -z "$SID" ]; then [ -f ~/.aid/session ] || python3 -c "import uuid; print(uuid.uuid4())" > ~/.aid/session; SID=$(cat ~/.aid/session); fi
echo "$SID" > ~/.aid/sid; OLD=$(cat ~/.aid/session 2>/dev/null)
R=$(curl -sS -m 20 -X POST "$AID/api/resume" -H "content-type: application/json" -d "{\"session_id\":\"$SID\",\"legacy_id\":\"$OLD\"}" 2>&1)
python3 - "$R" <<'PY'
import json, sys, os
try: d = json.loads(sys.argv[1])
except Exception: print("RESEAU"); sys.exit()
if d.get("ok"):
    open(os.path.expanduser("~/.aid/token"),"w").write(d["token"]); print("ACTIVE"); print("NOM="+(d.get("label") or ""))
else: print("CODE_REQUIS: "+d.get("error",""))
PY
```
- `ACTIVE` → étape 2, puis message **M2**.
- `CODE_REQUIS: session jamais activée` → 1.2 puis 1.3.
- `CODE_REQUIS: code désactivé` / `code expiré` → **M7c**, puis 1.3.
- `RESEAU` → **mode compatible, étape 1.4** (ne rien demander, ne pas parler d'abonnement).

**1.2 Manuel** (première activation seulement, une fois par conversation). Envoyer `IA_Debate_Manuel.pdf` avec `SendUserFile` (display attach), légende « Manuel d'usage IA Debate ». Sinon le télécharger : `curl -fsSL https://raw.githubusercontent.com/mzakariarahmouni-web/ia-debate/main/IA_Debate_Manuel.pdf -o ~/aid/IA_Debate_Manuel.pdf`.

**1.3 Demander le code (mode direct)** : message **M1**, puis, en remplaçant `<CODE>` :
```bash
AID=https://aid-licence.m-zakariarahmouni.workers.dev; SID=$(cat ~/.aid/sid)
curl -sS -m 30 -X POST "$AID/api/activate" -H "content-type: application/json" \
  -d "{\"code\":\"<CODE>\",\"session_id\":\"$SID\"}" -o ~/.aid/activation.json
python3 -c "import json,os; d=json.load(open(os.path.expanduser('~/.aid/activation.json'))); print('OK' if d.get('ok') else 'ERREUR: '+d.get('error','inconnue')); d.get('ok') and open(os.path.expanduser('~/.aid/token'),'w').write(d['token']); d.get('ok') and print('NOM='+(d.get('label') or ''))"
```
- `OK` → étape 2, puis **M3**. Ne jamais réafficher le code.
- `code inconnu` → **M7a** (3 essais maxi).  `code déjà utilisé…` → **M7b**, stop.  `code désactivé`/`code expiré` → **M7c**, stop.
- Si le `curl` échoue (réseau) → **mode compatible, étape 1.4**.

**1.4 Mode compatible — activation dans le navigateur de l'utilisateur.** Aucun accès réseau de Claude n'est requis : l'utilisateur fait l'activation lui-même dans son navigateur, puis dépose le kit.
1. **Déjà activée dans cette conversation ?** Si `~/aid/_kit/VERSION` et `~/.aid/compatible` existent tous les deux, le kit est présent : message **M2** (sans numéro de version si inconnu) et aller à l'étape 3. Ne rien demander.
2. **Sinon, première activation.** Si jamais vu dans cette conversation, envoyer le manuel (1.2). Puis construire le lien d'activation et envoyer **M4** :
```bash
SID=$(cat ~/.aid/sid); OLD=$(cat ~/.aid/session 2>/dev/null)
echo "https://aid-licence.m-zakariarahmouni.workers.dev/activer#s=${SID}${OLD:+&o=$OLD}"
```
   Donner ce lien complet dans **M4** (il contient `#s=...` : ne jamais le raccourcir). L'utilisateur ouvre la page, saisit son code **sur la page** (le code ne passe pas par la conversation), télécharge le kit (.zip) et le dépose dans la conversation.
3. **Quand le fichier est déposé** (souvent dans `/mnt/user-data/uploads`) :
```bash
Z=$(ls -t /mnt/user-data/uploads/*.zip 2>/dev/null | head -1)
cd ~/aid && rm -rf _kit /tmp/aidk && mkdir -p /tmp/aidk && unzip -q "$Z" -d /tmp/aidk \
  && mv /tmp/aidk/aid-kit _kit && cp _kit/kit/* ~/aid/ && echo "kit $(cat _kit/VERSION)" && touch ~/.aid/compatible
```
   Si aucun ZIP valide (pas de dossier `aid-kit`) → redire **M4** en une phrase. Si OK → message **M3** (bienvenue, sans prénom si inconnu), puis, une fois par conversation, l'astuce **M8**. Aller à l'étape 3.

## Étape 2 — Mise à jour (mode direct uniquement)

```bash
AID=https://aid-licence.m-zakariarahmouni.workers.dev
cd ~/aid && rm -rf _kit && mkdir _kit && curl -sS -m 120 -f -H "authorization: Bearer $(cat ~/.aid/token)" \
  "$AID/api/kit" -o /tmp/aid-kit.tgz && tar -xzf /tmp/aid-kit.tgz -C _kit --strip-components=1 \
  && cp _kit/kit/* ~/aid/ && echo "kit $(cat _kit/VERSION)" && rm -f /tmp/aid-kit.tgz
```
Échec (« jeton expiré/inconnu ») → refaire 1.1 une fois, sans redemander le code. En **mode compatible**, ne pas exécuter cette étape : le kit déposé en 1.4 fait foi.

**Version du skill** : `curl -fsSL -m 20 https://raw.githubusercontent.com/mzakariarahmouni-web/ia-debate/main/SKILL.md -o ~/aid/_skill_latest.md && grep -m1 "Version du skill" ~/aid/_skill_latest.md`. Si plus récente que celle de ce fichier : appliquer les étapes 1 et 2 de la version publiée et envoyer **M5**, sans bloquer.

## Étape 3 — Procédure

`~/aid/_kit/skill/PROCEDURE.md` est le fichier de référence : le lire en entier avec Read à chaque appel, puis appliquer ses phases A à G dans l'ordre. Le kit est déjà récupéré : ignorer toute consigne qui redemanderait un code. Pour les voix, utiliser **M6** (connexion) et **M9** (récupération).

## Messages à l'utilisateur

**Prénom.** Vient de `NOM=` / `label` (champ « Pour qui ? » du code). L'utiliser seulement s'il ressemble à un nom (lettres, espaces, tirets, ≤ 30 car.) : retirer ce qui est entre parenthèses, majuscule initiale. Un @, des chiffres ou une note → pas de prénom. Ne jamais afficher d'e-mail. En mode compatible, le prénom est souvent inconnu : écrire les messages sans prénom.

**M1 — Demande du code (mode direct)**
> Bonjour et bienvenue ! Pour démarrer IA Debate, entre ton code d'activation (format AID-XXXX-XXXX-XXXX). Il n'est demandé qu'une seule fois : ensuite, cette conversation reste activée.

**M2 — Retour (déjà activée)**
> Re-bonjour <Prénom>, IA Debate est prêt<et à jour (version <v>)>. Quel sujet veux-tu débattre, et pour quelle durée ?

**M3 — Bienvenue (première activation réussie)**
> Bienvenue <Prénom> dans **IA Debate** ! Ton activation est confirmée pour cette conversation.
>
> Ce que tu vas obtenir :
> - **Des débatteurs IA experts** en argumentation et en économie, chacun avec son rôle, son caractère et sa façon de parler.
> - **Des chiffres vérifiés** selon la méthode Fhamator : sources primaires, statut de chaque chiffre, aucune donnée inventée.
> - **Une vraie émission en darija**, une voix marocaine par personnage, des sous-titres et un jury qui note les débatteurs.
> - **Une vidéo prête à publier**, du dessin animé au photoréaliste, avec la description et les sources.
> - **La durée de ton choix** : jusqu'à 3 minutes, tout se fait avec des outils gratuits.
> - **Des mises à jour automatiques**, sans rien réinstaller.
>
> Dis-moi le sujet du débat et la durée souhaitée.

**M4 — Activation dans le navigateur (mode compatible)**
> Pour activer IA Debate, ouvre ce lien dans ton navigateur : <lien>
> Sur la page : entre ton code d'activation, clique sur **Activer**, puis **télécharge le kit** (un fichier .zip). Reviens ici et **glisse ce fichier dans la conversation**. Dis-moi ensuite « fichier déposé ».
> Rien d'autre à installer, et ça marche avec tout compte Claude. Pour une mise à jour future, ce sera le même geste.

**M5 — Nouvelle version du skill**
> Une nouvelle version du skill IA Debate (<v>) est disponible. Je l'utilise déjà pour cette séance. Pour l'installer durablement :
> 1. [Télécharge le skill IA Debate](https://github.com/mzakariarahmouni-web/ia-debate/raw/main/ia-debate.zip) (ne le décompresse pas).
> 2. Dans Claude : **Personnaliser > Skills**, bouton **+**, **Importer un skill**, choisis le fichier.
>
> Ton activation est conservée : aucun code à ressaisir.

**M6 — Outil de voix à connecter**
> Pour donner une voix à chaque personnage, connecte l'outil de voix : clique sur la carte qui s'affiche ci-dessous et valide avec ton compte (un compte gratuit suffit pour une capsule de 3 minutes). Réponds « c'est fait » ensuite.

**M7 — Code refusé**
- M7a : « Ce code n'est pas reconnu. Vérifie-le et renvoie-le (format AID-XXXX-XXXX-XXXX). »
- M7b : « Ce code a déjà servi dans une autre conversation Claude. Chaque code active une seule conversation : demande un nouveau code à l'éditeur, ou qu'il libère l'ancien. »
- M7c : « L'activation de cette conversation a été désactivée par l'éditeur. Entre un nouveau code fourni par l'éditeur pour continuer. »

**M8 — Astuce facultative (mode compatible, une fois)**
> Astuce (facultatif) : si ton compte Claude le permet, tu peux tout rendre automatique en ajoutant `aid-licence.m-zakariarahmouni.workers.dev`, `storage.googleapis.com` et `api.elevenlabs.io` dans **Paramètres > Capacités**, domaines autorisés. Sinon, le dépôt de fichier marche très bien.

**M9 — Récupérer les voix (si le téléchargement automatique est impossible)**
> Les voix sont prêtes. Choisis l'option la plus simple :
> - **App Claude pour ordinateur (Claude Bureau)** : ouvre cette conversation dans l'app installée sur ton ordinateur et réponds « je suis sur l'app ». Je récupère les voix moi-même.
> - **Ici** : ouvre ce lien <url du projet de voix>, télécharge les fichiers audio (un par réplique) et glisse-les tous dans la conversation.

## Sans activation valide

Si l'activation échoue ou si l'utilisateur n'a pas de code : ne pas produire l'émission, ne rien reconstituer de mémoire, ne pas contourner le serveur. Envoyer **M7**, ou sans code : « IA Debate nécessite un code d'activation, à obtenir auprès de l'éditeur du skill. »
