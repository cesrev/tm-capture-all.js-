# tm-capture-all.js-
 * TICKETMASTER - CAPTURE TOUTES LES REQUÊTES  * Capture TOUT le trafic réseau vers un fichier
/**
 *
 * Usage: node tm-capture-all.js "URL"
 */

const { chromium } = require('playwright');
const fs = require('fs');
const path = require('path');

const PROFILE_DIR = path.join(require('os').homedir(), '.ticketmaster_profile');
const LOG_FILE = './tm_requests.log';

// Vider le fichier log
fs.writeFileSync(LOG_FILE, `=== CAPTURE DÉMARRÉE ${new Date().toISOString()} ===\n\n`);

function log(msg) {
  const line = `[${new Date().toISOString()}] ${msg}\n`;
  console.log(msg);
  fs.appendFileSync(LOG_FILE, line);
}

async function main() {
  const url = process.argv[2] || 'https://www.ticketmaster.fr/fr/manifestation/gims-billet/idmanif/639365';

  log(`URL: ${url}`);
  log('');
  log('=== INSTRUCTIONS ===');
  log('1. Ajoute des billets au panier');
  log('2. Clique sur Réserver');
  log('3. Toutes les requêtes sont loguées dans: ' + LOG_FILE);
  log('4. Ferme le navigateur quand terminé');
  log('====================');
  log('');

  // Supprimer le lock si existe
  const lockFile = path.join(PROFILE_DIR, 'SingletonLock');
  if (fs.existsSync(lockFile)) {
    fs.unlinkSync(lockFile);
    log('Lock supprimé');
  }

  const browser = await chromium.launchPersistentContext(PROFILE_DIR, {
    headless: false,
    viewport: null,
    locale: 'fr-FR',
    args: ['--disable-blink-features=AutomationControlled', '--start-maximized'],
  });

  const page = browser.pages()[0] || await browser.newPage();

  // Compteur de requêtes
  let reqCount = 0;

  // CAPTURER TOUTES LES REQUÊTES
  page.on('request', async (request) => {
    const url = request.url();
    const method = request.method();

    // Filtrer seulement Ticketmaster
    if (!url.includes('ticketmaster.fr')) return;

    reqCount++;
    const postData = request.postData();

    log(`\n>>> [${reqCount}] ${method} ${url}`);

    if (postData) {
      log(`    BODY: ${postData.substring(0, 500)}`);

      // Si c'est une requête importante, sauvegarder complètement
      if (url.includes('api') || url.includes('epsf') || url.includes('purchase') || url.includes('cart') || url.includes('basket') || url.includes('order')) {
        const filename = `./captures/req_${reqCount}_${method}_${Date.now()}.json`;
        if (!fs.existsSync('./captures')) fs.mkdirSync('./captures');
        fs.writeFileSync(filename, JSON.stringify({
          timestamp: new Date().toISOString(),
          method,
          url,
          headers: request.headers(),
          postData,
        }, null, 2));
        log(`    SAVED: ${filename}`);
      }
    }
  });

  // CAPTURER TOUTES LES RÉPONSES
  page.on('response', async (response) => {
    const url = response.url();
    const status = response.status();

    // Filtrer seulement Ticketmaster API
    if (!url.includes('ticketmaster.fr')) return;
    if (!url.includes('api') && !url.includes('epsf')) return;

    try {
      const body = await response.text();
      log(`<<< [${status}] ${url.substring(0, 80)}`);

      if (body && body.length > 0 && body.length < 5000) {
        log(`    RESPONSE: ${body.substring(0, 300)}`);
      }

      // Sauvegarder les réponses API
      if (url.includes('api') || url.includes('purchase') || url.includes('cart') || url.includes('basket')) {
        const filename = `./captures/resp_${status}_${Date.now()}.json`;
        if (!fs.existsSync('./captures')) fs.mkdirSync('./captures');
        fs.writeFileSync(filename, JSON.stringify({
          timestamp: new Date().toISOString(),
          status,
          url,
          headers: response.headers(),
          body: body.substring(0, 10000),
        }, null, 2));
      }
    } catch (e) {
      // Réponse non-texte (image, etc.)
    }
  });

  // Navigation
  log('Navigation...');
  await page.goto(url, { waitUntil: 'domcontentloaded' });
  log('Page chargée - AJOUTE DES BILLETS MAINTENANT!');

  // Attendre fermeture du navigateur
  await new Promise(resolve => browser.on('close', resolve));

  log('');
  log('=== CAPTURE TERMINÉE ===');
  log(`Total requêtes Ticketmaster: ${reqCount}`);
  log(`Log complet: ${LOG_FILE}`);
  log(`Captures: ./captures/`);
}

main().catch(e => {
  log(`ERREUR: ${e.message}`);
  process.exit(1);
});
