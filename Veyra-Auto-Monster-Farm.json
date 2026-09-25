// ==UserScript==
// @name         Veyra - PvE Target Catalogue
// @namespace    https://github.com/sideG1030/
// @version      1.4.7
// @description  Auto-catalogues PvE monsters, predicts damage/stamina, and directly attacks all selected targets to their configured damage thresholds.
// @homepageURL   https://github.com/sideG1030/Veyra-Auto-Monster-Farm
// @updateURL     https://raw.githubusercontent.com/sideG1030/Veyra-Auto-Monster-Farm/main/Veyra-Auto-Monster-Farm.user.js
// @downloadURL   https://raw.githubusercontent.com/sideG1030/Veyra-Auto-Monster-Farm/main/Veyra-Auto-Monster-Farm.user.js
// @author       sideG
// @match        https://demonicscans.org/guild_dash.php*
// @match        https://demonicscans.org/guild_dungeon.php*
// @match        https://demonicscans.org/guild_dungeon_instance.php*
// @match        https://demonicscans.org/guild_dungeon_location.php*
// @match        https://demonicscans.org/gates.php*
// @match        https://demonicscans.org/active_wave.php*
// @match        https://demonicscans.org/olympus.php*
// @match        https://demonicscans.org/battle.php*
// @grant        none
// @run-at       document-idle
// ==/UserScript==

(function () {
    'use strict';

    // Hidden battle frames get a tiny worker instead of the full manager.
    // The worker executes clicks from INSIDE the iframe's own context, which is
    // much more reliable than asking the top window to click a foreign
    // document's button. It never creates UI or another iframe.
    if (window.top !== window.self) {
        const WORKER_MSG = 'veyra-pve-battle-worker-v1';

        function workerNum(raw) {
            const n = Number(String(raw ?? '').replace(/[^0-9.-]/g, ''));
            return Number.isFinite(n) ? n : 0;
        }

        function workerSnapshot() {
            const dmg = workerNum(
                document.querySelector('#yourDamageValue')?.textContent || '0'
            );

            const staminaSpan = document.querySelector('#stamina_span');
            let stamina = 0;
            let maxStamina = 0;

            if (staminaSpan) {
                stamina = workerNum(staminaSpan.textContent);
                const pair = (staminaSpan.parentElement?.textContent || '')
                    .match(/([\d,]+)\s*\/\s*([\d,]+)/);
                if (pair) {
                    stamina = workerNum(pair[1]);
                    maxStamina = workerNum(pair[2]);
                }
            }

            const monsterHpText = document.querySelector('#hpText')?.textContent || '';
            const monsterHpPair = monsterHpText.match(/([\d,]+)\s*\/\s*([\d,]+)/);

            const playerHpText = document.querySelector('#pHpText')?.textContent || '';
            const playerHpPair = playerHpText.match(/([\d,]+)\s*\/\s*([\d,]+)/);

            return {
                damage: dmg,
                stamina,
                maxStamina,
                monsterHp: monsterHpPair ? workerNum(monsterHpPair[1]) : null,
                monsterMaxHp: monsterHpPair ? workerNum(monsterHpPair[2]) : null,
                playerHp: playerHpPair ? workerNum(playerHpPair[1]) : null,
                playerMaxHp: playerHpPair ? workerNum(playerHpPair[2]) : null,
                hasAttackButtons: !!document.querySelector('.attack-btn'),
                hasJoinButton: !!document.querySelector('#join-battle'),
                notification:
                    (document.querySelector('#notification')?.textContent || '')
                        .replace(/\s+/g, ' ')
                        .trim()
            };
        }

        function workerReply(id, payload = {}) {
            try {
                window.parent.postMessage({
                    type: WORKER_MSG,
                    id,
                    ...payload
                }, location.origin);
            } catch {}
        }

        window.addEventListener('message', event => {
            if (event.origin !== location.origin) return;
            if (event.source !== window.parent) return;

            const msg = event.data || {};
            if (msg.type !== WORKER_MSG || !msg.id) return;

            if (msg.action === 'snapshot') {
                workerReply(msg.id, {
                    ok: true,
                    action: 'snapshot',
                    snapshot: workerSnapshot()
                });
                return;
            }

            if (msg.action === 'join') {
                const button = document.querySelector('#join-battle');

                if (!button) {
                    workerReply(msg.id, {
                        ok: !!document.querySelector('.attack-btn'),
                        action: 'join',
                        reason: document.querySelector('.attack-btn')
                            ? 'already joined'
                            : 'join button not found',
                        snapshot: workerSnapshot()
                    });
                    return;
                }

                workerReply(msg.id, {
                    ok: true,
                    action: 'join-clicked',
                    snapshot: workerSnapshot()
                });

                // Execute the game's own registered click handler from inside
                // the battle frame.
                button.click();
                return;
            }

            if (msg.action === 'attack') {
                const skillId = String(msg.skillId);
                const button = document.querySelector(
                    `.attack-btn[data-skill-id="${skillId}"]`
                );

                if (!button) {
                    workerReply(msg.id, {
                        ok: false,
                        action: 'attack',
                        reason: `skill button ${skillId} not found`,
                        snapshot: workerSnapshot()
                    });
                    return;
                }

                if (button.disabled) {
                    workerReply(msg.id, {
                        ok: false,
                        action: 'attack',
                        reason: 'button disabled',
                        snapshot: workerSnapshot()
                    });
                    return;
                }

                const before = workerSnapshot();

                workerReply(msg.id, {
                    ok: true,
                    action: 'attack-clicked',
                    before
                });

                // Native game handler performs the actual request.
                button.click();
                return;
            }
        });

        const announceReady = () => {
            try {
                window.parent.postMessage({
                    type: WORKER_MSG,
                    action: 'ready',
                    snapshot: workerSnapshot()
                }, location.origin);
            } catch {}
        };

        if (document.readyState === 'loading') {
            document.addEventListener('DOMContentLoaded', () => {
                setTimeout(announceReady, 80);
            }, { once: true });
        } else {
            setTimeout(announceReady, 80);
        }

        return;
    }

    /* ============================================================
       CONFIG
    ============================================================ */

    const BASE_URL = 'https://demonicscans.org';

    // Guild dungeons
    const DUNGEONS_URL = `${BASE_URL}/guild_dungeon.php`;
    const DUNGEON_INSTANCE_URL = `${BASE_URL}/guild_dungeon_instance.php`;
    const DUNGEON_LOCATION_URL = `${BASE_URL}/guild_dungeon_location.php`;

    // Gates
    const GATES_URL = `${BASE_URL}/gates.php`;
    const OLYMPUS_MAP_URL = `${BASE_URL}/olympus.php`;

    // Cache / UI
    const STORAGE_CATALOGUE = 'veyra_pve_catalogue_v5';
    const STORAGE_TARGETS = 'veyra_pve_targets_v5';
    const STORAGE_LAST_SCAN = 'veyra_pve_last_scan_v5';
    const STORAGE_PANEL_MINIMIZED = 'veyra_pve_panel_minimized_v1';
    const STORAGE_ATTACK_AUTOMATION = 'veyra_pve_attack_automation_v1';

    const PANEL_ID = 'veyra-pve-target-manager';

    // Catalogue structure can change more often than loot thresholds.
    const CATALOGUE_REFRESH_MS = 10 * 60 * 1000;       // 10 min
    const LOOT_THRESHOLD_REFRESH_MS = 24 * 60 * 60 * 1000; // 24 h

    // Be polite to the site.
    const REQUEST_DELAY_MS = 100;
    const LOOT_REQUEST_DELAY_MS = 140;

    // Basic Slash ladder confirmed from the battle page.
    const SLASH_ATTACKS = [
        { cost: 1000, skillId: -5, name: 'World Breaker Slash' },
        { cost: 200,  skillId: -4, name: 'Legendary Slash' },
        { cost: 100,  skillId: -3, name: 'Ultimate Slash' },
        { cost: 50,   skillId: -2, name: 'Heroic Slash' },
        { cost: 10,   skillId: -1, name: 'Power Slash' },
        { cost: 1,    skillId: 0,  name: 'Slash' }
    ];

    const ATTACK_POST_CLICK_DELAY_MS = 650;
    const ATTACK_PROGRESS_TIMEOUT_MS = 6000;

    // TEMPORARY TEST MODE: only join Triton, Herald of the Deep Crown.
    const TEST_TRITON_NAME_KEY = 'triton herald of the deep crown';
    const TEST_TRITON_DAMAGE_URL = 'https://demonicscans.org/damage.php';

    function getCurrentTritonTargetDamage() {
        const catalogue = loadCatalogue();
        const targets = loadTargets();

        for (const [gateKey, gate] of Object.entries(catalogue.gates || {})) {
            for (const [sectionKey, section] of Object.entries(gate.sections || {})) {
                const monster = section.monsters?.[TEST_TRITON_NAME_KEY];
                if (!monster) continue;

                const manual = getManualDamageTarget(
                    targets,
                    TEST_TRITON_NAME_KEY
                );

                const targetDamage =
                    manual !== null
                        ? manual
                        : (Number(monster.autoDamageTarget) || null);

                if (!targetDamage) return null;

                return {
                    gateKey,
                    sectionKey,
                    monster,
                    targetDamage
                };
            }
        }

        return null;
    }

    function getCurrentTritonBattleUrl() {
        // 1) Best source: the catalogue already tracks the current timed-boss
        // battle URL and refreshes it when the spawn changes.
        try {
            const catalogue = loadCatalogue();

            for (const gate of Object.values(catalogue.gates || {})) {
                for (const section of Object.values(gate.sections || {})) {
                    const monster = section.monsters?.[TEST_TRITON_NAME_KEY];

                    if (
                        monster &&
                        monster.currentlyAlive !== false &&
                        monster.representativeBattleUrl
                    ) {
                        return monster.representativeBattleUrl;
                    }
                }
            }
        } catch {}

        // 2) If the user is currently looking at Triton, use this exact page.
        if (
            location.pathname.toLowerCase() === '/battle.php' &&
            normalizeKey(document.body?.innerText || '')
                .includes(TEST_TRITON_NAME_KEY)
        ) {
            return location.href;
        }

        return '';
    }

    // Polyhedral Crucible still needs a fallback because its layout differs.
    const POLYHEDRAL_FALLBACK_LOCATIONS = [
        { id: '11', name: 'Gate Prism' },
        { id: '12', name: 'Ash Lane' },
        { id: '13', name: 'Crown Lens' },
        { id: '14', name: 'Zenith Lock' }
    ];

    // Grakthar fallback waves. Dynamic wave discovery is still attempted first.
    const GRAKTHAR_FALLBACK_WAVES = [
        { gateId: '3', waveId: '3', name: 'Wave 1' },
        { gateId: '3', waveId: '5', name: 'Wave 2' },
        { gateId: '3', waveId: '8', name: 'Wave 3' }
    ];

    let scanning = false;
    let statusElement = null;
    let resultsElement = null;
    let logElement = null;
    let refreshButton = null;
    let attackButton = null;
    let attackLoopRunning = false;
    let headlessBattleFrame = null;

    /* ============================================================
       HELPERS
    ============================================================ */

    function sleep(ms) {
        return new Promise(resolve => setTimeout(resolve, ms));
    }

    function cleanText(value) {
        return String(value ?? '').replace(/\s+/g, ' ').trim();
    }

    function normalizeKey(value) {
        return cleanText(value)
            .toLowerCase()
            .replace(/[’‘]/g, "'")
            .replace(/[^a-z0-9' -]+/g, '')
            .replace(/\s+/g, ' ')
            .trim();
    }

    function escapeHtml(value) {
        return String(value ?? '')
            .replace(/&/g, '&amp;')
            .replace(/</g, '&lt;')
            .replace(/>/g, '&gt;')
            .replace(/"/g, '&quot;')
            .replace(/'/g, '&#039;');
    }

    function absoluteUrl(rawUrl, base = BASE_URL) {
        try {
            return new URL(rawUrl, base).href;
        } catch {
            return '';
        }
    }

    async function fetchHtml(url) {
        const response = await fetch(url, {
            credentials: 'same-origin',
            cache: 'no-store'
        });

        if (!response.ok) {
            throw new Error(`HTTP ${response.status}`);
        }

        return await response.text();
    }

    function parseHtml(html) {
        return new DOMParser().parseFromString(html, 'text/html');
    }

    function cleanLocationName(name) {
        return cleanText(name)
            .replace(/\s*[—–-]\s*\d+\s*\/\s*\d+\s*$/i, '')
            .trim();
    }

    function timestampIso() {
        return new Date().toISOString();
    }

    /* ============================================================
       DAMAGE PARSING / FORMATTING
    ============================================================ */

    // Supports 3000000, 3,000,000, 3m, 3M, 2.5B, 1T.
    function parseDamageValue(rawValue) {
        if (typeof rawValue === 'number' && Number.isFinite(rawValue)) {
            return rawValue >= 0 ? Math.round(rawValue) : null;
        }

        let text = String(rawValue ?? '').trim().toUpperCase();
        if (!text) return null;

        text = text.replace(/[,_\s]/g, '');

        const match = text.match(/^(\d+(?:\.\d+)?)([KMBT])?$/);
        if (!match) return null;

        const base = Number(match[1]);
        if (!Number.isFinite(base)) return null;

        const multiplier = {
            K: 1e3,
            M: 1e6,
            B: 1e9,
            T: 1e12
        }[match[2]] || 1;

        const value = base * multiplier;
        if (!Number.isFinite(value) || value < 0) return null;

        return Math.round(value);
    }

    function trimDecimal(value, decimals = 3) {
        return Number(value.toFixed(decimals)).toString();
    }

    function formatDamageCompact(value) {
        const num = Number(value);

        if (!Number.isFinite(num) || num < 0) return '';

        if (num >= 1e12) return `${trimDecimal(num / 1e12)}T`;
        if (num >= 1e9) return `${trimDecimal(num / 1e9)}B`;
        if (num >= 1e6) return `${trimDecimal(num / 1e6)}M`;
        if (num >= 1e3) return `${trimDecimal(num / 1e3)}K`;

        return Math.round(num).toLocaleString('en-US');
    }

    /* ============================================================
       STORAGE
    ============================================================ */

    function emptyCatalogue() {
        return {
            updatedAt: null,
            dungeons: {},
            gates: {}
        };
    }

    function loadCatalogue() {
        try {
            const raw = localStorage.getItem(STORAGE_CATALOGUE);
            if (!raw) return emptyCatalogue();

            const parsed = JSON.parse(raw);
            if (!parsed || typeof parsed !== 'object') return emptyCatalogue();

            parsed.dungeons ||= {};
            parsed.gates ||= {};

            return parsed;
        } catch {
            return emptyCatalogue();
        }
    }

    function saveCatalogue(catalogue) {
        catalogue.updatedAt = timestampIso();
        localStorage.setItem(STORAGE_CATALOGUE, JSON.stringify(catalogue));
    }

    function emptyTargets() {
        return {
            selections: {},
            manualDamageTargets: {}
        };
    }

    function loadTargets() {
        try {
            const raw = localStorage.getItem(STORAGE_TARGETS);
            if (!raw) return emptyTargets();

            const parsed = JSON.parse(raw);

            return {
                selections: parsed.selections || {},
                manualDamageTargets:
                    parsed.manualDamageTargets ||
                    parsed.damageTargets ||
                    {}
            };
        } catch {
            return emptyTargets();
        }
    }

    function saveTargets(targets) {
        localStorage.setItem(STORAGE_TARGETS, JSON.stringify(targets));
    }

    /* ============================================================
       LOGGING / STATUS
    ============================================================ */

    function addLog(message) {
        console.log('[Veyra PvE Catalogue]', message);

        if (!logElement) return;

        const line = document.createElement('div');
        line.textContent = `[${new Date().toLocaleTimeString()}] ${message}`;

        logElement.appendChild(line);
        logElement.scrollTop = logElement.scrollHeight;

        // Keep the complete multi-target run visible. The old 100-line
        // cap clipped the first dungeon targets before the run reached
        // Olympus, which made dungeon execution appear absent from exports.
        while (logElement.children.length > 500) {
            logElement.firstElementChild?.remove();
        }
    }

    function setStatus(message, isError = false) {
        if (!statusElement) return;

        statusElement.textContent = message;
        statusElement.style.color = isError ? '#ff7d7d' : '#d9ca98';
    }

    /* ============================================================
       EMBEDDED PvE DAMAGE / STAMINA ESTIMATOR

       Based on Veyra PvE Damage Predictor v0.2.0.  The original
       predictor calculated stamina for a fixed 1B target.  Here the
       exact same model is generalized to an arbitrary damage target.
    ============================================================ */

    const DamageEstimator = (() => {
      const CFG = {
        CACHE_KEY: 'veyraDamagePredictor.v1',
        CACHE_MAX_AGE_MS: 10 * 60 * 1000,
        TARGET_DAMAGE: 1_000_000_000,

        // Confirmed from testing / battle logs.
        HEART_PVE_BONUS_PERCENT: 1.5,
        BASE_CRIT_RATE_PERCENT: 0,
        BASE_CRIT_MULTIPLIER: 1.5,

        // Current best empirical soft-cap approximation for Grondakar/Bastior/Cinderback-style gap passives.
        SOFTCAP_START: 3000,
        SOFTCAP_LOG_A: 6300,
        SOFTCAP_LOG_B: 5000,

        // Orryphos fallback values if the current pet text cannot be parsed.
        ORRYPHOS_PROC_PERCENT: 13.68,
        ORRYPHOS_ECHO_PERCENT: 40,

        // Dawn Brand: 25 stacks = +1/3 final PvE damage => 1/75 per stack.
        DAWN_MAX_STACKS: 25,
        DAWN_PER_STACK: 1 / 75,

        // Internal diagnostics.
        DEBUG: false,
      };

      const ELEMENTS = ['FIRE', 'NATURE', 'FROST', 'THUNDER', 'WATER', 'VOID', 'LIGHT'];
      const COUNTERS = {
        FIRE: ['NATURE', 'FROST'],
        NATURE: ['LIGHT', 'VOID'],
        FROST: ['WATER', 'THUNDER'],
        THUNDER: ['WATER', 'NATURE'],
        WATER: ['FIRE', 'LIGHT'],
        VOID: ['FIRE', 'THUNDER'],
        LIGHT: ['VOID', 'FROST'],
      };

      const log = (...args) => CFG.DEBUG && console.log('[Veyra Damage Predictor]', ...args);
      const warn = (...args) => console.warn('[Veyra Damage Predictor]', ...args);

      function num(value, fallback = 0) {
        if (typeof value === 'number') return Number.isFinite(value) ? value : fallback;
        const cleaned = String(value ?? '').replace(/,/g, '').replace(/%/g, '').trim();
        const n = parseFloat(cleaned);
        return Number.isFinite(n) ? n : fallback;
      }

      function int(value, fallback = 0) {
        const n = num(value, NaN);
        return Number.isFinite(n) ? Math.trunc(n) : fallback;
      }

      function fmt(n, decimals = 0) {
        if (!Number.isFinite(n)) return '—';
        return n.toLocaleString(undefined, {
          minimumFractionDigits: decimals,
          maximumFractionDigits: decimals,
        });
      }

      function clamp(v, lo, hi) {
        return Math.max(lo, Math.min(hi, v));
      }

      function normalizeName(s) {
        return String(s ?? '').trim().replace(/\s+/g, ' ').toLowerCase();
      }

      function fetchDoc(url) {
        return fetch(url, { credentials: 'same-origin', cache: 'no-store' })
          .then(r => {
            if (!r.ok) throw new Error(`${url}: HTTP ${r.status}`);
            return r.text();
          })
          .then(html => new DOMParser().parseFromString(html, 'text/html'));
      }

      function saveCache(state) {
        try {
          localStorage.setItem(CFG.CACHE_KEY, JSON.stringify({ savedAt: Date.now(), state }));
        } catch (e) {
          warn('Could not save cache', e);
        }
      }

      function loadCache() {
        try {
          const parsed = JSON.parse(localStorage.getItem(CFG.CACHE_KEY) || 'null');
          if (!parsed?.state) return null;
          return parsed;
        } catch {
          return null;
        }
      }

      function parseStats(doc) {
        const attack = int(doc.querySelector('#v-attack')?.textContent);
        const defense = int(doc.querySelector('#v-defense')?.textContent);
        const stamina = int(doc.querySelector('#v-stamina')?.textContent);
        if (!attack && !defense && !stamina) throw new Error('Could not parse character stats.');
        return { attack, defense, stamina };
      }

      function parseClass(doc) {
        // On a fetched HTML document the page scripts are not executed, so
        // window.SkillTreeConfig is unavailable. Read either the visible modal
        // or the literal classPassive value embedded in the page source.
        let passiveText = (
          doc.querySelector('#passive-body')?.textContent ||
          Array.from(doc.querySelectorAll('*')).find(el => /passive/i.test(el.id || '') && /attack|def/i.test(el.textContent || ''))?.textContent ||
          ''
        ).trim();

        let className = '';
        const scriptText = Array.from(doc.scripts).map(x => x.textContent || '').join('\n');
        if (!passiveText) {
          passiveText = (scriptText.match(/classPassive\s*:\s*["']([^"']+)["']/i)?.[1] || '').trim();
        }
        className = (scriptText.match(/className\s*:\s*["']([^"']+)["']/i)?.[1] || '').trim();

        let attackPercent = 0;
        let defensePercent = 0;

        const atk = passiveText.match(/([+-]?\d+(?:\.\d+)?)\s*%\s*attack/i);
        const def = passiveText.match(/([+-]?\d+(?:\.\d+)?)\s*%\s*def/i);
        if (atk) attackPercent = num(atk[1]);
        if (def) defensePercent = num(def[1]);

        return { className, passiveText, attackPercent, defensePercent };
      }

      function pveOnlyText(text) {
        const lines = String(text ?? '').split(/\r?\n/);
        const kept = [];

        for (let line of lines) {
          line = line.trim();
          if (!line) continue;

          // Entirely PvP-only lines are irrelevant.
          if (/^⚡?\s*pvp only\b/i.test(line) || /^pvp only\b/i.test(line)) continue;

          // Remove explicit PvP clauses from mixed descriptions where possible.
          line = line
            .replace(/\bEffect is doubled in PvP combat\.?/ig, '')
            .replace(/\bPvP:\s*[^.]+\.?/ig, '')
            .trim();

          if (line) kept.push(line);
        }
        return kept.join('\n');
      }

      function emptyEffects() {
        return {
          extraMonsterPct: 0,
          extraElementalMonsterPct: 0,
          equipmentPenPct: 0,
          petPenPct: 0,
          critRatePct: 0,
          critDamagePct: 0,
          charAttackPct: 0,
          charDefensePct: 0,
          elementRateIncreasePct: Object.fromEntries(ELEMENTS.map(e => [e, 0])),
          conditionalEquipmentPen: [],
          conditionalPetPen: [],
          gapPassives: [],
          orryphos: null,
          ourovyryn: false,
        };
      }

      function mergeEffects(target, source) {
        for (const key of ['extraMonsterPct', 'extraElementalMonsterPct', 'equipmentPenPct', 'petPenPct', 'critRatePct', 'critDamagePct', 'charAttackPct', 'charDefensePct']) {
          target[key] += source[key] || 0;
        }
        for (const e of ELEMENTS) {
          target.elementRateIncreasePct[e] += source.elementRateIncreasePct?.[e] || 0;
        }
        target.conditionalEquipmentPen.push(...(source.conditionalEquipmentPen || []));
        target.conditionalPetPen.push(...(source.conditionalPetPen || []));
        target.gapPassives.push(...(source.gapPassives || []));
        if (source.orryphos) target.orryphos = source.orryphos;
        if (source.ourovyryn) target.ourovyryn = true;
        return target;
      }

      function parseEffects(rawText, sourceName = '') {
        const out = emptyEffects();
        const text = pveOnlyText(rawText);
        const name = String(sourceName || '');

        // Generic damage bonuses.
        for (const m of text.matchAll(/(?<!Elemental\s)(\d+(?:\.\d+)?)\s*%\s*Extra Damage To Monsters/gi)) {
          out.extraMonsterPct += num(m[1]);
        }
        for (const m of text.matchAll(/(\d+(?:\.\d+)?)\s*%\s*Extra Elemental Damage To Monsters/gi)) {
          out.extraElementalMonsterPct += num(m[1]);
        }

        // Crit.
        for (const m of text.matchAll(/Increase\s+Critical\s+Rate\s+By\s+(\d+(?:\.\d+)?)\s*%/gi)) {
          out.critRatePct += num(m[1]);
        }
        for (const m of text.matchAll(/Increase\s+Critical\s+Damage\s+By\s+(\d+(?:\.\d+)?)\s*%/gi)) {
          out.critDamagePct += num(m[1]);
        }

        // Character ATK/DEF percentage modifiers.
        for (const m of text.matchAll(/Increases?\s+your\s+Attack\s+and\s+Defense\s+stats?\s+by\s+(\d+(?:\.\d+)?)\s*%/gi)) {
          out.charAttackPct += num(m[1]);
          out.charDefensePct += num(m[1]);
        }
        for (const m of text.matchAll(/Increase\s+Your\s+Attack(?:\s+Stat)?\s+by\s+(\d+(?:\.\d+)?)\s*%/gi)) {
          out.charAttackPct += num(m[1]);
        }

        // Defense penetration. Values sometimes appear as percentages and sometimes decimals.
        const parsePenValue = (s, hadPercent) => {
          const v = num(s);
          return hadPercent ? v : (v <= 1 ? v * 100 : v);
        };

        for (const m of text.matchAll(/(?:Increase\s+Your\s+)?(?:Equipment|Equipement)\s+Defense\s+Penetration\s+by\s+(\d+(?:\.\d+)?)(\s*%)?/gi)) {
          out.equipmentPenPct += parsePenValue(m[1], !!m[2]);
        }
        for (const m of text.matchAll(/Increase\s+Your\s+Armor\s+Penetration\s+by\s+(\d+(?:\.\d+)?)(\s*%)?/gi)) {
          out.equipmentPenPct += parsePenValue(m[1], !!m[2]);
        }
        for (const m of text.matchAll(/Pet\s+Defense\s+Penetration\s+by\s+(\d+(?:\.\d+)?)(\s*%)?/gi)) {
          out.petPenPct += parsePenValue(m[1], !!m[2]);
        }

        // Element-rate increases are multiplicative to that element's accumulated rate.
        for (const e of ELEMENTS) {
          const re = new RegExp(`(\\d+(?:\\.\\d+)?)\\s*%\\s*${e}\\s+Element\\s+Rate\\s+Increase`, 'gi');
          for (const m of text.matchAll(re)) out.elementRateIncreasePct[e] += num(m[1]);
        }

        // Conditional equipment penetration pets.
        for (const m of text.matchAll(/When\s+your\s+Attack\s+exceeds\s+the\s+enemy\s+Attack\s+and\s+your\s+Attack\s+is\s+higher\s+than(?:\s+or\s+equals)?\s+your\s+Defense,?\s+gain\s+(\d+(?:\.\d+)?)(\s*%)?\s+Equipment\s+Defense\s+Penetration/gi)) {
          out.conditionalEquipmentPen.push({
            kind: 'attackDominant',
            percent: parsePenValue(m[1], !!m[2]),
          });
        }
        for (const m of text.matchAll(/When\s+your\s+Defense\s+exceeds\s+the\s+enemy\s+Defense\s+and\s+your\s+Defense\s+is\s+higher\s+than\s+your\s+Attack,?\s+gain\s+(\d+(?:\.\d+)?)(\s*%)?\s+Equipment\s+Defense\s+Penetration/gi)) {
          out.conditionalEquipmentPen.push({
            kind: 'defenseDominant',
            percent: parsePenValue(m[1], !!m[2]),
          });
        }

        // Grondakar-style Attack-gap passive.
        for (const m of text.matchAll(/bonus\s+PvE\s+damage\s+of\s+up\s+to\s+(\d+(?:\.\d+)?)\s*x\s*\(Your\s+Attack\s*-\s*Enemy\s+Attack\)/gi)) {
          out.gapPassives.push({
            source: name,
            stat: 'attack',
            coefficient: num(m[1]),
            requireDominant: 'attack',
            pvpBuildScaling: true,
          });
        }

        // Cinderback/Bastior-style Defense-gap passive.
        for (const m of text.matchAll(/bonus\s+damage\s+equal\s+to\s+(\d+(?:\.\d+)?)\s*x\s*\(your\s+defense\s*-\s*enemy\s+defense\)/gi)) {
          out.gapPassives.push({
            source: name,
            stat: 'defense',
            coefficient: num(m[1]),
            requireDominant: null,
            pvpBuildScaling: false,
            halfIfStaminaHigherThanDefense: /stamina\s+is\s+higher\s+than\s+your\s+defense[\s\S]*decreased\s+by\s+50%/i.test(text),
          });
        }

        // More generic defense-gap phrasing (e.g. Bastior variants).
        for (const m of text.matchAll(/(?:up\s+to\s+)?(\d+(?:\.\d+)?)\s*x\s*\(Your\s+Defense\s*-\s*Enemy\s+Defense\)/gi)) {
          if (!out.gapPassives.some(p => p.stat === 'defense' && Math.abs(p.coefficient - num(m[1])) < 1e-9)) {
            out.gapPassives.push({
              source: name,
              stat: 'defense',
              coefficient: num(m[1]),
              requireDominant: 'defense',
              pvpBuildScaling: true,
            });
          }
        }

        // Orryphos.
        if (/Orryphos\s+the\s+Eternal\s+Echo\s+Dragon/i.test(name) || /attacks\s+won't\s+consume\s+stamina/i.test(text)) {
          const proc = text.match(/(\d+(?:\.\d+)?)\s*%\s*Your attacks won't consume stamina/i);
          const echo = text.match(/(\d+(?:\.\d+)?)\s*%\s*PvE echo damage/i);
          out.orryphos = {
            procPct: proc ? num(proc[1]) : CFG.ORRYPHOS_PROC_PERCENT,
            echoPct: echo ? num(echo[1]) : CFG.ORRYPHOS_ECHO_PERCENT,
          };
        }

        // Ourovyrn / Dawn Brand.
        if (/Ourovyrn\s+the\s+Dawn\s+Worldcoil/i.test(name) || /Dawn Brand/i.test(text)) {
          out.ourovyryn = true;
        }

        return out;
      }

      function parseEquipment(doc) {
        const lower = doc.querySelector('[data-section-key="inv-attack-main-equipment"]');
        const upper = doc.querySelector('[data-section-key="inv-attack-main-equipped"]');
        if (!lower || !upper) throw new Error('Could not find PvE equipment sections.');

        const owned = Array.from(lower.querySelectorAll('.slot-box[data-equip="1"]')).map(card => {
          const info = card.querySelector('.info-btn');
          const name = info?.dataset.name || card.querySelector('.equipment-item-name')?.textContent || card.querySelector('img')?.alt || '';
          const type = card.dataset.itemType || card.dataset.typee || '';
          const attack = num(card.dataset.atk, num(card.querySelector('[data-atk], .boosted-stat')?.textContent));
          const defense = num(card.dataset.def, 0);
          const desc = info?.dataset.desc || '';
          const element = parseElementFromEquipmentDesc(desc);
          return {
            invId: card.dataset.invId || '',
            name: name.trim(),
            type: type.trim().toLowerCase(),
            attack,
            defense,
            desc,
            element,
            effects: parseEffects(desc, name),
          };
        });

        const used = new Set();
        const equipped = [];

        for (const card of upper.querySelectorAll('.slot-box')) {
          const name = card.querySelector('img')?.alt?.trim() || '';
          const label = card.querySelector('.label')?.textContent || '';
          const type = (label.match(/^\s*([A-Za-z_-]+)/)?.[1] || '').trim().toLowerCase();
          const boosted = Array.from(card.querySelectorAll('.boosted-stat')).map(x => num(x.textContent));
          const attack = boosted[0] ?? 0;
          const defense = boosted[1] ?? 0;

          let idx = owned.findIndex((x, i) => !used.has(i) && normalizeName(x.name) === normalizeName(name) && x.type === type && x.attack === attack && x.defense === defense);
          if (idx < 0) idx = owned.findIndex((x, i) => !used.has(i) && normalizeName(x.name) === normalizeName(name) && x.attack === attack && x.defense === defense);
          if (idx < 0) idx = owned.findIndex((x, i) => !used.has(i) && normalizeName(x.name) === normalizeName(name));

          if (idx >= 0) {
            used.add(idx);
            equipped.push({ ...owned[idx], equipped: true });
          } else {
            // We can still use the final ATK/DEF even if description matching failed.
            equipped.push({
              invId: '', name, type, attack, defense, desc: '',
              element: { type: 'NONE', rate: 0 }, effects: emptyEffects(), equipped: true,
            });
            warn('Could not match equipped item to inventory description:', { name, type, attack, defense });
          }
        }

        return { owned, equipped };
      }

      function parseElementFromEquipmentDesc(desc) {
        const m = String(desc || '').match(/Element:\s*([A-Z]+)(?:\s*\(\+([\d.]+)%\))?/i);
        if (!m) return { type: 'NONE', rate: 0 };
        const type = m[1].toUpperCase();
        return { type, rate: num(m[2], 0) };
      }

      function findSectionByTitle(doc, titleRe) {
        return Array.from(doc.querySelectorAll('.section')).find(section => titleRe.test(section.querySelector('.section-title')?.textContent || '')) || null;
      }

      async function parsePets(doc) {
        const equippedSection = findSectionByTitle(doc, /PvE\s+Attack\s+Team/i);
        if (!equippedSection) throw new Error('Could not find PvE Attack pet team.');

        const cards = Array.from(equippedSection.querySelectorAll('.pet-card[data-pet-inv-id]'));
        const pets = [];

        for (const card of cards) {
          const invId = int(card.dataset.petInvId);
          const info = card.querySelector('.info-btn');
          const name = info?.dataset.name || card.querySelector('img')?.alt || '';
          const elementType = String(card.dataset.element || 'NONE').toUpperCase();
          // Pet page stores 5% as 0.05; damage formula uses literal 5.
          const elementRate = num(card.dataset.elementRate) * 100;

          let attack = num(card.querySelector('[data-attack]')?.textContent);
          let defense = num(card.querySelector('[data-defense]')?.textContent);
          let effectText = card.querySelector('[data-power]')?.textContent || info?.dataset.desc || '';
          let links = [];

          if (invId) {
            try {
              const res = await fetch(`/pet_links_ajax.php?pet_inv_id=${encodeURIComponent(invId)}`, { credentials: 'same-origin', cache: 'no-store' });
              const data = await res.json();
              if (data?.status === 'success' && data.pet) {
                attack = num(data.pet.updated_attack, attack);
                defense = num(data.pet.updated_defense, defense);
                effectText = data.pet.total_effect_text || effectText;
                links = data.links || [];
              }
            } catch (e) {
              warn(`Could not load links for ${name}`, e);
            }
          }

          pets.push({
            invId,
            name: name.trim(),
            attack,
            defense,
            element: { type: elementType, rate: elementRate },
            effectText,
            effects: parseEffects(effectText, name),
            links,
          });
        }

        return { equipped: pets };
      }

      function aggregateLoadout(equipment, pets) {
        const effects = emptyEffects();
        const elementRates = Object.fromEntries(ELEMENTS.map(e => [e, 0]));

        let EA = 0;
        let equipmentDefense = 0;
        for (const item of equipment.equipped) {
          EA += item.attack || 0;
          equipmentDefense += item.defense || 0;
          mergeEffects(effects, item.effects);
          if (ELEMENTS.includes(item.element?.type) && item.element.rate > 0) {
            elementRates[item.element.type] += item.element.rate;
          }
        }

        let PA = 0;
        let petDefense = 0;
        for (const pet of pets.equipped) {
          PA += pet.attack || 0;
          petDefense += pet.defense || 0;
          mergeEffects(effects, pet.effects);
          if (ELEMENTS.includes(pet.element?.type) && pet.element.rate > 0) {
            elementRates[pet.element.type] += pet.element.rate;
          }
        }

        // Apply element-rate-increase effects after summing the base rate of each element.
        for (const e of ELEMENTS) {
          elementRates[e] *= 1 + (effects.elementRateIncreasePct[e] || 0) / 100;
        }

        return { EA, PA, equipmentDefense, petDefense, effects, elementRates };
      }

      function parseMonster(doc = document) {
        const modal = doc.querySelector('#monsterStatsModal');
        if (!modal) throw new Error('Monster stats modal not found on this page.');

        const data = {};
        for (const cell of modal.querySelectorAll('.monster-stat-cell')) {
          const label = cell.querySelector('.label')?.textContent?.trim().toLowerCase();
          const value = cell.querySelector('strong')?.textContent?.trim() || '';
          if (label) data[label] = value;
        }

        const monsterCard = doc.querySelector('.monster-card');
        const title = modal.querySelector('h2')?.textContent?.replace(/\s+Stats\s*$/i, '').trim()
          || monsterCard?.querySelector('.card-title')?.textContent?.replace(/^\s*🧟\s*/, '').replace(/\bDEAD\b/i, '').trim()
          || 'Monster';

        // Current/max HP are useful for HP-dependent monster passives.
        let currentHp = 0;
        let maxHp = 0;
        const hpText = monsterCard?.querySelector('.card-sub')?.textContent || '';
        const hpMatch = hpText.match(/HP\s*([\d,]+)\s*\/\s*([\d,]+)/i);
        if (hpMatch) {
          currentHp = num(hpMatch[1]);
          maxHp = num(hpMatch[2]);
        }

        // Monster effects are displayed as chips below the monster image.
        const effectTexts = Array.from(monsterCard?.querySelectorAll('.chip') || [])
          .map(x => (x.textContent || '').replace(/\s+/g, ' ').trim())
          .filter(x => /🧠\s*Effect:/i.test(x) || /^Effect:/i.test(x));
        const effectText = effectTexts.join(' ');

        const monster = {
          name: title,
          attack: num(data['attack']),
          defense: num(data['defense']),
          baseDefense: num(data['defense']),
          petDefense: num(data['pet defense']),
          equipmentDefense: num(data['equipment defense']),
          element: String(data['element'] || 'NONE').toUpperCase(),
          elementRate: num(data['element rate']),
          critRateResistancePct: num(data['crit rate resistance']),
          critDamageResistancePct: num(data['crit damage resistance']),
          finalDamageResistancePct: num(data['final damage resistance']),
          passiveDamageResistancePct: num(data['passive damage resistance']),
          currentHp,
          maxHp,
          effectText,
          passives: {
            divineMultiplier: 1,
            hpDefenseScaling: null,
            moonMark: null,
          },
        };

        // Generic Divine Shield parser. Prefer the exact multiplier printed by
        // the fight page (e.g. 55.0%), because it already accounts for relics.
        const pageText = doc.body?.innerText || '';
        const divineMatch = pageText.match(/Damage Multiplier:\s*([\d.]+)%/i)
          || pageText.match(/multiplier\s*=\s*([\d.]+)%/i);
        if (divineMatch && /DIVINE SHIELD/i.test(pageText)) {
          monster.passives.divineMultiplier = clamp(num(divineMatch[1]) / 100, 0, 1);
        }

        // Dungeon passive: "Increases its defense by a percentage equals to half
        // of its missing health."  At 40% missing HP, Active DEF becomes +20%.
        if (/increases its defense by a percentage equals to half of its missing health/i.test(effectText)) {
          const missingFrac = maxHp > 0 ? clamp(1 - currentHp / maxHp, 0, 1) : 0;
          const increasePct = missingFrac * 50;
          monster.passives.hpDefenseScaling = {
            kind: 'halfMissingHealth',
            missingHealthPct: missingFrac * 100,
            defenseIncreasePct: increasePct,
          };
          // Wording says "defense", so apply this to Active DEF only.
          monster.defense = monster.baseDefense * (1 + increasePct / 100);
        }

        // Poseidon/Artemis-style mark. The page gives both the effective proc
        // chance and the damage reduction/duration, so parse those generically.
        const mark = effectText.match(/Moon Mark\s*([\d.]+)%\s*\(([\d.]+)%\s*after relic reduction\).*?Marked players deal\s*([\d.]+)%\s*less damage for\s*(\d+)\s*turns?/i);
        if (mark) {
          monster.passives.moonMark = {
            baseChancePct: num(mark[1]),
            chancePct: num(mark[2]),
            damageReductionPct: num(mark[3]),
            durationTurns: int(mark[4]),
          };
        } else {
          const mark2 = effectText.match(/Moon Mark\s*([\d.]+)%.*?Marked players deal\s*([\d.]+)%\s*less damage for\s*(\d+)\s*turns?/i);
          if (mark2) {
            monster.passives.moonMark = {
              baseChancePct: num(mark2[1]),
              chancePct: num(mark2[1]),
              damageReductionPct: num(mark2[2]),
              durationTurns: int(mark2[3]),
            };
          }
        }

        return monster;
      }

      function expectedMoonMarkMultiplier(monster) {
        const mark = monster.passives?.moonMark;
        if (!mark || mark.chancePct <= 0 || mark.durationTurns <= 0) return 1;

        const p = clamp(mark.chancePct / 100, 0, 1);
        const reducedMult = 1 - clamp(mark.damageReductionPct, 0, 100) / 100;

        // Mark is applied after an attack and affects the following N attacks.
        // Iterate the small Markov chain to its stationary distribution.
        let dist = Array(mark.durationTurns + 1).fill(0);
        dist[0] = 1;
        for (let iter = 0; iter < 500; iter++) {
          const next = Array(mark.durationTurns + 1).fill(0);
          for (let turns = 0; turns < dist.length; turns++) {
            const q = dist[turns] || 0;
            if (!q) continue;
            const afterAttack = Math.max(0, turns - 1);
            next[mark.durationTurns] += q * p;
            next[afterAttack] += q * (1 - p);
          }
          dist = next;
        }

        const unmarked = dist[0] || 0;
        return unmarked + (1 - unmarked) * reducedMult;
      }

      function chooseElement(elementRates, monsterElement) {
        const entries = ELEMENTS.map(e => ({ type: e, rate: elementRates[e] || 0 }));
        const maxRate = Math.max(0, ...entries.map(x => x.rate));
        if (maxRate <= 0) return { type: 'NONE', rate: 0, counter: false, usedRate: 0 };

        const tied = entries.filter(x => Math.abs(x.rate - maxRate) < 1e-9);
        const advantageous = tied.find(x => (COUNTERS[x.type] || []).includes(monsterElement));
        const chosen = advantageous || tied[0];
        const counter = (COUNTERS[chosen.type] || []).includes(monsterElement);
        return { ...chosen, counter, usedRate: chosen.rate * (counter ? 2 : 1) };
      }

      function gapSoftcap(gap) {
        if (gap <= CFG.SOFTCAP_START) return Math.max(0, gap);
        return CFG.SOFTCAP_START + CFG.SOFTCAP_LOG_A * Math.log(1 + (gap - CFG.SOFTCAP_START) / CFG.SOFTCAP_LOG_B);
      }

      function pvpBuildMultiplier(stats) {
        const total = stats.attack + stats.defense + stats.stamina;
        if (total <= 0) return 0.3;
        const rate = (stats.attack + stats.defense) / total;
        if (rate > 0.60) return 1.0;
        if (rate > 0.40) return 0.60;
        return 0.30;
      }

      function activeConditionalPen(effects, stats, monster) {
        let eq = 0;
        let pet = 0;

        const applies = c => {
          if (c.kind === 'attackDominant') return stats.attack > monster.attack && stats.attack >= stats.defense;
          if (c.kind === 'defenseDominant') return stats.defense > monster.defense && stats.defense > stats.attack;
          return false;
        };

        for (const c of effects.conditionalEquipmentPen || []) if (applies(c)) eq += c.percent;
        for (const c of effects.conditionalPetPen || []) if (applies(c)) pet += c.percent;
        return { equipmentPenPct: eq, petPenPct: pet };
      }

      function calculateOneStaminaCore(state, monster) {
        const { stats, classInfo, loadout } = state;
        const effects = loadout.effects;

        // Main character ATK uses class passive and PvE-relevant character ATK modifiers.
        const classAtkFactor = 1 + (classInfo.attackPercent || 0) / 100;
        const effectAtkFactor = 1 + (effects.charAttackPct || 0) / 100;
        const effectiveCharacterAttack = stats.attack * classAtkFactor * effectAtkFactor;

        const condPen = activeConditionalPen(effects, stats, monster);
        const equipmentPenPct = clamp((effects.equipmentPenPct || 0) + condPen.equipmentPenPct, 0, 100);
        const petPenPct = clamp((effects.petPenPct || 0) + condPen.petPenPct, 0, 100);

        const eqDef = monster.equipmentDefense * (1 - equipmentPenPct / 100);
        const petDef = monster.petDefense * (1 - petPenPct / 100);

        const eqTerm = Math.max(30, loadout.EA - eqDef) * 20;
        const petTerm = Math.max(30, loadout.PA - petDef) * 20;
        const charDelta = Math.max(30, effectiveCharacterAttack - monster.defense);
        const charTerm = 1000 * Math.pow(charDelta, 0.25);
        const physicalRaw = 225 + eqTerm + petTerm + charTerm;

        const chosenElement = chooseElement(loadout.elementRates, monster.element);
        const elementalRaw = Math.max(0, (chosenElement.usedRate - monster.elementRate) * 1500);
        const elementalAfterSpecific = elementalRaw * (1 + (effects.extraElementalMonsterPct || 0) / 100);

        // Special passive damage is kept separate so monster Passive Damage Resistance can apply.
        let passiveRaw = 0;
        const pvpMult = pvpBuildMultiplier(stats);
        const passiveDetails = [];

        for (const p of effects.gapPassives || []) {
          let eligible = true;
          let rawGap = 0;

          if (p.stat === 'attack') {
            rawGap = stats.attack - monster.attack;
            if (stats.attack <= monster.attack) eligible = false;
            if (p.requireDominant === 'attack' && stats.attack < stats.defense) eligible = false;
          } else if (p.stat === 'defense') {
            rawGap = stats.defense - monster.defense;
            if (stats.defense <= monster.defense) eligible = false;
            if (p.requireDominant === 'defense' && stats.defense <= stats.attack) eligible = false;
          }

          if (!eligible || rawGap <= 0) continue;
          const effectiveGap = gapSoftcap(rawGap);
          let mult = p.pvpBuildScaling ? pvpMult : 1;
          if (p.halfIfStaminaHigherThanDefense && stats.stamina > stats.defense) mult *= 0.5;
          const dmg = p.coefficient * effectiveGap * mult;
          passiveRaw += dmg;
          passiveDetails.push({ ...p, rawGap, effectiveGap, multiplier: mult, damage: dmg });
        }

        passiveRaw *= 1 - clamp(monster.passiveDamageResistancePct, 0, 100) / 100;

        // Heart + Extra Damage To Monsters act as PvE final damage multipliers.
        const generalPveMultiplier = 1 + (CFG.HEART_PVE_BONUS_PERCENT + (effects.extraMonsterPct || 0)) / 100;
        let nonCrit = (physicalRaw + elementalAfterSpecific + passiveRaw) * generalPveMultiplier;
        nonCrit *= 1 - clamp(monster.finalDamageResistancePct, 0, 100) / 100;

        const critChancePct = clamp(
          CFG.BASE_CRIT_RATE_PERCENT + (effects.critRatePct || 0) - monster.critRateResistancePct,
          0,
          100
        );
        const critMultiplier = Math.max(
          1,
          CFG.BASE_CRIT_MULTIPLIER + ((effects.critDamagePct || 0) - monster.critDamageResistancePct) / 100
        );
        const effectiveBeforeOrryphos = nonCrit * (1 - critChancePct / 100) + nonCrit * critMultiplier * (critChancePct / 100);

        const orry = effects.orryphos;
        const procP = orry ? clamp(orry.procPct / 100, 0, 1) : 0;
        const echoMult = orry ? 1 + orry.echoPct / 100 : 1;
        const effectiveWithOrryphos = effectiveBeforeOrryphos * ((1 - procP) + procP * echoMult);

        // Monster-side final multipliers.
        const divineMultiplier = monster.passives?.divineMultiplier ?? 1;
        const moonMarkExpectedMultiplier = expectedMoonMarkMultiplier(monster);
        const effectiveAfterMonsterPassives = effectiveWithOrryphos * divineMultiplier * moonMarkExpectedMultiplier;

        return {
          physicalRaw,
          eqTerm,
          petTerm,
          charTerm,
          effectiveCharacterAttack,
          equipmentPenPct,
          petPenPct,
          chosenElement,
          elementalRaw,
          elementalAfterSpecific,
          passiveRaw,
          passiveDetails,
          generalPveMultiplier,
          nonCrit,
          critChancePct,
          critMultiplier,
          effectiveBeforeOrryphos,
          orryphos: orry,
          orryphosProcP: procP,
          orryphosEchoMult: echoMult,
          divineMultiplier,
          moonMarkExpectedMultiplier,
          effectiveBeforeMonsterPassives: effectiveWithOrryphos,
          effectiveDamage: effectiveAfterMonsterPassives,
        };
      }

      // Probability distribution over Dawn Brand stacks after N qualifying 1000-stamina attacks.
      function advanceDawnDistribution(dist, procP) {
        const next = Array(CFG.DAWN_MAX_STACKS + 1).fill(0);
        for (let s = 0; s <= CFG.DAWN_MAX_STACKS; s++) {
          const p = dist[s] || 0;
          if (!p) continue;
          const sNormal = Math.min(CFG.DAWN_MAX_STACKS, s + 1);
          const sProc = Math.min(CFG.DAWN_MAX_STACKS, s + 2);
          next[sNormal] += p * (1 - procP);
          next[sProc] += p * procP;
        }
        return next;
      }

      function expectedBrandMultiplier(dist) {
        let sum = 0;
        for (let s = 0; s < dist.length; s++) {
          sum += (dist[s] || 0) * (1 + s * CFG.DAWN_PER_STACK);
        }
        return sum;
      }

      function expectedAttackDamageAtDistribution(core, dist) {
        const brandMult = expectedBrandMultiplier(dist);
        const orry = core.orryphos;
        let dmg = core.effectiveBeforeOrryphos * brandMult;
        if (orry) {
          const p = clamp(orry.procPct / 100, 0, 1);
          const echo = 1 + orry.echoPct / 100;
          dmg *= ((1 - p) + p * echo);
        }
        dmg *= core.divineMultiplier ?? 1;
        dmg *= core.moonMarkExpectedMultiplier ?? 1;
        return dmg;
      }

      function expectedConsumedStamina(nominalCost, orry) {
        if (!orry) return nominalCost;
        const p = clamp(orry.procPct / 100, 0, 1);
        return nominalCost * (1 - p);
      }

      function staminaForTarget(core, hasOurovyrn, target) {
        target = Math.max(0, num(target, 0));
        const orry = core.orryphos;

        if (!hasOurovyrn) {
          const nominalNeeded = target / core.effectiveDamage;
          return {
            expectedStamina: nominalNeeded * (orry ? (1 - clamp(orry.procPct / 100, 0, 1)) : 1),
            nominalStamina: nominalNeeded,
            bigAttacks: 0,
            dawnStacksExpected: 0,
          };
        }

        // Evaluate every strategy using 0..25 qualifying 1000-stamina attacks,
        // then finish the remaining target with 1-stamina Slashes at the resulting brand distribution.
        let best = null;
        let dist = Array(CFG.DAWN_MAX_STACKS + 1).fill(0);
        dist[0] = 1;
        let cumulativeDamage = 0;
        let cumulativeConsumed = 0;

        for (let nBig = 0; nBig <= CFG.DAWN_MAX_STACKS; nBig++) {
          const perOne = expectedAttackDamageAtDistribution(core, dist);
          const remaining = Math.max(0, target - cumulativeDamage);
          const nominalOneStamAttacks = remaining / perOne;
          const finishConsumed = expectedConsumedStamina(nominalOneStamAttacks, orry);
          const totalConsumed = cumulativeConsumed + finishConsumed;

          let expectedStacks = 0;
          for (let s = 0; s < dist.length; s++) expectedStacks += s * (dist[s] || 0);

          const candidate = {
            expectedStamina: totalConsumed,
            nominalStamina: nBig * 1000 + nominalOneStamAttacks,
            bigAttacks: nBig,
            dawnStacksExpected: expectedStacks,
            cumulativeBigDamage: cumulativeDamage,
          };
          if (!best || candidate.expectedStamina < best.expectedStamina) best = candidate;

          if (nBig === CFG.DAWN_MAX_STACKS || cumulativeDamage >= target) break;

          // Damage of the next 1000-stamina qualifying attack uses the CURRENT stack state.
          const oneDamage = expectedAttackDamageAtDistribution(core, dist);
          cumulativeDamage += oneDamage * 1000;
          cumulativeConsumed += expectedConsumedStamina(1000, orry);
          dist = advanceDawnDistribution(dist, orry ? clamp(orry.procPct / 100, 0, 1) : 0);
        }

        return best;
      }

      function detectHeartBonusFromBattlePage() {
        const text = document.body?.innerText || '';
        const m = text.match(/Heart of the First-Year Chronicle increased PvE damage by\s*([\d.]+)%/i);
        if (m) CFG.HEART_PVE_BONUS_PERCENT = num(m[1], CFG.HEART_PVE_BONUS_PERCENT);
      }

      async function collectState(force = false) {
        const cached = loadCache();
        if (!force && cached && Date.now() - cached.savedAt < CFG.CACHE_MAX_AGE_MS) {
          log('Using cached state');
          return cached.state;
        }

        log('Refreshing character/loadout data...');
        const [statsDoc, classDoc, inventoryDoc, petsDoc] = await Promise.all([
          fetchDoc('/stats.php'),
          fetchDoc('/class_skill_tree.php'),
          fetchDoc('/inventory.php?set=attack'),
          fetchDoc('/pets.php?team=attack'),
        ]);

        const stats = parseStats(statsDoc);
        const classInfo = parseClass(classDoc);
        const equipment = parseEquipment(inventoryDoc);
        const pets = await parsePets(petsDoc);
        const loadout = aggregateLoadout(equipment, pets);

        const state = { stats, classInfo, equipment, pets, loadout, refreshedAt: Date.now() };
        saveCache(state);
        log('State refreshed', state);
        return state;
      }


        function detectHeartBonusFromDoc(doc) {
            const text = doc?.body?.innerText || '';
            const m = text.match(/Heart of the First-Year Chronicle increased PvE damage by\s*([\d.]+)%/i);
            if (m) CFG.HEART_PVE_BONUS_PERCENT = num(m[1], CFG.HEART_PVE_BONUS_PERCENT);
            return CFG.HEART_PVE_BONUS_PERCENT;
        }

        function calculateForTarget(state, monster, targetDamage) {
            const core = calculateOneStaminaCore(state, monster);
            const hasOurovyrn = !!state.loadout.effects.ourovyryn;
            const stamina = staminaForTarget(core, hasOurovyrn, targetDamage);
            return { core, stamina, hasOurovyrn };
        }

        return {
            collectState,
            parseMonster,
            detectHeartBonusFromDoc,
            calculateForTarget
        };
    })();

    /* ============================================================
       GENERIC LOOT-THRESHOLD SCANNING
    ============================================================ */

    function extractMaxLootDamageFromDoc(doc) {
        // Prefer a container explicitly labelled "Possible Loot".
        const possibleLootContainer =
            [...doc.querySelectorAll('section, .panel, .card, div')]
                .find(el => {
                    const heading = el.querySelector(':scope > h1, :scope > h2, :scope > h3, :scope > .panel-title, :scope > .title');
                    return heading && /possible\s+loot/i.test(cleanText(heading.textContent));
                }) ||
            [...doc.querySelectorAll('.panel')]
                .find(el => /possible\s+loot/i.test(cleanText(el.textContent))) ||
            doc;

        let maximum = 0;

        // Common modern layout.
        for (const chip of possibleLootContainer.querySelectorAll('.loot-stats .chip, .loot-card .chip, .chip')) {
            const text = cleanText(chip.textContent);
            const match = text.match(/DMG\s*req\s*:\s*([\d,._\s]+(?:[KMBT])?)/i);

            if (!match) continue;

            const value = parseDamageValue(match[1]);
            if (value !== null && value > maximum) maximum = value;
        }

        // Fallback: scan text in case markup changes.
        if (maximum <= 0) {
            const text = possibleLootContainer.textContent || '';
            const regex = /DMG\s*req\s*:\s*([\d,._\s]+(?:[KMBT])?)/gi;

            let match;
            while ((match = regex.exec(text)) !== null) {
                const value = parseDamageValue(match[1]);
                if (value !== null && value > maximum) maximum = value;
            }
        }

        return maximum > 0 ? maximum : null;
    }

    async function fetchMaxLootDamageFromBattleUrl(battleUrl) {
        if (!battleUrl) return null;

        const html = await fetchHtml(battleUrl);
        const doc = parseHtml(html);

        return extractMaxLootDamageFromDoc(doc);
    }

    function lootTargetNeedsRefresh(monster) {
        if (!monster.autoDamageTarget) return true;

        const updatedAt = Date.parse(monster.lootThresholdUpdatedAt || '');
        if (!Number.isFinite(updatedAt)) return true;

        return (Date.now() - updatedAt) > LOOT_THRESHOLD_REFRESH_MS;
    }

    /* ============================================================
       CACHED MONSTER METADATA
    ============================================================ */

    function iterateAllMonsterNodes(catalogue, callback) {
        // Dungeons
        for (const dungeon of Object.values(catalogue.dungeons || {})) {
            for (const location of Object.values(dungeon.locations || {})) {
                for (const [monsterKey, monster] of Object.entries(location.monsters || {})) {
                    callback(monsterKey, monster, {
                        sourceType: 'dungeon',
                        sourceName: dungeon.name,
                        sectionName: location.name
                    });
                }
            }
        }

        // Gates
        for (const gate of Object.values(catalogue.gates || {})) {
            for (const section of Object.values(gate.sections || {})) {
                for (const [monsterKey, monster] of Object.entries(section.monsters || {})) {
                    callback(monsterKey, monster, {
                        sourceType: 'gate',
                        sourceName: gate.name,
                        sectionName: section.name
                    });
                }
            }
        }
    }

    function collectCachedMonsterInfo(catalogue) {
        const map = new Map();

        iterateAllMonsterNodes(catalogue, (monsterKey, monster) => {
            if (!map.has(monsterKey)) map.set(monsterKey, {});
            const cached = map.get(monsterKey);

            if (monster.autoDamageTarget && !cached.autoDamageTarget) {
                cached.autoDamageTarget = Number(monster.autoDamageTarget);
            }

            if (monster.lootThresholdUpdatedAt && !cached.lootThresholdUpdatedAt) {
                cached.lootThresholdUpdatedAt = monster.lootThresholdUpdatedAt;
            }

            if (monster.damageModel && !cached.damageModel) {
                cached.damageModel = monster.damageModel;
            }

            if (monster.damageModelUpdatedAt && !cached.damageModelUpdatedAt) {
                cached.damageModelUpdatedAt = monster.damageModelUpdatedAt;
            }

            if (monster.representativeBattleUrl && !cached.representativeBattleUrl) {
                cached.representativeBattleUrl = monster.representativeBattleUrl;
            }

            // Preserve the actual server-side attack identity across scans.
            if (monster.monsterId && !cached.monsterId) {
                cached.monsterId = String(monster.monsterId);
            }

            // Dungeon battles use dgmid + instance_id instead of monster_id.
            if (monster.dgmid && !cached.dgmid) {
                cached.dgmid = String(monster.dgmid);
            }
            if (monster.instanceId && !cached.instanceId) {
                cached.instanceId = String(monster.instanceId);
            }
        });

        return map;
    }

    function applyLootThresholdEverywhere(catalogue, monsterKey, damage) {
        const updatedAt = timestampIso();

        iterateAllMonsterNodes(catalogue, (key, monster) => {
            if (key !== monsterKey) return;

            monster.autoDamageTarget = damage;
            monster.lootThresholdUpdatedAt = updatedAt;
        });
    }

    function applyDamageModelEverywhere(catalogue, monsterKey, damageModel, battleUrl = '') {
        const updatedAt = timestampIso();

        iterateAllMonsterNodes(catalogue, (key, monster) => {
            if (key !== monsterKey) return;

            monster.damageModel = damageModel;
            monster.damageModelUpdatedAt = updatedAt;
            if (battleUrl) monster.representativeBattleUrl = battleUrl;
        });
    }

    function damageModelNeedsRefresh(monster) {
        if (!monster.damageModel) return true;

        const updatedAt = Date.parse(monster.damageModelUpdatedAt || '');
        if (!Number.isFinite(updatedAt)) return true;

        return (Date.now() - updatedAt) > CATALOGUE_REFRESH_MS;
    }

    function getUniqueMonsters(catalogue) {
        const map = new Map();

        iterateAllMonsterNodes(catalogue, (monsterKey, monster) => {
            if (!map.has(monsterKey)) {
                map.set(monsterKey, monster);
            } else {
                const existing = map.get(monsterKey);

                if (!existing.autoDamageTarget && monster.autoDamageTarget) {
                    existing.autoDamageTarget = monster.autoDamageTarget;
                    existing.lootThresholdUpdatedAt = monster.lootThresholdUpdatedAt;
                }

                if (!existing.damageModel && monster.damageModel) {
                    existing.damageModel = monster.damageModel;
                    existing.damageModelUpdatedAt = monster.damageModelUpdatedAt;
                }

                if (!existing.representativeBattleUrl && monster.representativeBattleUrl) {
                    existing.representativeBattleUrl = monster.representativeBattleUrl;
                }

                if (!existing.monsterId && monster.monsterId) {
                    existing.monsterId = String(monster.monsterId);
                }
                if (!existing.dgmid && monster.dgmid) {
                    existing.dgmid = String(monster.dgmid);
                }
                if (!existing.instanceId && monster.instanceId) {
                    existing.instanceId = String(monster.instanceId);
                }
            }
        });

        return map;
    }

    /* ============================================================
       DUNGEON DISCOVERY
    ============================================================ */

    function extractDungeonInstances(doc) {
        const result = new Map();

        for (const link of doc.querySelectorAll('a[href*="guild_dungeon_enter.php?id="]')) {
            let instanceId;

            try {
                instanceId = new URL(link.href, location.origin).searchParams.get('id');
            } catch {
                continue;
            }

            if (!instanceId) continue;

            const card = link.closest('.card') || link.parentElement;
            if (!card) continue;

            const titleElement = card.querySelector('.h, .title, h2, h3, strong');
            let dungeonName = cleanText(titleElement?.textContent);

            if (!dungeonName) dungeonName = `Unknown Dungeon ${instanceId}`;

            const ended = Boolean(card.closest('.grid.ended'));

            result.set(String(instanceId), {
                instanceId: String(instanceId),
                dungeonName,
                status: ended ? 'ended' : 'active'
            });
        }

        return [...result.values()];
    }

    function extractLocationsFromDungeonInstance(doc, instanceId) {
        const result = new Map();

        for (const link of doc.querySelectorAll('a[href*="guild_dungeon_location.php"]')) {
            let url;

            try {
                url = new URL(link.href, location.origin);
            } catch {
                continue;
            }

            const linkedInstance = url.searchParams.get('instance_id');
            const locationId = url.searchParams.get('location_id');

            if (!locationId) continue;
            if (linkedInstance && String(linkedInstance) !== String(instanceId)) continue;

            let locationName = cleanText(link.textContent);

            const explicit = link.querySelector('.h, .title, .name, strong');
            if (explicit) locationName = cleanText(explicit.textContent);

            if (!locationName || locationName.length > 100) {
                const parent = link.closest('.card, .location, .loc');
                const parentName = parent?.querySelector('.h, .title, .name, h3, strong');
                if (parentName) locationName = cleanText(parentName.textContent);
            }

            locationName = cleanLocationName(locationName);
            if (!locationName) locationName = `Location ${locationId}`;

            result.set(String(locationId), {
                id: String(locationId),
                name: locationName
            });
        }

        return [...result.values()];
    }

    function applyDungeonLocationFallbacks(dungeonName, locations) {
        const result = new Map(locations.map(loc => [String(loc.id), loc]));
        const key = normalizeKey(dungeonName);

        if (key.includes('polyhedral crucible')) {
            for (const fallback of POLYHEDRAL_FALLBACK_LOCATIONS) {
                if (!result.has(fallback.id)) {
                    result.set(fallback.id, fallback);
                }
            }
        }

        return [...result.values()];
    }

    function extractDungeonMonsterName(card) {
        const nameContainer = card.children[1]?.children[0];

        if (nameContainer) {
            const textNode = [...nameContainer.childNodes]
                .find(node =>
                    node.nodeType === Node.TEXT_NODE &&
                    cleanText(node.textContent).length > 0
                );

            if (textNode) return cleanText(textNode.textContent);
        }

        const fallback = card.querySelector(
            '.monster-name, .mon-name, .name, .title, h3, strong'
        );

        return fallback ? cleanText(fallback.textContent) : '';
    }

    function extractDungeonMonsters(doc, defaultInstanceId) {
        const result = new Map();

        for (const card of doc.querySelectorAll('.mon')) {
            const name = extractDungeonMonsterName(card);
            if (!name) continue;

            const monsterKey = normalizeKey(name);
            const image = card.querySelector('img')?.getAttribute('src') || '';

            const battleLink = card.querySelector('a[href*="dgmid="]');
            let dgmid = null;
            let instanceId = defaultInstanceId;
            let battleUrl = '';

            if (battleLink) {
                try {
                    const url = new URL(battleLink.href, location.origin);

                    dgmid = url.searchParams.get('dgmid');
                    instanceId = url.searchParams.get('instance_id') || instanceId;

                    if (dgmid && instanceId) {
                        battleUrl = `${BASE_URL}/battle.php?dgmid=${encodeURIComponent(dgmid)}&instance_id=${encodeURIComponent(instanceId)}`;
                    } else {
                        battleUrl = url.href;
                    }
                } catch {
                    // Keep catalogue entry even if link parsing fails.
                }
            }

            const encounter =
                battleUrl && dgmid && instanceId
                    ? {
                        battleUrl,
                        dgmid: String(dgmid),
                        instanceId: String(instanceId)
                    }
                    : null;

            if (!result.has(monsterKey)) {
                result.set(monsterKey, {
                    name,
                    image,
                    battleUrl,
                    dgmid: dgmid ? String(dgmid) : '',
                    instanceId: instanceId ? String(instanceId) : '',
                    type: 'regular',
                    // One UI/catalogue monster can correspond to many live
                    // dungeon cards with the same name. Keep every dgmid so
                    // selecting the monster can attack every encounter.
                    encounters: encounter ? [encounter] : []
                });
            } else {
                const existing = result.get(monsterKey);
                if (!existing.image && image) existing.image = image;
                if (!existing.battleUrl && battleUrl) existing.battleUrl = battleUrl;
                if (!existing.dgmid && dgmid) existing.dgmid = String(dgmid);
                if (!existing.instanceId && instanceId) existing.instanceId = String(instanceId);

                if (encounter) {
                    existing.encounters ||= [];
                    const duplicate = existing.encounters.some(item =>
                        String(item.dgmid) === String(encounter.dgmid) &&
                        String(item.instanceId) === String(encounter.instanceId)
                    );
                    if (!duplicate) existing.encounters.push(encounter);
                }
            }
        }

        return [...result.values()];
    }

    function ensureDungeon(catalogue, dungeonName) {
        const key = normalizeKey(dungeonName);

        if (!catalogue.dungeons[key]) {
            catalogue.dungeons[key] = {
                name: dungeonName,
                instances: {},
                locations: {}
            };
        }

        return catalogue.dungeons[key];
    }

    function ensureDungeonLocation(dungeon, locationId, locationName) {
        const key = String(locationId);

        if (!dungeon.locations[key]) {
            dungeon.locations[key] = {
                id: key,
                name: cleanLocationName(locationName),
                monsters: {}
            };
        } else if (dungeon.locations[key].name.startsWith('Location ') && locationName) {
            dungeon.locations[key].name = cleanLocationName(locationName);
        }

        return dungeon.locations[key];
    }

    /* ============================================================
       GATE DISCOVERY
    ============================================================ */

    function extractGateCards(doc) {
        const gates = [];

        for (const card of doc.querySelectorAll('a.gate-card')) {
            const name =
                cleanText(card.querySelector('.title')?.textContent) ||
                cleanText(card.getAttribute('aria-label'));

            if (!name) continue;

            const href = absoluteUrl(card.getAttribute('href') || '', GATES_URL);
            const badgeText = cleanText(card.querySelector('.badge')?.textContent);

            gates.push({
                name,
                href,
                active: /active/i.test(badgeText) || Boolean(href)
            });
        }

        return gates;
    }

    function ensureGate(catalogue, gateName) {
        const key = normalizeKey(gateName);

        if (!catalogue.gates[key]) {
            catalogue.gates[key] = {
                name: gateName,
                sections: {}
            };
        }

        return catalogue.gates[key];
    }

    function ensureGateSection(gate, sectionKey, sectionName, meta = {}) {
        const key = String(sectionKey);

        if (!gate.sections[key]) {
            gate.sections[key] = {
                id: key,
                name: sectionName,
                locked: Boolean(meta.locked),
                gateId: meta.gateId ? String(meta.gateId) : null,
                waveId: meta.waveId ? String(meta.waveId) : null,
                url: meta.url || '',
                monsters: {}
            };
        } else {
            if (sectionName) gate.sections[key].name = sectionName;
            if (meta.url) gate.sections[key].url = meta.url;
            if (meta.gateId) gate.sections[key].gateId = String(meta.gateId);
            if (meta.waveId) gate.sections[key].waveId = String(meta.waveId);
            if (meta.locked !== undefined) gate.sections[key].locked = Boolean(meta.locked);
        }

        return gate.sections[key];
    }

    function extractWaveLinks(doc, baseUrl) {
        const result = new Map();

        for (const link of doc.querySelectorAll('.waves-nav a.wave-chip, a.wave-chip')) {
            const text = cleanText(link.textContent);
            if (!text || !/wave/i.test(text)) continue;

            const url = absoluteUrl(link.getAttribute('href') || '', baseUrl);
            if (!url) continue;

            try {
                const parsed = new URL(url);
                const gateId = parsed.searchParams.get('gate');
                const waveId = parsed.searchParams.get('wave');

                if (!gateId || !waveId) continue;

                result.set(`${gateId}:${waveId}`, {
                    gateId,
                    waveId,
                    name: text,
                    url
                });
            } catch {
                // ignore
            }
        }

        return [...result.values()];
    }

    function extractOlympusMapSections(doc) {
        const result = new Map();

        // Wave 1 is linked by the "Leave the Gate" button.
        const wave1Link = doc.querySelector('a.map-back-btn[href*="active_wave.php"][href*="gate=5"][href*="wave="]');
        if (wave1Link) {
            const url = absoluteUrl(wave1Link.getAttribute('href'), OLYMPUS_MAP_URL);
            try {
                const parsed = new URL(url);
                const gateId = parsed.searchParams.get('gate');
                const waveId = parsed.searchParams.get('wave');

                if (gateId && waveId) {
                    result.set(`wave:${waveId}`, {
                        key: `wave:${waveId}`,
                        name: 'Wave 1',
                        gateId,
                        waveId,
                        url,
                        locked: false
                    });
                }
            } catch {
                // ignore
            }
        }

        // Unlocked domains.
        for (const marker of doc.querySelectorAll('a.domain-marker[href*="active_wave.php"]')) {
            const name =
                cleanText(marker.getAttribute('title')) ||
                cleanText(marker.getAttribute('aria-label')) ||
                cleanText(marker.querySelector('.marker-label')?.textContent);

            const url = absoluteUrl(marker.getAttribute('href'), OLYMPUS_MAP_URL);
            if (!url) continue;

            try {
                const parsed = new URL(url);
                const gateId = parsed.searchParams.get('gate');
                const waveId = parsed.searchParams.get('wave');

                if (!gateId || !waveId) continue;

                const displayName =
                    name.replace(/^Enter\s+/i, '').trim();

                result.set(`domain:${waveId}`, {
                    key: `domain:${waveId}`,
                    name: displayName,
                    gateId,
                    waveId,
                    url,
                    locked: false
                });
            } catch {
                // ignore
            }
        }

        // Locked domains are visible but have no wave URL yet.
        for (const marker of doc.querySelectorAll('button.domain-marker.locked')) {
            const raw =
                cleanText(marker.getAttribute('title')) ||
                cleanText(marker.getAttribute('aria-label')) ||
                cleanText(marker.querySelector('.marker-label')?.textContent);

            if (!raw) continue;

            const displayName = raw
                .replace(/\s+is sealed$/i, '')
                .replace(/^Enter\s+/i, '')
                .trim();

            const domainKey =
                marker.getAttribute('data-domain') ||
                normalizeKey(displayName);

            result.set(`locked:${domainKey}`, {
                key: `locked:${domainKey}`,
                name: displayName,
                gateId: '5',
                waveId: null,
                url: '',
                locked: true
            });
        }

        return [...result.values()];
    }

    function getWavePageTitle(doc, fallbackName) {
        const waveTitle = doc.querySelector('.wave-title');
        if (!waveTitle) return fallbackName;

        // Clone so nested recommendation text does not pollute the title.
        const clone = waveTitle.cloneNode(true);
        clone.querySelectorAll('div, br').forEach(el => el.remove());

        const title = cleanText(clone.textContent)
            .replace(/^🌊\s*/u, '')
            .trim();

        return title || fallbackName;
    }

    function extractGateMonsters(doc, pageUrl) {
        const result = new Map();

        /*
         * 1) Actual monster cards.
         *
         * A gate monster uses:
         *   .monster-card
         *   data-monster-id
         *   data-name
         *   data-boss
         * and usually links to battle.php?id=<monster id>
         */
        for (const card of doc.querySelectorAll('.monster-card')) {
            const displayName =
                cleanText(card.querySelector('h3')?.textContent) ||
                cleanText(card.dataset.name);

            if (!displayName) continue;

            const monsterKey = normalizeKey(displayName);
            const monsterId = card.dataset.monsterId || '';

            const image = card.querySelector('img.monster-img, img')?.getAttribute('src') || '';

            let battleUrl = '';
            const battleLink =
                [...card.querySelectorAll('a[href*="battle.php"]')]
                    .find(a => /battle\.php\?id=/i.test(a.getAttribute('href') || '')) ||
                card.querySelector('a[href*="battle.php"]');

            if (battleLink) {
                battleUrl = absoluteUrl(battleLink.getAttribute('href'), pageUrl);
            } else if (monsterId) {
                battleUrl = `${BASE_URL}/battle.php?id=${encodeURIComponent(monsterId)}`;
            }

            result.set(monsterKey, {
                name: displayName,
                image,
                battleUrl,
                monsterId,
                type: card.dataset.boss === '1' ? 'boss' : 'regular',
                timed: false,
                nextSpawnTs: null,
                currentlyAlive: card.dataset.dead !== '1'
            });
        }

        /*
         * 2) Auto-summon cards.
         *
         * These identify timed bosses even while they are not currently
         * represented by an attackable monster-card.
         */
        for (const timerCard of doc.querySelectorAll('.auto-summon-card')) {
            const displayName = cleanText(
                timerCard.querySelector('.auto-summon-name')?.textContent
            );

            if (!displayName) continue;

            const monsterKey = normalizeKey(displayName);
            const image = timerCard.querySelector('img')?.getAttribute('src') || '';
            const alive = timerCard.dataset.alive === '1';
            const nextTs = Number(timerCard.dataset.nextTs || 0) || null;

            // Some timed-boss layouts place data-monster-id on the timer card
            // itself or on a nearby ancestor rather than on .monster-card.
            let timedMonsterId =
                timerCard.dataset?.monsterId ||
                timerCard.getAttribute?.('data-monster-id') ||
                '';
            let timedBattleUrl = '';

            let timedNode = timerCard;
            for (let depth = 0; timedNode && depth < 8; depth++, timedNode = timedNode.parentElement) {
                if (!timedMonsterId) {
                    timedMonsterId =
                        timedNode.dataset?.monsterId ||
                        timedNode.getAttribute?.('data-monster-id') ||
                        '';
                }

                if (!timedBattleUrl) {
                    const a = timedNode.matches?.('a[href*="battle.php"]')
                        ? timedNode
                        : timedNode.querySelector?.('a[href*="battle.php"]');
                    if (a?.getAttribute?.('href')) {
                        timedBattleUrl = absoluteUrl(a.getAttribute('href'), pageUrl);
                    }
                }

                if (timedMonsterId && timedBattleUrl) break;
            }

            if (result.has(monsterKey)) {
                const existing = result.get(monsterKey);
                existing.timed = true;
                existing.type = 'timed_boss';
                existing.currentlyAlive = alive;
                existing.nextSpawnTs = nextTs;
                if (!existing.image && image) existing.image = image;
                if (!existing.monsterId && timedMonsterId) {
                    existing.monsterId = String(timedMonsterId);
                }
                if (!existing.battleUrl && timedBattleUrl) {
                    existing.battleUrl = timedBattleUrl;
                }
            } else {
                result.set(monsterKey, {
                    name: displayName,
                    image,
                    battleUrl: timedBattleUrl,
                    monsterId: timedMonsterId ? String(timedMonsterId) : '',
                    type: 'timed_boss',
                    timed: true,
                    nextSpawnTs: nextTs,
                    currentlyAlive: alive
                });
            }
        }

        return [...result.values()];
    }

    /* ============================================================
       MONSTER MERGING
    ============================================================ */

    function mergeMonster(section, monster, cachedInfo, battleRepresentatives, meta = {}) {
        const key = normalizeKey(monster.name);
        const cached = cachedInfo.get(key) || {};
        const representativeBattleUrl = monster.battleUrl || cached.representativeBattleUrl || '';
        const monsterId = monster.monsterId || cached.monsterId || '';
        const dgmid = monster.dgmid || cached.dgmid || '';
        const instanceId = monster.instanceId || cached.instanceId || '';

        if (!section.monsters[key]) {
            section.monsters[key] = {
                name: monster.name,
                image: monster.image || '',
                type: monster.type || 'regular',
                timed: Boolean(monster.timed),
                nextSpawnTs: monster.nextSpawnTs || null,
                currentlyAlive:
                    monster.currentlyAlive === undefined
                        ? null
                        : Boolean(monster.currentlyAlive),
                autoDamageTarget: cached.autoDamageTarget || null,
                lootThresholdUpdatedAt: cached.lootThresholdUpdatedAt || null,
                damageModel: cached.damageModel || null,
                damageModelUpdatedAt: cached.damageModelUpdatedAt || null,
                // Actual server-side IDs captured from the same source card
                // that supplied the monster name. Never infer monsterId from
                // battle.php?id=.
                monsterId: monsterId ? String(monsterId) : '',
                dgmid: dgmid ? String(dgmid) : '',
                instanceId: instanceId ? String(instanceId) : '',
                representativeBattleUrl,
                activeBattleUrl:
                    meta.sourceType === 'dungeon' && meta.instanceStatus === 'active'
                        ? (monster.battleUrl || '')
                        : '',
                activeEncounters:
                    meta.sourceType === 'dungeon' && meta.instanceStatus === 'active'
                        ? (
                            Array.isArray(monster.encounters) && monster.encounters.length
                                ? monster.encounters.map(encounter => ({ ...encounter }))
                                : (
                                    monster.battleUrl && dgmid && instanceId
                                        ? [{
                                            battleUrl: monster.battleUrl,
                                            dgmid: String(dgmid),
                                            instanceId: String(instanceId)
                                        }]
                                        : []
                                )
                        )
                        : []
            };

            // If the first occurrence is from the active dungeon instance,
            // make the active URL authoritative immediately.
            if (
                meta.sourceType === 'dungeon' &&
                meta.instanceStatus === 'active' &&
                monster.battleUrl
            ) {
                const activeIds = dungeonIdsFromBattleUrl(monster.battleUrl);
                const created = section.monsters[key];
                created.representativeBattleUrl = monster.battleUrl;
                if (activeIds.dgmid) created.dgmid = String(activeIds.dgmid);
                if (activeIds.instanceId) created.instanceId = String(activeIds.instanceId);
            }
        } else {
            const existing = section.monsters[key];

            if (!existing.image && monster.image) existing.image = monster.image;
            if (monster.type) existing.type = monster.type;
            if (monster.timed) existing.timed = true;
            if (monster.nextSpawnTs) existing.nextSpawnTs = monster.nextSpawnTs;
            if (monster.currentlyAlive !== undefined) {
                existing.currentlyAlive = Boolean(monster.currentlyAlive);
            }
            const isDungeon = meta.sourceType === 'dungeon';
            const isActiveDungeon = isDungeon && meta.instanceStatus === 'active';

            // For dungeons, ended instances are scanned for catalogue discovery
            // only. They must never overwrite the live target identity. This
            // was the source of the stale dgmid/instanceId values: an active
            // instance could be scanned first and then an ended instance would
            // replace its IDs while activeBattleUrl stayed correct.
            if (!isDungeon) {
                if (representativeBattleUrl) existing.representativeBattleUrl = representativeBattleUrl;
                if (monsterId) existing.monsterId = String(monsterId);
                if (dgmid) existing.dgmid = String(dgmid);
                if (instanceId) existing.instanceId = String(instanceId);
            } else if (isActiveDungeon && monster.battleUrl) {
                const activeIds = dungeonIdsFromBattleUrl(monster.battleUrl);
                const liveDgmid = activeIds.dgmid || dgmid || '';
                const liveInstanceId = activeIds.instanceId || instanceId || '';

                existing.representativeBattleUrl = monster.battleUrl;
                existing.activeBattleUrl = monster.battleUrl;
                if (liveDgmid) existing.dgmid = String(liveDgmid);
                if (liveInstanceId) existing.instanceId = String(liveInstanceId);

                const incomingEncounters =
                    Array.isArray(monster.encounters) && monster.encounters.length
                        ? monster.encounters
                        : [{
                            battleUrl: monster.battleUrl,
                            dgmid: String(liveDgmid),
                            instanceId: String(liveInstanceId)
                        }];

                existing.activeEncounters = [];
                for (const encounter of incomingEncounters) {
                    if (!encounter?.battleUrl) continue;
                    const ids = dungeonIdsFromBattleUrl(encounter.battleUrl);
                    const encounterDgmid = ids.dgmid || encounter.dgmid || '';
                    const encounterInstanceId = ids.instanceId || encounter.instanceId || '';

                    if (!encounterDgmid || !encounterInstanceId) continue;

                    const duplicate = existing.activeEncounters.some(item =>
                        String(item.dgmid) === String(encounterDgmid) &&
                        String(item.instanceId) === String(encounterInstanceId)
                    );
                    if (duplicate) continue;

                    existing.activeEncounters.push({
                        battleUrl: encounter.battleUrl,
                        dgmid: String(encounterDgmid),
                        instanceId: String(encounterInstanceId)
                    });
                }

                addLog(
                    `${monster.name}: persisted active dungeon identity → ` +
                    `dgmid=${liveDgmid || '?'} instance_id=${liveInstanceId || '?'} ` +
                    `(${existing.activeEncounters.length || 1} encounter(s)).`
                );
            } else {
                // Ended dungeon instance: retain its metadata only when this
                // monster has never had any dungeon identity at all.
                if (!existing.representativeBattleUrl && representativeBattleUrl) {
                    existing.representativeBattleUrl = representativeBattleUrl;
                }
                if (!existing.dgmid && dgmid) existing.dgmid = String(dgmid);
                if (!existing.instanceId && instanceId) existing.instanceId = String(instanceId);
            }
        }

        // Only live dungeon instances should become attack representatives.
        // Ended instances remain useful for catalogue discovery but must never
        // supply an automation route.
        const canRepresentForAttack =
            meta.sourceType !== 'dungeon' ||
            meta.instanceStatus === 'active';

        if (
            representativeBattleUrl &&
            canRepresentForAttack &&
            !battleRepresentatives.has(key)
        ) {
            battleRepresentatives.set(key, {
                name: monster.name,
                battleUrl: representativeBattleUrl
            });
        }
    }

    /* ============================================================
       SCAN DUNGEONS
    ============================================================ */

    async function scanDungeons(catalogue, cachedInfo, battleRepresentatives) {
        setStatus('Scanning guild dungeons...');

        const html = await fetchHtml(DUNGEONS_URL);
        const doc = parseHtml(html);

        const instances = extractDungeonInstances(doc);

        // activeBattleUrl is intentionally ephemeral. Clear routes from the
        // previous scan first so an instance that has since completed cannot
        // remain attackable from stale catalogue data.
        for (const dungeon of Object.values(catalogue.dungeons || {})) {
            for (const locationData of Object.values(dungeon.locations || {})) {
                for (const monster of Object.values(locationData.monsters || {})) {
                    monster.activeBattleUrl = '';
                    monster.activeEncounters = [];
                }
            }
        }

        addLog(`Dungeons: found ${instances.length} instance(s).`);

        for (const instance of instances) {
            const dungeon = ensureDungeon(catalogue, instance.dungeonName);

            dungeon.instances[instance.instanceId] = {
                status: instance.status
            };

            let instanceHtml;
            try {
                instanceHtml = await fetchHtml(
                    `${DUNGEON_INSTANCE_URL}?id=${encodeURIComponent(instance.instanceId)}`
                );
            } catch (error) {
                addLog(`${instance.dungeonName} #${instance.instanceId}: instance scan failed (${error.message}).`);
                continue;
            }

            const instanceDoc = parseHtml(instanceHtml);

            let locations = extractLocationsFromDungeonInstance(
                instanceDoc,
                instance.instanceId
            );

            locations = applyDungeonLocationFallbacks(
                instance.dungeonName,
                locations
            );

            for (const locationInfo of locations) {
                const locationData = ensureDungeonLocation(
                    dungeon,
                    locationInfo.id,
                    locationInfo.name
                );

                const locationUrl =
                    `${DUNGEON_LOCATION_URL}` +
                    `?instance_id=${encodeURIComponent(instance.instanceId)}` +
                    `&location_id=${encodeURIComponent(locationInfo.id)}`;

                try {
                    const locationHtml = await fetchHtml(locationUrl);
                    const locationDoc = parseHtml(locationHtml);
                    const monsters = extractDungeonMonsters(
                        locationDoc,
                        instance.instanceId
                    );

                    for (const monster of monsters) {
                        mergeMonster(
                            locationData,
                            monster,
                            cachedInfo,
                            battleRepresentatives,
                            {
                                sourceType: 'dungeon',
                                instanceStatus: instance.status,
                                instanceId: instance.instanceId
                            }
                        );
                    }
                } catch (error) {
                    addLog(`${instance.dungeonName} → ${locationData.name}: failed (${error.message}).`);
                }

                await sleep(REQUEST_DELAY_MS);
            }

            saveCatalogue(catalogue);
            renderTargetManager();

            await sleep(REQUEST_DELAY_MS);
        }
    }

    /* ============================================================
       SCAN GATES
    ============================================================ */

    async function scanStandardGate(gate, gateCard, cachedInfo, battleRepresentatives) {
        let firstPageHtml;
        try {
            firstPageHtml = await fetchHtml(gateCard.href);
        } catch (error) {
            addLog(`${gate.name}: gate page failed (${error.message}).`);
            return;
        }

        const firstPageDoc = parseHtml(firstPageHtml);

        let waves = extractWaveLinks(firstPageDoc, gateCard.href);

        if (normalizeKey(gate.name) === 'grakthar') {
            const map = new Map(waves.map(w => [`${w.gateId}:${w.waveId}`, w]));

            for (const fallback of GRAKTHAR_FALLBACK_WAVES) {
                const k = `${fallback.gateId}:${fallback.waveId}`;

                if (!map.has(k)) {
                    map.set(k, {
                        ...fallback,
                        url: `${BASE_URL}/active_wave.php?gate=${fallback.gateId}&wave=${fallback.waveId}`
                    });
                }
            }

            waves = [...map.values()];
        }

        // If the page itself is a wave and the nav is incomplete, include it.
        try {
            const parsed = new URL(gateCard.href);
            const gateId = parsed.searchParams.get('gate');
            const waveId = parsed.searchParams.get('wave');

            if (gateId && waveId && !waves.some(w => w.gateId === gateId && w.waveId === waveId)) {
                waves.push({
                    gateId,
                    waveId,
                    name: getWavePageTitle(firstPageDoc, `Wave ${waveId}`),
                    url: gateCard.href
                });
            }
        } catch {
            // ignore
        }

        for (const wave of waves) {
            let waveHtml;
            let waveDoc;

            try {
                if (wave.url === gateCard.href) {
                    waveHtml = firstPageHtml;
                    waveDoc = firstPageDoc;
                } else {
                    waveHtml = await fetchHtml(wave.url);
                    waveDoc = parseHtml(waveHtml);
                }
            } catch (error) {
                addLog(`${gate.name} → ${wave.name}: failed (${error.message}).`);
                continue;
            }

            const sectionName = getWavePageTitle(waveDoc, wave.name);
            const section = ensureGateSection(
                gate,
                `wave:${wave.waveId}`,
                sectionName,
                {
                    gateId: wave.gateId,
                    waveId: wave.waveId,
                    url: wave.url,
                    locked: false
                }
            );

            const monsters = extractGateMonsters(waveDoc, wave.url);

            const idsFound = monsters.filter(monster => monster.monsterId).length;
            addLog(`${gate.name} → ${sectionName}: scanned ${monsters.length} monster(s), ${idsFound} live monster ID(s).`);

            for (const monster of monsters) {
                mergeMonster(
                    section,
                    monster,
                    cachedInfo,
                    battleRepresentatives
                );
            }

            await sleep(REQUEST_DELAY_MS);
        }
    }

    async function scanOlympusGate(gate, cachedInfo, battleRepresentatives) {
        let mapHtml;

        try {
            mapHtml = await fetchHtml(OLYMPUS_MAP_URL);
        } catch (error) {
            addLog(`Olympus map failed (${error.message}).`);
            return;
        }

        const mapDoc = parseHtml(mapHtml);
        const sections = extractOlympusMapSections(mapDoc);

        addLog(`Olympus: found ${sections.length} wave/domain section(s).`);

        for (const sectionInfo of sections) {
            const section = ensureGateSection(
                gate,
                sectionInfo.key,
                sectionInfo.name,
                sectionInfo
            );

            // Locked domains are catalogued but cannot be scanned yet.
            if (sectionInfo.locked || !sectionInfo.url) {
                continue;
            }

            try {
                const branchHtml = await fetchHtml(sectionInfo.url);
                const branchDoc = parseHtml(branchHtml);

                section.name = getWavePageTitle(
                    branchDoc,
                    sectionInfo.name
                );

                const monsters = extractGateMonsters(
                    branchDoc,
                    sectionInfo.url
                );

                const idsFound = monsters.filter(monster => monster.monsterId).length;
                addLog(`Olympus → ${section.name}: scanned ${monsters.length} monster(s), ${idsFound} live monster ID(s).`);

                for (const monster of monsters) {
                    mergeMonster(
                        section,
                        monster,
                        cachedInfo,
                        battleRepresentatives
                    );
                }
            } catch (error) {
                addLog(`Olympus → ${sectionInfo.name}: failed (${error.message}).`);
            }

            await sleep(REQUEST_DELAY_MS);
        }
    }

    async function scanGates(catalogue, cachedInfo, battleRepresentatives) {
        setStatus('Scanning Gates...');

        const html = await fetchHtml(GATES_URL);
        const doc = parseHtml(html);

        const gateCards = extractGateCards(doc);

        addLog(`Gates: found ${gateCards.length} gate(s).`);

        for (const gateCard of gateCards) {
            const gate = ensureGate(catalogue, gateCard.name);

            if (normalizeKey(gateCard.name) === 'olympus') {
                await scanOlympusGate(
                    gate,
                    cachedInfo,
                    battleRepresentatives
                );
            } else if (gateCard.href) {
                await scanStandardGate(
                    gate,
                    gateCard,
                    cachedInfo,
                    battleRepresentatives
                );
            }

            saveCatalogue(catalogue);
            renderTargetManager();

            await sleep(REQUEST_DELAY_MS);
        }
    }

    /* ============================================================
       AUTO THRESHOLD SCAN
    ============================================================ */

    async function scanLootThresholds(
        catalogue,
        battleRepresentatives,
        forceLootRefresh
    ) {
        const uniqueMonsters = getUniqueMonsters(catalogue);

        let index = 0;
        const total = uniqueMonsters.size;

        for (const [monsterKey, monster] of uniqueMonsters) {
            index++;

            const representative = battleRepresentatives.get(monsterKey);
            const battleUrl = representative?.battleUrl || monster.representativeBattleUrl || '';
            if (!battleUrl) continue;

            const needLoot = forceLootRefresh || lootTargetNeedsRefresh(monster);
            const needModel = forceLootRefresh || damageModelNeedsRefresh(monster);

            if (!needLoot && !needModel) continue;

            setStatus(
                `Checking battle data ${index}/${total}: ${monster.name}`
            );

            try {
                const battleHtml = await fetchHtml(battleUrl);
                const battleDoc = parseHtml(battleHtml);
                DamageEstimator.detectHeartBonusFromDoc(battleDoc);

                if (needLoot) {
                    const maxDamage = extractMaxLootDamageFromDoc(battleDoc);

                    if (maxDamage !== null) {
                        applyLootThresholdEverywhere(
                            catalogue,
                            monsterKey,
                            maxDamage
                        );

                        addLog(
                            `${monster.name}: max-loot target = ${formatDamageCompact(maxDamage)}`
                        );
                    } else {
                        addLog(`${monster.name}: no DMG req threshold found.`);
                    }
                }

                if (needModel) {
                    try {
                        const damageModel = DamageEstimator.parseMonster(battleDoc);
                        applyDamageModelEverywhere(
                            catalogue,
                            monsterKey,
                            damageModel,
                            battleUrl
                        );
                    } catch (modelError) {
                        addLog(`${monster.name}: damage model unavailable (${modelError.message}).`);
                    }
                }

                saveCatalogue(catalogue);
                renderTargetManager();
            } catch (error) {
                addLog(
                    `${monster.name}: battle-data scan failed (${error.message}).`
                );
            }

            await sleep(LOOT_REQUEST_DELAY_MS);
        }
    }

    /* ============================================================
       FULL BACKGROUND SCAN
    ============================================================ */

    async function scanEverything({ forceLootRefresh = false } = {}) {
        if (scanning) return;

        scanning = true;
        updateButtons();

        const oldCatalogue = loadCatalogue();
        const cachedInfo = collectCachedMonsterInfo(oldCatalogue);

        const catalogue = emptyCatalogue();

        // monsterKey -> one usable battle URL
        const battleRepresentatives = new Map();

        try {
            setStatus('Automatic background scan started...');

            // If one source errors, still try the other.
            try {
                await scanDungeons(
                    catalogue,
                    cachedInfo,
                    battleRepresentatives
                );
            } catch (error) {
                addLog(`Dungeon scan error: ${error.message}`);
            }

            try {
                await scanGates(
                    catalogue,
                    cachedInfo,
                    battleRepresentatives
                );
            } catch (error) {
                addLog(`Gate scan error: ${error.message}`);
            }

            saveCatalogue(catalogue);
            renderTargetManager();

            await scanLootThresholds(
                catalogue,
                battleRepresentatives,
                forceLootRefresh
            );

            saveCatalogue(catalogue);

            localStorage.setItem(
                STORAGE_LAST_SCAN,
                String(Date.now())
            );

            renderTargetManager();

            setStatus('Catalogue is up to date.');
            addLog('Automatic background scan complete.');

        } catch (error) {
            setStatus(`Scan error: ${error.message}`, true);
            addLog(`ERROR: ${error.message}`);
        } finally {
            scanning = false;
            updateButtons();
        }
    }

    function startAutomaticScan() {
        const catalogue = loadCatalogue();

        const hasAnyCatalogue =
            Object.keys(catalogue.dungeons || {}).length > 0 ||
            Object.keys(catalogue.gates || {}).length > 0;

        if (!hasAnyCatalogue) {
            setStatus('Building PvE catalogue for the first time...');
            addLog('No cache found. Starting first automatic scan.');

            setTimeout(() => scanEverything(), 250);
            return;
        }

        // Existing catalogue is cache-first. Do NOT launch a full dungeon /
        // gate rescan merely because the page was reloaded. Full rescans are
        // expensive and are now performed only by the Refresh button (or on
        // the very first run when no catalogue exists).
        //
        // Live monster IDs used for autoattack are still resolved directly
        // from each monster's source page immediately before attacking, so
        // disabling the page-load catalogue rescan does not make attacks use
        // stale monster IDs.
        setStatus('Catalogue loaded from cache.');
    }

    /* ============================================================
       TARGET SETTINGS
    ============================================================ */

    /*
     * Path examples:
     *
     * dungeon|shadowbridge warrens|1|gribble junk-magus
     * gate|olympus|domain:11|moonlit huntress
     */
    function makeSelectionPath(sourceType, sourceKey, sectionKey, monsterKey) {
        return [
            sourceType,
            sourceKey,
            String(sectionKey),
            monsterKey
        ].join('|');
    }

    function isMonsterSelected(targets, path) {
        return Boolean(targets.selections[path]);
    }

    function setMonsterSelected(targets, path, selected) {
        targets.selections[path] = Boolean(selected);
    }

    function getManualDamageTarget(targets, monsterKey) {
        const value = Number(targets.manualDamageTargets[monsterKey]);

        return Number.isFinite(value)
            ? value
            : null;
    }

    function setManualDamageTarget(targets, monsterKey, rawValue) {
        const text = String(rawValue ?? '').trim();

        // Blank => return to Auto.
        if (!text) {
            delete targets.manualDamageTargets[monsterKey];

            return {
                valid: true,
                manual: false,
                value: null
            };
        }

        const parsed = parseDamageValue(text);

        if (parsed === null) {
            return {
                valid: false,
                manual: true,
                value: null
            };
        }

        targets.manualDamageTargets[monsterKey] = parsed;

        return {
            valid: true,
            manual: true,
            value: parsed
        };
    }

    /* ============================================================
       UI DATA MODEL
    ============================================================ */

    function buildUiSources(catalogue) {
        const sources = [];

        const dungeonChildren = Object.entries(catalogue.dungeons || {})
            .map(([sourceKey, dungeon]) => ({
                sourceType: 'dungeon',
                sourceKey,
                name: dungeon.name,
                sections: Object.entries(dungeon.locations || {})
                    .map(([sectionKey, section]) => ({
                        sectionKey,
                        name: section.name,
                        locked: false,
                        monsters: section.monsters || {}
                    }))
                    .sort((a, b) => Number(a.sectionKey) - Number(b.sectionKey))
            }))
            .sort((a, b) => a.name.localeCompare(b.name));

        if (dungeonChildren.length) {
            sources.push({
                groupKey: 'dungeons',
                name: 'Guild Dungeons',
                icon: '🏰',
                children: dungeonChildren
            });
        }

        const gateChildren = Object.entries(catalogue.gates || {})
            .map(([sourceKey, gate]) => ({
                sourceType: 'gate',
                sourceKey,
                name: gate.name,
                sections: Object.entries(gate.sections || {})
                    .map(([sectionKey, section]) => ({
                        sectionKey,
                        name: section.name,
                        locked: Boolean(section.locked),
                        monsters: section.monsters || {}
                    }))
                    .sort((a, b) => {
                        const aw = Number(a.sectionKey.split(':').pop());
                        const bw = Number(b.sectionKey.split(':').pop());

                        if (Number.isFinite(aw) && Number.isFinite(bw)) {
                            return aw - bw;
                        }

                        return a.name.localeCompare(b.name);
                    })
            }))
            .sort((a, b) => a.name.localeCompare(b.name));

        if (gateChildren.length) {
            sources.push({
                groupKey: 'gates',
                name: 'Gates',
                icon: '🚪',
                children: gateChildren
            });
        }

        return sources;
    }

    /* ============================================================
       CHECKBOX STATE
    ============================================================ */

    function calculateCheckboxState(checkboxes) {
        const list = [...checkboxes].filter(Boolean);

        if (!list.length) {
            return {
                checked: false,
                indeterminate: false
            };
        }

        const checkedCount = list.filter(x => x.checked).length;

        return {
            checked: checkedCount === list.length,
            indeterminate:
                checkedCount > 0 &&
                checkedCount < list.length
        };
    }

    function updateParentCheckboxStates() {
        if (!resultsElement) return;

        // Sections / locations / waves.
        for (const section of resultsElement.querySelectorAll('.pve-section-block')) {
            const parent = section.querySelector(
                ':scope > .pve-section-header .pve-section-checkbox'
            );

            if (!parent) continue;

            const children = section.querySelectorAll(
                ':scope > .pve-monster-list .pve-monster-checkbox'
            );

            const state = calculateCheckboxState(children);

            parent.checked = state.checked;
            parent.indeterminate = state.indeterminate;
        }

        // Individual dungeon / gate.
        for (const source of resultsElement.querySelectorAll('.pve-source-block')) {
            const parent = source.querySelector(
                ':scope > .pve-source-header .pve-source-checkbox'
            );

            if (!parent) continue;

            const children = source.querySelectorAll('.pve-monster-checkbox');
            const state = calculateCheckboxState(children);

            parent.checked = state.checked;
            parent.indeterminate = state.indeterminate;
        }

        // Guild Dungeons / Gates top-level groups.
        for (const group of resultsElement.querySelectorAll('.pve-group-block')) {
            const parent = group.querySelector(
                ':scope > .pve-group-header .pve-group-checkbox'
            );

            if (!parent) continue;

            const children = group.querySelectorAll('.pve-monster-checkbox');
            const state = calculateCheckboxState(children);

            parent.checked = state.checked;
            parent.indeterminate = state.indeterminate;
        }
    }

    let staminaRefreshTimer = null;
    let damageEstimatorStatePromise = null;

    function formatStamina(value) {
        const n = Number(value);
        if (!Number.isFinite(n) || n < 0) return '—';
        return Math.ceil(n).toLocaleString('en-US');
    }

    function scheduleStaminaEstimateRefresh(forceState = false) {
        clearTimeout(staminaRefreshTimer);
        staminaRefreshTimer = setTimeout(() => {
            refreshStaminaEstimates(forceState).catch(error => {
                addLog(`Stamina estimator error: ${error.message || error}`);
            });
        }, 120);
    }

    async function getDamageEstimatorState(force = false) {
        if (force) damageEstimatorStatePromise = null;

        if (!damageEstimatorStatePromise) {
            damageEstimatorStatePromise = DamageEstimator.collectState(force)
                .catch(error => {
                    damageEstimatorStatePromise = null;
                    throw error;
                });
        }

        return damageEstimatorStatePromise;
    }

    async function refreshStaminaEstimates(forceState = false) {
        if (!resultsElement) return;

        const cells = [...resultsElement.querySelectorAll('.pve-stamina-estimate')];
        if (!cells.length) return;

        const catalogue = loadCatalogue();
        const targets = loadTargets();
        const uniqueMonsters = getUniqueMonsters(catalogue);

        const computable = cells.filter(cell => {
            const monster = uniqueMonsters.get(cell.dataset.monsterKey);
            if (!monster?.damageModel) return false;

            const manual = getManualDamageTarget(targets, cell.dataset.monsterKey);
            const targetDamage = manual !== null
                ? manual
                : (Number(monster.autoDamageTarget) || null);

            return targetDamage !== null;
        });

        for (const cell of cells) {
            cell.textContent = '💪 —';
            cell.title = 'Waiting for a damage target and monster battle data.';
        }

        if (!computable.length) return;

        let state;
        try {
            state = await getDamageEstimatorState(forceState);
        } catch (error) {
            for (const cell of computable) {
                cell.textContent = '💪 ERR';
                cell.title = `Could not load current character/equipment/pet data: ${error.message || error}`;
            }
            return;
        }

        for (const cell of computable) {
            const monsterKey = cell.dataset.monsterKey;
            const monster = uniqueMonsters.get(monsterKey);
            if (!monster?.damageModel) continue;

            const manual = getManualDamageTarget(targets, monsterKey);
            const targetDamage = manual !== null
                ? manual
                : (Number(monster.autoDamageTarget) || null);

            if (targetDamage === null) continue;

            try {
                const result = DamageEstimator.calculateForTarget(
                    state,
                    monster.damageModel,
                    targetDamage
                );

                const expected = result.stamina?.expectedStamina;
                if (!Number.isFinite(expected)) throw new Error('No finite stamina estimate');

                cell.textContent = `💪 ${formatStamina(expected)}`;
                cell.title = [
                    `Expected stamina to deal ${formatDamageCompact(targetDamage)} to ${monster.name}: ${formatStamina(expected)}`,
                    `Expected effective 1-stamina Slash: ${Math.round(result.core.effectiveDamage).toLocaleString('en-US')}`,
                    result.hasOurovyrn
                        ? `Dawn Brand strategy: ${result.stamina.bigAttacks} × 1000-stamina setup Slash${result.stamina.bigAttacks === 1 ? '' : 'es'}`
                        : '',
                    result.core.orryphos
                        ? `Includes Orryphos free-stamina/echo expectation`
                        : ''
                ].filter(Boolean).join('\n');
            } catch (error) {
                cell.textContent = '💪 —';
                cell.title = `Estimate unavailable: ${error.message || error}`;
            }
        }
    }

    /* ============================================================
       RENDER
    ============================================================ */

    function renderMonsterRowHtml(source, section, monsterKey, monster, targets, flattened = false) {
        const selectionPath = makeSelectionPath(
            source.sourceType,
            source.sourceKey,
            section.sectionKey,
            monsterKey
        );

        const selected = isMonsterSelected(targets, selectionPath);
        const manualDamage = getManualDamageTarget(targets, monsterKey);
        const autoDamage = Number(monster.autoDamageTarget) || null;
        const isManual = manualDamage !== null;
        const effectiveDamage = isManual ? manualDamage : autoDamage;
        const displayDamage = effectiveDamage !== null
            ? formatDamageCompact(effectiveDamage)
            : '';

        let imageHtml = '<span class="pve-img-spacer"></span>';
        if (monster.image) {
            const imageUrl = absoluteUrl(monster.image);
            imageHtml = `
                <img
                    src="${escapeHtml(imageUrl)}"
                    class="pve-monster-img"
                    alt=""
                >
            `;
        }

        let typeBadge = '';
        if (monster.type === 'timed_boss') {
            typeBadge = '<span class="pve-badge timed">Timed</span>';
        } else if (monster.type === 'boss') {
            typeBadge = '<span class="pve-badge boss">Boss</span>';
        }

        return `
            <div
                class="pve-monster-row ${flattened ? 'pve-flat-monster' : ''}"
                data-monster-key="${escapeHtml(monsterKey)}"
                data-selection-path="${escapeHtml(selectionPath)}"
            >
                <input
                    type="checkbox"
                    class="pve-monster-checkbox"
                    ${selected ? 'checked' : ''}
                >

                ${imageHtml}

                <div class="pve-monster-label">
                    <span class="pve-monster-name">
                        ${escapeHtml(monster.name)}
                    </span>
                    ${typeBadge}
                </div>

                <div class="pve-damage-wrap">
                    <input
                        type="text"
                        class="pve-damage-input ${isManual ? 'manual' : 'auto'}"
                        data-monster-key="${escapeHtml(monsterKey)}"
                        value="${escapeHtml(displayDamage)}"
                        placeholder="${
                            autoDamage !== null
                                ? escapeHtml(formatDamageCompact(autoDamage))
                                : 'Scanning...'
                        }"
                        title="Examples: 3M, 2.5B, 1T. Clear to restore automatic threshold."
                    >

                    <span class="pve-damage-mode ${isManual ? 'manual' : 'auto'}">
                        ${isManual ? 'MANUAL' : 'AUTO'}
                    </span>
                </div>

                <span
                    class="pve-stamina-estimate"
                    data-monster-key="${escapeHtml(monsterKey)}"
                    title="Calculating expected stamina..."
                >💪 …</span>
            </div>
        `;
    }

    function renderTargetManager() {
        if (!resultsElement) return;

        const catalogue = loadCatalogue();
        const targets = loadTargets();
        const groups = buildUiSources(catalogue);

        if (!groups.length) {
            resultsElement.innerHTML = `
                <div class="pve-empty">
                    Building catalogue automatically...
                </div>
            `;
            return;
        }

        let html = '';

        for (const group of groups) {
            html += `
                <div class="pve-group-block" data-group="${escapeHtml(group.groupKey)}">
                    <div class="pve-group-header">
                        <input type="checkbox" class="pve-group-checkbox">
                        <span class="pve-arrow">▸</span>
                        <strong>${group.icon} ${escapeHtml(group.name)}</strong>
                    </div>

                    <div class="pve-group-content" style="display:none;">
            `;

            for (const source of group.children) {
                html += `
                    <div
                        class="pve-source-block"
                        data-source-type="${escapeHtml(source.sourceType)}"
                        data-source-key="${escapeHtml(source.sourceKey)}"
                    >
                        <div class="pve-source-header">
                            <input type="checkbox" class="pve-source-checkbox">
                            <span class="pve-arrow">▸</span>
                            <strong>${escapeHtml(source.name)}</strong>
                        </div>

                        <div class="pve-source-content" style="display:none;">
                `;

                for (const section of source.sections) {
                    const monsterEntries = Object.entries(section.monsters || {})
                        .sort(([, a], [, b]) => a.name.localeCompare(b.name));

                    // UX rule: if a zone contains exactly one monster, skip the
                    // redundant sub-location/domain row and show the monster directly.
                    if (!section.locked && monsterEntries.length === 1) {
                        const [monsterKey, monster] = monsterEntries[0];
                        html += `
                            <div class="pve-flat-monster-wrap" data-section-key="${escapeHtml(section.sectionKey)}">
                                ${renderMonsterRowHtml(source, section, monsterKey, monster, targets, true)}
                            </div>
                        `;
                        continue;
                    }

                    const showSectionCheckbox = !section.locked && monsterEntries.length > 1;
                    const lockedText = section.locked
                        ? '<span class="pve-locked">Locked</span>'
                        : '';

                    html += `
                        <div
                            class="pve-section-block"
                            data-section-key="${escapeHtml(section.sectionKey)}"
                        >
                            <div class="pve-section-header">
                                ${
                                    showSectionCheckbox
                                        ? '<input type="checkbox" class="pve-section-checkbox">'
                                        : '<span class="pve-checkbox-spacer"></span>'
                                }

                                <span class="pve-arrow">▸</span>
                                <span>${escapeHtml(section.name)}</span>
                                ${lockedText}

                                <small>
                                    ${
                                        section.locked
                                            ? ''
                                            : `${monsterEntries.length} monsters`
                                    }
                                </small>
                            </div>

                            <div class="pve-monster-list" style="display:none;">
                    `;

                    if (section.locked) {
                        html += `
                            <div class="pve-empty">
                                Domain is visible but not unlocked yet.
                            </div>
                        `;
                    } else if (!monsterEntries.length) {
                        html += `
                            <div class="pve-empty">
                                No monsters discovered yet.
                            </div>
                        `;
                    }

                    for (const [monsterKey, monster] of monsterEntries) {
                        html += renderMonsterRowHtml(source, section, monsterKey, monster, targets, false);
                    }

                    html += `
                            </div>
                        </div>
                    `;
                }

                html += `
                        </div>
                    </div>
                `;
            }

            html += `
                    </div>
                </div>
            `;
        }

        resultsElement.innerHTML = html;

        bindTargetEvents();
        updateParentCheckboxStates();
        scheduleStaminaEstimateRefresh(false);
    }

    /* ============================================================
       UI EVENTS
    ============================================================ */

    function toggleBlock(header, contentSelector) {
        const block = header.parentElement;
        const content = block.querySelector(contentSelector);
        const arrow = header.querySelector('.pve-arrow');

        if (!content) return;

        const hidden = content.style.display === 'none';

        content.style.display = hidden ? '' : 'none';
        if (arrow) arrow.textContent = hidden ? '▾' : '▸';
    }

    function bindTargetEvents() {
        if (!resultsElement) return;

        // Monster checkbox.
        for (const checkbox of resultsElement.querySelectorAll('.pve-monster-checkbox')) {
            checkbox.addEventListener('change', event => {
                const row = event.target.closest('.pve-monster-row');
                if (!row) return;

                const targets = loadTargets();

                setMonsterSelected(
                    targets,
                    row.dataset.selectionPath,
                    event.target.checked
                );

                saveTargets(targets);
                updateParentCheckboxStates();
            });
        }

        // Section checkbox - only exists when there are 2+ monsters.
        for (const checkbox of resultsElement.querySelectorAll('.pve-section-checkbox')) {
            checkbox.addEventListener('change', event => {
                const section = event.target.closest('.pve-section-block');
                if (!section) return;

                const targets = loadTargets();

                for (const child of section.querySelectorAll('.pve-monster-checkbox')) {
                    child.checked = event.target.checked;

                    const row = child.closest('.pve-monster-row');

                    setMonsterSelected(
                        targets,
                        row.dataset.selectionPath,
                        event.target.checked
                    );
                }

                saveTargets(targets);
                updateParentCheckboxStates();
            });
        }

        // Source checkbox.
        for (const checkbox of resultsElement.querySelectorAll('.pve-source-checkbox')) {
            checkbox.addEventListener('change', event => {
                const source = event.target.closest('.pve-source-block');
                if (!source) return;

                const targets = loadTargets();

                for (const child of source.querySelectorAll('.pve-monster-checkbox')) {
                    child.checked = event.target.checked;

                    const row = child.closest('.pve-monster-row');

                    setMonsterSelected(
                        targets,
                        row.dataset.selectionPath,
                        event.target.checked
                    );
                }

                saveTargets(targets);
                updateParentCheckboxStates();
            });
        }

        // Group checkbox.
        for (const checkbox of resultsElement.querySelectorAll('.pve-group-checkbox')) {
            checkbox.addEventListener('change', event => {
                const group = event.target.closest('.pve-group-block');
                if (!group) return;

                const targets = loadTargets();

                for (const child of group.querySelectorAll('.pve-monster-checkbox')) {
                    child.checked = event.target.checked;

                    const row = child.closest('.pve-monster-row');

                    setMonsterSelected(
                        targets,
                        row.dataset.selectionPath,
                        event.target.checked
                    );
                }

                saveTargets(targets);
                updateParentCheckboxStates();
            });
        }

        // Auto/manual damage input.
        for (const input of resultsElement.querySelectorAll('.pve-damage-input')) {
            input.addEventListener('focus', event => {
                if (event.target.classList.contains('auto')) {
                    event.target.select();
                }
            });

            input.addEventListener('input', event => {
                const text = event.target.value.trim();
                const invalid =
                    Boolean(text && parseDamageValue(text) === null);

                event.target.classList.toggle(
                    'invalid',
                    invalid
                );
            });

            input.addEventListener('change', event => {
                commitDamageInput(event.target);
            });
        }

        // All lists start collapsed after every page load / rerender.
        for (const header of resultsElement.querySelectorAll('.pve-group-header')) {
            header.addEventListener('click', event => {
                if (event.target.matches('input')) return;

                toggleBlock(
                    header,
                    ':scope > .pve-group-content'
                );
            });
        }

        for (const header of resultsElement.querySelectorAll('.pve-source-header')) {
            header.addEventListener('click', event => {
                if (event.target.matches('input')) return;

                toggleBlock(
                    header,
                    ':scope > .pve-source-content'
                );
            });
        }

        for (const header of resultsElement.querySelectorAll('.pve-section-header')) {
            header.addEventListener('click', event => {
                if (event.target.matches('input')) return;

                toggleBlock(
                    header,
                    ':scope > .pve-monster-list'
                );
            });
        }
    }

    function commitDamageInput(input) {
        const monsterKey = input.dataset.monsterKey;
        const targets = loadTargets();

        const result = setManualDamageTarget(
            targets,
            monsterKey,
            input.value
        );

        if (!result.valid) {
            input.classList.add('invalid');

            setStatus(
                'Invalid damage value. Use e.g. 3M, 2.5B, 1T or a full number.',
                true
            );

            return;
        }

        saveTargets(targets);

        // Do NOT rerender the whole target tree here. A rerender collapses every
        // open group/source/location, which is very annoying when editing several
        // thresholds. Update only this row in place and keep the user's current
        // expansion state untouched.
        const wrap = input.closest('.pve-damage-wrap');
        const mode = wrap?.querySelector('.pve-damage-mode');

        input.classList.remove('invalid', 'manual', 'auto');

        if (result.manual) {
            input.classList.add('manual');
            input.value = formatDamageCompact(result.value);

            if (mode) {
                mode.classList.remove('auto');
                mode.classList.add('manual');
                mode.textContent = 'MANUAL';
            }

            setStatus(
                `Manual target set to ${formatDamageCompact(result.value)}.`
            );
        } else {
            input.classList.add('auto');

            // The placeholder is the current automatic max-loot target.
            input.value = input.placeholder === 'Scanning...'
                ? ''
                : input.placeholder;

            if (mode) {
                mode.classList.remove('manual');
                mode.classList.add('auto');
                mode.textContent = 'AUTO';
            }

            setStatus('Automatic max-loot target restored.');
        }

        scheduleStaminaEstimateRefresh(false);
    }

    /* ============================================================
       EXPORT
    ============================================================ */

    function exportJson() {
        const data = {
            catalogue: loadCatalogue(),
            targets: loadTargets()
        };

        const blob = new Blob(
            [JSON.stringify(data, null, 2)],
            { type: 'application/json' }
        );

        const url = URL.createObjectURL(blob);
        const a = document.createElement('a');

        a.href = url;
        a.download = 'veyra-pve-targets.json';

        document.body.appendChild(a);
        a.click();
        a.remove();

        URL.revokeObjectURL(url);
    }

    /* ============================================================
       PANEL STATE
    ============================================================ */

    function isPanelMinimized() {
        return localStorage.getItem(STORAGE_PANEL_MINIMIZED) === 'true';
    }

    function setPanelMinimized(value) {
        localStorage.setItem(
            STORAGE_PANEL_MINIMIZED,
            value ? 'true' : 'false'
        );
    }

    function updateButtons() {
        if (!refreshButton) return;

        refreshButton.disabled = scanning;
        refreshButton.textContent =
            scanning
                ? '⏳ Refreshing...'
                : '↻ Refresh now';
    }


    /* ============================================================
       AUTOMATIC ATTACK CONTROLLER
    ============================================================ */

    function loadAttackAutomation() {
        try {
            const parsed = JSON.parse(localStorage.getItem(STORAGE_ATTACK_AUTOMATION) || 'null');
            if (!parsed || typeof parsed !== 'object') {
                return { active: false, queue: [], index: 0 };
            }

            parsed.queue ||= [];
            parsed.index = Math.max(0, Number(parsed.index) || 0);
            parsed.active = Boolean(parsed.active);

            return parsed;
        } catch {
            return { active: false, queue: [], index: 0 };
        }
    }

    function saveAttackAutomation(state) {
        localStorage.setItem(STORAGE_ATTACK_AUTOMATION, JSON.stringify(state));
        updateAttackButton();
    }

    function stopAttackAutomation(message = 'Attack automation stopped.') {
        const state = loadAttackAutomation();
        state.active = false;
        state.lastStatus = message;
        saveAttackAutomation(state);
        setStatus(message);
        addLog(message);
    }

    function updateAttackButton() {
        if (!attackButton) return;

        const state = loadAttackAutomation();

        if (state.active) {
            attackButton.textContent = '■ Stop Autoattack';
            attackButton.classList.add('active');
            attackButton.title = 'Stop the current multi-target attack queue.';
        } else {
            attackButton.textContent = '⚔ Autoattack Selected';
            attackButton.classList.remove('active');
            attackButton.title = 'Attack every selected, currently attackable monster to its configured damage threshold.';
        }
    }

    function selectedAttackQueue() {
        const catalogue = loadCatalogue();
        const targets = loadTargets();
        const queue = [];

        function consider(sourceType, sourceKey, sectionKey, monsterKey, monster) {
            const selectionPath = makeSelectionPath(
                sourceType,
                sourceKey,
                sectionKey,
                monsterKey
            );

            if (!isMonsterSelected(targets, selectionPath)) return;

            const manual = getManualDamageTarget(targets, monsterKey);
            const targetDamage = manual !== null
                ? manual
                : (Number(monster.autoDamageTarget) || null);

            if (!targetDamage) {
                addLog(`${monster.name}: selected, but no damage target is available yet.`);
                return;
            }

            const attackBattleUrl =
                sourceType === 'dungeon'
                    ? (monster.activeBattleUrl || '')
                    : (monster.representativeBattleUrl || '');

            const sourceUrl =
                sourceType === 'gate'
                    ? (catalogue.gates?.[sourceKey]?.sections?.[String(sectionKey)]?.url || '')
                    : '';

            if (sourceType === 'dungeon' && !attackBattleUrl) {
                addLog(
                    `${monster.name}: selected, but there is no active dungeon instance for this monster.`
                );
                return;
            }

            if (sourceType !== 'dungeon' && !sourceUrl) {
                addLog(
                    `${monster.name}: selected, but its live source page is not cached yet.`
                );
                return;
            }

            if (!monster.damageModel) {
                addLog(`${monster.name}: selected, but its battle damage model has not been scanned yet.`);
                return;
            }

            let queueDgmid = monster.dgmid ? String(monster.dgmid) : '';
            let queueInstanceId = monster.instanceId ? String(monster.instanceId) : '';

            // Dungeon catalogue entries can retain IDs from an older ended
            // instance while activeBattleUrl correctly points at the current
            // live instance. For automation, the active URL is authoritative.
            if (sourceType === 'dungeon' && attackBattleUrl) {
                const activeIds = dungeonIdsFromBattleUrl(attackBattleUrl);
                if (activeIds.dgmid) queueDgmid = String(activeIds.dgmid);
                if (activeIds.instanceId) queueInstanceId = String(activeIds.instanceId);
            }

            const baseQueueItem = {
                sourceType,
                sourceKey,
                sectionKey: String(sectionKey),
                monsterKey,
                name: monster.name,
                targetDamage,
                battleUrl: attackBattleUrl,
                sourceUrl,
                monsterId: monster.monsterId ? String(monster.monsterId) : '',
                dgmid: queueDgmid,
                instanceId: queueInstanceId,
                damageModel: monster.damageModel
            };

            if (sourceType === 'dungeon') {
                const activeEncounters =
                    Array.isArray(monster.activeEncounters) && monster.activeEncounters.length
                        ? monster.activeEncounters
                        : [{
                            battleUrl: attackBattleUrl,
                            dgmid: queueDgmid,
                            instanceId: queueInstanceId
                        }];

                const seen = new Set();

                for (const encounter of activeEncounters) {
                    if (!encounter?.battleUrl) continue;

                    const ids = dungeonIdsFromBattleUrl(encounter.battleUrl);
                    const encounterDgmid = String(ids.dgmid || encounter.dgmid || '');
                    const encounterInstanceId = String(ids.instanceId || encounter.instanceId || '');
                    const encounterKey = `${encounterInstanceId}:${encounterDgmid}`;

                    if (!encounterDgmid || !encounterInstanceId || seen.has(encounterKey)) {
                        continue;
                    }
                    seen.add(encounterKey);

                    queue.push({
                        ...baseQueueItem,
                        battleUrl: encounter.battleUrl,
                        dgmid: encounterDgmid,
                        instanceId: encounterInstanceId
                    });
                }

                if (!seen.size) {
                    queue.push(baseQueueItem);
                }
            } else {
                queue.push(baseQueueItem);
            }
        }

        for (const [dungeonKey, dungeon] of Object.entries(catalogue.dungeons || {})) {
            for (const [locationKey, locationData] of Object.entries(dungeon.locations || {})) {
                for (const [monsterKey, monster] of Object.entries(locationData.monsters || {})) {
                    consider('dungeon', dungeonKey, locationKey, monsterKey, monster);
                }
            }
        }

        for (const [gateKey, gate] of Object.entries(catalogue.gates || {})) {
            for (const [sectionKey, section] of Object.entries(gate.sections || {})) {
                for (const [monsterKey, monster] of Object.entries(section.monsters || {})) {
                    consider('gate', gateKey, sectionKey, monsterKey, monster);
                }
            }
        }

        const dungeonItems = queue.filter(item => item.sourceType === 'dungeon');
        const gateItems = queue.filter(item => item.sourceType !== 'dungeon');

        addLog(
            `Queue build: ${queue.length} attackable selected target(s) · ` +
            `${dungeonItems.length} dungeon · ${gateItems.length} gate/timed.`
        );

        if (dungeonItems.length) {
            addLog(
                `Dungeon queue: ` +
                dungeonItems.map((item, index) => {
                    const ids = dungeonIdsFromBattleUrl(item.battleUrl);
                    return `${index + 1}. ${item.name} ` +
                        `[dgmid=${ids.dgmid || item.dgmid || '?'} ` +
                        `instance=${ids.instanceId || item.instanceId || '?'}]`;
                }).join(' → ')
            );
        } else {
            addLog(`Dungeon queue: NONE.`);
        }

        return queue;
    }

    function startAttackAutomation() {
        const queue = selectedAttackQueue();

        if (!queue.length) {
            setStatus(
                'No attackable selected monsters. Select monsters and wait for their damage target/battle data to finish scanning.',
                true
            );
            return;
        }

        const state = {
            active: true,
            queue,
            index: 0,
            startedAt: Date.now(),
            lastStatus: `Starting direct autoattacks for ${queue.length} selected monster${queue.length === 1 ? '' : 's'}.`
        };

        saveAttackAutomation(state);
        addLog(state.lastStatus);
        addLog(
            `Attack queue (${queue.length}): ` +
            queue.map(
                (item, index) =>
                    `${index + 1}. [${item.sourceType === 'dungeon' ? 'DUNGEON' : 'GATE'}] ${item.name}`
            ).join(' → ')
        );
        setStatus(state.lastStatus);

        runHeadlessAttackAutomation().catch(error => {
            stopAttackAutomation(`Attack automation error: ${error.message || error}`);
        });
    }

    function currentAttackItem(state = loadAttackAutomation()) {
        return state.queue?.[state.index] || null;
    }

    function dungeonInstanceIdFromBattleUrl(url) {
        try {
            return new URL(url, location.origin).searchParams.get('instance_id') || '';
        } catch {
            return '';
        }
    }

    async function isDungeonQueueItemStillActive(item) {
        if (!item || item.sourceType !== 'dungeon') return true;

        const instanceId = dungeonInstanceIdFromBattleUrl(item.battleUrl);
        if (!instanceId) return false;

        try {
            const html = await fetchHtml(DUNGEONS_URL);
            const doc = parseHtml(html);
            const instances = extractDungeonInstances(doc);

            return instances.some(instance =>
                String(instance.instanceId) === String(instanceId) &&
                instance.status === 'active'
            );
        } catch (error) {
            // Network failure is not evidence that the dungeon ended. Keep the
            // current target rather than skipping it based on an unavailable
            // status page.
            addLog(
                `${item.name}: could not re-check dungeon instance status (${error.message}); keeping target.`
            );
            return true;
        }
    }

    async function skipInactiveDungeonTargets(state) {
        while (state.active) {
            const item = currentAttackItem(state);
            if (!item) return state;

            if (item.sourceType !== 'dungeon') return state;

            const active = await isDungeonQueueItemStillActive(item);
            if (active) return state;

            addLog(`${item.name}: dungeon instance is completed/ended; skipping.`);
            setStatus(`${item.name}: dungeon already completed — skipping.`);

            state.index += 1;
            state.runtimeBattleUrl = null;
            state.runtimeDamage = null;
            state.runtimeStamina = null;

            if (state.index >= state.queue.length) {
                state.active = false;
                state.lastStatus = 'All selected active targets processed.';
                saveAttackAutomation(state);
                return state;
            }

            saveAttackAutomation(state);
        }

        return state;
    }

    function parseVisibleNumber(raw) {
        const cleaned = String(raw ?? '').replace(/[^0-9.-]/g, '');
        const value = Number(cleaned);
        return Number.isFinite(value) ? value : 0;
    }

    function getCurrentBattleDamage(state = null, item = null) {
        if (
            state &&
            item &&
            state.runtimeBattleUrl &&
            sameBattleUrl(state.runtimeBattleUrl, item.battleUrl) &&
            Number.isFinite(Number(state.runtimeDamage))
        ) {
            return Number(state.runtimeDamage);
        }

        return parseVisibleNumber(
            document.querySelector('#yourDamageValue')?.textContent || '0'
        );
    }

    function getVisibleStaminaStats() {
        const staminaSpan = document.querySelector('#stamina_span');

        if (staminaSpan) {
            const current = parseVisibleNumber(staminaSpan.textContent);

            // Normal battle layout:
            // <span class="gtb-value"><span id="stamina_span">7,222</span> / 12,230</span>
            const parentText = staminaSpan.parentElement?.textContent || '';
            const pair = parentText.match(/([\d,]+)\s*\/\s*([\d,]+)/);

            if (pair) {
                return {
                    current: parseVisibleNumber(pair[1]),
                    max: parseVisibleNumber(pair[2])
                };
            }

            // The page also exposes maxStam in its stamina timer script. We
            // normally do not need this fallback, but keep current usable even
            // if a layout variant changes the parent text.
            return { current, max: 0 };
        }

        // Fallback for side-drawer / alternate layouts.
        const candidates = [...document.querySelectorAll('.gtb-stat, .side-chip')];
        for (const el of candidates) {
            const raw = el.textContent || '';
            const pair = raw.match(/([\d,]+)\s*\/\s*([\d,]+)/);
            if (!pair) continue;

            return {
                current: parseVisibleNumber(pair[1]),
                max: parseVisibleNumber(pair[2])
            };
        }

        return { current: 0, max: 0 };
    }

    function getCurrentVisibleStamina() {
        return getVisibleStaminaStats().current;
    }

    function getAutomationStaminaStats(state = null, item = null) {
        const visible = getVisibleStaminaStats();

        if (
            state &&
            item &&
            state.runtimeBattleUrl &&
            sameBattleUrl(state.runtimeBattleUrl, item.battleUrl) &&
            Number.isFinite(Number(state.runtimeStamina))
        ) {
            return {
                current: Number(state.runtimeStamina),
                max: visible.max
            };
        }

        return visible;
    }

    async function refreshAutomationBattleSnapshot(state, item) {
        const response = await fetch(location.href, {
            method: 'GET',
            credentials: 'same-origin',
            cache: 'no-store'
        });

        if (!response.ok) return state;

        const html = await response.text();
        const doc = new DOMParser().parseFromString(html, 'text/html');

        const dmg = parseVisibleNumber(
            doc.querySelector('#yourDamageValue')?.textContent || ''
        );

        const staminaSpan = doc.querySelector('#stamina_span');
        let stamina = NaN;
        if (staminaSpan) {
            stamina = parseVisibleNumber(staminaSpan.textContent);
        }

        state.runtimeBattleUrl = item.battleUrl;
        if (Number.isFinite(dmg)) state.runtimeDamage = dmg;
        if (Number.isFinite(stamina)) state.runtimeStamina = stamina;
        saveAttackAutomation(state);

        return state;
    }

    function chooseDrainSlash(currentStamina, estimatedStaminaNeeded) {
        const current = Math.max(0, Math.floor(Number(currentStamina) || 0));
        const need = Math.max(1, Number(estimatedStaminaNeeded) || 1);

        // Drain with the largest available Slash that:
        //   1) we can actually afford with the stamina currently left, and
        //   2) is not larger than the predicted stamina still required.
        //
        // This naturally gives sequences such as:
        // 430 -> 200 -> 200 -> 10 -> 10 -> 10 -> potion.
        return SLASH_ATTACKS.find(
            attack => attack.cost <= current && attack.cost <= need
        ) || (
            current >= 1
                ? SLASH_ATTACKS[SLASH_ATTACKS.length - 1]
                : null
        );
    }

    function chooseSlashAttack(estimatedStaminaNeeded) {
        const need = Math.max(0, Number(estimatedStaminaNeeded) || 0);
        return SLASH_ATTACKS.find(attack => need >= attack.cost) || SLASH_ATTACKS[SLASH_ATTACKS.length - 1];
    }

    function isCurrentBattleClearlyDead() {
        const hpText = document.querySelector('#hpText')?.textContent || '';
        const hpPair = hpText.match(/([\d,]+)\s*\/\s*([\d,]+)/);
        if (hpPair && parseVisibleNumber(hpPair[1]) <= 0) return true;

        const bodyText = cleanText(document.body?.innerText || '').toLowerCase();
        if (
            bodyText.includes('monster is already dead') ||
            bodyText.includes('monster has been defeated') ||
            bodyText.includes('this monster is dead')
        ) {
            return true;
        }

        // A dead/finished battle commonly exposes loot but no attack bar.
        if (
            !document.querySelector('.attack-btn') &&
            document.querySelector('#loot-button')
        ) {
            return true;
        }

        return false;
    }

    function parseAttackResponse(raw) {
        const text = String(raw ?? '').replace(/^\uFEFF/, '').trim();
        if (!text) return null;

        try {
            return JSON.parse(text);
        } catch {}

        const first = text.indexOf('{');
        if (first < 0) return null;

        let depth = 0;
        let inString = false;
        let escaped = false;

        for (let i = first; i < text.length; i++) {
            const ch = text[i];

            if (inString) {
                if (escaped) escaped = false;
                else if (ch === '\\') escaped = true;
                else if (ch === '"') inString = false;
                continue;
            }

            if (ch === '"') inString = true;
            else if (ch === '{') depth++;
            else if (ch === '}') {
                depth--;
                if (depth === 0) {
                    try {
                        return JSON.parse(text.slice(first, i + 1));
                    } catch {
                        return null;
                    }
                }
            }
        }

        return null;
    }

    function buildDirectAttackPayload(slash) {
        if (
            typeof baseQS !== 'function' ||
            typeof ep !== 'function'
        ) {
            throw new Error('The battle page attack API helpers are not available.');
        }

        const fd = baseQS();
        fd.set('skill_id', String(slash.skillId));
        fd.set('stamina_cost', String(slash.cost));

        if (typeof IS_DG !== 'undefined' && IS_DG) {
            if (typeof BCFG === 'undefined') {
                throw new Error('Dungeon battle configuration is unavailable.');
            }
            fd.set('instance_id', String(BCFG.instanceId));
            fd.set('dgmid', String(BCFG.dgmid));
        }

        if (
            typeof BCFG !== 'undefined' &&
            BCFG.privateBoss &&
            BCFG.privateBoss.enabled
        ) {
            fd.set('csrf_token', String(BCFG.privateBoss.csrf || ''));
            fd.set(
                'request_token',
                (window.crypto && typeof window.crypto.randomUUID === 'function')
                    ? window.crypto.randomUUID()
                    : `${Date.now()}-${Math.random().toString(16).slice(2)}-${Math.random().toString(16).slice(2)}`
            );
        }

        return {
            endpoint: ep('ATTACK'),
            body: fd.toString()
        };
    }

    async function performBackgroundAttack(slash, attempt = 1) {
        const payload = buildDirectAttackPayload(slash);

        const response = await fetch(payload.endpoint, {
            method: 'POST',
            credentials: 'same-origin',
            headers: {
                'Content-Type': 'application/x-www-form-urlencoded'
            },
            body: payload.body
        });

        const raw = await response.text();
        const data = parseAttackResponse(raw);
        const message = cleanText(data?.message || raw || '');

        if (
            message === 'Invalid request.' &&
            attempt < 2
        ) {
            await sleep(350);
            return performBackgroundAttack(slash, attempt + 1);
        }

        if (!response.ok || !data) {
            throw new Error(
                message.slice(0, 180) ||
                `Attack request failed with HTTP ${response.status}.`
            );
        }

        return data;
    }

    async function useLargeStaminaPotion(state, item) {
        const potionBtn = [...document.querySelectorAll('.potion-use-btn')]
            .find(btn => normalizeKey(btn.dataset.name) === 'large stamina potion');

        if (!potionBtn) {
            throw new Error('Large Stamina Potion is not available on this battle page.');
        }

        const invId = potionBtn.dataset.inv;
        const qtySpan = invId
            ? document.getElementById(`pqty_${invId}`)
            : null;

        const quantity = qtySpan
            ? parseVisibleNumber(qtySpan.textContent)
            : parseVisibleNumber(potionBtn.dataset.max);

        if (!invId || quantity <= 0) {
            throw new Error('No Large Stamina Potions remaining.');
        }

        setStatus(`Using Large Stamina Potion (${Math.round(quantity)} remaining)...`);
        addLog('Using 1 × Large Stamina Potion because current stamina is below the chosen Slash cost.');

        const params = new URLSearchParams();
        params.set('inv_id', invId);

        const response = await fetch('/use_item.php', {
            method: 'POST',
            credentials: 'same-origin',
            headers: {
                'Content-Type': 'application/x-www-form-urlencoded'
            },
            body: params.toString()
        });

        const raw = await response.text();

        if (!response.ok || !raw.trim().toLowerCase().includes('success')) {
            throw new Error(`Large Stamina Potion failed: ${cleanText(raw).slice(0, 160) || `HTTP ${response.status}`}`);
        }

        // Keep the site visually unchanged. Re-fetch the battle page in the
        // background to obtain the authoritative post-potion stamina value,
        // then continue the controller without a visible reload.
        await sleep(250);
        await refreshAutomationBattleSnapshot(state, item);
        return true;
    }

    function sameBattleUrl(a, b) {
        try {
            const ua = new URL(a, location.origin);
            const ub = new URL(b, location.origin);

            if (ua.pathname !== ub.pathname) return false;

            const keys = ['id', 'dgmid', 'instance_id'];
            return keys.every(key => (ua.searchParams.get(key) || '') === (ub.searchParams.get(key) || ''));
        } catch {
            return false;
        }
    }

    async function moveToNextAttackTarget(state) {
        state.index += 1;

        if (state.index >= state.queue.length) {
            state.active = false;
            state.lastStatus = 'All selected monster damage targets reached.';
            saveAttackAutomation(state);
            setStatus(state.lastStatus);
            addLog(state.lastStatus);
            return;
        }

        const next = currentAttackItem(state);
        state.runtimeBattleUrl = null;
        state.runtimeDamage = null;
        state.runtimeStamina = null;
        state.lastStatus = `Moving to ${next.name} (${state.index + 1}/${state.queue.length}).`;
        saveAttackAutomation(state);
        setStatus(state.lastStatus);
        addLog(state.lastStatus);

        // Headless controller handles the next target without changing the
        // visible page.
    }


    const BATTLE_WORKER_MSG = 'veyra-pve-battle-worker-v1';
    let battleWorkerSeq = 0;

    function frameWorkerRequest(frame, action, payload = {}, timeoutMs = 3500) {
        return new Promise((resolve, reject) => {
            const id = `pve-${Date.now()}-${++battleWorkerSeq}`;
            let done = false;

            const cleanup = () => {
                window.removeEventListener('message', onMessage);
                clearTimeout(timer);
            };

            const onMessage = event => {
                if (event.origin !== location.origin) return;
                if (event.source !== frame.contentWindow) return;

                const msg = event.data || {};
                if (msg.type !== BATTLE_WORKER_MSG || msg.id !== id) return;

                done = true;
                cleanup();
                resolve(msg);
            };

            const timer = setTimeout(() => {
                if (done) return;
                cleanup();
                reject(
                    new Error(
                        `Hidden battle worker did not answer "${action}" within ${timeoutMs}ms.`
                    )
                );
            }, timeoutMs);

            window.addEventListener('message', onMessage);

            frame.contentWindow.postMessage({
                type: BATTLE_WORKER_MSG,
                id,
                action,
                ...payload
            }, location.origin);
        });
    }

    async function workerSnapshot(frame) {
        const response = await frameWorkerRequest(
            frame,
            'snapshot',
            {},
            3500
        );
        return response.snapshot || null;
    }

    function getHeadlessBattleFrame() {
        if (headlessBattleFrame && document.body.contains(headlessBattleFrame)) {
            return headlessBattleFrame;
        }

        const frame = document.createElement('iframe');
        frame.id = 'veyra-pve-headless-battle';
        frame.setAttribute('aria-hidden', 'true');
        frame.setAttribute('data-veyra-headless', '1');
        frame.tabIndex = -1;
        frame.style.cssText = [
            'position:fixed',
            'width:1px',
            'height:1px',
            'left:-10000px',
            'top:-10000px',
            'opacity:0',
            'pointer-events:none',
            'border:0'
        ].join(';');

        document.body.appendChild(frame);
        headlessBattleFrame = frame;
        return frame;
    }

    function waitForFrameLoad(frame, timeoutMs = 12000) {
        return new Promise((resolve, reject) => {
            let done = false;

            const timer = setTimeout(() => {
                if (done) return;
                done = true;
                reject(new Error('Hidden battle page timed out while loading.'));
            }, timeoutMs);

            frame.addEventListener('load', () => {
                if (done) return;
                done = true;
                clearTimeout(timer);
                setTimeout(() => resolve(frame), 120);
            }, { once: true });
        });
    }

    async function loadHeadlessBattle(url, maxLoadAttempts = 2) {
        const frame = getHeadlessBattleFrame();
        const absolute = new URL(url, location.origin).href;

        let current = '';
        try {
            current = frame.contentWindow?.location?.href || '';
        } catch {}

        if (!current || current === 'about:blank' || !sameBattleUrl(current, absolute)) {
            let loadedOk = false;
            let lastLoadError = null;

            for (let attempt = 1; attempt <= Math.max(1, maxLoadAttempts); attempt++) {
                try {
                    const loaded = waitForFrameLoad(frame, 12000);
                    frame.src = absolute;
                    await loaded;
                    loadedOk = true;
                    break;
                } catch (error) {
                    lastLoadError = error;

                    if (attempt < maxLoadAttempts) {
                        addLog(
                            `Hidden dungeon page load timed out; retrying ` +
                            `(${attempt + 1}/${maxLoadAttempts}).`
                        );

                        // Force a fresh navigation so the next retry gets a
                        // new load event instead of waiting on the old request.
                        try {
                            frame.src = 'about:blank';
                        } catch {}
                        await sleep(250);
                    }
                }
            }

            if (!loadedOk) {
                throw lastLoadError || new Error('Hidden battle page timed out while loading.');
            }
        }

        // Do not continue until the iframe copy of this userscript has installed
        // its lightweight worker. This also gives us an explicit diagnostic if
        // the battle page cannot be automated inside a frame.
        let snapshot = null;
        let lastError = null;

        for (let attempt = 0; attempt < 6; attempt++) {
            try {
                snapshot = await workerSnapshot(frame);
                if (snapshot) break;
            } catch (error) {
                lastError = error;
            }
            await sleep(250);
        }

        if (!snapshot) {
            throw new Error(
                `Hidden battle worker unavailable for ${absolute}. ` +
                `${lastError?.message || 'No worker response.'}`
            );
        }

        return frame;
    }

    function frameDocument(frame) {
        try {
            return frame?.contentDocument || frame?.contentWindow?.document || null;
        } catch {
            return null;
        }
    }

    function battleHpFromDoc(doc) {
        if (!doc) return { current: NaN, max: NaN };

        const hpText = doc.querySelector('#hpText')?.textContent || '';
        let pair = hpText.match(/([\d,]+)\s*\/\s*([\d,]+)/);

        if (!pair) {
            const sub = [...doc.querySelectorAll('.monster-card .card-sub, .card-headline .card-sub')]
                .map(el => el.textContent || '')
                .find(t => /\bHP\b/i.test(t) && /\//.test(t)) || '';
            pair = sub.match(/([\d,]+)\s*\/\s*([\d,]+)/);
        }

        return pair
            ? {
                current: parseVisibleNumber(pair[1]),
                max: parseVisibleNumber(pair[2])
            }
            : { current: NaN, max: NaN };
    }

    function battleIsDeadDoc(doc) {
        if (!doc) return false;

        // IMPORTANT:
        // Do NOT use DamageEstimator.parseMonster() as a death signal here.
        // When it cannot parse an HP value it can legitimately produce 0,
        // which previously caused a live monster (including Triton) to be
        // classified as dead before join/attack ever ran.
        //
        // A monster is dead only when the battle page itself gives us an
        // explicit, authoritative signal.

        const hp = battleHpFromDoc(doc);

        // Only trust HP=0 if we actually parsed a real max-HP value too.
        if (
            Number.isFinite(hp.current) &&
            Number.isFinite(hp.max) &&
            hp.max > 0 &&
            hp.current <= 0
        ) {
            return true;
        }

        const bodyText = cleanText(doc.body?.innerText || '').toLowerCase();

        return (
            bodyText.includes('monster is already dead') ||
            bodyText.includes('monster has been defeated') ||
            bodyText.includes('this monster is dead')
        );
    }

    async function preflightBattleTarget(item) {
        if (item.sourceType === 'dungeon') {
            const active = await isDungeonQueueItemStillActive(item);
            if (!active) {
                return { alive: false, reason: 'dungeon completed', hp: null };
            }
        }

        const catalogue = loadCatalogue();
        let catalogueMonster = null;

        try {
            catalogueMonster = item.sourceType === 'dungeon'
                ? catalogue.dungeons?.[item.sourceKey]
                    ?.locations?.[String(item.sectionKey)]
                    ?.monsters?.[item.monsterKey] || null
                : catalogue.gates?.[item.sourceKey]
                    ?.sections?.[String(item.sectionKey)]
                    ?.monsters?.[item.monsterKey] || null;
        } catch {}

        // Timed-boss source cards expose an explicit alive flag. A false value
        // is authoritative and lets us skip without ever opening battle.php.
        if (catalogueMonster?.timed && catalogueMonster.currentlyAlive === false) {
            return {
                alive: false,
                reason: 'timed boss not alive',
                hp: {
                    current: Number(catalogueMonster.damageModel?.currentHp) || 0,
                    max: Number(catalogueMonster.damageModel?.maxHp) || 0
                }
            };
        }

        // Do NOT use cached dungeon currentHp=0 as a death signal.
        //
        // Dungeon catalogue scans often store currentHp=0 for monsters that
        // still have a valid live activeBattleUrl. The exported catalogue
        // proved this happens across active dungeon instances, and this check
        // was therefore skipping those targets BEFORE ensureJoinedInFrame()
        // ever had a chance to run.
        //
        // For dungeon targets, the live battle.php GET below is authoritative.
        // Keep the cached-HP shortcut only for non-dungeon targets.
        const cachedHp = Number(catalogueMonster?.damageModel?.currentHp);
        const cachedMaxHp = Number(catalogueMonster?.damageModel?.maxHp);
        if (
            item.sourceType !== 'dungeon' &&
            Number.isFinite(cachedHp) &&
            Number.isFinite(cachedMaxHp) &&
            cachedMaxHp > 0 &&
            cachedHp <= 0
        ) {
            return {
                alive: false,
                reason: 'monster HP is 0',
                hp: { current: cachedHp, max: cachedMaxHp }
            };
        }

        // Silent GET only; this never changes the visible browser page.
        const response = await fetch(item.battleUrl, {
            method: 'GET',
            credentials: 'same-origin',
            cache: 'no-store'
        });

        if (!response.ok) {
            throw new Error(`${item.name}: battle preflight failed (HTTP ${response.status}).`);
        }

        const html = await response.text();
        const doc = parseHtml(html);

        let parsedMonster = null;
        try {
            parsedMonster = DamageEstimator.parseMonster(doc);
        } catch {}

        let hp = {
            current: Number(parsedMonster?.currentHp),
            max: Number(parsedMonster?.maxHp)
        };

        if (!Number.isFinite(hp.current) || !Number.isFinite(hp.max) || hp.max <= 0) {
            hp = battleHpFromDoc(doc);
        }

        if (
            Number.isFinite(hp.current) &&
            Number.isFinite(hp.max) &&
            hp.max > 0 &&
            hp.current <= 0
        ) {
            return { alive: false, reason: 'monster HP is 0', hp };
        }

        const bodyText = cleanText(doc.body?.innerText || '').toLowerCase();
        if (
            bodyText.includes('monster is already dead') ||
            bodyText.includes('monster has been defeated') ||
            bodyText.includes('this monster is dead')
        ) {
            return { alive: false, reason: 'monster dead', hp };
        }

        // Important: no buttons does NOT mean dead. Fetched pages can omit
        // battle controls depending on membership state.
        return {
            alive: true,
            hp,
            joined: !!doc.querySelector('.attack-btn'),
            joinable: !!doc.querySelector('#join-battle')
        };
    }

    function iframeNotificationText(doc) {
        return cleanText(doc?.querySelector('#notification')?.textContent || '');
    }

    function getIframeStaminaStats(doc) {
        if (!doc) return { current: 0, max: 0 };

        const span = doc.querySelector('#stamina_span');
        if (span) {
            const parentText = span.parentElement?.textContent || '';
            const pair = parentText.match(/([\d,]+)\s*\/\s*([\d,]+)/);
            if (pair) {
                return {
                    current: parseVisibleNumber(pair[1]),
                    max: parseVisibleNumber(pair[2])
                };
            }

            return {
                current: parseVisibleNumber(span.textContent),
                max: 0
            };
        }

        return { current: 0, max: 0 };
    }

    function getIframeDamage(doc) {
        return parseVisibleNumber(
            doc?.querySelector('#yourDamageValue')?.textContent || '0'
        );
    }


    function getFrameGameApi(frame) {
        const w = frame?.contentWindow;
        if (!w) return null;

        try {
            const hasBaseQS = typeof w.baseQS === 'function';
            const hasEp = typeof w.ep === 'function';

            if (!hasBaseQS || !hasEp) return null;

            return {
                window: w,
                baseQS: w.baseQS.bind(w),
                ep: w.ep.bind(w),
                isDungeon: Boolean(w.IS_DG),
                config: w.BCFG || null
            };
        } catch {
            return null;
        }
    }

    async function directJoinInFrame(frame, item) {
        const api = getFrameGameApi(frame);
        if (!api) return { ok: false, unavailable: true, message: 'battle API helpers unavailable' };

        try {
            const p = api.baseQS();

            // The site's own join handler explicitly adds user_id.
            // Prefer an exposed USER_ID, otherwise use the same `demon`
            // cookie that the proven regular-battle join path uses.
            const userId =
                (typeof api.window.USER_ID !== 'undefined' && api.window.USER_ID)
                    ? String(api.window.USER_ID)
                    : String(getCookieValue('demon') || '');

            if (userId) {
                p.set('user_id', userId);
            }

            const response = await api.window.fetch(api.ep('JOIN'), {
                method: 'POST',
                credentials: 'same-origin',
                headers: {
                    'Content-Type': 'application/x-www-form-urlencoded'
                },
                body: p.toString()
            });

            const raw = await response.text();
            const clean = cleanText(raw);
            const ok =
                response.ok &&
                clean.toLowerCase().startsWith('you have successfully');

            return { ok, unavailable: false, message: clean };
        } catch (error) {
            return {
                ok: false,
                unavailable: false,
                message: error?.message || String(error)
            };
        }
    }

    async function directAttackInFrame(frame, slash) {
        const api = getFrameGameApi(frame);
        if (!api) {
            return {
                ok: false,
                unavailable: true,
                data: null,
                message: 'battle API helpers unavailable'
            };
        }

        try {
            const fd = api.baseQS();
            fd.set('skill_id', String(slash.skillId));
            fd.set('stamina_cost', String(slash.cost));

            if (api.isDungeon) {
                if (!api.config) {
                    return {
                        ok: false,
                        unavailable: false,
                        data: null,
                        message: 'Dungeon battle configuration unavailable.'
                    };
                }

                fd.set('instance_id', String(api.config.instanceId));
                fd.set('dgmid', String(api.config.dgmid));
            }

            if (
                api.config &&
                api.config.privateBoss &&
                api.config.privateBoss.enabled
            ) {
                fd.set(
                    'csrf_token',
                    String(api.config.privateBoss.csrf || '')
                );
                fd.set(
                    'request_token',
                    (api.window.crypto &&
                     typeof api.window.crypto.randomUUID === 'function')
                        ? api.window.crypto.randomUUID()
                        : `${Date.now()}-${Math.random().toString(16).slice(2)}-${Math.random().toString(16).slice(2)}`
                );
            }

            const response = await api.window.fetch(api.ep('ATTACK'), {
                method: 'POST',
                credentials: 'same-origin',
                headers: {
                    'Content-Type': 'application/x-www-form-urlencoded'
                },
                body: fd.toString()
            });

            const raw = await response.text();
            const data = parseAttackResponse(raw);
            const message = cleanText(data?.message || raw || '');

            return {
                ok: Boolean(
                    response.ok &&
                    data &&
                    String(data.status).trim() === 'success'
                ),
                unavailable: false,
                response,
                raw,
                data,
                message
            };
        } catch (error) {
            return {
                ok: false,
                unavailable: false,
                data: null,
                message: error?.message || String(error)
            };
        }
    }

    async function ensureJoinedInFrame(frame, item) {
        for (let attempt = 0; attempt < 5; attempt++) {
            let snapshot = null;

            try {
                snapshot = await workerSnapshot(frame);
            } catch {
                await sleep(250);
                continue;
            }

            if (
                Number.isFinite(Number(snapshot?.monsterHp)) &&
                Number(snapshot.monsterHp) <= 0
            ) {
                return { ok: false, dead: true };
            }

            if (snapshot?.hasAttackButtons) {
                return { ok: true, dead: false };
            }

            if (snapshot?.hasJoinButton) {
                setStatus(`${item.name}: joining battle in background...`);

                const monsterId = monsterIdFromBattleUrl(item.battleUrl);

                if (monsterId) {
                    addLog(`${item.name}: direct join → user_join_battle.php (monster_id=${monsterId}).`);

                    const result = await directJoinRegularBattle(item);
                    if (!result.ok) {
                        throw new Error(
                            `${item.name}: direct join failed — ${result.message || `HTTP ${result.status}`}`
                        );
                    }

                    // The direct request does not reload the iframe itself.
                    frame.src = item.battleUrl;
                    await waitForFrameLoad(frame, 10000);
                } else {
                    // Dungeon battle pages already contain the exact native
                    // join handler we need. Drive that handler directly through
                    // the same-origin iframe DOM, exactly as the older dungeon
                    // implementation did, rather than reconstructing its
                    // baseQS()/ep('JOIN') request or routing through postMessage.
                    const ids = dungeonIdsFromBattleUrl(item.battleUrl);
                    const doc = frameDocument(frame);
                    const join = doc?.querySelector('#join-battle');

                    addLog(
                        `${item.name}: native dungeon DOM join → ` +
                        `dgmid=${ids.dgmid || item.dgmid || '?'} ` +
                        `instance_id=${ids.instanceId || item.instanceId || '?'}.`
                    );

                    if (!join) {
                        throw new Error(`${item.name}: native dungeon Join button is not present in hidden battle page.`);
                    }

                    const reloaded = waitForFrameLoad(frame, 12000).catch(() => null);
                    join.click();

                    // The site's handler performs the real join POST and, on
                    // success, schedules location.reload() after ~900 ms.
                    await Promise.race([reloaded, sleep(1700)]);
                    await sleep(250);

                    // If this page variant did not reload itself, inspect the
                    // notification and then force only the hidden frame to
                    // refresh so membership can be verified.
                    let joinedDoc = frameDocument(frame);
                    if (!joinedDoc?.querySelector('.attack-btn')) {
                        const note = iframeNotificationText(joinedDoc);
                        if (note) {
                            addLog(`${item.name}: dungeon join response → ${note}`);
                        }

                        const loaded = waitForFrameLoad(frame, 12000).catch(() => null);
                        try {
                            frame.contentWindow.location.reload();
                        } catch {
                            frame.src = item.battleUrl;
                        }
                        await Promise.race([loaded, sleep(1900)]);
                        await sleep(180);
                    }
                }

                await sleep(1200);

                try {
                    snapshot = await workerSnapshot(frame);
                    if (snapshot?.hasAttackButtons) {
                        addLog(`${item.name}: background join completed.`);
                        return { ok: true, dead: false };
                    }
                } catch {}

                // Force only the hidden frame to refresh and recheck.
                const loaded = waitForFrameLoad(frame, 12000).catch(() => null);
                try {
                    frame.contentWindow.location.reload();
                } catch {}
                await Promise.race([loaded, sleep(1800)]);
                await sleep(180);
                continue;
            }

            // Neither joined nor joinable yet: allow page scripts a moment,
            // then refresh the hidden frame once.
            await sleep(500);

            const loaded = waitForFrameLoad(frame, 12000).catch(() => null);
            try {
                frame.contentWindow.location.reload();
            } catch {}
            await Promise.race([loaded, sleep(1800)]);
            await sleep(180);
        }

        const finalSnapshot = await workerSnapshot(frame).catch(() => null);

        if (
            finalSnapshot &&
            Number.isFinite(Number(finalSnapshot.monsterHp)) &&
            Number(finalSnapshot.monsterHp) <= 0
        ) {
            return { ok: false, dead: true };
        }

        return {
            ok: !!finalSnapshot?.hasAttackButtons,
            dead: false
        };
    }

    function iframeAttackButton(doc, skillId) {
        return doc?.querySelector(`.attack-btn[data-skill-id="${skillId}"]`) || null;
    }

    async function waitForDungeonAttackButtonReady(frame, skillId, timeoutMs = 2200) {
        const started = Date.now();

        while (Date.now() - started < timeoutMs) {
            const doc = frameDocument(frame);
            const btn = iframeAttackButton(doc, skillId);

            if (btn && !btn.disabled && btn.getAttribute('aria-busy') !== 'true') {
                return btn;
            }

            await sleep(60);
        }

        return iframeAttackButton(frameDocument(frame), skillId);
    }

    async function waitForHiddenAttackResult(
        frame,
        beforeDamage,
        beforeStamina,
        beforeNote = '',
        timeoutMs = 2600
    ) {
        const started = Date.now();

        while (Date.now() - started < timeoutMs) {
            const doc = frameDocument(frame);

            if (!doc) {
                await sleep(90);
                continue;
            }

            if (battleIsDeadDoc(doc)) {
                return { kind: 'dead', doc };
            }

            const damage = getIframeDamage(doc);
            const stamina = getIframeStaminaStats(doc).current;
            const note = cleanText(iframeNotificationText(doc) || '');
            const newNote = note && note !== cleanText(beforeNote || '');

            if (
                damage !== beforeDamage ||
                stamina !== beforeStamina
            ) {
                return { kind: 'changed', doc, damage, stamina, note };
            }

            if (newNote && /removed due to inactivity|please rejoin/i.test(note)) {
                return { kind: 'rejoin', doc, note };
            }

            if (newNote && /monster is already dead/i.test(note)) {
                return { kind: 'dead', doc, note };
            }

            if (newNote && /couldn.?t pierce|no stamina spent/i.test(note)) {
                return { kind: 'negated', doc, note };
            }

            if (newNote && /slow down|cooldown|attacking too quickly|take your time/i.test(note)) {
                return { kind: 'cooldown', doc, note };
            }

            await sleep(90);
        }

        return {
            kind: 'unchanged',
            doc: frameDocument(frame)
        };
    }

    async function useLargePotionInFrame(frame, item) {
        const doc = frameDocument(frame);
        const potionBtn = [...(doc?.querySelectorAll('.potion-use-btn') || [])]
            .find(btn => normalizeKey(btn.dataset.name) === 'large stamina potion');

        if (!potionBtn) {
            throw new Error(`${item.name}: Large Stamina Potion is not available.`);
        }

        const invId = potionBtn.dataset.inv;
        const qtySpan = invId ? doc.getElementById(`pqty_${invId}`) : null;
        const qty = qtySpan
            ? parseVisibleNumber(qtySpan.textContent)
            : parseVisibleNumber(potionBtn.dataset.max);

        if (!invId || qty <= 0) {
            throw new Error('No Large Stamina Potions remaining.');
        }

        setStatus(`${item.name}: using Large Stamina Potion in background...`);
        addLog(`${item.name}: using 1 × Large Stamina Potion.`);

        const params = new URLSearchParams();
        params.set('inv_id', invId);

        const response = await frame.contentWindow.fetch('/use_item.php', {
            method: 'POST',
            credentials: 'same-origin',
            headers: {
                'Content-Type': 'application/x-www-form-urlencoded'
            },
            body: params.toString()
        });

        const raw = await response.text();
        if (!response.ok || !raw.trim().toLowerCase().includes('success')) {
            throw new Error(`Large Stamina Potion failed: ${cleanText(raw).slice(0, 160)}`);
        }

        const loaded = waitForFrameLoad(frame, 12000).catch(() => null);
        frame.contentWindow.location.reload();
        await Promise.race([loaded, sleep(1700)]);
        await sleep(180);
    }


    function monsterIdFromBattleUrl(url) {
        try {
            return new URL(url, location.origin).searchParams.get('id') || '';
        } catch {
            return '';
        }
    }

    function dungeonIdsFromBattleUrl(url) {
        try {
            const u = new URL(url, location.origin);
            return {
                dgmid: u.searchParams.get('dgmid') || '',
                instanceId: u.searchParams.get('instance_id') || ''
            };
        } catch {
            return { dgmid: '', instanceId: '' };
        }
    }

    function getCookieValue(name) {
        const prefix = `${name}=`;
        const part = document.cookie
            .split(';')
            .map(x => x.trim())
            .find(x => x.startsWith(prefix));
        return part ? decodeURIComponent(part.slice(prefix.length)) : '';
    }

    async function resolveLiveMonsterTarget(item) {
        // Dungeon battles have a separate dgmid + instance_id request scheme.
        if (item.sourceType === 'dungeon') {
            const activeIds = dungeonIdsFromBattleUrl(item.battleUrl);
            const dgmid = activeIds.dgmid || item.dgmid || '';
            const instanceId = activeIds.instanceId || item.instanceId || '';

            return {
                monsterId: '',
                dgmid,
                instanceId,
                battleUrl: item.battleUrl,
                source: 'dungeon-active-url'
            };
        }

        const sourceUrl = item.sourceUrl || '';
        const storedMonsterId = item.monsterId ? String(item.monsterId) : '';

        // Refresh from the source page immediately before attacking. This uses
        // the exact identity mechanism used by the working scripts:
        // data-monster-id from the monster's own source card. The stored ID is
        // a fallback only if the source fetch temporarily fails.
        if (!sourceUrl) {
            if (storedMonsterId) {
                addLog(`${item.name}: using scanned monster_id=${storedMonsterId} (no source URL).`);
                return {
                    monsterId: storedMonsterId,
                    battleUrl: item.battleUrl,
                    source: 'catalogue'
                };
            }
            throw new Error(`${item.name}: no source page URL and no scanned monster_id.`);
        }

        let doc;
        try {
            const response = await fetch(sourceUrl, {
                method: 'GET',
                credentials: 'same-origin',
                cache: 'no-store'
            });

            if (!response.ok) {
                throw new Error(`HTTP ${response.status}`);
            }

            doc = parseHtml(await response.text());
        } catch (error) {
            if (storedMonsterId) {
                addLog(
                    `${item.name}: live ID refresh failed (${error.message}); ` +
                    `using scanned monster_id=${storedMonsterId}.`
                );
                return {
                    monsterId: storedMonsterId,
                    battleUrl: item.battleUrl,
                    source: 'catalogue fallback'
                };
            }
            throw new Error(`${item.name}: source-page lookup failed (${error.message}).`);
        }

        const wanted = normalizeKey(item.name);
        let monsterId = '';
        let battleUrl = '';

        // First choice: actual attackable .monster-card. This is the same
        // source used by the working autofarm/direct-attack scripts.
        for (const card of doc.querySelectorAll('.monster-card')) {
            const cardName =
                cleanText(card.dataset?.name || '') ||
                cleanText(card.querySelector('.monster-name, .name, h2, h3, h4, .card-title')?.textContent || '');

            if (normalizeKey(cardName) !== wanted) continue;

            monsterId =
                card.dataset?.monsterId ||
                card.getAttribute?.('data-monster-id') ||
                '';

            const a = [...card.querySelectorAll('a[href*="battle.php"]')]
                .find(link => /battle\.php\?id=/i.test(link.getAttribute('href') || '')) ||
                card.querySelector('a[href*="battle.php"]');

            if (a?.getAttribute?.('href')) {
                battleUrl = absoluteUrl(a.getAttribute('href'), sourceUrl);
            }

            if (monsterId) break;
        }

        // Timed-boss fallback: exact name node -> nearby ancestor containing
        // data-monster-id. This mirrors the proven Triton resolver.
        if (!monsterId) {
            const exactNameNode = Array.from(doc.querySelectorAll('body *')).find(el => {
                return normalizeKey(el.textContent || '') === wanted;
            });

            if (exactNameNode) {
                let node = exactNameNode;
                for (let depth = 0; node && depth < 10; depth++, node = node.parentElement) {
                    if (!monsterId) {
                        monsterId =
                            node.dataset?.monsterId ||
                            node.getAttribute?.('data-monster-id') ||
                            '';
                    }

                    if (!battleUrl) {
                        const a = node.matches?.('a[href*="battle.php"]')
                            ? node
                            : node.querySelector?.('a[href*="battle.php"]');
                        if (a?.getAttribute?.('href')) {
                            battleUrl = absoluteUrl(a.getAttribute('href'), sourceUrl);
                        }
                    }

                    if (monsterId && battleUrl) break;
                }
            }
        }

        if (!monsterId && storedMonsterId) {
            addLog(
                `${item.name}: source page did not expose an ID right now; ` +
                `using scanned monster_id=${storedMonsterId}.`
            );
            monsterId = storedMonsterId;
        }

        if (!monsterId) {
            throw new Error(
                `${item.name}: no data-monster-id found on its source card and no scanned ID is cached.`
            );
        }

        if (storedMonsterId && storedMonsterId !== String(monsterId)) {
            addLog(
                `${item.name}: live monster_id changed ${storedMonsterId} → ${monsterId}; ` +
                `using the live ID.`
            );
        } else {
            addLog(`${item.name}: monster_id=${monsterId} confirmed from source card.`);
        }

        return {
            monsterId: String(monsterId),
            battleUrl: battleUrl || item.battleUrl,
            source: sourceUrl
        };
    }

    async function directJoinByMonsterId(item, monsterId) {
        const userId = getCookieValue('demon');
        if (!userId) {
            throw new Error(`${item.name}: cookie "demon" was not found.`);
        }

        const body =
            `monster_id=${encodeURIComponent(monsterId)}` +
            `&user_id=${encodeURIComponent(userId)}`;

        const response = await fetch(`${BASE_URL}/user_join_battle.php`, {
            method: 'POST',
            headers: {
                'Content-Type': 'application/x-www-form-urlencoded; charset=UTF-8'
            },
            body,
            credentials: 'same-origin'
        });

        const raw = cleanText(await response.text());
        const lower = raw.toLowerCase();

        const ok =
            raw.includes('You have successfully joined the battle.') ||
            lower.includes('already joined') ||
            lower.includes('already in this battle') ||
            lower.includes('already in battle') ||
            lower.includes('already participating') ||
            lower.includes('have joined this battle') ||
            lower.includes('you joined this battle') ||
            lower.includes('currently joined') ||
            lower.includes('cannot join') && (
                lower.includes('already') ||
                lower.includes('joined')
            );

        return { ok, message: raw, status: response.status };
    }

    async function directAttackByMonsterId(monsterId, slash) {
        const body =
            `monster_id=${encodeURIComponent(monsterId)}` +
            `&skill_id=${encodeURIComponent(slash.skillId)}` +
            `&stamina_cost=${encodeURIComponent(slash.cost)}`;

        const response = await fetch(`${BASE_URL}/damage.php`, {
            method: 'POST',
            headers: {
                'Content-Type': 'application/x-www-form-urlencoded; charset=UTF-8'
            },
            body,
            credentials: 'same-origin'
        });

        const raw = await response.text();

        let data = null;
        try {
            data = JSON.parse(raw);
        } catch {
            data = parseAttackResponse(raw);
        }

        return {
            ok: !!data && String(data.status || '').trim().toLowerCase() === 'success',
            status: response.status,
            data,
            raw
        };
    }

    function getMonsterHpFromDoc(doc) {
        if (!doc) return null;

        const text = cleanText(doc.querySelector('#hpText')?.textContent || '');
        const match = text.match(/([\d,]+)\s*\/\s*([\d,]+)/);
        if (!match) return null;

        return {
            current: parseVisibleNumber(match[1]),
            max: parseVisibleNumber(match[2])
        };
    }

    function getPlayerHpFromDoc(doc) {
        if (!doc) return { current: null, max: null };

        const hpText = cleanText(
            doc.querySelector('#pHpText')?.textContent ||
            doc.querySelector('.player-card .card-sub')?.textContent ||
            ''
        );

        const pair = hpText.match(/([\d,]+)\s*\/\s*([\d,]+)/);
        if (!pair) return { current: null, max: null };

        return {
            current: parseVisibleNumber(pair[1]),
            max: parseVisibleNumber(pair[2])
        };
    }

    function getHpPotionFromDoc(doc) {
        if (!doc) return null;

        const button = doc.querySelector('.potion-use-btn[data-item="108"]');
        if (!button) return null;

        const invId = cleanText(button.getAttribute('data-inv') || '');
        const qty = parseVisibleNumber(
            doc.getElementById(`pqty_${invId}`)?.textContent ||
            button.getAttribute('data-max') ||
            '0'
        );

        if (!invId || qty <= 0) return null;

        return { invId, qty };
    }

    function getLargeStaminaPotionFromDoc(doc) {
        if (!doc) return null;

        const button = [...doc.querySelectorAll('.potion-use-btn')]
            .find(btn => normalizeKey(btn.getAttribute('data-name') || btn.dataset?.name || '') === 'large stamina potion');

        if (!button) return null;

        const invId = cleanText(button.getAttribute('data-inv') || '');
        const qty = parseVisibleNumber(
            doc.getElementById(`pqty_${invId}`)?.textContent ||
            button.getAttribute('data-max') ||
            '0'
        );

        if (!invId || qty <= 0) return null;
        return { invId, qty };
    }

    async function useLargeStaminaPotionDirect(item, potion) {
        if (!potion?.invId) {
            throw new Error(`${item.name}: Large Stamina Potion inventory ID is unavailable.`);
        }

        setStatus(`${item.name}: stamina too low — using Large Stamina Potion...`);
        addLog(
            `${item.name}: using Large Stamina Potion ` +
            `(inv_id=${potion.invId}, ${Math.round(potion.qty).toLocaleString('en-US')} available).`
        );

        const params = new URLSearchParams();
        params.set('inv_id', potion.invId);

        const response = await fetch(`${BASE_URL}/use_item.php`, {
            method: 'POST',
            credentials: 'same-origin',
            headers: {
                'Content-Type': 'application/x-www-form-urlencoded; charset=UTF-8'
            },
            body: params.toString()
        });

        const raw = cleanText(await response.text());
        const lower = raw.toLowerCase();
        const ok = response.ok && lower.includes('success');

        if (!ok) {
            throw new Error(
                `${item.name}: Large Stamina Potion failed — ${raw.slice(0, 180) || `HTTP ${response.status}`}`
            );
        }

        addLog(`${item.name}: Large Stamina Potion used successfully.`);
        await sleep(300);
        return true;
    }

    async function useHpPotion(item, potion) {
        if (!potion?.invId) {
            throw new Error(`${item.name}: Full HP Potion inventory ID is unavailable.`);
        }

        setStatus(`${item.name}: HP is 0 — using Full HP Potion...`);
        addLog(
            `${item.name}: player HP is 0; using Full HP Potion ` +
            `(inv_id=${potion.invId}, ${Math.round(potion.qty).toLocaleString('en-US')} available).`
        );

        const params = new URLSearchParams();
        params.set('inv_id', potion.invId);

        const response = await fetch(`${BASE_URL}/user_heal_potion.php`, {
            method: 'POST',
            credentials: 'same-origin',
            headers: {
                'Content-Type': 'application/x-www-form-urlencoded; charset=UTF-8'
            },
            body: params.toString()
        });

        const raw = cleanText(await response.text());
        const lower = raw.toLowerCase();

        let data = null;
        try { data = JSON.parse(raw); } catch {}

        const ok =
            response.ok && (
                String(data?.status || '').trim().toLowerCase() === 'success' ||
                lower.includes('success') ||
                lower.includes('healed')
            );

        if (!ok) {
            throw new Error(
                `${item.name}: HP potion failed — ${cleanText(data?.message || raw || `HTTP ${response.status}`).slice(0, 180)}`
            );
        }

        addLog(`${item.name}: HP potion used successfully.`);
        await sleep(300);
        return true;
    }

    async function ensureRegularPlayerAlive(item, battleUrl) {
        const state = await readRegularBattleState(battleUrl);
        const hp = state.playerHp;

        if (!Number.isFinite(Number(hp?.current))) {
            addLog(`${item.name}: player HP could not be read; continuing without HP recovery.`);
            return state;
        }

        if (Number(hp.current) > 0) return state;

        const potion = state.hpPotion;
        if (!potion) {
            throw new Error(`${item.name}: player HP is 0 and no Full HP Potion is available.`);
        }

        await useHpPotion(item, potion);
        const refreshed = await readRegularBattleState(battleUrl);

        if (
            Number.isFinite(Number(refreshed.playerHp?.current)) &&
            Number(refreshed.playerHp.current) <= 0
        ) {
            throw new Error(`${item.name}: HP remained at 0 after using a Full HP Potion.`);
        }

        addLog(
            `${item.name}: player recovered to ` +
            `${Math.round(Number(refreshed.playerHp?.current) || 0).toLocaleString('en-US')} HP; resuming attack.`
        );
        return refreshed;
    }

    async function ensureFramePlayerAlive(frame, item) {
        let doc = frameDocument(frame);
        const hp = getPlayerHpFromDoc(doc);

        if (!Number.isFinite(Number(hp.current)) || Number(hp.current) > 0) {
            return false;
        }

        const potion = getHpPotionFromDoc(doc);
        if (!potion) {
            throw new Error(`${item.name}: player HP is 0 and no Full HP Potion is available.`);
        }

        await useHpPotion(item, potion);

        // Reload only the hidden battle frame so its HP/potion state is
        // authoritative before the next attack. The visible page is untouched.
        const loadPromise = waitForIframeLoad(frame);
        frame.src = item.battleUrl;
        await loadPromise;

        doc = frameDocument(frame);
        const refreshedHp = getPlayerHpFromDoc(doc);
        if (
            Number.isFinite(Number(refreshedHp.current)) &&
            Number(refreshedHp.current) <= 0
        ) {
            throw new Error(`${item.name}: HP remained at 0 after using a Full HP Potion.`);
        }

        addLog(
            `${item.name}: player recovered to ` +
            `${Math.round(Number(refreshedHp.current) || 0).toLocaleString('en-US')} HP; resuming attack.`
        );
        return true;
    }

    async function readRegularBattleState(battleUrl) {
        const response = await fetch(battleUrl, {
            method: 'GET',
            credentials: 'same-origin',
            cache: 'no-store'
        });

        if (!response.ok) {
            throw new Error(`Battle-state refresh failed (HTTP ${response.status}).`);
        }

        const html = await response.text();
        const doc = parseHtml(html);

        return {
            damage: getIframeDamage(doc),
            stamina: getIframeStaminaStats(doc),
            dead: battleIsDeadDoc(doc),
            joined: !!doc.querySelector('.attack-btn'),
            joinable: !!doc.querySelector('#join-battle'),
            playerHp: getPlayerHpFromDoc(doc),
            hpPotion: getHpPotionFromDoc(doc),
            largeStaminaPotion: getLargeStaminaPotionFromDoc(doc)
        };
    }

    function inspectDivineBattleDoc(doc, battleUrl = '') {
        if (!doc) return { isDivine: false, phase: 'normal', duelUrl: '', effectText: '' };

        const isDivine = !!doc.querySelector('.monster-frame.divine, .divine-badge');
        if (!isDivine) return { isDivine: false, phase: 'normal', duelUrl: '', effectText: '' };

        const duelLink = [...doc.querySelectorAll('a[href*="pvp_style_battle.php"]')]
            .find(a => {
                const href = a.getAttribute('href') || '';
                return /source=monster_phase/i.test(href) && /active_id=/i.test(href);
            });

        const duelUrl = duelLink?.getAttribute('href')
            ? absoluteUrl(duelLink.getAttribute('href'), battleUrl || BASE_URL)
            : '';

        const bodyText = cleanText(doc.body?.innerText || '');
        const effectText = cleanText(
            doc.querySelector('.effect-text, .monster-effect, [class*="effect"]')?.textContent || bodyText
        );

        // Final PvE must take priority over any stale/ambient duel wording
        // that can remain elsewhere in the page after the phase duel is cleared.
        // The live final-form page explicitly identifies itself as
        // "Ascended PvE form", so treat that as authoritative.
        if (/Ascended PvE form/i.test(bodyText)) {
            return { isDivine: true, phase: 'final', duelUrl: '', effectText };
        }

        // Only an actual actionable monster-phase duel link identifies the
        // duel state. Do not classify from generic "Duel Phase" page text.
        if (duelUrl) {
            return { isDivine: true, phase: 'duel', duelUrl, effectText };
        }

        // Any other Divine battle page is the initial PvE form. Never spend
        // stamina here; the later PvE form's damage counts for the useful loot.
        return { isDivine: true, phase: 'initial', duelUrl: '', effectText };
    }

    async function readBattleDocument(battleUrl) {
        const response = await fetch(battleUrl, {
            method: 'GET',
            credentials: 'same-origin',
            cache: 'no-store'
        });
        if (!response.ok) throw new Error(`Battle page refresh failed (HTTP ${response.status}).`);
        return parseHtml(await response.text());
    }

    function pvpEnemyTargetKey(state) {
        const enemyMap = state?.teams?.enemy?.players_by_num || {};
        for (const player of Object.values(enemyMap)) {
            if (!player) continue;
            const alive = player.alive === true || Number(player.alive) === 1;
            const key = cleanText(player.key || player.target_key || '');
            if (alive && key) return key;
        }
        return '';
    }

    async function divineDuelState(activeId) {
        const params = new URLSearchParams({
            source: 'monster_phase',
            active_id: String(activeId),
            since_log_id: '0'
        });
        const response = await fetch(`${BASE_URL}/pvp_style_state.php?${params.toString()}`, {
            credentials: 'same-origin',
            cache: 'no-store'
        });
        const json = await response.json().catch(() => null);
        if (!response.ok || !json || !json.ok) {
            throw new Error(cleanText(json?.error || json?.message || `Phase Duel state failed (HTTP ${response.status}).`));
        }
        return json;
    }

    async function divineDuelAction(activeId, action, extra = {}) {
        const body = new URLSearchParams({
            source: 'monster_phase',
            active_id: String(activeId),
            action: String(action)
        });
        for (const [key, value] of Object.entries(extra)) body.set(key, String(value));

        const response = await fetch(`${BASE_URL}/pvp_style_action.php`, {
            method: 'POST',
            credentials: 'same-origin',
            headers: { 'Content-Type': 'application/x-www-form-urlencoded; charset=UTF-8' },
            body: body.toString()
        });
        const json = await response.json().catch(() => null);
        if (!response.ok || !json || !json.ok) {
            throw new Error(cleanText(json?.error || json?.message || `Phase Duel action failed (HTTP ${response.status}).`));
        }
        return json;
    }

    async function runDivinePhaseDuel(item, duelUrl) {
        let parsed;
        try { parsed = new URL(duelUrl, BASE_URL); }
        catch { throw new Error(`${item.name}: invalid Enter Phase Duel URL.`); }

        const activeId = parsed.searchParams.get('active_id') || '';
        if (!activeId) throw new Error(`${item.name}: Enter Phase Duel URL has no active_id.`);

        addLog(`${item.name}: Divine Phase Duel detected · active_id=${activeId}.`);
        setStatus(`${item.name}: fighting Divine Phase Duel...`);

        let retries = 0;
        const maxRetries = 3;

        while (true) {
            const state = await divineDuelState(activeId);
            const ended = !!state?.match?.ended;
            const winner = String(state?.match?.winner_side || '');
            const roomStatus = String(state?.room_status || '');

            if (ended && winner === 'ally' && roomStatus === 'cleared') {
                addLog(`${item.name}: Divine Phase Duel cleared.`);
                setStatus(`${item.name}: Phase Duel cleared — loading Ascended PvE form...`);
                await sleep(900);
                return true;
            }

            if (ended && winner === 'enemy' && roomStatus === 'lost') {
                if (retries >= maxRetries) {
                    throw new Error(`${item.name}: Divine Phase Duel lost ${retries + 1} times; stopping retries.`);
                }
                retries += 1;
                addLog(`${item.name}: Phase Duel lost; retrying (${retries}/${maxRetries}).`);
                await divineDuelAction(activeId, 'retry');
                await sleep(700);
                continue;
            }

            const me = state?.me || {};
            const turn = state?.turn || {};
            const myTurn = !!(me.in_match && turn.side === 'ally' && String(turn.user_id) === String(me.user_id || getCookieValue('demon')));
            const myAlive = me.alive === true || Number(me.alive) === 1;

            if (!myTurn || !myAlive) {
                await sleep(450);
                continue;
            }

            const skills = Array.isArray(me.skills) ? me.skills : [];
            const tokens = Math.max(0, parseInt(me.tokens || 0, 10));
            const mana = skills.find(skill => String(skill.id) === 'adv:10' && String(skill.target) === 'enemy');
            const slash = skills.find(skill => String(skill.id) === '0' && String(skill.target) === 'enemy');
            const targetKey = pvpEnemyTargetKey(state);

            if (!targetKey) throw new Error(`${item.name}: no living Phase Duel enemy target was found.`);

            const manaCost = mana ? Math.max(0, parseInt(mana.cost || 0, 10)) : Infinity;
            const chosen = mana && tokens >= manaCost ? mana : slash;
            if (!chosen) throw new Error(`${item.name}: neither Mana Collapse nor Slash is available in the Phase Duel.`);

            addLog(
                `${item.name}: Phase Duel turn → ${String(chosen.id) === 'adv:10' ? 'Mana Collapse' : 'Slash'} ` +
                `(tokens ${tokens}, cost ${parseInt(chosen.cost || 0, 10)}).`
            );

            await divineDuelAction(activeId, 'use_skill', {
                skill_id: String(chosen.id),
                target_key: targetKey
            });
            await sleep(500);
        }
    }

    function responseMarksCurrentAttacker(message) {
        // This is deliberately evaluated ONLY on the response returned by our
        // own damage.php request. Never inspect the shared battle log, because
        // that log can say that another player was marked.
        return /\bmarked you for 2 turns\b/i.test(cleanText(message || ''));
    }

    async function runDirectRegularTarget(state, item) {
        // Use the exact sequence that already worked in the standalone
        // Triton test:
        //   1) resolve LIVE monster_id
        //   2) POST join unconditionally (already-joined is accepted)
        //   3) immediately POST a 1-stamina Slash
        // Only after that proven direct request succeeds do we involve the
        // predictor/tier-selection logic.

        const live = await resolveLiveMonsterTarget(item);

        addLog(
            `${item.name}: LIVE monster_id=${live.monsterId}` +
            `${live.battleUrl ? ` · ${live.battleUrl}` : ''}.`
        );

        let divineState = { isDivine: false, phase: 'normal', duelUrl: '' };
        try {
            const liveBattleDoc = await readBattleDocument(live.battleUrl);
            divineState = inspectDivineBattleDoc(liveBattleDoc, live.battleUrl);
        } catch (error) {
            addLog(`${item.name}: Divine-state inspection warning (${error.message || error}); continuing normal path.`);
        }

        if (divineState.isDivine && divineState.phase === 'initial') {
            addLog(`${item.name}: Divine initial PvE phase detected; spending 0 stamina and waiting for Phase Duel.`);
            setStatus(`${item.name}: Divine initial phase — skipped until Phase Duel.`);
            await moveToNextAttackTargetHeadless(state);
            return;
        }

        if (divineState.isDivine && divineState.phase === 'duel') {
            if (!divineState.duelUrl) {
                addLog(`${item.name}: Divine Duel Phase detected but Enter Phase Duel link is not available yet; skipping.`);
                await moveToNextAttackTargetHeadless(state);
                return;
            }

            await runDivinePhaseDuel(item, divineState.duelUrl);

            // The active_id/battle URL stays authoritative. Verify that the
            // duel actually unlocked the Ascended PvE form before spending stamina.
            const postDuelDoc = await readBattleDocument(live.battleUrl);
            divineState = inspectDivineBattleDoc(postDuelDoc, live.battleUrl);
            if (divineState.phase !== 'final') {
                addLog(`${item.name}: Phase Duel cleared, but Ascended PvE form is not visible yet; skipping safely.`);
                await moveToNextAttackTargetHeadless(state);
                return;
            }
            addLog(`${item.name}: Ascended PvE form confirmed after Phase Duel; starting normal PvE attacks.`);
        }

        let markedTurns = 0;

        setStatus(`${item.name}: joining...`);
        addLog(`${item.name}: POST user_join_battle.php → monster_id=${live.monsterId}.`);

        const join = await directJoinByMonsterId(item, live.monsterId);

        addLog(
            `${item.name}: join response [HTTP ${join.status}]: ` +
            `${join.message || '(empty response)'}`
        );

        if (!join.ok) {
            // Do NOT stop here. user_join_battle.php can reject a second join
            // even though the player is already participating. The attack
            // endpoint is authoritative: if we are already joined, damage.php
            // will accept the request. If we really are not joined, its
            // response will tell us and we can handle that below.
            addLog(
                `${item.name}: join was not accepted, but continuing to attack ` +
                `in case this means we are already joined.`
            );
        }

        await sleep(350);

        // A player at 0 HP cannot attack. Check the live battle page before
        // the very first attack and heal automatically when necessary.
        await ensureRegularPlayerAlive(item, live.battleUrl);

        // The first probe still costs 1 stamina. If we are completely empty,
        // refill before sending it rather than letting the direct path fail.
        {
            let initialState = await readRegularBattleState(live.battleUrl);
            if (Number(initialState.stamina?.current) < 1) {
                if (!initialState.largeStaminaPotion) {
                    throw new Error(`${item.name}: stamina is 0 and no Large Stamina Potion is available.`);
                }
                await useLargeStaminaPotionDirect(item, initialState.largeStaminaPotion);
                initialState = await readRegularBattleState(live.battleUrl);
                if (Number(initialState.stamina?.current) < 1) {
                    throw new Error(`${item.name}: stamina remained too low after using a Large Stamina Potion.`);
                }
            }
        }

        // PROVEN first attack: exact same 1-stamina Slash used by the
        // standalone test that the user confirmed works.
        const probeSlash = {
            cost: 1,
            skillId: 0,
            name: 'Slash'
        };

        addLog(
            `${item.name}: PROVEN TEST ATTACK → monster_id=${live.monsterId}, ` +
            `skill_id=0, stamina_cost=1.`
        );

        const probe = await directAttackByMonsterId(live.monsterId, probeSlash);
        const probeData = probe.data;
        const probeMessage = cleanText(probeData?.message || probe.raw || '');

        addLog(
            `${item.name}: probe attack [HTTP ${probe.status}]: ` +
            `${probeMessage || JSON.stringify(probeData || {})}`
        );

        if (probe.ok && divineState.isDivine && responseMarksCurrentAttacker(probeMessage)) {
            markedTurns = 2;
            addLog(`${item.name}: YOU were Moon Marked → forcing 1-STAM Slash for the next 2 successful attacks.`);
        }

        if (!probe.ok) {
            if (/monster is already dead/i.test(probeMessage)) {
                addLog(`${item.name}: server reports monster already dead; moving on.`);
                await moveToNextAttackTargetHeadless(state);
                return;
            }

            if (
                /removed due to inactivity|please rejoin|not joined|join the battle|must join|need to join/i.test(
                    probeMessage
                )
            ) {
                const rejoin = await directJoinByMonsterId(item, live.monsterId);

                addLog(
                    `${item.name}: rejoin response [HTTP ${rejoin.status}]: ` +
                    `${rejoin.message || '(empty response)'}`
                );

                // Even if the join endpoint still claims failure, try the
                // attack once more. This covers the "already joined" case.
                await sleep(350);

                const retryProbe = await directAttackByMonsterId(live.monsterId, probeSlash);
                const retryMessage = cleanText(
                    retryProbe.data?.message || retryProbe.raw || ''
                );

                addLog(
                    `${item.name}: retry probe [HTTP ${retryProbe.status}]: ` +
                    `${retryMessage || JSON.stringify(retryProbe.data || {})}`
                );

                if (!retryProbe.ok) {
                    throw new Error(
                        `${item.name}: proven 1-STAM attack still failed — ` +
                        `${retryMessage || `HTTP ${retryProbe.status}`}`
                    );
                }

                if (Number.isFinite(Number(retryProbe.data?.totaldmgdealt))) {
                    probeData.totaldmgdealt = Number(retryProbe.data.totaldmgdealt);
                }
            } else if (/couldn.?t pierce|no stamina spent|divine shield/i.test(probeMessage)) {
                // A negate proves the request itself worked, so continue.
                addLog(`${item.name}: probe was negated, but damage.php accepted the attack request.`);
            } else {
                throw new Error(
                    `${item.name}: proven 1-STAM attack failed — ` +
                    `${probeMessage || `HTTP ${probe.status}`}`
                );
            }
        }

        let currentDamage = Number(probeData?.totaldmgdealt);

        if (!Number.isFinite(currentDamage)) {
            // Only use the battle page as a state read AFTER the proven direct
            // attack has run. It is no longer allowed to prevent attacking.
            try {
                const battleState = await readRegularBattleState(live.battleUrl);
                currentDamage = Number(battleState.damage) || 0;
            } catch {
                currentDamage = 0;
            }
        }

        addLog(
            `${item.name}: direct attack path confirmed; current damage=` +
            `${Math.round(currentDamage).toLocaleString('en-US')}.`
        );

        while (true) {
            state = loadAttackAutomation();
            if (!state.active) return;

            if (currentDamage >= item.targetDamage) {
                addLog(
                    `${item.name}: target reached (` +
                    `${Math.round(currentDamage).toLocaleString('en-US')} / ` +
                    `${Math.round(item.targetDamage).toLocaleString('en-US')}).`
                );
                await moveToNextAttackTargetHeadless(state);
                return;
            }

            const remainingDamage = Math.max(0, item.targetDamage - currentDamage);

            let slash = probeSlash;

            // Predictor is now advisory. It can never prevent the script from
            // attacking. If anything in prediction fails, fall back to Slash.
            try {
                const estimatorState = await getDamageEstimatorState(false);
                const prediction = DamageEstimator.calculateForTarget(
                    estimatorState,
                    item.damageModel,
                    remainingDamage
                );

                const estimatedStaminaNeeded =
                    Number(prediction.stamina?.expectedStamina);

                if (Number.isFinite(estimatedStaminaNeeded)) {
                    slash = chooseSlashAttack(estimatedStaminaNeeded);
                } else {
                    addLog(`${item.name}: predictor returned no stamina estimate; using 1-STAM Slash.`);
                }
            } catch (error) {
                addLog(
                    `${item.name}: predictor warning (${error.message || error}); ` +
                    `using 1-STAM Slash instead.`
                );
            }

            if (divineState.isDivine && markedTurns > 0) {
                slash = probeSlash;
                addLog(
                    `${item.name}: Moon Mark active (${markedTurns} attack${markedTurns === 1 ? '' : 's'} remaining) ` +
                    `→ forcing 1-STAM Slash.`
                );
            }

            // Check again immediately before each attack. Normally the previous
            // damage response tells us the retaliation HP, but this authoritative
            // page check also covers deaths caused outside our own last request.
            await ensureRegularPlayerAlive(item, live.battleUrl);

            // Re-read authoritative stamina immediately before the attack.
            // For max stamina below 6000, preserve the original behavior:
            // drain the remaining stamina with the largest affordable Slash,
            // then use a Large Stamina Potion once no useful attack is affordable.
            // For max stamina >= 6000, refill immediately when the selected
            // attack tier is unaffordable.
            let battleState = await readRegularBattleState(live.battleUrl);
            let currentStamina = Number(battleState.stamina?.current) || 0;
            const maxStamina = Number(battleState.stamina?.max) || 0;

            if (currentStamina < slash.cost) {
                const shouldDrainBeforePotion =
                    maxStamina > 0 &&
                    maxStamina < 6000 &&
                    currentStamina > 0;

                if (shouldDrainBeforePotion) {
                    const drainSlash = chooseDrainSlash(
                        currentStamina,
                        Math.max(1, slash.cost)
                    );
                    if (drainSlash) {
                        addLog(
                            `${item.name}: only ${Math.floor(currentStamina)} stamina remains; ` +
                            `draining with ${drainSlash.cost}-STAM Slash before potion.`
                        );
                        slash = drainSlash;
                    }
                }

                if (currentStamina < slash.cost) {
                    if (!battleState.largeStaminaPotion) {
                        throw new Error(
                            `${item.name}: only ${Math.floor(currentStamina)} stamina remains and no Large Stamina Potion is available.`
                        );
                    }

                    await useLargeStaminaPotionDirect(item, battleState.largeStaminaPotion);
                    battleState = await readRegularBattleState(live.battleUrl);
                    currentStamina = Number(battleState.stamina?.current) || 0;

                    if (currentStamina < slash.cost) {
                        throw new Error(
                            `${item.name}: Large Stamina Potion was used, but stamina is still below ${slash.cost}.`
                        );
                    }

                    addLog(
                        `${item.name}: stamina restored to ${Math.floor(currentStamina).toLocaleString('en-US')}; resuming attack.`
                    );
                }
            }

            setStatus(
                `${item.name}: ${Math.round(currentDamage).toLocaleString('en-US')} / ` +
                `${Math.round(item.targetDamage).toLocaleString('en-US')} DMG · ` +
                `next ${slash.cost} STAM`
            );

            addLog(
                `${item.name}: POST damage.php → monster_id=${live.monsterId}, ` +
                `skill_id=${slash.skillId}, stamina_cost=${slash.cost}.`
            );

            const result = await directAttackByMonsterId(live.monsterId, slash);
            const data = result.data;
            const message = cleanText(data?.message || result.raw || '');

            addLog(
                `${item.name}: attack response [HTTP ${result.status}]: ` +
                `${message || JSON.stringify(data || {})}`
            );

            if (!result.ok) {
                if (/0 hp|zero hp|you are dead|you have died|cannot attack.*dead|can't attack.*dead/i.test(message)) {
                    addLog(`${item.name}: server reports player cannot attack at 0 HP; recovering.`);
                    await ensureRegularPlayerAlive(item, live.battleUrl);
                    await sleep(350);
                    continue;
                }

                if (
                    /removed due to inactivity|please rejoin|not joined|join the battle|must join|need to join/i.test(
                        message
                    )
                ) {
                    const rejoin = await directJoinByMonsterId(item, live.monsterId);

                    addLog(
                        `${item.name}: rejoin response [HTTP ${rejoin.status}]: ` +
                        `${rejoin.message || '(empty response)'}`
                    );

                    // Do not abort on the join response. A second join can be
                    // rejected simply because we are already participating.
                    // The next damage.php request decides whether we can attack.
                    await sleep(400);
                    continue;
                }

                if (/monster is already dead/i.test(message)) {
                    await moveToNextAttackTargetHeadless(state);
                    return;
                }

                if (/couldn.?t pierce|no stamina spent|divine shield/i.test(message)) {
                    await sleep(700);
                    continue;
                }

                if (/slow down|cooldown|attacking too quickly|take your time/i.test(message)) {
                    await sleep(1000);
                    continue;
                }

                if (/not enough stamina/i.test(message)) {
                    addLog(`${item.name}: server reports not enough stamina; refreshing stamina and attempting potion recovery.`);
                    const refreshed = await readRegularBattleState(live.battleUrl);
                    if (!refreshed.largeStaminaPotion) {
                        throw new Error(`${item.name}: not enough stamina and no Large Stamina Potion is available.`);
                    }
                    await useLargeStaminaPotionDirect(item, refreshed.largeStaminaPotion);
                    await sleep(350);
                    continue;
                }

                throw new Error(
                    `${item.name}: attack failed — ${message || `HTTP ${result.status}`}`
                );
            }

            if (divineState.isDivine) {
                // A successful attack consumes one existing marked turn. If this
                // same attack marks us again, reset to exactly 2 afterwards.
                // Marks never stack/prolong to 3, 4, ... turns.
                if (markedTurns > 0) markedTurns -= 1;
                if (responseMarksCurrentAttacker(message)) {
                    markedTurns = 2;
                    addLog(`${item.name}: YOU were Moon Marked → markedTurns reset to 2.`);
                } else if (markedTurns > 0) {
                    addLog(`${item.name}: Moon Mark consumed → ${markedTurns} marked attack${markedTurns === 1 ? '' : 's'} remaining.`);
                } else {
                    addLog(`${item.name}: Moon Mark cleared; normal Slash tier selection resumes.`);
                }
            }

            if (Number.isFinite(Number(data?.totaldmgdealt))) {
                currentDamage = Number(data.totaldmgdealt);
            } else {
                try {
                    const battleState = await readRegularBattleState(live.battleUrl);
                    currentDamage = Number(battleState.damage) || currentDamage;
                } catch {}
            }

            addLog(
                `${item.name}: now ${Math.round(currentDamage).toLocaleString('en-US')} / ` +
                `${Math.round(item.targetDamage).toLocaleString('en-US')} DMG.`
            );

            await sleep(850);
        }
    }

    async function directJoinRegularBattle(item) {
        const monsterId = monsterIdFromBattleUrl(item.battleUrl);
        if (!monsterId) {
            return { ok: false, unavailable: true, message: 'not a regular monster_id battle' };
        }

        const userId = getCookieValue('demon');
        if (!userId) {
            throw new Error(`${item.name}: cookie "demon" was not found.`);
        }

        const body =
            `monster_id=${encodeURIComponent(monsterId)}` +
            `&user_id=${encodeURIComponent(userId)}`;

        const response = await fetch(`${BASE_URL}/user_join_battle.php`, {
            method: 'POST',
            headers: {
                'Content-Type': 'application/x-www-form-urlencoded; charset=UTF-8'
            },
            body,
            credentials: 'same-origin'
        });

        const raw = cleanText(await response.text());
        const lower = raw.toLowerCase();

        const ok =
            raw.includes('You have successfully joined the battle.') ||
            lower.includes('already joined') ||
            lower.includes('already in this battle') ||
            lower.includes('already in battle') ||
            lower.includes('already participating') ||
            lower.includes('have joined this battle') ||
            lower.includes('you joined this battle') ||
            lower.includes('currently joined') ||
            lower.includes('cannot join') && (
                lower.includes('already') ||
                lower.includes('joined')
            );

        return { ok, unavailable: false, message: raw, status: response.status };
    }

    async function performDirectAttack(item, slash, frame = null) {
        const monsterId = monsterIdFromBattleUrl(item.battleUrl);

        if (monsterId) {
            const body =
                `monster_id=${encodeURIComponent(monsterId)}` +
                `&skill_id=${encodeURIComponent(slash.skillId)}` +
                `&stamina_cost=${encodeURIComponent(slash.cost)}`;

            const response = await fetch(`${BASE_URL}/damage.php`, {
                method: 'POST',
                headers: {
                    'Content-Type': 'application/x-www-form-urlencoded; charset=UTF-8'
                },
                body,
                credentials: 'same-origin'
            });

            const raw = await response.text();

            let data = null;
            try {
                data = JSON.parse(raw);
            } catch {
                data = parseAttackResponse(raw);
            }

            return {
                ok: !!data && String(data.status || '').trim().toLowerCase() === 'success',
                status: response.status,
                data,
                raw,
                nativeWorker: false
            };
        }

        // Dungeon battles: click the battle page's own native attack button.
        //
        // The uploaded dungeon page source shows that this click handler builds
        // its request with baseQS(), then adds skill_id/stamina_cost and the
        // live BCFG instance_id/dgmid before calling ep('ATTACK'). By clicking
        // the actual .attack-btn we let that exact page code run, without
        // guessing or trying to access lexical page helpers from the manager.
        if (!frame) {
            throw new Error(`${item.name}: dungeon attack requires the hidden battle frame.`);
        }

        let doc = frameDocument(frame);
        if (!doc) {
            throw new Error(`${item.name}: dungeon battle frame document is unavailable.`);
        }

        // The game's native battle page disables every .attack-btn for
        // 1000 ms after each accepted click. Calling HTMLElement.click() while
        // the button is disabled does nothing, leaving the previous success
        // notification on screen. Wait for the game's own lock to clear.
        const btn = await waitForDungeonAttackButtonReady(frame, slash.skillId, 2200);
        if (!btn || btn.disabled || btn.getAttribute('aria-busy') === 'true') {
            return {
                ok: false,
                status: 0,
                data: {
                    status: 'error',
                    message: 'Dungeon attack controls are still cooling down.'
                },
                raw: 'Dungeon attack controls are still cooling down.',
                nativeWorker: true,
                retryable: true
            };
        }

        doc = frameDocument(frame);
        const beforeDamage = getIframeDamage(doc);
        const beforeStamina = getIframeStaminaStats(doc).current;
        const beforeNote = cleanText(iframeNotificationText(doc) || '');

        btn.click();

        const observed = await waitForHiddenAttackResult(
            frame,
            beforeDamage,
            beforeStamina,
            beforeNote,
            4200
        );

        doc = observed?.doc || frameDocument(frame);
        const afterDamage = getIframeDamage(doc);
        const afterStamina = getIframeStaminaStats(doc).current;
        const note = cleanText(
            observed?.note ||
            iframeNotificationText(doc) ||
            ''
        );

        if (observed?.kind === 'changed') {
            return {
                ok: true,
                status: 200,
                data: {
                    status: 'success',
                    message: note || 'Dungeon native attack completed.',
                    totaldmgdealt: afterDamage,
                    stamina: afterStamina
                },
                raw: note || 'Dungeon native attack completed.',
                nativeWorker: true
            };
        }

        if (observed?.kind === 'dead') {
            return {
                ok: false,
                status: 200,
                data: {
                    status: 'error',
                    message: note || 'Monster is already dead.'
                },
                raw: note || 'Monster is already dead.',
                nativeWorker: true
            };
        }

        if (observed?.kind === 'rejoin') {
            return {
                ok: false,
                status: 200,
                data: {
                    status: 'error',
                    message: note || 'You were removed due to inactivity. Please rejoin the battle.'
                },
                raw: note || 'You were removed due to inactivity. Please rejoin the battle.',
                nativeWorker: true
            };
        }

        if (observed?.kind === 'negated' || observed?.kind === 'cooldown') {
            return {
                ok: false,
                status: 200,
                data: {
                    status: 'error',
                    message: note || (
                        observed.kind === 'negated'
                            ? "Your attack couldn't pierce the monster's defense. No stamina spent."
                            : 'Slow down — wait for the cooldown.'
                    )
                },
                raw: note || observed.kind,
                nativeWorker: true
            };
        }

        // Some server responses update the notification without changing
        // damage/stamina (e.g. not enough stamina, dead player, auto farm).
        // Only accept it here if it changed after this click; otherwise it is
        // merely the previous attack's notification.
        const currentNote = cleanText(note || '');
        if (currentNote && currentNote !== beforeNote) {
            return {
                ok: false,
                status: 200,
                data: {
                    status: 'error',
                    message: currentNote
                },
                raw: currentNote,
                nativeWorker: true
            };
        }

        return {
            ok: false,
            status: 0,
            data: {
                status: 'error',
                message: 'Dungeon native attack produced no new observable response.'
            },
            raw: 'Dungeon native attack produced no new observable response.',
            nativeWorker: true,
            retryable: true
        };
    }

    function applyDirectAttackResultToFrame(frame, data) {
        const doc = frameDocument(frame);
        if (!doc || !data) return;

        if (Number.isFinite(Number(data.totaldmgdealt))) {
            const damage = Number(data.totaldmgdealt);
            doc.querySelectorAll('#yourDamageValue').forEach(el => {
                el.textContent = Math.round(damage).toLocaleString('en-US');
            });
        }

        if (Number.isFinite(Number(data.stamina))) {
            const stamina = Number(data.stamina);
            const el = doc.querySelector('#stamina_span');
            if (el) el.textContent = Math.round(stamina).toLocaleString('en-US');
        }

        if (data.hp && Number.isFinite(Number(data.hp.value))) {
            const hpText = doc.querySelector('#hpText');
            if (hpText) {
                const max = Number(data.hp.max);
                hpText.textContent = Number.isFinite(max)
                    ? `${Math.round(Number(data.hp.value)).toLocaleString('en-US')} / ${Math.round(max).toLocaleString('en-US')}`
                    : Math.round(Number(data.hp.value)).toLocaleString('en-US');
            }
        }
    }

    async function runHeadlessAttackAutomation() {
        if (attackLoopRunning) return;
        attackLoopRunning = true;

        try {
            while (true) {
                let state = loadAttackAutomation();
                if (!state.active) return;

                // Do not pre-skip dungeon queue items based on guild_dungeon.php.
                // We have observed valid, alive, joinable battle.php targets being
                // skipped before the join code was ever reached. The live battle
                // preflight below is the authoritative source for dungeon viability.
                const item = currentAttackItem(state);
                if (!item) {
                    stopAttackAutomation('Attack queue is empty.');
                    return;
                }

                if (item.sourceType === 'dungeon') {
                    const ids = dungeonIdsFromBattleUrl(item.battleUrl);
                    addLog(
                        `${item.name}: dungeon queue item reached → ` +
                        `dgmid=${ids.dgmid || item.dgmid || '?'} ` +
                        `instance_id=${ids.instanceId || item.instanceId || '?'}`
                    );
                }

                // Regular and timed battles now use the exact mechanism that
                // was proven by the standalone Triton test: resolve the LIVE
                // data-monster-id from the source page, then POST join/attack.
                // No hidden iframe is involved in this path.
                if (item.sourceType !== 'dungeon') {
                    try {
                        await runDirectRegularTarget(state, item);
                    } catch (error) {
                        addLog(
                            `${item.name}: target failed (${error.message || error}); ` +
                            `skipping to the next selected monster instead of stopping the queue.`
                        );
                        setStatus(
                            `${item.name}: failed — moving to next selected monster.`,
                            true
                        );

                        // Reload the latest persisted state before advancing,
                        // because runDirectRegularTarget may have updated it.
                        const latestState = loadAttackAutomation();
                        if (!latestState.active) return;

                        await moveToNextAttackTargetHeadless(latestState);
                    }
                    continue;
                }

                // Dungeon battles still use their dgmid + instance_id route.
                // Crucially, inspect with fetch before ever loading the hidden
                // battle frame. Dead/finished monsters are skipped here.
                // Dungeon battle.php is authoritative.
                //
                // Do NOT let preflightBattleTarget() classify a dungeon monster
                // as dead. The live logs proved that its HTML heuristics can
                // return both false HP=0 and false "monster dead" results even
                // when the active dungeon battle still has HP remaining.
                //
                // We already have an activeBattleUrl/dgmid/instance_id in the
                // selected queue item. Open that actual battle page and inspect
                // its live DOM instead. ensureJoinedInFrame() will distinguish:
                //   - a real dead battle,
                //   - an already-joined live battle,
                //   - a live battle requiring Join.
                addLog(
                    `${item.name}: dungeon preflight bypassed; opening authoritative live battle page.`
                );

                let frame;
                try {
                    frame = await loadHeadlessBattle(item.battleUrl, 2);
                } catch (error) {
                    addLog(
                        `${item.name}: hidden dungeon page could not be loaded after retry ` +
                        `(${error.message || error}); skipping this encounter and continuing queue.`
                    );
                    setStatus(
                        `${item.name}: dungeon page load failed — moving to next target.`,
                        true
                    );
                    await moveToNextAttackTargetHeadless(state);
                    continue;
                }

                {
                    const liveDoc = frameDocument(frame);
                    const liveMonsterHp = getMonsterHpFromDoc(liveDoc);
                    const liveHpText =
                        liveMonsterHp &&
                        Number.isFinite(Number(liveMonsterHp.current)) &&
                        Number.isFinite(Number(liveMonsterHp.max))
                            ? ` · HP ${Math.round(Number(liveMonsterHp.current)).toLocaleString('en-US')}/${Math.round(Number(liveMonsterHp.max)).toLocaleString('en-US')}`
                            : '';

                    addLog(
                        `${item.name}: authoritative dungeon page loaded` +
                        liveHpText +
                        `${liveDoc?.querySelector('.attack-btn') ? ' · attack controls present' : ''}` +
                        `${liveDoc?.querySelector('#join-battle') ? ' · Join button present' : ''}.`
                    );
                }

                const joinState = await ensureJoinedInFrame(frame, item);
                if (joinState.dead) {
                    addLog(`${item.name}: died before join; skipping.`);
                    await moveToNextAttackTargetHeadless(state);
                    continue;
                }
                if (!joinState.ok) {
                    throw new Error(`${item.name}: could not join battle in background.`);
                }

                addLog(`${item.name}: hidden battle ready; attack controls detected.`);

                let doc = frameDocument(frame);
                let currentDamage = getIframeDamage(doc);

                if (currentDamage >= item.targetDamage) {
                    addLog(
                        `${item.name}: target already reached (${Math.round(currentDamage).toLocaleString('en-US')} / ${Math.round(item.targetDamage).toLocaleString('en-US')}).`
                    );
                    await moveToNextAttackTargetHeadless(state);
                    continue;
                }

                while (state.active) {
                    doc = frameDocument(frame);

                    if (battleIsDeadDoc(doc)) {
                        addLog(`${item.name}: monster died; moving to next target.`);
                        setStatus(`${item.name}: monster died — skipping.`);
                        await moveToNextAttackTargetHeadless(state);
                        break;
                    }

                    if (await ensureFramePlayerAlive(frame, item)) {
                        doc = frameDocument(frame);
                    }

                    if (!doc?.querySelector('.attack-btn')) {
                        const joined = await ensureJoinedInFrame(frame, item);

                        if (joined.dead) {
                            await moveToNextAttackTargetHeadless(state);
                            break;
                        }

                        if (!joined.ok) {
                            throw new Error(`${item.name}: lost battle membership and could not rejoin.`);
                        }

                        doc = frameDocument(frame);
                    }

                    currentDamage = getIframeDamage(doc);

                    if (currentDamage >= item.targetDamage) {
                        addLog(
                            `${item.name}: target reached (${Math.round(currentDamage).toLocaleString('en-US')} / ${Math.round(item.targetDamage).toLocaleString('en-US')}).`
                        );
                        setStatus(
                            `${item.name}: ${Math.round(currentDamage).toLocaleString('en-US')} / ` +
                            `${Math.round(item.targetDamage).toLocaleString('en-US')} DMG ✓`
                        );
                        await moveToNextAttackTargetHeadless(state);
                        break;
                    }

                    const remainingDamage = Math.max(0, item.targetDamage - currentDamage);

                    let estimatorState;
                    try {
                        estimatorState = await getDamageEstimatorState(false);
                    } catch (error) {
                        throw new Error(`Could not load current damage predictor state: ${error.message || error}`);
                    }

                    const prediction = DamageEstimator.calculateForTarget(
                        estimatorState,
                        item.damageModel,
                        remainingDamage
                    );

                    const estimatedStaminaNeeded = Number(
                        prediction.stamina?.expectedStamina
                    );

                    if (!Number.isFinite(estimatedStaminaNeeded)) {
                        throw new Error(`Could not estimate remaining stamina for ${item.name}.`);
                    }

                    let slash = chooseSlashAttack(estimatedStaminaNeeded);
                    const stamina = getIframeStaminaStats(doc);

                    if (
                        stamina.max > 0 &&
                        stamina.max < 6000 &&
                        stamina.current > 0 &&
                        stamina.current < slash.cost
                    ) {
                        const drainSlash = chooseDrainSlash(
                            stamina.current,
                            estimatedStaminaNeeded
                        );
                        if (drainSlash) slash = drainSlash;
                    }

                    if (stamina.current < slash.cost) {
                        const shouldKeepDraining =
                            stamina.max > 0 &&
                            stamina.max < 6000 &&
                            stamina.current > 0;

                        if (!shouldKeepDraining) {
                            await useLargePotionInFrame(frame, item);
                            continue;
                        }
                    }

                    const beforeDamage = currentDamage;
                    const beforeStamina = stamina.current;

                    setStatus(
                        `${item.name}: ${Math.round(beforeDamage).toLocaleString('en-US')} / ` +
                        `${Math.round(item.targetDamage).toLocaleString('en-US')} DMG · ` +
                        `next ${slash.cost} STAM`
                    );

                    addLog(
                        `${item.name}: ${monsterIdFromBattleUrl(item.battleUrl) ? "direct" : "native dungeon"} ${slash.name} (${slash.cost} STAM).`
                    );

                    const attackResult = await performDirectAttack(item, slash, frame);
                    const attackData = attackResult.data;
                    const message = cleanText(attackData?.message || attackResult.raw || '');

                    if (!attackResult.ok) {
                        if (attackResult.retryable) {
                            addLog(`${item.name}: dungeon attack not accepted yet; waiting for native attack lock and retrying.`);
                            await sleep(450);
                            continue;
                        }

                        if (/0 hp|zero hp|you are dead|you have died|cannot attack.*dead|can't attack.*dead/i.test(message)) {
                            addLog(`${item.name}: server reports player cannot attack at 0 HP; recovering.`);
                            await ensureFramePlayerAlive(frame, item);
                            await sleep(350);
                            continue;
                        }

                        if (/removed due to inactivity|please rejoin/i.test(message)) {
                            addLog(`${item.name}: battle membership expired; rejoining automatically.`);
                            const joined = await ensureJoinedInFrame(frame, item);
                            if (joined.dead) {
                                await moveToNextAttackTargetHeadless(state);
                                break;
                            }
                            await sleep(700);
                            continue;
                        }

                        if (/monster is already dead/i.test(message)) {
                            addLog(`${item.name}: monster is dead; moving to next target.`);
                            await moveToNextAttackTargetHeadless(state);
                            break;
                        }

                        if (/couldn.?t pierce|no stamina spent|divine shield/i.test(message)) {
                            addLog(`${item.name}: attack negated; continuing.`);
                            await sleep(650);
                            continue;
                        }

                        if (/slow down|cooldown|attacking too quickly|take your time/i.test(message)) {
                            await sleep(700);
                            continue;
                        }

                        if (/not enough stamina/i.test(message)) {
                            throw new Error(`${item.name}: server reports not enough stamina.`);
                        }

                        throw new Error(
                            `${item.name}: direct attack failed — ${message || `HTTP ${attackResult.status}`}`
                        );
                    }

                    applyDirectAttackResultToFrame(frame, attackData);

                    const afterDamage = Number.isFinite(Number(attackData.totaldmgdealt))
                        ? Number(attackData.totaldmgdealt)
                        : getIframeDamage(frameDocument(frame));

                    const dealt = Math.max(0, afterDamage - beforeDamage);

                    setStatus(
                        `${item.name}: ${Math.round(afterDamage).toLocaleString('en-US')} / ` +
                        `${Math.round(item.targetDamage).toLocaleString('en-US')} DMG · ` +
                        `last +${Math.round(dealt).toLocaleString('en-US')}`
                    );

                    addLog(
                        `${item.name}: +${Math.round(dealt).toLocaleString('en-US')} DMG · ` +
                        `${Math.round(afterDamage).toLocaleString('en-US')} / ` +
                        `${Math.round(item.targetDamage).toLocaleString('en-US')}.`
                    );

                    await sleep(650);
                    state = loadAttackAutomation();
                    if (!state.active) return;
                    continue;

                }
            }
        } finally {
            attackLoopRunning = false;
        }
    }

    async function moveToNextAttackTargetHeadless(state) {
        state.index += 1;
        state.runtimeBattleUrl = null;
        state.runtimeDamage = null;
        state.runtimeStamina = null;

        if (state.index >= state.queue.length) {
            state.active = false;
            state.lastStatus = 'All selected active targets processed.';
            saveAttackAutomation(state);
            setStatus(state.lastStatus);
            addLog(state.lastStatus);
            return;
        }

        const next = currentAttackItem(state);
        state.lastStatus = `Next: ${next.name} (${state.index + 1}/${state.queue.length}).`;
        saveAttackAutomation(state);
        setStatus(state.lastStatus);
        addLog(state.lastStatus);
        addLog(
            `Queue advance confirmed: index=${state.index}, ` +
            `remaining=${Math.max(0, state.queue.length - state.index)}.`
        );
    }

    async function runCurrentBattleAttackLoop() {
        if (attackLoopRunning) return;
        attackLoopRunning = true;

        try {
            while (true) {
                let automation = loadAttackAutomation();
                if (!automation.active) return;

                let item = currentAttackItem(automation);
                if (!item) {
                    stopAttackAutomation('Attack queue is empty.');
                    return;
                }

                if (item.sourceType === 'dungeon') {
                    automation = await skipInactiveDungeonTargets(automation);
                    if (!automation.active) {
                        setStatus(automation.lastStatus || 'All selected active targets processed.');
                        return;
                    }
                    item = currentAttackItem(automation);
                }

                if (location.pathname.toLowerCase() !== '/battle.php') {
                    location.href = item.battleUrl;
                    return;
                }

                if (!sameBattleUrl(location.href, item.battleUrl)) {
                    location.href = item.battleUrl;
                    return;
                }

                // If this battle page is already clearly dead/finished, do not
                // look for attack buttons and do not treat it as an error.
                if (isCurrentBattleClearlyDead()) {
                    addLog(`${item.name}: monster is already dead; skipping.`);
                    setStatus(`${item.name}: already dead — moving to next selected target.`);
                    await moveToNextAttackTarget(automation);
                    return;
                }

                // Initialize a private runtime snapshot from the visible page
                // only once. After that, background attack responses are the
                // authoritative values and the game UI itself is left untouched.
                if (
                    !automation.runtimeBattleUrl ||
                    !sameBattleUrl(automation.runtimeBattleUrl, item.battleUrl)
                ) {
                    automation.runtimeBattleUrl = item.battleUrl;
                    automation.runtimeDamage = parseVisibleNumber(
                        document.querySelector('#yourDamageValue')?.textContent || '0'
                    );
                    automation.runtimeStamina = getVisibleStaminaStats().current;
                    saveAttackAutomation(automation);
                }

                let currentDamage = getCurrentBattleDamage(automation, item);

                if (currentDamage >= item.targetDamage) {
                    addLog(
                        `${item.name}: target reached (${formatDamageCompact(currentDamage)} / ${formatDamageCompact(item.targetDamage)}).`
                    );
                    await moveToNextAttackTarget(automation);
                    return;
                }

                const remainingDamage = Math.max(
                    0,
                    item.targetDamage - currentDamage
                );

                let estimatorState;
                try {
                    estimatorState = await getDamageEstimatorState(false);
                } catch (error) {
                    throw new Error(`Could not load current damage predictor state: ${error.message || error}`);
                }

                const prediction = DamageEstimator.calculateForTarget(
                    estimatorState,
                    item.damageModel,
                    remainingDamage
                );

                const estimatedStaminaNeeded =
                    Number(prediction.stamina?.expectedStamina);

                if (!Number.isFinite(estimatedStaminaNeeded)) {
                    throw new Error(`Could not estimate remaining stamina for ${item.name}.`);
                }

                let slash = chooseSlashAttack(estimatedStaminaNeeded);
                const stamina = getAutomationStaminaStats(automation, item);
                const currentStamina = stamina.current;
                const maxStamina = stamina.max;

                const shouldDrainBeforePotion =
                    maxStamina > 0 &&
                    maxStamina < 6000 &&
                    currentStamina > 0 &&
                    currentStamina < slash.cost;

                if (shouldDrainBeforePotion) {
                    const drainSlash = chooseDrainSlash(
                        currentStamina,
                        estimatedStaminaNeeded
                    );

                    if (drainSlash) {
                        slash = drainSlash;
                    }
                }

                if (currentStamina < slash.cost) {
                    const smallMaxDrainStillPossible =
                        maxStamina > 0 &&
                        maxStamina < 6000 &&
                        currentStamina > 0;

                    if (!smallMaxDrainStillPossible) {
                        await useLargeStaminaPotion(automation, item);
                        await sleep(250);
                        continue;
                    }
                }

                setStatus(
                    `${item.name}: ${Math.round(currentDamage).toLocaleString('en-US')} / ` +
                    `${Math.round(item.targetDamage).toLocaleString('en-US')} DMG · ` +
                    `next ${slash.cost} STAM`
                );

                addLog(
                    `${item.name}: background ${slash.cost}-stamina Slash; ` +
                    `${formatDamageCompact(remainingDamage)} damage remaining.`
                );

                const data = await performBackgroundAttack(slash);
                const message = cleanText(data?.message || '');

                if (String(data.status).trim() !== 'success') {
                    if (message === 'Monster is already dead.') {
                        addLog(`${item.name}: monster died before our attack landed; skipping.`);
                        setStatus(`${item.name}: already dead — moving to next selected target.`);
                        await moveToNextAttackTarget(automation);
                        return;
                    }

                    if (
                        /slow down|cooldown/i.test(message)
                    ) {
                        await sleep(650);
                        continue;
                    }

                    if (
                        /couldn't pierce|no stamina spent/i.test(message)
                    ) {
                        addLog(`${item.name}: attack was negated; no damage/stamina change.`);
                        await sleep(ATTACK_POST_CLICK_DELAY_MS);
                        continue;
                    }

                    if (/not enough stamina/i.test(message)) {
                        await refreshAutomationBattleSnapshot(automation, item);
                        await sleep(250);
                        continue;
                    }

                    throw new Error(message || 'Background attack failed.');
                }

                // Keep all attack effects inside our control panel. Do not call
                // updateAttackSuccessUI(), do not click the site's attack button,
                // and do not change HP bars, animations, logs, or leaderboard.
                if (Number.isFinite(Number(data.totaldmgdealt))) {
                    automation.runtimeDamage = Number(data.totaldmgdealt);
                } else {
                    // Fallback if this response variant omits totaldmgdealt.
                    // Refresh only in the background, never visibly.
                    await refreshAutomationBattleSnapshot(automation, item);
                    automation = loadAttackAutomation();
                }

                if (Number.isFinite(Number(data.stamina))) {
                    automation.runtimeStamina = Number(data.stamina);
                }

                automation.runtimeBattleUrl = item.battleUrl;
                saveAttackAutomation(automation);

                const afterDamage = getCurrentBattleDamage(automation, item);
                const dealtThisAttack = Math.max(0, afterDamage - currentDamage);

                setStatus(
                    `${item.name}: ${Math.round(afterDamage).toLocaleString('en-US')} / ` +
                    `${Math.round(item.targetDamage).toLocaleString('en-US')} DMG · ` +
                    `last attack +${Math.round(dealtThisAttack).toLocaleString('en-US')}`
                );

                addLog(
                    `${item.name}: +${Math.round(dealtThisAttack).toLocaleString('en-US')} DMG · ` +
                    `${Math.round(afterDamage).toLocaleString('en-US')} / ${Math.round(item.targetDamage).toLocaleString('en-US')}.`
                );

                await sleep(ATTACK_POST_CLICK_DELAY_MS);
            }
        } finally {
            attackLoopRunning = false;
        }
    }

    async function resumeAttackAutomation() {
        const state = loadAttackAutomation();
        updateAttackButton();

        if (!state.active) return;

        // No visible navigation at all. All battle pages are loaded into a
        // hidden same-origin iframe and processed there.
        await runHeadlessAttackAutomation();
    }

    /* ============================================================
       TEMPORARY TRITON ATTACK TEST
       This deliberately ignores the normal multi-target attack queue.
    ============================================================ */

    let tritonJoinFrame = null;
    let tritonJoinRunning = false;

    function getTritonJoinFrame() {
        if (tritonJoinFrame && document.body.contains(tritonJoinFrame)) {
            return tritonJoinFrame;
        }
        const frame = document.createElement('iframe');
        frame.id = 'veyra-triton-join-test-frame';
        frame.setAttribute('aria-hidden', 'true');
        frame.tabIndex = -1;
        frame.style.cssText = [
            'position:fixed','left:-10000px','top:-10000px','width:2px','height:2px',
            'opacity:0','pointer-events:none','border:0'
        ].join(';');
        document.body.appendChild(frame);
        tritonJoinFrame = frame;
        return frame;
    }

    function waitForIframeLoad(frame, timeoutMs = 12000) {
        return new Promise((resolve, reject) => {
            let finished = false;
            const timer = setTimeout(() => {
                if (finished) return;
                finished = true;
                reject(new Error('Triton battle page timed out while loading.'));
            }, timeoutMs);
            frame.addEventListener('load', () => {
                if (finished) return;
                finished = true;
                clearTimeout(timer);
                setTimeout(resolve, 300);
            }, { once: true });
        });
    }

    async function waitForTritonState(frame, timeoutMs = 10000) {
        const started = Date.now();
        while (Date.now() - started < timeoutMs) {
            const doc = frame.contentDocument;
            if (doc) {
                const attacks = doc.querySelectorAll('.attack-btn');
                if (attacks.length) return { state:'joined', doc, attacks:attacks.length };
                const joinButton = doc.querySelector('#join-battle');
                if (joinButton) return { state:'joinable', doc, joinButton };
                const hpText = cleanText(doc.querySelector('#hpText')?.textContent || '');
                const pair = hpText.match(/([\d,]+)\s*\/\s*([\d,]+)\s*HP/i);
                if (pair && parseVisibleNumber(pair[1]) <= 0) return { state:'dead', doc };
            }
            await sleep(150);
        }
        return { state:'unknown', doc:frame.contentDocument };
    }


    function getTritonBattleId(battleUrl) {
        try {
            return new URL(battleUrl, location.origin).searchParams.get('id') || '';
        } catch {
            return '';
        }
    }

    function parseTritonAttackJson(raw) {
        const text = String(raw ?? '').replace(/^\uFEFF/, '').trim();
        if (!text) return null;

        try {
            return JSON.parse(text);
        } catch {}

        const start = text.indexOf('{');
        const end = text.lastIndexOf('}');
        if (start >= 0 && end > start) {
            try {
                return JSON.parse(text.slice(start, end + 1));
            } catch {}
        }

        return null;
    }

    async function nativeTritonAttack(frame, slash) {
        // Important: execute the click INSIDE the battle iframe via the small
        // userscript worker. This triggers the game's own registered
        // .attack-btn handler, which builds baseQS(), chooses ep('ATTACK'),
        // and sends whatever hidden/non-dungeon fields the game requires.
        const before = await workerSnapshot(frame);

        const reply = await frameWorkerRequest(
            frame,
            'attack',
            { skillId: slash.skillId },
            3000
        );

        if (!reply?.ok) {
            return {
                ok: false,
                reason: reply?.reason || 'worker attack click failed',
                before,
                after: await workerSnapshot(frame)
            };
        }

        const started = Date.now();
        let last = before;

        while (Date.now() - started < 5000) {
            await sleep(120);

            let snap;
            try {
                snap = await workerSnapshot(frame);
            } catch {
                continue;
            }

            last = snap;

            if (
                Number(snap.damage) !== Number(before.damage) ||
                Number(snap.stamina) !== Number(before.stamina)
            ) {
                return {
                    ok: true,
                    kind: 'changed',
                    before,
                    after: snap
                };
            }

            const note = cleanText(snap.notification || '');

            if (/removed due to inactivity|please rejoin/i.test(note)) {
                return { ok: false, kind: 'rejoin', reason: note, before, after: snap };
            }

            if (/monster is already dead/i.test(note)) {
                return { ok: false, kind: 'dead', reason: note, before, after: snap };
            }

            if (/couldn.?t pierce|no stamina spent/i.test(note)) {
                return { ok: true, kind: 'negated', reason: note, before, after: snap };
            }

            if (/slow down|cooldown|attacking too quickly|take your time/i.test(note)) {
                return { ok: false, kind: 'cooldown', reason: note, before, after: snap };
            }
        }

        return {
            ok: false,
            kind: 'unchanged',
            reason:
                `Native attack button was clicked inside the iframe, but after 5s ` +
                `damage stayed ${before.damage} and stamina stayed ${before.stamina}. ` +
                `Notification: ${cleanText(last?.notification || '') || '(none)'}`,
            before,
            after: last
        };
    }

    function getTritonFrameDamage(doc) {
        return parseVisibleNumber(
            doc?.querySelector('#yourDamageValue')?.textContent || '0'
        );
    }

    function getTritonFrameStamina(doc) {
        const span = doc?.querySelector('#stamina_span');
        if (!span) return { current: 0, max: 0 };

        const current = parseVisibleNumber(span.textContent);
        const pair = (span.parentElement?.textContent || '')
            .match(/([\d,]+)\s*\/\s*([\d,]+)/);

        return {
            current: pair ? parseVisibleNumber(pair[1]) : current,
            max: pair ? parseVisibleNumber(pair[2]) : 0
        };
    }

    function getTritonAttackButton(doc, skillId) {
        return doc?.querySelector(
            `.attack-btn[data-skill-id="${skillId}"]`
        ) || null;
    }

    async function waitForTritonAttackResult(
        frame,
        beforeDamage,
        beforeStamina,
        timeoutMs = 4500
    ) {
        const started = Date.now();

        while (Date.now() - started < timeoutMs) {
            const doc = frame.contentDocument;
            if (!doc) {
                await sleep(100);
                continue;
            }

            const damage = getTritonFrameDamage(doc);
            const stamina = getTritonFrameStamina(doc).current;
            const note = cleanText(
                doc.querySelector('#notification')?.textContent || ''
            );

            if (damage !== beforeDamage || stamina !== beforeStamina) {
                return {
                    kind: 'changed',
                    damage,
                    stamina,
                    note
                };
            }

            if (/removed due to inactivity|please rejoin/i.test(note)) {
                return { kind: 'rejoin', damage, stamina, note };
            }

            if (/monster is already dead/i.test(note)) {
                return { kind: 'dead', damage, stamina, note };
            }

            if (/couldn't pierce|no stamina spent/i.test(note)) {
                return { kind: 'negated', damage, stamina, note };
            }

            if (/slow down|cooldown|attacking too quickly/i.test(note)) {
                return { kind: 'cooldown', damage, stamina, note };
            }

            await sleep(100);
        }

        return {
            kind: 'unchanged',
            damage: getTritonFrameDamage(frame.contentDocument),
            stamina: getTritonFrameStamina(frame.contentDocument).current,
            note: cleanText(
                frame.contentDocument
                    ?.querySelector('#notification')?.textContent || ''
            )
        };
    }

    async function ensureTritonJoined(frame) {
        let state = await waitForTritonState(frame);

        if (state.state === 'joined') {
            return true;
        }

        if (state.state === 'dead') {
            return false;
        }

        if (state.state !== 'joinable') {
            throw new Error(
                'Triton page is neither joined nor joinable.'
            );
        }

        addLog('Triton test: clicking Join the Battle.');
        setStatus('Triton test: joining battle...');

        state.joinButton.click();

        const started = Date.now();

        while (Date.now() - started < 12000) {
            await sleep(300);

            const doc = frame.contentDocument;
            if (!doc) continue;

            const attacks = doc.querySelectorAll('.attack-btn');
            if (attacks.length) {
                addLog(
                    `Triton test: joined successfully; ${attacks.length} attack button(s) visible.`
                );
                return true;
            }
        }

        throw new Error(
            'Join button was clicked, but Triton never entered joined state.'
        );
    }

    async function attackTritonUntilTarget(frame, targetInfo, tritonUrl) {
        const targetDamage = targetInfo.targetDamage;
        const monster = targetInfo.monster;

        addLog(
            `Triton test: attack target is ${Math.round(targetDamage).toLocaleString('en-US')} DMG.`
        );

        let currentDamage = getTritonFrameDamage(frame.contentDocument);

        while (true) {
            if (currentDamage >= targetDamage) {
                setStatus(
                    `Triton target reached: ${Math.round(currentDamage).toLocaleString('en-US')} / ` +
                    `${Math.round(targetDamage).toLocaleString('en-US')} DMG`
                );
                addLog(
                    `Triton test: TARGET REACHED — ${Math.round(currentDamage).toLocaleString('en-US')} / ` +
                    `${Math.round(targetDamage).toLocaleString('en-US')} DMG.`
                );
                return;
            }

            const remainingDamage = Math.max(0, targetDamage - currentDamage);

            let slash = SLASH_ATTACKS[SLASH_ATTACKS.length - 1];

            try {
                const estimatorState = await getDamageEstimatorState(false);
                const prediction = DamageEstimator.calculateForTarget(
                    estimatorState,
                    monster.damageModel,
                    remainingDamage
                );

                const expectedStamina = Number(
                    prediction.stamina?.expectedStamina
                );

                if (Number.isFinite(expectedStamina)) {
                    slash = chooseSlashAttack(expectedStamina);
                }
            } catch (error) {
                addLog(
                    `Triton test: predictor unavailable (${error.message || error}); using 1-stamina Slash.`
                );
            }

            // Read stamina from the joined battle page before the hit.
            const staminaSnapshot = getTritonFrameStamina(frame.contentDocument);

            if (staminaSnapshot.current < slash.cost) {
                const affordable =
                    SLASH_ATTACKS.find(a => a.cost <= staminaSnapshot.current) ||
                    null;

                if (!affordable) {
                    throw new Error(
                        'No stamina remaining for another Triton attack.'
                    );
                }

                slash = affordable;
            }

            setStatus(
                `Triton: ${Math.round(currentDamage).toLocaleString('en-US')} / ` +
                `${Math.round(targetDamage).toLocaleString('en-US')} DMG · ` +
                `next ${slash.cost} STAM`
            );

            addLog(
                `Triton test: native iframe click → ${slash.name} (${slash.cost} STAM).`
            );

            const result = await nativeTritonAttack(frame, slash);

            if (!result.ok) {
                if (result.kind === 'rejoin') {
                    addLog('Triton test: removed for inactivity; rejoining.');
                    const joined = await ensureTritonJoined(frame);
                    if (!joined) {
                        throw new Error('Triton died before target was reached.');
                    }
                    await sleep(900);
                    continue;
                }

                if (result.kind === 'dead') {
                    throw new Error(
                        'Triton died before the configured damage threshold was reached.'
                    );
                }

                if (result.kind === 'cooldown') {
                    addLog(`Triton test: cooldown — ${result.reason}`);
                    await sleep(750);
                    continue;
                }

                throw new Error(
                    `Triton native attack failed: ${result.reason || 'unknown error'}`
                );
            }

            if (result.kind === 'negated') {
                addLog(`Triton test: attack negated — ${result.reason}`);
                await sleep(650);
                continue;
            }

            const beforeDamage = Number(result.before?.damage || currentDamage);
            currentDamage = Number(result.after?.damage || beforeDamage);
            const dealt = Math.max(0, currentDamage - beforeDamage);

            setStatus(
                `Triton: ${Math.round(currentDamage).toLocaleString('en-US')} / ` +
                `${Math.round(targetDamage).toLocaleString('en-US')} DMG · ` +
                `last +${Math.round(dealt).toLocaleString('en-US')}`
            );

            addLog(
                `Triton test: SUCCESS +${Math.round(dealt).toLocaleString('en-US')} DMG · ` +
                `${Math.round(currentDamage).toLocaleString('en-US')} / ` +
                `${Math.round(targetDamage).toLocaleString('en-US')}.`
            );


            // The live battle page enforces a 500 ms minimum attack interval.
            await sleep(650);
        }
    }

    async function joinTritonOnly() {
        if (tritonJoinRunning) return;
        tritonJoinRunning = true;

        try {
            const targetInfo = getCurrentTritonTargetDamage();
            if (!targetInfo) {
                throw new Error(
                    'No current Triton damage threshold is available.'
                );
            }

            const tritonUrl = getCurrentTritonBattleUrl();

            if (!tritonUrl) {
                throw new Error(
                    'No current Triton battle URL is available in the catalogue.'
                );
            }

            setStatus('Triton test: loading current battle in background...');
            addLog(`Triton test: loading ${tritonUrl} in hidden iframe.`);

            const frame = getTritonJoinFrame();

            const loadPromise = waitForIframeLoad(frame);
            frame.src = tritonUrl;
            await loadPromise;

            const joined = await ensureTritonJoined(frame);

            if (!joined) {
                throw new Error('Triton is dead; nothing to attack.');
            }

            await attackTritonUntilTarget(frame, targetInfo, tritonUrl);
        } finally {
            tritonJoinRunning = false;
        }
    }

    /* ============================================================
       GUI
    ============================================================ */

    function createGui() {
        if (document.getElementById(PANEL_ID)) return;

        const panel = document.createElement('div');
        panel.id = PANEL_ID;

        const minimized = isPanelMinimized();

        panel.innerHTML = `
            <div class="pve-main-header">
                <div id="pve-title">
                    ${minimized ? '🗺️ PvE · v1.4.7' : '🗺️ PvE Target Manager · v1.4.7'}
                </div>

                <button
                    id="pve-minimize"
                    type="button"
                    title="${minimized ? 'Expand' : 'Minimize'}"
                >
                    ${minimized ? '+' : '−'}
                </button>
            </div>

            <div
                id="pve-main-body"
                ${minimized ? 'style="display:none;"' : ''}
            >
                <div class="pve-toolbar">
                    <button
                        id="pve-attack"
                        type="button"
                        title="Temporary test: join Triton if needed, then trigger Triton's native attack handler from inside the hidden battle frame until its configured damage threshold is reached."
                    >
                        ⚔ Attack Triton
                    </button>

                    <button
                        id="pve-refresh"
                        type="button"
                        title="Normal scanning is automatic. This forces an immediate catalogue and loot-threshold refresh."
                    >
                        ↻ Refresh now
                    </button>

                    <button
                        id="pve-export"
                        type="button"
                    >
                        Export JSON
                    </button>
                </div>

                <div class="pve-help">
                    Everything scans automatically, including the first run.
                    New monsters discovered in dungeons or Gates are added automatically. Ended dungeon instances are used only to discover catalogue entries and are never attack targets.
                    Their highest <b>DMG req</b> is then used as the default damage target.
                    Automatic values are dim gray.
                    Type <b>3M</b>, <b>2.5B</b>, <b>1T</b>, etc. to override a monster manually.
                    Clear a manual value to restore Auto.
                    The <b>💪</b> value estimates expected stamina needed to deal that exact target damage with your current PvE gear/pets.
                    <b>Autoattack:</b> attacks all selected, currently available monsters to their configured damage thresholds.
                </div>

                <div id="pve-status">
                    Loading...
                </div>

                <div class="pve-section-title">
                    TARGETS
                </div>

                <div id="pve-results"></div>

                <div class="pve-section-title">
                    BACKGROUND LOG
                </div>

                <div id="pve-log"></div>
            </div>
        `;

        document.body.appendChild(panel);

        statusElement = panel.querySelector('#pve-status');
        resultsElement = panel.querySelector('#pve-results');
        logElement = panel.querySelector('#pve-log');
        refreshButton = panel.querySelector('#pve-refresh');
        attackButton = panel.querySelector('#pve-attack');

        attackButton.addEventListener('click', () => {
            const state = loadAttackAutomation();

            if (state.active) {
                stopAttackAutomation('Attack automation stopped by user.');
                return;
            }

            startAttackAutomation();
        });

        refreshButton.addEventListener('click', () => {
            damageEstimatorStatePromise = null;
            scanEverything({
                forceLootRefresh: true
            }).finally(() => scheduleStaminaEstimateRefresh(true));
        });

        panel.querySelector('#pve-export').addEventListener(
            'click',
            exportJson
        );

        const body = panel.querySelector('#pve-main-body');
        const minimizeButton = panel.querySelector('#pve-minimize');
        const title = panel.querySelector('#pve-title');

        function applyPanelMinimizedState(hidden) {
            body.style.display = hidden ? 'none' : '';
            minimizeButton.textContent = hidden ? '+' : '−';
            minimizeButton.title = hidden ? 'Expand' : 'Minimize';
            title.textContent = hidden ? '🗺️ PvE · v1.4.7' : '🗺️ PvE Target Manager · v1.4.7';
            panel.classList.toggle('pve-minimized', hidden);
        }

        applyPanelMinimizedState(minimized);

        minimizeButton.addEventListener('click', () => {
            const nowHidden = body.style.display !== 'none';

            applyPanelMinimizedState(nowHidden);
            setPanelMinimized(nowHidden);
        });

        // Render cached data instantly.
        renderTargetManager();
        updateButtons();
        updateAttackButton();

        // First run and later stale-cache refreshes both happen automatically.
        startAutomaticScan();
        const savedAttackState = loadAttackAutomation();
        if (savedAttackState.active) {
            addLog('Resuming saved multi-target attack queue.');
            runHeadlessAttackAutomation().catch(error => {
                stopAttackAutomation(`Attack automation error: ${error.message || error}`);
            });
        }
    }

    /* ============================================================
       STYLES
    ============================================================ */

    const style = document.createElement('style');

    style.textContent = `
        #${PANEL_ID} {
            position: fixed;
            right: 15px;
            bottom: 20px;
            width: 780px;
            max-width: calc(100vw - 30px);
            max-height: 90vh;
            background: rgba(18,18,25,.98);
            color: #eee;
            border: 1px solid #3e4358;
            border-radius: 10px;
            box-shadow: 0 10px 30px rgba(0,0,0,.65);
            font-family: Arial, sans-serif;
            font-size: 13px;
            overflow: hidden;
            z-index: 999999;
        }

        #${PANEL_ID}.pve-minimized {
            top: 6px;
            right: 6px;
            bottom: auto;
            width: auto;
            max-width: none;
            max-height: none;
            border-radius: 8px;
        }

        #${PANEL_ID}.pve-minimized .pve-main-header {
            gap: 8px;
            padding: 4px 7px;
            border-bottom: 0;
            font-size: 12px;
        }

        #${PANEL_ID}.pve-minimized #pve-minimize {
            padding: 0 3px;
            font-size: 16px;
            line-height: 1.2;
        }

        #${PANEL_ID} .pve-main-header {
            display: flex;
            align-items: center;
            justify-content: space-between;
            padding: 11px 14px;
            color: #f0cc67;
            background: #1d2030;
            border-bottom: 1px solid #3b4055;
            font-size: 15px;
            font-weight: bold;
        }

        #pve-minimize {
            color: #eee;
            background: transparent;
            border: 0;
            font-size: 20px;
            cursor: pointer;
        }

        #pve-main-body {
            padding: 12px;
            max-height: calc(90vh - 48px);
            overflow-y: auto;
        }

        .pve-toolbar {
            display: flex;
            gap: 6px;
            margin-bottom: 9px;
        }

        .pve-toolbar button {
            padding: 7px 11px;
            color: #fff;
            background: #363c4f;
            border: 1px solid #555d74;
            border-radius: 5px;
            cursor: pointer;
        }

        .pve-toolbar button:hover {
            filter: brightness(1.13);
        }

        .pve-toolbar button:disabled {
            opacity: .5;
            cursor: not-allowed;
        }

        #pve-attack {
            background: #254b32;
            border-color: #3d7650;
            font-weight: 700;
        }

        #pve-attack.active {
            background: #5a2929;
            border-color: #8a4444;
        }

        .pve-help {
            margin-bottom: 9px;
            padding: 8px 10px;
            color: #aeb5ca;
            background: #151824;
            border: 1px solid #30364a;
            border-radius: 6px;
            font-size: 11px;
            line-height: 1.45;
        }

        #pve-status {
            margin-bottom: 12px;
            padding: 7px 9px;
            color: #dec98c;
            background: #12141c;
            border: 1px solid #303443;
            border-radius: 5px;
        }

        .pve-section-title {
            margin: 12px 0 7px;
            color: #777f99;
            font-size: 10px;
            font-weight: bold;
            letter-spacing: 1px;
        }

        .pve-group-block {
            margin-bottom: 10px;
            border: 1px solid #4c536d;
            border-radius: 8px;
            overflow: hidden;
        }

        .pve-group-header {
            display: flex;
            align-items: center;
            gap: 8px;
            padding: 10px;
            background: #272c40;
            color: #e8d68e;
            cursor: pointer;
            user-select: none;
        }

        .pve-group-header strong {
            font-size: 14px;
        }

        .pve-source-block {
            margin: 7px;
            border: 1px solid #3c4257;
            border-radius: 7px;
            overflow: hidden;
        }

        .pve-source-header {
            display: flex;
            align-items: center;
            gap: 8px;
            padding: 9px 10px;
            background: #22273a;
            color: #80b5ff;
            cursor: pointer;
            user-select: none;
        }

        .pve-source-header strong {
            font-size: 13px;
        }

        .pve-section-block {
            margin: 6px 8px;
            background: #191c29;
            border: 1px solid #30364a;
            border-radius: 6px;
        }

        .pve-section-header {
            display: flex;
            align-items: center;
            gap: 8px;
            padding: 7px 9px;
            color: #e0ce91;
            background: #202435;
            cursor: pointer;
            user-select: none;
        }

        .pve-section-header small {
            margin-left: auto;
            color: #72798f;
        }

        .pve-arrow {
            display: inline-block;
            width: 12px;
            color: #8c94ac;
            flex: 0 0 12px;
        }

        .pve-checkbox-spacer {
            width: 16px;
            flex: 0 0 16px;
        }

        .pve-monster-list {
            padding: 5px 8px 8px 30px;
        }

        .pve-monster-row {
            display: grid;
            grid-template-columns:
                22px
                30px
                minmax(210px, 1fr)
                190px
                105px;
            align-items: center;
            gap: 7px;
            min-height: 38px;
            padding: 3px 0;
        }

        .pve-monster-row:hover {
            background: rgba(255,255,255,.025);
        }

        .pve-flat-monster-wrap {
            margin: 5px 10px 5px 28px;
            padding: 2px 0;
            border-bottom: 1px solid rgba(255,255,255,.035);
        }

        .pve-stamina-estimate {
            justify-self: end;
            min-width: 92px;
            padding: 5px 7px;
            color: #b9f0c9;
            background: #14221a;
            border: 1px solid #294432;
            border-radius: 5px;
            font-family: monospace;
            font-size: 11px;
            font-weight: bold;
            text-align: right;
            white-space: nowrap;
            cursor: help;
        }

        .pve-monster-img {
            width: 27px;
            height: 27px;
            object-fit: cover;
            border-radius: 4px;
            border: 1px solid #444a5f;
        }

        .pve-img-spacer {
            width: 27px;
            height: 27px;
        }

        .pve-monster-label {
            min-width: 0;
            display: flex;
            align-items: center;
            gap: 6px;
            flex-wrap: wrap;
        }

        .pve-monster-name {
            color: #ddd;
        }

        .pve-badge {
            padding: 2px 5px;
            border-radius: 999px;
            font-size: 9px;
            font-weight: bold;
            letter-spacing: .2px;
        }

        .pve-badge.timed {
            color: #f1d689;
            background: #3a3017;
            border: 1px solid #685728;
        }

        .pve-badge.boss {
            color: #dda7ff;
            background: #342041;
            border: 1px solid #64427b;
        }

        .pve-locked {
            margin-left: 4px;
            padding: 2px 6px;
            border-radius: 999px;
            color: #9da4b8;
            background: #292c35;
            border: 1px solid #424754;
            font-size: 9px;
            font-weight: bold;
            text-transform: uppercase;
        }

        .pve-damage-wrap {
            display: grid;
            grid-template-columns: 1fr 52px;
            gap: 5px;
            align-items: center;
        }

        .pve-damage-input {
            width: 100%;
            box-sizing: border-box;
            padding: 5px 7px;
            background: #11141d;
            border: 1px solid #3a4054;
            border-radius: 4px;
            font-family: monospace;
            text-align: right;
        }

        /* Automatic values are deliberately dimmer. */
        .pve-damage-input.auto {
            color: #7f8694;
            background: #10121a;
            border-color: #303544;
        }

        .pve-damage-input.manual {
            color: #fff;
            background: #151821;
            border-color: #57617b;
        }

        .pve-damage-input:focus {
            outline: none;
            color: #fff;
            border-color: #5c91d5;
        }

        .pve-damage-input.invalid {
            color: #ffaaaa;
            border-color: #d85757;
            background: #251518;
        }

        .pve-damage-mode {
            padding: 4px 3px;
            border-radius: 4px;
            font-size: 9px;
            font-weight: bold;
            text-align: center;
        }

        .pve-damage-mode.auto {
            color: #728a77;
            background: #17251b;
            border: 1px solid #2b4732;
        }

        .pve-damage-mode.manual {
            color: #e6c47d;
            background: #342b18;
            border: 1px solid #655326;
        }

        .pve-group-checkbox,
        .pve-source-checkbox,
        .pve-section-checkbox,
        .pve-monster-checkbox {
            width: 16px;
            height: 16px;
            accent-color: #4caf50;
            cursor: pointer;
            flex: 0 0 16px;
        }

        .pve-empty {
            padding: 8px;
            color: #71788e;
            font-style: italic;
        }

        #pve-log {
            height: 110px;
            overflow-y: auto;
            padding: 7px;
            color: #9298a8;
            background: #0e1017;
            border: 1px solid #2f3444;
            border-radius: 5px;
            font-family: monospace;
            font-size: 10px;
        }

        #pve-log > div {
            padding: 2px 0;
        }

        @media (max-width: 700px) {
            #${PANEL_ID}:not(.pve-minimized) {
                right: 5px;
                bottom: 5px;
                width: calc(100vw - 10px);
                max-width: none;
            }

            #${PANEL_ID}.pve-minimized {
                top: 4px;
                right: 4px;
                bottom: auto;
                width: auto;
            }

            .pve-monster-row {
                grid-template-columns: 22px 28px minmax(120px, 1fr);
            }

            .pve-damage-wrap {
                grid-column: 3;
                width: 100%;
                margin-bottom: 5px;
            }

            .pve-stamina-estimate {
                grid-column: 3;
                justify-self: stretch;
                text-align: left;
                margin-bottom: 5px;
            }
        }
    `;

    document.head.appendChild(style);

    /* ============================================================
       START
    ============================================================ */

    createGui();

})();
