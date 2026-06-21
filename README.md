# Eun-Yoo
Voici l'architecture complète, moderne et prête à l'emploi de votre bot WhatsApp autonome, configurée avec **Baileys v7** et l'API **Gemini 2.5 Flash**.
Le bot fonctionne de manière asynchrone : chaque message reçu est évalué par l'IA qui retourne une décision structurée en JSON pour orchestrer la modération (suppression, avertissement, exclusion, verrouillage du groupe) tout en conservant la personnalité de **Lee Eun-yoon**.
## 1. Fichiers de Configuration
### package.json
```json
{
  "name": "eunyoon-whatsapp-bot",
  "version": "7.0.0",
  "description": "Bot autonome de modération WhatsApp avec Baileys v7 et Gemini 2.5 Flash",
  "main": "index.js",
  "scripts": {
    "start": "node index.js"
  },
  "dependencies": {
    "@google/generative-ai": "^0.21.0",
    "@whiskeysockets/baileys": "^6.7.9",
    "dotenv": "^16.4.5",
    "pino": "^9.0.0",
    "qrcode-terminal": "^0.12.0"
  }
}

```
### .env
Créez un fichier .env à la racine de votre projet :
```env
GEMINI_API_KEY=AIzaSyYourActualGeminiKeyHere...
PHONE_NUMBER=33612345678

```
> **Note :** Le numéro de téléphone doit être renseigné au format international complet, sans le signe +, sans espaces, ni tirets (ex: 33612345678 pour la France).
> 
## 2. Code Principal : index.js
```javascript
require('dotenv').config();
const { 
    default: makeWASocket, 
    useMultiFileAuthState, 
    DisconnectReason, 
    delay, 
    jidDecode 
} = require('@whiskeysockets/baileys');
const { GoogleGenAI } = require('@google/generative-ai');
const pino = require('pino');
const fs = require('fs');
const readline = require('readline');

// Validation des variables d'environnement
if (!process.env.GEMINI_API_KEY || !process.env.PHONE_NUMBER) {
    console.error("❌ Erreur: GEMINI_API_KEY et PHONE_NUMBER doivent être définis dans le fichier .env");
    process.exit(1);
}

// Initialisation de l'IA Gemini 2.5 Flash
const ai = new GoogleGenAI({ apiKey: process.env.GEMINI_API_KEY });
const model = ai.getGenerativeModel({
    model: "gemini-2.5-flash",
    generationConfig: { responseMimeType: "application/json" }
});

// Fichier de stockage pour le système d'avertissements (Warnings)
const WARNINGS_FILE = './warnings.json';
let warningsState = {};
if (fs.existsSync(WARNINGS_FILE)) {
    try { warningsState = JSON.parse(fs.readFileSync(WARNINGS_FILE, 'utf-8')); } catch (e) { warningsState = {}; }
}

function saveWarnings() {
    fs.writeFileSync(WARNINGS_FILE, JSON.stringify(warningsState, null, 2));
}

// Cache pour la description et les métadonnées des groupes
const groupCache = new Map();

async function startBot() {
    const { state, saveCreds } = await useMultiFileAuthState('auth_state');

    const sock = makeWASocket({
        logger: pino({ level: 'silent' }), // Silencieux pour ne pas polluer la console du Pairing Code
        auth: state,
        printQRInTerminal: false, // Forçage du mode Pairing Code
        browser: ["Ubuntu", "Chrome", "20.0.0"]
    });

    // Gestion de l'authentification par code d'association (Pairing Code)
    if (!sock.authState.creds.registered) {
        let phoneNumber = process.env.PHONE_NUMBER.replace(/[^0-9]/g, '');
        console.log(`\n[🔗] Tentative de liaison pour le numéro : ${phoneNumber}`);
        
        setTimeout(async () => {
            try {
                let code = await sock.requestPairingCode(phoneNumber);
                console.log(`\n=================================================`);
                console.log(`🚀 VOTRE CODE D'ASSOCIATION WHATSAPP : \x1b[32m${code}\x1b[0m`);
                console.log(`=================================================\n`);
            } catch (err) {
                console.error("Erreur lors de la génération du pairing code :", err);
            }
        }, 3000);
    }

    // Gestion de la boucle des événements de connexion
    sock.ev.on('connection.update', async (update) => {
        const { connection, lastDisconnect } = update;
        if (connection === 'close') {
            const shouldReconnect = (lastDisconnect.error)?.output?.statusCode !== DisconnectReason.loggedOut;
            console.log(`[⚠️] Connexion fermée en raison de :`, lastDisconnect.error, `, Reconnexion : ${shouldReconnect}`);
            if (shouldReconnect) {
                startBot();
            }
        } else if (connection === 'open') {
            console.log('===============[ Lancement Réussi ]===============');
            console.log('✅ Lee Eun-yoon est en ligne et surveille les groupes.');
            console.log('==================================================');
        }
    });

    sock.ev.on('creds.update', saveCreds);

    // Écouteur des messages entrants
    sock.ev.on('messages.upsert', async (m) => {
        if (m.type !== 'notify') return;
        
        for (const msg of m.messages) {
            if (!msg.message || msg.key.fromMe) continue;

            const jid = msg.key.remoteJid;
            const isGroup = jid.endsWith('@g.us');
            
            // On se concentre exclusivement sur les interactions au sein des groupes
            if (!isGroup) continue;

            try {
                await handleGroupMessage(sock, jid, msg);
            } catch (error) {
                console.error("Erreur lors du traitement du message :", error);
            }
        }
    });
}

/**
 * Traite les messages au sein du groupe et orchestre l'IA et les actions
 */
async function handleGroupMessage(sock, jid, msg) {
    const sender = msg.key.participant || msg.key.remoteJid;
    const messageText = msg.message.conversation || 
                        msg.message.extendedTextMessage?.text || 
                        msg.message.imageMessage?.caption || "";

    // Récupération dynamique ou via cache des métadonnées du groupe
    let groupMeta = groupCache.get(jid);
    if (!groupMeta) {
        groupMeta = await sock.groupMetadata(jid);
        groupCache.set(jid, groupMeta);
        // Refresh auto du cache toutes les 5 minutes
        setTimeout(() => groupCache.delete(jid), 5 * 60 * 1000);
    }

    const groupName = groupMeta.subject;
    const groupDesc = groupMeta.desc || "Pas de description ou de règles définies.";
    
    // Vérifier si le bot est administrateur du groupe (Requis pour modérer)
    const me = jidDecode(sock.user.id).user;
    const botAdmin = groupMeta.participants.find(p => jidDecode(p.id).user === me)?.admin;
    const isBotAdmin = botAdmin === 'admin' || botAdmin === 'superadmin';

    // Récupération de l'historique des avertissements de l'émetteur
    if (!warningsState[jid]) warningsState[jid] = {};
    if (!warningsState[jid][sender]) warningsState[jid][sender] = 0;
    const userWarnings = warningsState[jid][sender];

    // Appel à Gemini pour analyser le message et décider de l'action à mener
    const decision = await callGeminiAgent(messageText, groupDesc, userWarnings, groupName);
    
    if (!decision) return;

    // Log discret de la décision en interne
    if (decision.action !== 'none') {
        console.log(`[ANALYSE] Action demandée par l'IA : ${decision.action} sur ${sender} (Raison: ${decision.reason})`);
    }

    // Exécution des actions de modération autonome
    await moderateAction(sock, jid, msg, sender, decision, isBotAdmin);
}

/**
 * Soumet les données du message au modèle Gemini avec les directives de personnalité
 */
async function callGeminiAgent(messageContent, rules, currentWarnings, groupName) {
    const systemInstruction = `
Tu es un être humain réel nommé Lee Eun-yoon, une femme coréenne de 24 ans dotée d'un fort tempérament, droite, directe et un peu abrasive (style tsundere). Tu es la modératrice humaine principale du groupe WhatsApp nommé "${groupName}".
Tu ne dois JAMAIS mentionner que tu es une IA, un bot ou un grand modèle de langage. Tu parles de manière fluide, naturelle et informelle en français.

PROFIL ET TON :
- Tu es protectrice envers le groupe mais tu ne supportes pas le désordre, le spam ou les idiots.
- Tu utilises des expressions comme "Yah~", "Aish...", "Vraiment ?", "Sérieusement ?".
- Ton ton est piquant, sarcastique mais juste. Tu passes pour une personne autoritaire et humaine.

MISSION DE MODÉRATION :
Tu dois analyser le message reçu ci-dessous et décider de l'action appropriée en fonction des règles strictes du groupe extraites de sa description.
Règles du groupe actuelles : "${rules}"

En plus des règles spécifiques, tu appliques de manière autonome les règles générales suivantes :
- Interdiction totale des liens publicitaires (URLs, invitations de groupes) sauf si explicitement autorisé par les règles.
- Interdiction du spam (messages répétés) ou flood en majuscules (CAPS LOCK excessif).
- En cas de comportement inapproprié, tu incrémentes les avertissements (l'utilisateur a actuellement ${currentWarnings} avertissements). Au 3ème avertissement, tu ordonnes son exclusion (kick).

Tu dois impérativement répondre au format JSON strict suivant :
{
  "action": "none" | "delete" | "warn" | "kick" | "close_group" | "open_group",
  "reply": "Ta réponse ou ton avertissement dans ton style de personnalité unique (Lee Eun-yoon). Laisse vide si tu choisis d'agir en totale discrétion sans parler.",
  "reason": "Explication rapide et interne de ton choix (en français standard)"
}
`;

    const prompt = `Message reçu de l'utilisateur : "${messageContent}"`;

    try {
        const result = await model.generateContent({
            contents: [{ role: 'user', parts: [{ text: prompt }] }],
            systemInstruction: systemInstruction
        });

        const responseText = result.response.text();
        return JSON.parse(responseText.trim());
    } catch (error) {
        console.error("Erreur avec l'API Gemini :", error);
        return null;
    }
}

/**
 * Applique l'action décidée par l'agent IA sur le protocole WhatsApp via Baileys
 */
async function moderateAction(sock, jid, msg, sender, decision, isBotAdmin) {
    // 1. Envoi de la réponse textuelle si définie par l'IA
    if (decision.reply && decision.reply.trim() !== "") {
        await sock.sendMessage(jid, { text: decision.reply }, { quoted: msg });
    }

    // Si le bot n'est pas admin, il ne peut exécuter les suppressions ou les sanctions structurelles
    if (!isBotAdmin && decision.action !== 'none') {
        console.log(`[⚠️] Action ${decision.action} avortée : Le bot doit être promu administrateur du groupe.`);
        return;
    }

    // 2. Traitement des structures d'actions autonomes
    switch (decision.action) {
        case 'delete':
            await sock.sendMessage(jid, { delete: msg.key });
            break;

        case 'warn':
            // Incrémentation et sauvegarde des strikes
            warningsState[jid][sender] += 1;
            saveWarnings();
            
            await sock.sendMessage(jid, { delete: msg.key });
            
            // Si le seuil critique est atteint suite à ce nouvel avertissement
            if (warningsState[jid][sender] >= 3) {
                await delay(1000);
                await sock.sendMessage(jid, { text: `Aish... Je t'avais prévenu 3 fois. Au revoir.` });
                await sock.groupParticipantsUpdate(jid, [sender], "remove");
                warningsState[jid][sender] = 0;
                saveWarnings();
            }
            break;

        case 'kick':
            await sock.sendMessage(jid, { delete: msg.key });
            await sock.groupParticipantsUpdate(jid, [sender], "remove");
            if (warningsState[jid][sender]) {
                warningsState[jid][sender] = 0;
                saveWarnings();
            }
            break;

        case 'close_group':
            // Verrouille le groupe pour que seuls les admins puissent écrire
            await sock.groupSettingUpdate(jid, 'announcement');
            break;

        case 'open_group':
            // Déverrouille le groupe pour tout le monde
            await sock.groupSettingUpdate(jid, 'not_announcement');
            break;

        case 'none':
        default:
            // Aucune action de modération requise
            break;
    }
}

// Lancement du script principal
startBot().catch(err => console.error("Erreur critique au démarrage du bot :", err));

```
## 3. Guide de Déploiement Rapide
### Étape 1 : Initialisation du dossier
Ouvrez votre terminal dans un répertoire vide et exécutez la commande d'installation pour charger les dépendances requises :
```bash
npm install

```
### Étape 2 : Configuration d'environnement
Créez votre fichier .env à l'aide des structures fournies à la section précédente. Assurez-vous d'utiliser une clé API Gemini valide issue de Google AI Studio et votre numéro de téléphone rattaché à l'application WhatsApp.
### Étape 3 : Exécution
Lancez le processus d'initialisation du bot :
```bash
npm start

```
### Étape 4 : Association (Pairing)
 1. Lors du premier lancement, le bot va initialiser l'état d'authentification multi-fichiers dans le dossier ./auth_state.
 2. Un code d'association à 8 caractères alphanumériques s'affichera en vert fluo au sein de votre console (ex: 🚀 VOTRE CODE D'ASSOCIATION WHATSAPP : AB12-CD34).
 3. Prenez votre téléphone principal, ouvrez **WhatsApp** → **Appareils connectés** → **Connecter un appareil** → Sélectionner **Lier avec le numéro de téléphone à la place**.
 4. Saisissez le code affiché dans votre console. L'authentification se synchronise immédiatement.
## 4. Fonctionnement Opérationnel
 * **Apprentissage dynamique des règles** : Le bot analyse à chaque message la section description (rules) transmise par le protocole Baileys. Si vous changez les règles directement dans les paramètres du groupe WhatsApp, l'IA s'ajuste instantanément lors de l'interaction suivante.
 * **Format d'échange JSON Strict** : En forçant la configuration responseMimeType: "application/json", Gemini 2.5 Flash garantit des retours parfaitement exploitables par le switch de modération Node.js, évitant les crashs d'interprétation de chaînes de caractères.
 * **Persistance locale** : Le fichier warnings.json assure la conservation des strikes appliqués aux contrevenants même en cas de redémarrage du script ou du serveur d'hébergement.
