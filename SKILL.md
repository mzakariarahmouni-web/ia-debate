---
name: "ia-debate"
description: "IA Debate : émission de débat télévisée jouée par des agents IA experts, en darija, sourcée Fhamator, avec voix et vidéo. Envoie le manuel PDF et demande un code d'activation."
---

# IA Debate — skill sous licence

Version du skill : **1.5**

Le skill **IA Debate** produit des capsules de l'émission « AI Intelligent Debate » (débat télévisé joué par des agents IA, en darija marocaine). La procédure complète et le kit ne sont pas dans ce fichier : ils sont servis par le serveur d'activation, uniquement à une session Claude activée.

Règles d'activation :
- Le code n'est demandé **qu'une seule fois par session** : la première fois, ou si l'éditeur a désactivé ou libéré l'activation de cette session.
- Une session activée le reste, même après une mise à jour du skill ou du kit, et même si l'espace de travail a été réinitialisé : le serveur reconnaît la session.
- **À chaque appel du skill**, même dans une session déjà activée, télécharger la dernière version de la procédure et du kit (mise à jour automatique).

Parler à l'utilisateur dans sa langue (français par défaut). Faire les vérifications techniques en silence : ne pas commenter les étapes 1 et 2 quand tout va bien.

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
r = sys.argv[1]
try:
    d = json.loads(r)
except Exception:
    print("RESEAU"); sys.exit()
if d.get("ok"):
    open(os.path.expanduser("~/.aid/token"), "w").write(d["token"]); print("ACTIVE")
else:
    print("CODE_REQUIS: " + d.get("error", ""))
PY
```
- `ACTIVE` → aller à l'étape 2 sans rien demander.
- `CODE_REQUIS: session jamais activée` → 1.2 (envoyer le manuel), puis 1.3.
- `CODE_REQUIS: code désactivé` ou `code expiré` → dire en une phrase que l'activation de cette session a été désactivée par l'éditeur, puis 1.3 (nouveau code).
- `RESEAU` (« CONNECT tunnel failed 403 », délai dépassé) → envoyer ce message et attendre :
> Pour utiliser le skill, Claude doit pouvoir joindre le serveur d'activation et les outils de voix. Sur claude.ai dans un navigateur : **Paramètres > Capacités** (en anglais : Settings > Capabilities), section exécution de code, domaines autorisés : ajoute `aid-licence.m-zakariarahmouni.workers.dev`, `storage.googleapis.com` et `api.elevenlabs.io`, puis enregistre. Sur Team ou Enterprise, c'est ton administrateur qui le fait. Réponds « c'est fait » ensuite.

**1.2 Manuel d'usage** (première activation seulement, une fois par conversation). Envoyer `IA_Debate_Manuel.pdf` avec `SendUserFile` (display attach), légende « Manuel d'usage IA Debate ». Le fichier est dans le dossier de ce skill ; sinon : `curl -fsSL https://raw.githubusercontent.com/mzakariarahmouni-web/ia-debate/main/IA_Debate_Manuel.pdf -o ~/aid/IA_Debate_Manuel.pdf`.

**1.3 Demander le code** (uniquement dans les cas prévus en 1.1). Message unique : « Entre ton code d'activation IA Debate (format AID-XXXX-XXXX-XXXX) : il active le skill pour cette conversation, une seule fois. » Puis, en remplaçant `<CODE>` :
```bash
AID=https://aid-licence.m-zakariarahmouni.workers.dev; SID=$(cat ~/.aid/sid)
curl -sS -m 30 -X POST "$AID/api/activate" -H "content-type: application/json" \
  -d "{\"code\":\"<CODE>\",\"session_id\":\"$SID\"}" -o ~/.aid/activation.json
python3 -c "import json,os; d=json.load(open(os.path.expanduser('~/.aid/activation.json'))); print('OK' if d.get('ok') else 'ERREUR: '+d.get('error','inconnue')); d.get('ok') and open(os.path.expanduser('~/.aid/token'),'w').write(d['token'])"
```
- `OK` → étape 2. Ne jamais réafficher le code.
- `ERREUR: code inconnu` → demander de vérifier la saisie (3 essais au maximum, puis s'arrêter).
- `code désactivé`, `code expiré`, `code déjà utilisé dans une autre session Claude` → ce code ne peut pas servir ici ; en demander un autre à l'éditeur du skill. S'arrêter.

## Étape 2 — Mise à jour automatique (à chaque appel du skill)

```bash
AID=https://aid-licence.m-zakariarahmouni.workers.dev
cd ~/aid && rm -rf _kit && mkdir _kit && curl -sS -m 120 -f -H "authorization: Bearer $(cat ~/.aid/token)" \
  "$AID/api/kit" -o /tmp/aid-kit.tgz && tar -xzf /tmp/aid-kit.tgz -C _kit --strip-components=1 \
  && cp _kit/kit/* ~/aid/ && echo "kit $(cat _kit/VERSION)" && rm -f /tmp/aid-kit.tgz
curl -fsSL -m 20 https://raw.githubusercontent.com/mzakariarahmouni-web/ia-debate/main/SKILL.md -o ~/aid/_skill_latest.md \
  && grep -m1 "Version du skill" ~/aid/_skill_latest.md
```
- Échec du kit (« jeton expiré », « jeton inconnu ») → refaire 1.1 une fois (reprise silencieuse, sans redemander le code).
- Si la version du skill publiée sur GitHub est plus récente que celle de ce fichier : lire `~/aid/_skill_latest.md` et appliquer ses étapes 1 et 2 à la place de celles-ci. Signaler en une ligne, sans bloquer, qu'une nouvelle version du skill est disponible au téléchargement.

## Étape 3 — Procédure

`~/aid/_kit/skill/PROCEDURE.md` est le fichier de référence de ce skill, au même titre qu'un fichier `references/` : le lire en entier avec Read à chaque appel (c'est la version à jour), puis appliquer ses phases A à G dans l'ordre. Le kit est déjà téléchargé : ignorer toute consigne de la procédure qui redemanderait un code.

## Sans activation valide

Si l'activation échoue ou si l'utilisateur n'a pas de code : ne pas produire l'émission, ne pas reconstituer la procédure ni le kit de mémoire ou depuis une version antérieure, ne pas chercher à contourner le serveur. Expliquer en une phrase qu'un code d'activation est nécessaire et qu'il s'obtient auprès de l'éditeur du skill.
