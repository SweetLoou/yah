// ==UserScript==
// @name         EdgeSum Plus +
// @namespace    http://tampermonkey.net/
// @version      5.2.0
// @description  Optimized and refactored BlackJack helper
// @author       Refactored & Optimized by AI Assistant
// @match        https://luckybird.io/blackJack
// @match        https://luckybird.io/game/blackJack
// @match        https://luckybird.io/blackJack_m
// @match        https://luckybird.io/game/blackJack_m
// @grant        GM_addStyle
// @grant        GM_getValue
// @grant        GM_setValue
// ==/UserScript==

/**
 * @file EdgeSum Plus - Blackjack Helper
 *
 * This script automates play at LuckyBird's blackjack table using a fixed basic strategy.
 * It communicates directly with the game's underlying Vue component and Cocos2d scene to read
 * the player's and dealer's hands, decide the mathematically optimal action on each turn, and
 * send commands such as hit, stand, double or split.  The logic is organized into modular
 * classes for logging, state management, card reading, strategy evaluation, game integration,
 * looping, and user interface rendering.  A small on‑screen control panel allows users to
 * start and stop the bot and displays status messages.
 *
 * Important: the LuckyBird game exposes a method named `clickSpilt` (notice the misspelling).
 * That method name must be used verbatim when invoking the Vue component to perform a split.
 * Although it looks like a typo, it reflects the actual API and should not be changed.
 *
 * This script does not add new features beyond the original functionality.  It preserves
 * existing behavior while improving internal structure and documentation.  Configuration values
 * are centralized, and the code is written using modern ES6 syntax (const/let, template
 * literals, arrow functions) to improve readability and maintainability.
 */

(function() {
    'use strict';

    // ============================================================================
    // LOGGER UTILITY
    // Centralised logging helper with timestamped output and coloured prefixes.
    // ============================================================================
    // Improved Logger with indented output and timestamp. Maintains a simple
    // indentation level to simulate console.group without nesting endless groups.
    const Logger = {
        levelColors: { INFO: '#888', WARN: 'orange', ERROR: 'red' },
        indentLevel: 0,
        /**
         * Internal logging helper. Prepends timestamp and indentation.
         * @param {string} level Log level (INFO, WARN, ERROR)
         */
        _log(level, ...args) {
            const timestamp = new Date().toLocaleTimeString();
            const indent = ' '.repeat(this.indentLevel * 2);
            const prefix = `[${timestamp}][${level}]`;
            const style = `color: ${this.levelColors[level] || '#888'};${level === 'ERROR' ? ' font-weight:bold;' : ''}`;
            console.log(`%c${prefix} ${indent}`, style, ...args);
        },
        info(...args)  { this._log('INFO',  ...args); },
        warn(...args)  { this._log('WARN',  ...args); },
        error(...args) { this._log('ERROR', ...args); },
        /**
         * Begins a new grouped section. Simply increases indentation and logs the label.
         * @param {string} label Group label
         */
        group(label) {
            this.info(label);
            this.indentLevel++;
        },
        /**
         * Ends the current group by reducing indentation.
         */
        groupEnd() {
            if (this.indentLevel > 0) this.indentLevel--;
        }
    };

    // ============================================================================
    // CONFIGURATION & CONSTANTS
    // All configurable values and constants are centralised here for clarity.
    // ============================================================================
    const CONFIG = {
        POLL_INTERVAL_MS: 250,
        ACTION_DELAY_MS: 500,
        INSURANCE_ACTION_DELAY_MS: 750,
        DEAL_CLICK_POST_DELAY_MS: 250,
        DELAY_AFTER_HAND_TRANSITION_MS: 1000,
        DELAY_BETWEEN_SPLIT_HANDS_MS: 1000,
        MAX_RETRIES_FIND_GAME_INSTANCE: 25,
        MAX_RETRIES_GAME_PROPERTY: 8,
        MAX_WAIT_FOR_ACTION_CYCLES: 12,
        MAX_DEAL_WAIT_CYCLES: 20,
        CARD_READ_MAX_ATTEMPTS: 5,
        CARD_READ_RETRY_INTERVAL_MS: 500,
        AUTO_START_BETTING: true,
        MAX_SPLIT_HANDS_ALLOWED: 1,
        HIDE_UI_ELEMENTS: true,
    };

    // Platform identifiers for desktop vs mobile. The CURRENT property will be set at runtime.
    const PLATFORM_CONFIG = {
        DESKTOP: { VUE_COMPONENT_ID: '672225', URL_SUFFIX: '/blackJack' },
        MOBILE:  { VUE_COMPONENT_ID: '185298', URL_SUFFIX: '/blackJack_m' },
        CURRENT: null
    };

    // Enumerations for bot states and player actions.
    const BOT_STATES = {
        IDLE:                   'IDLE',
        INITIALIZING_ROUND:    'INITIALIZING_ROUND',
        PLAYER_TURN:           'PLAYER_TURN',
        WAITING_FOR_ACTION_RESULT: 'WAITING_FOR_ACTION_RESULT',
        ROUND_ENDING:          'ROUND_ENDING',
        ERROR:                 'ERROR'
    };
    const PLAYER_ACTIONS = {
        PLAY:         'play',
        HIT:          'hit',
        STAND:        'stand',
        SPLIT:        'split',
        DOUBLE:       'double',
        INSURANCE_NO: 'insuranceNoBtn',
        BUST:         'bust',
        BLACKJACK:    'blackjack'
    };
    const VUE_ACTION_METHODS = {
        [PLAYER_ACTIONS.PLAY]:        'spinEvent',
        [PLAYER_ACTIONS.HIT]:         'clickHit',
        [PLAYER_ACTIONS.STAND]:       'clickStand',
        // Note: The LuckyBird game uses a misspelled method name `clickSpilt` for splitting
        [PLAYER_ACTIONS.SPLIT]:       'clickSpilt',
        [PLAYER_ACTIONS.DOUBLE]:      'clickDouble',
        [PLAYER_ACTIONS.INSURANCE_NO]:'clickNoInsurance'
    };
    // Card constants for rank and suit translations.
    const CARD_CONSTANTS = {
        RANKS_DISPLAY: ['A','2','3','4','5','6','7','8','9','10','J','Q','K'],
        SUITS_DISPLAY: [null, 'S','H','C','D']
    };

    // ============================================================================
    // FIXED BASIC STRATEGY ENGINE
    // Implementation of a simple basic Blackjack strategy using lookup tables.
    // ============================================================================
    const FixedStrategy = {
        chart: {
            hard: {
                '9':  { 2:'Dh', 3:'Dh', 4:'Dh', 5:'Dh', 6:'Dh', 7:'H', 8:'H', 9:'H', 10:'H', 1:'H' },
                '10': { 2:'Dh', 3:'Dh', 4:'Dh', 5:'Dh', 6:'Dh', 7:'Dh', 8:'Dh', 9:'Dh', 10:'H', 1:'H' },
                '11': { 2:'Dh', 3:'Dh', 4:'Dh', 5:'Dh', 6:'Dh', 7:'Dh', 8:'Dh', 9:'Dh', 10:'Dh', 1:'H' },
                '12': { 2:'H',  3:'H',  4:'S',  5:'S',  6:'S',  7:'H', 8:'H', 9:'H', 10:'H', 1:'H' },
                '13': { 2:'S',  3:'S',  4:'S',  5:'S',  6:'S',  7:'H', 8:'H', 9:'H', 10:'H', 1:'H' },
                '14': { 2:'S',  3:'S',  4:'S',  5:'S',  6:'S',  7:'H', 8:'H', 9:'H', 10:'H', 1:'H' },
                '15': { 2:'S',  3:'S',  4:'S',  5:'S',  6:'S',  7:'H', 8:'H', 9:'H', 10:'H', 1:'H' },
                '16': { 2:'S',  3:'S',  4:'S',  5:'S',  6:'S',  7:'H', 8:'H', 9:'H', 10:'H', 1:'H' }
            },
            soft: {
                '13': { 2:'H', 3:'H', 4:'H', 5:'Dh',6:'Dh',7:'H',8:'H',9:'H',10:'H',1:'H' },
                '14': { 2:'H', 3:'H', 4:'H', 5:'Dh',6:'Dh',7:'H',8:'H',9:'H',10:'H',1:'H' },
                '15': { 2:'H', 3:'H', 4:'Dh',5:'Dh',6:'Dh',7:'H',8:'H',9:'H',10:'H',1:'H' },
                '16': { 2:'H', 3:'H', 4:'Dh',5:'Dh',6:'Dh',7:'H',8:'H',9:'H',10:'H',1:'H' },
                '17': { 2:'H', 3:'Dh',4:'Dh',5:'Dh',6:'Dh',7:'H',8:'H',9:'H',10:'H',1:'H' },
                '18': { 2:'S', 3:'Ds',4:'Ds',5:'Ds',6:'Ds',7:'S',8:'S',9:'H',10:'H',1:'H' },
                '19': { 2:'S', 3:'S', 4:'S', 5:'S', 6:'S', 7:'S', 8:'S', 9:'S',10:'S',1:'S' },
                '20': { 2:'S', 3:'S', 4:'S', 5:'S', 6:'S', 7:'S', 8:'S', 9:'S',10:'S',1:'S' }
            },
            pairs: {
                '2': { 2:'P',3:'P',4:'P',5:'P',6:'P',7:'P',8:'H',9:'H',10:'H',1:'H' },
                '3': { 2:'P',3:'P',4:'P',5:'P',6:'P',7:'P',8:'H',9:'H',10:'H',1:'H' },
                '4': { 2:'H',3:'H',4:'H',5:'P',6:'P',7:'H',8:'H',9:'H',10:'H',1:'H' },
                '5': { 2:'Dh',3:'Dh',4:'Dh',5:'Dh',6:'Dh',7:'Dh',8:'Dh',9:'Dh',10:'H',1:'H' },
                '6': { 2:'P',3:'P',4:'P',5:'P',6:'P',7:'H',8:'H',9:'H',10:'H',1:'H' },
                '7': { 2:'P',3:'P',4:'P',5:'P',6:'P',7:'P',8:'H',9:'H',10:'H',1:'H' },
                '8': { 2:'P',3:'P',4:'P',5:'P',6:'P',7:'P',8:'P',9:'P',10:'P',1:'P' },
                '9': { 2:'P',3:'P',4:'P',5:'P',6:'P',7:'S',8:'P',9:'P',10:'S',1:'S' },
                '10':{ 2:'S',3:'S',4:'S',5:'S',6:'S',7:'S',8:'S',9:'S',10:'S',1:'S' },
                '1': { 2:'P',3:'P',4:'P',5:'P',6:'P',7:'P',8:'P',9:'P',10:'P',1:'P' }
            }
        },
        /**
         * Determine the best action for the current hand according to basic strategy.
         * @param {string[]} playerCards Array of card strings, e.g. ['7H','8S']
         * @param {number} dealerCard Dealer upcard numeric value (A=1, K/Q/J=10)
         * @param {Object} handInfo Computed hand info from StrategyEngine.calculateScore
         * @returns {string} Player action key
         */
        getAction(playerCards, dealerCard, handInfo) {
            // Double is only allowed on two-card hands
            const canDouble = playerCards.length === 2;
            // Check if we can split
            if (handInfo.isPair) {
                const pairValue = StrategyEngine.getCardNumericalValue(playerCards[0]);
                const pairActionCode = this.chart.pairs[pairValue]?.[dealerCard];
                if (pairActionCode === 'P') return PLAYER_ACTIONS.SPLIT;
            }
            let actionCode;
            // Soft totals use separate table
            if (handInfo.soft) {
                actionCode = this.chart.soft[handInfo.score]?.[dealerCard] || 'S';
            } else {
                // Hard totals less than 9 always hit
                if (handInfo.score <= 8) actionCode = 'H';
                else if (handInfo.score >= 17) actionCode = 'S';
                else actionCode = this.chart.hard[handInfo.score]?.[dealerCard];
            }
            // Map strategy codes to specific player actions
            switch (actionCode) {
                case 'H':  return PLAYER_ACTIONS.HIT;
                case 'S':  return PLAYER_ACTIONS.STAND;
                case 'P':  return PLAYER_ACTIONS.SPLIT;
                case 'Dh': return canDouble ? PLAYER_ACTIONS.DOUBLE : PLAYER_ACTIONS.HIT;
                case 'Ds': return canDouble ? PLAYER_ACTIONS.DOUBLE : PLAYER_ACTIONS.STAND;
                default:   return PLAYER_ACTIONS.STAND;
            }
        }
    };

    // ============================================================================
    // GLOBAL STATE MANAGEMENT
    // Encapsulates all mutable state for the bot.
    // ============================================================================
    class BotState {
        constructor() {
            this.active = false;
            this.currentState = BOT_STATES.IDLE;
            this.actionLock = false;
            this.gameInstance = null;
            this.vueGameComponent = null;
            this.cocosPlayerFrame1Ref = null;
            this.cocosPlayerFrame2Ref = null;
            this.playButtonClickedThisDeal = false;
            this.dealAttemptTimeoutCounter = 0;
            this.currentSplitCount = 0;
            this.handsPlayedThisRound = 0;
            this.handBeforeAction = [];
            this.waitingForActionCounter = 0;
            this.actionThatLedToWait = null;
            this.scoreBeforeAction = 0;
            this.roundJustEnded = false;
            this.isWaitingOnSplitAces = false;
            this.isInSplitMode = false;
            this.activeSplitHandIndex = 0;
            this.firstSplitHandResolved = false;
            this.dealerUpcardForSplit = null;
            this.retryCountFindGame = 0;
            this.retryCountGameProperty = 0;
            this.lastActionTime = 0;
            this.actionCount = 0;
            this.roundCount = 0;
            this.bankrollAtRoundStart = 0;
            this.timerIntervalId = null;
            this.timerStartTime = 0;
            this.timerPausedElapsed = 0;
        }
        setState(newState) {
            // Change state only when different, without emitting UI state messages.
            if (this.currentState !== newState) {
                this.currentState = newState;
                this._handleStateTransition(newState);
            }
        }
        _handleStateTransition(newState) {
            switch (newState) {
                case BOT_STATES.IDLE:
                    Logger.info('Ready. Monitoring for new round...');
                    this._resetRoundTrackers();
                    break;
                case BOT_STATES.PLAYER_TURN:
                    this._resetActionTrackers();
                    break;
                case BOT_STATES.INITIALIZING_ROUND:
                    this.roundCount++;
                    // Start of a new round; log but do not group to avoid nested round logs
                    Logger.info(`Round ${this.roundCount}: Starting new round...`);
                    this.dealAttemptTimeoutCounter = 0;
                    this.roundJustEnded = false;
                    if (this.vueGameComponent?.userInfo?.balance) {
                        this.bankrollAtRoundStart = this.vueGameComponent.userInfo.balance;
                        Logger.info(`[Betting] Placing bet. Current bankroll: $${this.bankrollAtRoundStart}`);
                    }
                    break;
                case BOT_STATES.ERROR:
                    Logger.error('Bot entered error state');
                    break;
            }
        }
        _resetRoundTrackers() {
            this.playButtonClickedThisDeal = false;
            this.dealAttemptTimeoutCounter = 0;
            this.handsPlayedThisRound = 0;
            this.isInSplitMode = false;
            this.activeSplitHandIndex = 0;
            this.firstSplitHandResolved = false;
            this.scoreBeforeAction = 0;
            this.isWaitingOnSplitAces = false;
        }
        _resetActionTrackers() {
            this.actionThatLedToWait = null;
            this.waitingForActionCounter = 0;
            this.actionLock = false;
        }
        reset() {
            // Reset any indentation level (no grouping used currently)
            this.setState(BOT_STATES.IDLE);
            this.handBeforeAction = [];
            this._resetActionTrackers();
            this._resetRoundTrackers();
            this.currentSplitCount = 0;
            this.dealerUpcardForSplit = null;
            this.timerPausedElapsed = 0;
            if (this.timerIntervalId) clearInterval(this.timerIntervalId);
            Logger.info('Bot state fully reset');
        }
        recordAction(action) {
            this.lastActionTime = Date.now();
            this.actionCount++;
            Logger.info(`Action ${this.actionCount}: ${action}`);
        }
    }
    const botState = new BotState();

    // ============================================================================
    // UTILITY FUNCTIONS & CARD READER
    // A set of helpers for delays, element lookup, card value extraction etc.
    // ============================================================================
    const Utils = {
        /**
         * Pause execution for a specified number of milliseconds.
         * @param {number} ms Number of milliseconds to delay
         */
        delay: (ms) => new Promise(resolve => setTimeout(resolve, ms)),
        /**
         * Debounce function to limit the rate at which a function is called.
         * @param {Function} func Function to debounce
         * @param {number} delay Delay in milliseconds
         */
        debounce(func, delay) {
            let timeout;
            return function(...args) {
                const context = this;
                clearTimeout(timeout);
                timeout = setTimeout(() => func.apply(context, args), delay);
            };
        },
        /**
         * Retrieve an element by CSS selector or XPath.
         * @param {string} selectorString CSS selector or XPath string prefixed with 'xpath:'
         * @returns {Element|null} DOM element or null if not found
         */
        getElement(selectorString) {
            if (!selectorString) return null;
            try {
                if (selectorString.toLowerCase().startsWith('xpath:')) {
                    const result = document.evaluate(selectorString.substring(6), document, null, XPathResult.FIRST_ORDERED_NODE_TYPE, null);
                    return result.singleNodeValue;
                }
                return document.querySelector(selectorString);
            } catch (error) {
                Logger.error('Error finding element: ' + selectorString, error);
                return null;
            }
        },
        /**
         * Determine whether a DOM element can be clicked (visible, enabled, etc.).
         * @param {Element} element DOM element
         * @returns {boolean} True if element appears clickable
         */
        isElementClickable(element) {
            if (!element) return false;
            const style = window.getComputedStyle(element);
            const rect = element.getBoundingClientRect();
            return style.display !== 'none' &&
                style.visibility !== 'hidden' &&
                element.offsetParent !== null &&
                parseFloat(style.opacity) > 0 &&
                !element.classList.contains('tw-button-disabled') &&
                !element.disabled &&
                element.getAttribute('aria-disabled') !== 'true' &&
                style.pointerEvents !== 'none' &&
                rect.width > 0 && rect.height > 0;
        },
        /**
         * Detect if a CAPTCHA overlay is present on the page.
         * @returns {boolean} True if a CAPTCHA appears to be shown
         */
        checkForCaptcha() {
            const captchaEl = this.getElement('#recaptcha-container, .captcha-modal');
            return captchaEl && window.getComputedStyle(captchaEl).display !== 'none';
        },
        /**
         * Hide non-essential UI elements in the game for a cleaner view when the bot runs.
         */
        hideUIElements() {
            if (!CONFIG.HIDE_UI_ELEMENTS) return;
            const elementsToHide = ['.game-stats, .statistics-panel', '.sidebar, .side-panel, .right-panel', '.notifications, .toast-container, .alert-container', '.chat-container, .chat-panel, .live-chat'];
            elementsToHide.forEach(selector => {
                const elements = document.querySelectorAll(selector);
                elements.forEach(el => {
                    el.style.display = 'none';
                });
            });
        }
    };
    class CardReader {
        /**
         * Convert an internal Cocos card node to a readable rank/suit string.
         * @param {cc.Node} pokerCardNode Node from the game engine
         * @returns {string|null} Card string or null if unavailable
         */
        static getCardValueFromCocosCard(pokerCardNode) {
            if (!(pokerCardNode instanceof cc.Node)) return null;
            if (typeof pokerCardNode.getPokerData === 'function') {
                try {
                    const cardData = pokerCardNode.getPokerData();
                    if (cardData && cardData.hasOwnProperty('number') && cardData.hasOwnProperty('color')) {
                        return this._buildCardString(cardData.number, cardData.color);
                    }
                } catch (e) {
                    Logger.warn('CardReader: Error calling getPokerData()', e);
                }
            }
            return null;
        }
        /**
         * Build a card string from numeric values.
         * @param {number|string} numberVal Rank (1–13)
         * @param {number|string} suitVal Suit (1–4)
         * @returns {string|null} Card string like 'AH' or null if invalid
         */
        static _buildCardString(numberVal, suitVal) {
            const num = typeof numberVal === 'string' ? parseInt(numberVal, 10) : numberVal;
            const suit = typeof suitVal === 'string' ? parseInt(suitVal, 10) : suitVal;
            if (isNaN(num) || num < 1 || num > CARD_CONSTANTS.RANKS_DISPLAY.length) return null;
            if (isNaN(suit) || suit < 1 || suit >= CARD_CONSTANTS.SUITS_DISPLAY.length) return null;
            const rank = CARD_CONSTANTS.RANKS_DISPLAY[num - 1];
            const suitChar = CARD_CONSTANTS.SUITS_DISPLAY[suit];
            return rank && suitChar ? rank + suitChar : null;
        }
        /**
         * Read player's hand from the game, handling main vs split hands.
         * Uses retry loops to wait for cards to appear/stabilise.
         * @returns {string[]} Array of card strings
         */
        static getPlayerHandInternal() {
            let targetCocosFrame;
            if (botState.isInSplitMode) {
                targetCocosFrame = botState.activeSplitHandIndex === 0 ? botState.cocosPlayerFrame1Ref : botState.cocosPlayerFrame2Ref;
                if (!targetCocosFrame && botState.gameInstance) {
                    botState.cocosPlayerFrame2Ref = botState.gameInstance.playerFrame2;
                    targetCocosFrame = botState.cocosPlayerFrame2Ref;
                    if (targetCocosFrame) Logger.info('Dynamically fetched playerFrame2 for card reading.');
                }
            } else {
                targetCocosFrame = botState.cocosPlayerFrame1Ref ?? botState.gameInstance?.playerFrame;
            }
            const pokerArray = targetCocosFrame?.pokerArray;
            if (!Array.isArray(pokerArray)) return [];
            return pokerArray.map(node => this.getCardValueFromCocosCard(node)).filter(Boolean);
        }
        /**
         * Attempt to read the player's hand multiple times to ensure a stable read.
         * @returns {Promise<string[]>} Array of card strings
         */
        static async getPlayerHandWithRetries() {
            let previousHandString = '';
            let lastGoodRead = [];
            for (let attempt = 1; attempt <= CONFIG.CARD_READ_MAX_ATTEMPTS; attempt++) {
                const hand = this.getPlayerHandInternal();
                const currentHandString = hand.join(',');
                let targetFrame;
                if (botState.isInSplitMode) {
                    targetFrame = botState.activeSplitHandIndex === 0 ? botState.cocosPlayerFrame1Ref : botState.cocosPlayerFrame2Ref;
                } else {
                    targetFrame = botState.cocosPlayerFrame1Ref ?? botState.gameInstance?.playerFrame;
                }
                const expectedCardCount = targetFrame?.pokerArray?.length ?? 0;
                // If both expected count and current are zero, no cards to read
                if (expectedCardCount === 0 && hand.length === 0) return [];
                // If full set of cards read correctly
                if (hand.length === expectedCardCount && hand.every(c => c)) {
                    lastGoodRead = [...hand];
                    // On first attempt, wait half interval to confirm stability
                    if (attempt === 1 && CONFIG.CARD_READ_MAX_ATTEMPTS > 1) {
                        previousHandString = currentHandString;
                        await Utils.delay(CONFIG.CARD_READ_RETRY_INTERVAL_MS / 2);
                        continue;
                    }
                    // For subsequent attempts, return if identical consecutive reads
                    if (attempt > 1 && currentHandString === previousHandString) {
                        return hand;
                    }
                    if (attempt === CONFIG.CARD_READ_MAX_ATTEMPTS || CONFIG.CARD_READ_MAX_ATTEMPTS === 1) {
                        return hand;
                    }
                }
                if (attempt === CONFIG.CARD_READ_MAX_ATTEMPTS) {
                    if (lastGoodRead.length === expectedCardCount && lastGoodRead.every(c => c)) {
                        Logger.warn(`Card reading stabilization failed but returning last good full read: ${lastGoodRead.join(',')}`);
                        return lastGoodRead;
                    }
                    const nonNull = hand.filter(Boolean);
                    Logger.warn(`Card reading stabilization failed after ${CONFIG.CARD_READ_MAX_ATTEMPTS} attempts. Expected: ${expectedCardCount}, Best Read: ${nonNull.join(',') || 'None'}`);
                    return nonNull;
                }
                previousHandString = currentHandString;
                await Utils.delay(CONFIG.CARD_READ_RETRY_INTERVAL_MS);
            }
            Logger.warn('Card reading retries exhausted. Returning last good read or empty.');
            return lastGoodRead.length > 0 ? lastGoodRead : [];
        }
        /**
         * Get the dealer's upcard from the game instance.
         * @returns {string|null} Dealer's upcard or null if not available
         */
        static getDealerUpcardInternal() {
            const pokerArray = botState.gameInstance?.bankerFrame?.pokerArray;
            if (!Array.isArray(pokerArray) || pokerArray.length === 0) return null;
            for (const node of pokerArray) {
                if (node instanceof cc.Node) {
                    const card = this.getCardValueFromCocosCard(node);
                    if (card) return card;
                }
            }
            return null;
        }
    }

    // ============================================================================
    // STRATEGY ENGINE
    // Provides functions to convert card values and compute scores.
    // ============================================================================
    class StrategyEngine {
        /**
         * Convert a card string into its numeric value for strategy comparisons.
         * @param {string} cardString Card representation, e.g. '10H', 'AS'
         * @returns {number} Card numeric value (A=1, K/Q/J/10=10, 2–9 as is)
         */
        static getCardNumericalValue(cardString) {
            if (!cardString || typeof cardString !== 'string' || cardString.length < 1) return 0;
            const rank = cardString.startsWith('10') ? '10' : cardString.charAt(0).toUpperCase();
            if (rank === 'A') return 1;
            if (['K','Q','J','10'].includes(rank)) return 10;
            const num = parseInt(rank, 10);
            return (num >= 2 && num <= 9) ? num : 0;
        }
        /**
         * Calculate the score of a blackjack hand, determining if it is soft or a pair.
         * @param {string[]} hand Array of cards
         * @returns {{score:number, soft:boolean, isPair:boolean}} Score and metadata
         */
        static calculateScore(hand) {
            if (!hand || hand.length === 0) return { score: 0, soft: false, isPair: false };
            let score = 0;
            let aces = 0;
            for (const card of hand) {
                const value = this.getCardNumericalValue(card);
                if (value === 1) {
                    score += 11;
                    aces++;
                } else {
                    score += value;
                }
            }
            // Reduce ace values from 11 to 1 until under 21
            while (score > 21 && aces > 0) {
                score -= 10;
                aces--;
            }
            return {
                score,
                soft: aces > 0,
                isPair: (hand.length === 2 && this.getCardNumericalValue(hand[0]) === this.getCardNumericalValue(hand[1]))
            };
        }
        /**
         * Decide the next action using fixed basic strategy. Returns a PLAYER_ACTIONS key.
         * @param {string[]} playerHand Player's hand
         * @param {string} dealerUpcardString Dealer's upcard string
         * @returns {string} Player action key
         */
        static decideAction(playerHand, dealerUpcardString) {
            if (!playerHand || playerHand.length === 0) {
                Logger.warn('DecideAction: Empty player hand.');
                return PLAYER_ACTIONS.STAND;
            }
            if (!dealerUpcardString) {
                Logger.warn('DecideAction: Empty dealer upcard.');
                return PLAYER_ACTIONS.STAND;
            }
            const handInfo = this.calculateScore(playerHand);
            if (handInfo.score >= 21) return PLAYER_ACTIONS.STAND;
            const dealerCardAsNumber = this.getCardNumericalValue(dealerUpcardString);
            if (dealerCardAsNumber === 0) {
                Logger.warn(`Invalid dealer card for strategy: ${dealerUpcardString}. Defaulting to stand.`);
                return PLAYER_ACTIONS.STAND;
            }
            return FixedStrategy.getAction(playerHand, dealerCardAsNumber, handInfo);
        }
    }

    // ============================================================================
    // GAME INTEGRATION & GAME LOOP
    // Handles connecting to Vue components, managing the main game loop, and executing actions.
    // ============================================================================
    class GameIntegration {
        /**
         * Recursively search through Vue component tree to find the component containing the game instance.
         * @param {Object} vueInstance Root Vue instance
         * @param {number} depth Current recursion depth
         * @param {number} maxDepth Maximum recursion depth
         * @param {string} path Debug path
         * @returns {Object|null} Vue component that contains the game
         */
        static findVueComponentWithGame(vueInstance, depth = 0, maxDepth = 15, path = 'root') {
            if (!vueInstance || depth > maxDepth) return null;
            const gameCandidate = vueInstance.game ?? vueInstance.$data?.game;
            if (this._isValidGameInstance(gameCandidate)) {
                Logger.info(`Game instance found at Vue path: ${path}.`);
                return vueInstance;
            }
            if (vueInstance.$children?.length > 0) {
                for (let i = 0; i < vueInstance.$children.length; i++) {
                    const child = vueInstance.$children[i];
                    const found = this.findVueComponentWithGame(child, depth + 1, maxDepth, `${path}.$children[${i}]`);
                    if (found) return found;
                }
            }
            return null;
        }
        /**
         * Validate that an object appears to be a game instance.
         * @param {Object} gameCandidate Candidate game instance
         * @returns {boolean} True if valid game instance
         */
        static _isValidGameInstance(gameCandidate) {
            return !!gameCandidate &&
                typeof gameCandidate === 'object' &&
                gameCandidate._className?.toLowerCase().includes('scene') &&
                gameCandidate.playerFrame?.hasOwnProperty('pokerArray') &&
                gameCandidate.bankerFrame?.hasOwnProperty('pokerArray');
        }
        /**
         * Entry point: attempts to locate the game instance and start the game loop.
         */
        static async findGameInstanceAndStart() {
            if (!botState.active) return;
            UI.updateStatus('Initializing game connection...');
            // If we already have valid references, start the loop
            if (botState.gameInstance && botState.vueGameComponent && this._validateInstances()) {
                UI.updateStatus('Game instances validated. Starting main loop.');
                GameLoop.start();
                return;
            }
            // Wait for game canvas to be available
            if (!document.getElementById('gameCanvas')) {
                await this._handleCanvasNotFound();
                return;
            }
            // Find Vue component if not present
            if (!botState.vueGameComponent) {
                await this._findVueComponent();
                if (!botState.vueGameComponent) {
                    await this._handleVueComponentNotFound();
                    return;
                }
            }
            // Reset retry count and try to get game property
            botState.retryCountGameProperty = 0;
            this._retryForGameProperty();
        }
        /**
         * Validate that we still have valid game references.
         * @returns {boolean} True if game instances are still valid
         */
        static _validateInstances() {
            try {
                return botState.gameInstance &&
                    botState.vueGameComponent &&
                    this._isValidGameInstance(botState.gameInstance) &&
                    botState.cocosPlayerFrame1Ref;
            } catch (error) {
                Logger.warn('Instance validation failed:', error);
                return false;
            }
        }
        /**
         * Handle scenario where game canvas is not found.
         */
        static async _handleCanvasNotFound() {
            botState.retryCountFindGame++;
            if (botState.retryCountFindGame > CONFIG.MAX_RETRIES_FIND_GAME_INSTANCE) {
                UI.updateStatus(`Game Canvas not found after ${CONFIG.MAX_RETRIES_FIND_GAME_INSTANCE} retries. Stopping bot.`);
                this._stopBot();
                return;
            }
            UI.updateStatus(`Searching for Game Canvas (${botState.retryCountFindGame}/${CONFIG.MAX_RETRIES_FIND_GAME_INSTANCE})...`);
            setTimeout(() => this.findGameInstanceAndStart(), CONFIG.POLL_INTERVAL_MS);
        }
        /**
         * Attempt to locate the Vue component containing the game.
         */
        static async _findVueComponent() {
            UI.updateStatus('Canvas found. Locating Vue component...');
            const appEl = document.querySelector('#app');
            if (appEl?.__vue__) {
                const foundComp = this.findVueComponentWithGame(appEl.__vue__);
                if (foundComp) {
                    botState.vueGameComponent = foundComp;
                    Logger.info('Vue component successfully located via #app');
                    return;
                }
            }
            // Fallback: search all DOM elements with __vue__ property
            const allElements = document.querySelectorAll('*');
            for (const el of allElements) {
                if (el.__vue__) {
                    const foundComp = this.findVueComponentWithGame(el.__vue__);
                    if (foundComp) {
                        botState.vueGameComponent = foundComp;
                        Logger.info('Vue component found via fallback search');
                        return;
                    }
                }
            }
        }
        /**
         * Handle scenario where Vue component could not be found.
         */
        static async _handleVueComponentNotFound() {
            botState.retryCountFindGame++;
            if (botState.retryCountFindGame > CONFIG.MAX_RETRIES_FIND_GAME_INSTANCE) {
                UI.updateStatus(`Vue component not found after ${CONFIG.MAX_RETRIES_FIND_GAME_INSTANCE} retries. Stopping bot.`);
                this._stopBot();
                return;
            }
            setTimeout(() => this.findGameInstanceAndStart(), CONFIG.POLL_INTERVAL_MS);
        }
        /**
         * Retry obtaining the game instance from the Vue component.
         */
        static _retryForGameProperty() {
            if (!botState.active) return;
            if (!botState.vueGameComponent) {
                UI.updateStatus('CRITICAL: Vue component lost. Restarting search.');
                this._restartSearch();
                return;
            }
            botState.retryCountGameProperty++;
            UI.updateStatus(`Vue component found. Waiting for game property (${botState.retryCountGameProperty}/${CONFIG.MAX_RETRIES_GAME_PROPERTY})`);
            const gameCandidate = botState.vueGameComponent?.game ?? botState.vueGameComponent?.$data?.game;
            if (this._isValidGameInstance(gameCandidate)) {
                botState.gameInstance = gameCandidate;
                botState.cocosPlayerFrame1Ref = botState.gameInstance?.playerFrame;
                botState.cocosPlayerFrame2Ref = botState.gameInstance?.playerFrame2;
                if (!botState.cocosPlayerFrame1Ref) {
                    Logger.error('CRITICAL: gameInstance.playerFrame (main hand) is missing!');
                    this._stopBot();
                    return;
                }
                Logger.info(`SUCCESS: Cocos2D Game Scene acquired. playerFrame1Ref is set. playerFrame2Ref is ${botState.cocosPlayerFrame2Ref ? 'found' : 'not found (expected)'}.`);
                UI.updateStatus('Game instance ready! Starting bot...');
                botState.setState(BOT_STATES.IDLE);
                if (botState.active) GameLoop.start();
                return;
            }
            if (botState.retryCountGameProperty >= CONFIG.MAX_RETRIES_GAME_PROPERTY) {
                UI.updateStatus(`Game property validation failed after ${CONFIG.MAX_RETRIES_GAME_PROPERTY} attempts. Stopping.`);
                this._stopBot();
                return;
            }
            setTimeout(() => this._retryForGameProperty(), CONFIG.POLL_INTERVAL_MS / 2);
        }
        /**
         * Restart the search for the game and Vue instances.
         */
        static _restartSearch() {
            botState.gameInstance = null;
            botState.vueGameComponent = null;
            botState.cocosPlayerFrame1Ref = null;
            botState.cocosPlayerFrame2Ref = null;
            botState.retryCountFindGame = 0;
            botState.reset();
            this.findGameInstanceAndStart();
        }
        /**
         * Stop the bot and update the UI accordingly.
         */
        static _stopBot() {
            botState.active = false;
            botState.setState(BOT_STATES.IDLE);
            if (UI.toggleButton) UI.toggleButton.textContent = 'Start';
            UI.updateStatus('Bot stopped.');
        }
    }
    class GameLoop {
        static lastTimestamp = 0;
        static accumulator = 0;
        /** Start the requestAnimationFrame loop if the bot is active. */
        static start() {
            if (!botState.active) {
                UI.updateStatus('Bot deactivated.');
                return;
            }
            this.lastTimestamp = 0;
            this.accumulator = 0;
            requestAnimationFrame(this._frameLoop.bind(this));
        }
        /**
         * The rAF loop with fixed timestep logic. Handles game cycles at defined intervals.
         * @param {DOMHighResTimeStamp} timestamp Current timestamp
         */
        static async _frameLoop(timestamp) {
            if (!botState.active) {
                UI.updateStatus('Bot loop stopped.');
                return;
            }
            if (this.lastTimestamp === 0) this.lastTimestamp = timestamp;
            const deltaTime = timestamp - this.lastTimestamp;
            this.lastTimestamp = timestamp;
            this.accumulator += deltaTime;
            const interval = this._getLoopDelay();
            // Execute as many cycles as fit in the accumulated time
            while (this.accumulator >= interval) {
                try {
                    await this._executeGameCycle();
                } catch (error) {
                    Logger.error('FATAL ERROR in game loop:', error.message, error.stack);
                    UI.updateStatus('Critical error detected. Please check console. Stopping bot.');
                    botState.setState(BOT_STATES.ERROR);
                    GameIntegration._stopBot();
                    return;
                }
                this.accumulator -= interval;
            }
            if (botState.active) requestAnimationFrame(this._frameLoop.bind(this));
        }
        /**
         * Perform a single iteration of the game cycle. Handles all bot states.
         */
        static async _executeGameCycle() {
            // If captcha pops up, pause bot until resolved
            if (Utils.checkForCaptcha()) {
                UI.updateStatus('CAPTCHA detected! Please solve manually.');
                botState.setState(BOT_STATES.IDLE);
                return;
            }
            // Validate game references
            if (!this._validateGameInstances()) {
                UI.updateStatus('Game instances invalid. Reconnecting...');
                GameIntegration.findGameInstanceAndStart();
                return;
            }
            // Wait if split aces are being automatically played
            if (botState.isWaitingOnSplitAces) {
                UI.updateStatus('Waiting for split Aces to be played automatically...');
                // When no buttons available, assume done
                if (!this._anyActionButtonsAvailableViaVueState()) {
                    Logger.info('Split Aces turn finished. Ending round.');
                    botState.isWaitingOnSplitAces = false;
                    botState.setState(BOT_STATES.ROUND_ENDING);
                }
                return;
            }
            // Optionally hide UI
            Utils.hideUIElements();
            // Handle state-specific logic
            switch (botState.currentState) {
                case BOT_STATES.IDLE:
                    await this._handleIdleState();
                    break;
                case BOT_STATES.INITIALIZING_ROUND:
                    await this._handleInitializingRoundState();
                    break;
                case BOT_STATES.PLAYER_TURN:
                    await this._handlePlayerTurnState();
                    break;
                case BOT_STATES.WAITING_FOR_ACTION_RESULT:
                    await this._handleWaitingForActionState();
                    break;
                case BOT_STATES.ROUND_ENDING:
                    await this._handleRoundEndingState();
                    break;
                case BOT_STATES.ERROR:
                    await this._handleErrorState();
                    break;
                default:
                    Logger.warn(`Unknown bot state: ${botState.currentState}`);
                    botState.setState(BOT_STATES.IDLE);
            }
        }
        /**
         * Logic executed when idle: decide whether to start a new round or recover.
         */
        static async _handleIdleState() {
            // Reset split status if out of sync
            if (botState.isInSplitMode) {
                Logger.warn('In IDLE state but isInSplitMode is true. Resetting split state.');
                botState.reset();
                return;
            }
            // Start new round if last round ended and can click start
            if (botState.roundJustEnded && this._canStartNewRound()) {
                botState.setState(BOT_STATES.INITIALIZING_ROUND);
                return;
            }
            botState.currentSplitCount = 0;
            botState.dealerUpcardForSplit = null;
            // Auto start if enabled and ready
            if (CONFIG.AUTO_START_BETTING && this._canStartNewRound()) {
                botState.setState(BOT_STATES.INITIALIZING_ROUND);
                return;
            }
            // Recover from unexpected turn state
            const playerHand = await CardReader.getPlayerHandWithRetries();
            if (playerHand.length > 0 && !this._canStartNewRound()) {
                Logger.warn('[Bot Recovery] Round in progress detected while in IDLE state. Forcing PLAYER_TURN.');
                botState.setState(BOT_STATES.PLAYER_TURN);
            }
        }
        /**
         * Handles the start of a round: clicks play and waits for cards to be dealt.
         */
        static async _handleInitializingRoundState() {
            if (!botState.playButtonClickedThisDeal) {
                const playClicked = await this._executeVueAction(PLAYER_ACTIONS.PLAY);
                if (playClicked) {
                    botState.playButtonClickedThisDeal = true;
                    await Utils.delay(CONFIG.DEAL_CLICK_POST_DELAY_MS);
                } else {
                    return;
                }
            }
            const playerHand = await CardReader.getPlayerHandWithRetries();
            const actionsAvailable = this._anyActionButtonsAvailableViaVueState();
            // Once two cards or actions available, proceed to player turn
            if (playerHand.length >= 2 || actionsAvailable) {
                if (playerHand.length >= 2) {
                    Logger.info('Cards dealt. Player hand:', playerHand.join(','));
                } else {
                    Logger.info('Action buttons detected before cards were read. Assuming deal is complete.');
                }
                botState.setState(BOT_STATES.PLAYER_TURN);
                return;
            }
            botState.dealAttemptTimeoutCounter++;
            if (botState.dealAttemptTimeoutCounter > CONFIG.MAX_DEAL_WAIT_CYCLES) {
                Logger.warn('Deal timeout (failsafe). Returning to idle to re-evaluate.');
                botState.setState(BOT_STATES.IDLE);
            }
        }
        /**
         * Executes logic during the player's decision phase.
         */
        static async _handlePlayerTurnState() {
            if (botState.actionLock) return;
            const playerHand = await CardReader.getPlayerHandWithRetries();
            // If no cards, likely round ended
            if (this._checkForEarlyExitConditions(playerHand)) return;
            const currentDealerUpcard = CardReader.getDealerUpcardInternal();
            // Handle insurance/blackjack scenarios
            if (await this._handleInsuranceAndDealerBlackjack(playerHand, currentDealerUpcard)) return;
            // If player hand is complete (bust or 21), transition
            if (await this._handlePlayerHandCompletion(playerHand)) return;
            // Use stored upcard for split or current dealer card for strategy
            const dealerCardForStrategy = botState.isInSplitMode ? botState.dealerUpcardForSplit : currentDealerUpcard;
            if (!dealerCardForStrategy) {
                Logger.warn('Dealer upcard not available for strategy. Waiting...');
                return;
            }
            await this._determineAndExecuteMove(playerHand, dealerCardForStrategy);
        }
        /**
         * If there are no player cards, check if we can start a new round and exit early.
         * @param {string[]} playerHand Current player hand
         * @returns {boolean} True if early exit occurs
         */
        static _checkForEarlyExitConditions(playerHand) {
            const handIdentifier = botState.isInSplitMode ? `Split Hand ${botState.activeSplitHandIndex + 1}` : 'Player Hand';
            if (!playerHand || playerHand.length === 0) {
                if (this._canStartNewRound()) {
                    Logger.info(`${handIdentifier}: No cards found and new round can start. Round likely ended. To IDLE.`);
                    botState.setState(BOT_STATES.IDLE);
                }
                return true;
            }
            return false;
        }
        /**
         * Handle insurance and dealer blackjack cases.
         * @param {string[]} playerHand Player's hand
         * @param {string|null} currentDealerUpcard Dealer upcard
         * @returns {Promise<boolean>} True if a special case was handled and no further action is needed
         */
        static async _handleInsuranceAndDealerBlackjack(playerHand, currentDealerUpcard) {
            // If dealer has two cards, check for blackjack when player has two and not in split
            if (!botState.isInSplitMode && playerHand.length === 2) {
                const dealerHand = botState.gameInstance?.bankerFrame?.pokerArray?.map(c => CardReader.getCardValueFromCocosCard(c)).filter(Boolean) ?? [];
                if (dealerHand.length === 2) {
                    const { score: dealerScore } = StrategyEngine.calculateScore(dealerHand);
                    if (dealerScore === 21) {
                        Logger.info('[Bot Result] Dealer has Blackjack. Ending round.');
                        botState.setState(BOT_STATES.ROUND_ENDING);
                        return true;
                    }
                }
            }
            // Offer insurance only when not in split mode and dealer upcard is Ace
            if (!botState.isInSplitMode && currentDealerUpcard && /^A/i.test(currentDealerUpcard) && this._isInsuranceOffered()) {
                UI.updateStatus('Declining insurance...');
                const actionSuccess = await this._executeVueAction(PLAYER_ACTIONS.INSURANCE_NO);
                if (actionSuccess) {
                    botState.handBeforeAction = [...playerHand];
                    botState.actionThatLedToWait = PLAYER_ACTIONS.INSURANCE_NO;
                    botState.setState(BOT_STATES.WAITING_FOR_ACTION_RESULT);
                }
                return true;
            }
            return false;
        }
        /**
         * Handle automatic hand completion when player busts or reaches 21.
         * @param {string[]} playerHand Player's hand
         * @returns {Promise<boolean>} True if hand is completed and no further action is needed
         */
        static async _handlePlayerHandCompletion(playerHand) {
            const { score: playerScore, soft: isSoft } = StrategyEngine.calculateScore(playerHand);
            if (playerScore < 21) return false;
            const handIdentifier = botState.isInSplitMode ? `Split Hand ${botState.activeSplitHandIndex + 1}` : 'Player Hand';
            const handDescription = `[${playerHand.join(', ')}] (${playerScore} ${isSoft ? 'Soft' : 'Hard'})`;
            Logger.info(`${handIdentifier} auto-finalized: Score ${playerScore} (${handDescription})`);
            botState.handBeforeAction = [...playerHand];
            botState.scoreBeforeAction = playerScore;
            botState.actionThatLedToWait = playerScore > 21 ? PLAYER_ACTIONS.BUST : PLAYER_ACTIONS.BLACKJACK;
            botState.actionLock = true;
            botState.setState(BOT_STATES.WAITING_FOR_ACTION_RESULT);
            return true;
        }
        /**
         * Determine the optimal move using the strategy engine and execute it via Vue.
         * @param {string[]} playerHand Player's hand
         * @param {string} dealerCardForStrategy Dealer card used for strategy evaluation
         */
        static async _determineAndExecuteMove(playerHand, dealerCardForStrategy) {
            let optimalAction = StrategyEngine.decideAction(playerHand, dealerCardForStrategy);
            // Verify that action is allowed at this moment
            if (!this._canPerformAction(optimalAction)) {
                // Provide fallback when split unavailable
                if (optimalAction === PLAYER_ACTIONS.SPLIT) {
                    Logger.error(`[CRITICAL STRATEGY DEVIATION] Recommended action was SPLIT, but it is unavailable. Hand: [${playerHand.join(', ')}]. Defaulting to STAND.`);
                } else {
                    Logger.warn(`Strategy action '${optimalAction}' is unavailable. Defaulting to STAND.`);
                }
                optimalAction = PLAYER_ACTIONS.STAND;
            }
            // Special handling when splitting
            if (optimalAction === PLAYER_ACTIONS.SPLIT && !botState.isInSplitMode) {
                const isAces = playerHand.length === 2 && StrategyEngine.getCardNumericalValue(playerHand[0]) === 1;
                if (isAces) {
                    Logger.info('Detected Ace split. Engaging special handling.');
                    UI.updateStatus('Splitting Aces... entering wait mode.');
                    botState.isWaitingOnSplitAces = true;
                    await this._executeVueAction(PLAYER_ACTIONS.SPLIT);
                    return;
                }
                // Store dealer upcard for subsequent split hands
                const currentDealerUpcard = CardReader.getDealerUpcardInternal();
                if (currentDealerUpcard) {
                    botState.dealerUpcardForSplit = currentDealerUpcard;
                    Logger.info(`Decision is to SPLIT. Storing dealer upcard: ${botState.dealerUpcardForSplit} for subsequent hands.`);
                } else {
                    Logger.error('CRITICAL: Cannot SPLIT because current dealer upcard is null. Strategy will fallback.');
                    const fallbackAction = StrategyEngine.calculateScore(playerHand).score < 17 ? PLAYER_ACTIONS.HIT : PLAYER_ACTIONS.STAND;
                    await this._prepareAndExecuteAction(fallbackAction, playerHand);
                    return;
                }
            }
            await this._prepareAndExecuteAction(optimalAction, playerHand);
        }
        /**
         * Compose status message and execute the Vue action.
         * @param {string} action Action key
         * @param {string[]} playerHand Current player hand
         */
        static async _prepareAndExecuteAction(action, playerHand) {
            const handIdentifier = botState.isInSplitMode ? `Split Hand ${botState.activeSplitHandIndex + 1}` : 'Player Hand';
            const dealerCardForStrategy = botState.dealerUpcardForSplit ?? CardReader.getDealerUpcardInternal();
            const { score: playerScore, soft: isSoft } = StrategyEngine.calculateScore(playerHand);
            const handDescription = `[${playerHand.join(', ')}] (${playerScore} ${isSoft ? 'Soft' : 'Hard'})`;
            const statusMsg = `${handIdentifier}: ${handDescription} vs Dealer ${dealerCardForStrategy} → ${action.toUpperCase()}`;
            UI.updateStatus(statusMsg);
            Logger.info(statusMsg);
            const actionSuccess = await this._executeVueAction(action);
            if (actionSuccess) {
                botState.handBeforeAction = [...playerHand];
                botState.scoreBeforeAction = playerScore;
                botState.actionThatLedToWait = action;
                botState.setState(BOT_STATES.WAITING_FOR_ACTION_RESULT);
                await Utils.delay(CONFIG.ACTION_DELAY_MS);
            } else {
                Logger.warn(`${handIdentifier}: Vue Action "${action}" failed. Re-evaluating in next cycle.`);
            }
        }
        /**
         * Manage the wait after executing an action, checking if the turn is resolved.
         */
        static async _handleWaitingForActionState() {
            const handIdentifier = botState.isInSplitMode ? `Split Hand ${botState.activeSplitHandIndex + 1}` : 'Main Hand';
            botState.waitingForActionCounter++;
            // Gather current information once to avoid repeated calls
            const currentHand = await CardReader.getPlayerHandWithRetries();
            const { score: currentScore } = StrategyEngine.calculateScore(currentHand);
            const handChanged = JSON.stringify(currentHand) !== JSON.stringify(botState.handBeforeAction);
            const dealerHand = botState.gameInstance?.bankerFrame?.pokerArray?.map(c => CardReader.getCardValueFromCocosCard(c)).filter(Boolean) ?? [];
            const { score: dealerScore } = StrategyEngine.calculateScore(dealerHand);
            // Check for dealer blackjack overriding wait
            if (dealerHand.length === 2 && dealerScore === 21) {
                Logger.info(`[Bot Logic] Detected dealer natural Blackjack (Score: 21). Overriding wait and ending turn.`);
                this._transitionToNextHandOrEndRound();
                return;
            }
            // Delegate to specific handlers based on the last action
            const lastAction = botState.actionThatLedToWait;
            let resolved = false;
            switch (lastAction) {
                case PLAYER_ACTIONS.INSURANCE_NO:
                    resolved = await this._handleInsuranceDecline();
                    break;
                case PLAYER_ACTIONS.SPLIT:
                    resolved = await this._handleSplitResolution();
                    break;
                case PLAYER_ACTIONS.HIT:
                    resolved = await this._handleHitResolution(handIdentifier, currentHand, currentScore, handChanged);
                    break;
                case PLAYER_ACTIONS.DOUBLE:
                case PLAYER_ACTIONS.STAND:
                case PLAYER_ACTIONS.BUST:
                case PLAYER_ACTIONS.BLACKJACK:
                    resolved = await this._handleStandOrCompletion(handIdentifier, currentScore);
                    break;
                default:
                    break;
            }
            if (resolved) return;
            // Safety timeout: if waiting too long, force end of turn
            if (botState.waitingForActionCounter > (CONFIG.MAX_WAIT_FOR_ACTION_CYCLES * 5)) {
                Logger.warn(`${handIdentifier}: Action result timeout after '${lastAction}'. Forcing end of turn.`);
                this._transitionToNextHandOrEndRound();
            }
        }
        /**
         * Handle decline insurance flow: wait until insurance prompt disappears and check dealer blackjack.
         * @returns {Promise<boolean>} True if handled
         */
        static async _handleInsuranceDecline() {
            if (!this._isInsuranceOffered()) {
                Logger.info('Insurance prompt gone. Checking for dealer blackjack before proceeding.');
                await Utils.delay(CONFIG.ACTION_DELAY_MS / 2);
                const postInsuranceDealerHand = botState.gameInstance?.bankerFrame?.pokerArray?.map(c => CardReader.getCardValueFromCocosCard(c)).filter(Boolean) ?? [];
                if (postInsuranceDealerHand.length === 2 && StrategyEngine.calculateScore(postInsuranceDealerHand).score === 21) {
                    Logger.info('[Bot Result] Dealer has Blackjack after insurance was declined. Ending round.');
                    botState.setState(BOT_STATES.ROUND_ENDING);
                } else {
                    Logger.info('Dealer does not have Blackjack. Continuing player turn.');
                    botState.actionLock = false;
                    botState.setState(BOT_STATES.PLAYER_TURN);
                }
                return true;
            }
            return false;
        }
        /**
         * Handle post-split resolution: confirm playerFrame2 exists and two cards in first hand then enter split mode.
         * @returns {Promise<boolean>} True if handled
         */
        static async _handleSplitResolution() {
            if (!botState.isInSplitMode) {
                if (!botState.cocosPlayerFrame2Ref) botState.cocosPlayerFrame2Ref = botState.gameInstance?.playerFrame2;
                if (botState.cocosPlayerFrame2Ref) {
                    const firstHand = await CardReader.getPlayerHandWithRetries();
                    if (firstHand.length === 2) {
                        Logger.info('Split confirmed: playerFrame2 exists and first hand has 2 cards. Initializing split mode.');
                        UI.updateStatus('Split successful. Playing first hand...');
                        botState.isInSplitMode = true;
                        botState.currentSplitCount++;
                        botState.activeSplitHandIndex = 0;
                        botState.actionLock = false;
                        botState.setState(BOT_STATES.PLAYER_TURN);
                        return true;
                    }
                }
            }
            return false;
        }
        /**
         * Handle results after hitting: if hand changed and bust or 21 then resolve turn, otherwise continue playing.
         * @param {string} handIdentifier Name of the hand
         * @param {string[]} currentHand Current player hand
         * @param {number} currentScore Current score
         * @param {boolean} handChanged Whether the hand changed since last action
         * @returns {Promise<boolean>} True if handled
         */
        static async _handleHitResolution(handIdentifier, currentHand, currentScore, handChanged) {
            if (handChanged) {
                Logger.info(`${handIdentifier} (Hit): Hand changed to [${currentHand.join(',')}] (Score: ${currentScore}).`);
                if (currentScore >= 21) {
                    Logger.info(`${handIdentifier} turn ends (Bust or 21).`);
                    this._transitionToNextHandOrEndRound();
                    return true;
                }
                Logger.info('Continuing play.');
                botState.actionLock = false;
                botState.setState(BOT_STATES.PLAYER_TURN);
                return true;
            }
            return false;
        }
        /**
         * Handle stand/double/bust/blackjack wait: check if action buttons are gone to detect end of turn.
         * @param {string} handIdentifier Name of the hand
         * @param {number} currentScore Current score
         * @returns {Promise<boolean>} True if handled
         */
        static async _handleStandOrCompletion(handIdentifier, currentScore) {
            // On first split hand, immediately move to second after action
            if (botState.isInSplitMode && botState.activeSplitHandIndex === 0 && !botState.firstSplitHandResolved) {
                Logger.info(`${handIdentifier} ('${botState.actionThatLedToWait}') resolved. Immediately transitioning to second hand.`);
                this._transitionToNextHandOrEndRound();
                return true;
            }
            // For main or second split hand, wait until no action buttons are available
            if (!this._anyActionButtonsAvailableViaVueState()) {
                Logger.info(`${handIdentifier} ('${botState.actionThatLedToWait}') resolved. Final score: ${currentScore}.`);
                this._transitionToNextHandOrEndRound();
                return true;
            }
            return false;
        }
        /**
         * Transition to next split hand or end the round when all hands played.
         */
        static async _transitionToNextHandOrEndRound() {
            botState.actionLock = false;
            if (botState.isInSplitMode) {
                if (botState.activeSplitHandIndex === 0 && !botState.firstSplitHandResolved) {
                    Logger.info('First split hand resolved. Transitioning to second hand.');
                    UI.updateStatus('First hand done. Playing second...');
                    botState.activeSplitHandIndex = 1;
                    botState.firstSplitHandResolved = true;
                    await Utils.delay(CONFIG.DELAY_BETWEEN_SPLIT_HANDS_MS);
                    botState.setState(BOT_STATES.PLAYER_TURN);
                } else {
                    Logger.info('Second split hand resolved. Ending round.');
                    UI.updateStatus('Both split hands resolved. Waiting for dealer.');
                    botState.setState(BOT_STATES.ROUND_ENDING);
                }
            } else {
                Logger.info('Main hand resolved. Ending round.');
                botState.setState(BOT_STATES.ROUND_ENDING);
            }
        }
        /**
         * After the round finishes, log results and prepare for next round.
         */
        static async _handleRoundEndingState() {
            UI.updateStatus('Round completed. Waiting for game to be ready...');
            let waitCycles = 0;
            const maxWaitCycles = 20;
            while (!this._canStartNewRound() && waitCycles < maxWaitCycles) {
                await Utils.delay(CONFIG.POLL_INTERVAL_MS);
                waitCycles++;
            }
            if (waitCycles >= maxWaitCycles) {
                Logger.warn('Timeout waiting for canStartNewRound signal. Forcing reset anyway.');
            } else {
                Logger.info('Game is ready for the next round. Analyzing results.');
            }
            // Gather results for main and split hands
            const dealerHand = botState.gameInstance?.bankerFrame?.pokerArray?.map(c => CardReader.getCardValueFromCocosCard(c)).filter(Boolean) ?? [];
            const { score: dealerScore } = StrategyEngine.calculateScore(dealerHand);
            const playerHand1 = botState.gameInstance?.playerFrame?.pokerArray?.map(c => CardReader.getCardValueFromCocosCard(c)).filter(Boolean) ?? [];
            const { score: player1Score } = StrategyEngine.calculateScore(playerHand1);
            const logResult = (handName, playerScore, playerHand) => {
                let result;
                if (playerScore > 21) {
                    result = 'Player LOSES (Bust)';
                    const dealerUpcard = botState.dealerUpcardForSplit || CardReader.getDealerUpcardInternal() || 'N/A';
                    const dealerUpcardScore = StrategyEngine.calculateScore([dealerUpcard]).score;
                    Logger.info(`[Bot Result] ${handName}: ${result}. Player: ${playerScore} ([${playerHand.join(', ')}]), Dealer shows: ${dealerUpcardScore} ([${dealerUpcard}])`);
                    return;
                }
                if (dealerScore > 21) result = 'Player WINS (Dealer Bust)';
                else if (playerScore > dealerScore) result = 'Player WINS';
                else if (playerScore < dealerScore) result = 'Player LOSES';
                else result = 'Player PUSHES';
                Logger.info(`[Bot Result] ${handName}: ${result}. Player: ${playerScore} ([${playerHand.join(', ')}]), Dealer: ${dealerScore} ([${dealerHand.join(', ')}])`);
            };
            logResult('Hand 1', player1Score, playerHand1);
            if (botState.currentSplitCount > 0) {
                const playerHand2 = botState.gameInstance?.playerFrame2?.pokerArray?.map(c => CardReader.getCardValueFromCocosCard(c)).filter(Boolean) ?? [];
                const { score: player2Score } = StrategyEngine.calculateScore(playerHand2);
                logResult('Hand 2 (Split)', player2Score, playerHand2);
            }
            const endBalance = botState.vueGameComponent?.userInfo?.balance;
            if (endBalance && botState.bankrollAtRoundStart > 0) {
                const change = endBalance - botState.bankrollAtRoundStart;
                Logger.info(`[Result] Bankroll change: ${change >= 0 ? '+' : ''}$${change.toFixed(2)}. New bankroll: $${endBalance}`);
            }
            botState.roundJustEnded = true;
            botState.reset();
            await Utils.delay(CONFIG.DELAY_AFTER_HAND_TRANSITION_MS / 2);
            botState.setState(BOT_STATES.IDLE);
        }
        /**
         * Simple error recovery: wait and then revert to idle.
         */
        static async _handleErrorState() {
            UI.updateStatus('Error state. Attempting recovery...');
            await Utils.delay(CONFIG.POLL_INTERVAL_MS * 3);
            botState.setState(BOT_STATES.IDLE);
        }
        /**
         * Execute an action by invoking the mapped Vue method on the game component.
         * @param {string} actionKey Action to execute
         * @returns {Promise<boolean>} True if action executed
         */
        static async _executeVueAction(actionKey) {
            if (!botState.vueGameComponent) {
                Logger.warn(`_executeVueAction: Vue component not available for action: ${actionKey}`);
                return false;
            }
            const methodName = VUE_ACTION_METHODS[actionKey];
            if (!methodName || typeof botState.vueGameComponent[methodName] !== 'function') {
                Logger.warn(`_executeVueAction: Vue method '${methodName}' (for action '${actionKey}') not found or not a function.`);
                return false;
            }
            try {
                const actionLog = botState.isInSplitMode ? `Executing Vue action (Split Hand ${botState.activeSplitHandIndex + 1}): ${actionKey} (${methodName})` : `Executing Vue action: ${actionKey} (${methodName})`;
                Logger.info(actionLog);
                botState.vueGameComponent[methodName]();
                botState.recordAction(`${actionKey} (Vue)`);
                botState.actionLock = true;
                return true;
            } catch (error) {
                Logger.error(`Error executing Vue action ${methodName} for ${actionKey}:`, error);
                return false;
            }
        }
        /**
         * Validate that we still have valid game references.
         * @returns {boolean} True if game instance and component references are valid
         */
        static _validateGameInstances() {
            return botState.gameInstance &&
                botState.vueGameComponent &&
                typeof botState.gameInstance === 'object' &&
                botState.gameInstance.playerFrame &&
                botState.gameInstance.bankerFrame &&
                botState.cocosPlayerFrame1Ref &&
                typeof botState.vueGameComponent === 'object';
        }
        /**
         * Check if the game allows starting a new round.
         * @returns {boolean} True if start button is available
         */
        static _canStartNewRound() {
            return !!botState.vueGameComponent?.canClickStart;
        }
        /**
         * Check if the insurance dialog is currently shown.
         * @returns {boolean} True if insurance is offered
         */
        static _isInsuranceOffered() {
            return !!botState.vueGameComponent?.isShowInsurance;
        }
        /**
         * Verify that a given player action can currently be executed (based on Vue flags).
         * @param {string} actionKey Action to verify
         * @returns {boolean} True if action is allowed
         */
        static _canPerformAction(actionKey) {
            if (!botState.vueGameComponent) return false;
            const vueStateFlags = {
                [PLAYER_ACTIONS.HIT]:    'isHit',
                [PLAYER_ACTIONS.STAND]:  'isStand',
                [PLAYER_ACTIONS.SPLIT]:  'isSplit',
                [PLAYER_ACTIONS.DOUBLE]: 'isDouble'
            };
            const flagName = vueStateFlags[actionKey];
            if (!flagName || !botState.vueGameComponent.hasOwnProperty(flagName)) return false;
            if (!botState.vueGameComponent[flagName]) return false;
            if (actionKey === PLAYER_ACTIONS.SPLIT && (botState.isInSplitMode || botState.currentSplitCount >= CONFIG.MAX_SPLIT_HANDS_ALLOWED)) return false;
            return true;
        }
        /**
         * Determine if any action buttons are active; used to detect end of turn.
         * @returns {boolean} True if any actionable buttons are available
         */
        static _anyActionButtonsAvailableViaVueState() {
            return this._canPerformAction(PLAYER_ACTIONS.HIT) ||
                this._canPerformAction(PLAYER_ACTIONS.STAND) ||
                this._canPerformAction(PLAYER_ACTIONS.DOUBLE) ||
                this._canPerformAction(PLAYER_ACTIONS.SPLIT);
        }
        /**
         * Determine polling delay based on current state.
         * @returns {number} Delay in milliseconds
         */
        static _getLoopDelay() {
            switch (botState.currentState) {
                case BOT_STATES.WAITING_FOR_ACTION_RESULT:
                    return CONFIG.POLL_INTERVAL_MS / 2;
                case BOT_STATES.ERROR:
                    return CONFIG.POLL_INTERVAL_MS * 2;
                default:
                    return CONFIG.POLL_INTERVAL_MS;
            }
        }
    }

    // ============================================================================
    // USER INTERFACE
    // Draws and manages the on-screen bot control panel.
    // ============================================================================
    class UI {
        /** Initialize the UI. */
        static init() {
            this.createBotUI();
            this.applyStyles();
            this.bindEvents();
        }
        /** Create the container and elements for the bot UI. */
        static createBotUI() {
            // Create base container
            this.container = document.createElement('div');
            this.container.id = 'blackjack-bot-ui';
            // Start/Stop button
            this.toggleButton = document.createElement('button');
            this.toggleButton.id = 'bot-toggle';
            this.toggleButton.textContent = 'Start';
            // Status display
            this.statusDisplay = document.createElement('div');
            this.statusDisplay.id = 'bot-status';
            this.statusDisplay.textContent = 'Bot Idle';
            // Assemble UI (no timer or config checkboxes)
            this.container.appendChild(this.toggleButton);
            this.container.appendChild(this.statusDisplay);
            document.body.appendChild(this.container);
        }
        /** Inject CSS styles for the bot UI. */
        static applyStyles() {
            GM_addStyle(`
                #blackjack-bot-ui {
                    position: fixed;
                    top: 20px;
                    right: 20px;
                    width: 280px;
                    background: linear-gradient(135deg, #1f2937 0%, #111827 100%);
                    border: 1px solid #374151;
                    border-radius: 12px;
                    padding: 20px;
                    font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
                    box-shadow: 0 10px 30px rgba(0,0,0,0.3);
                    z-index: 10000;
                    color: #d1d5db;
                }
                #bot-toggle {
                    width: 100%;
                    padding: 12px;
                    font-size: 16px;
                    font-weight: bold;
                    border: none;
                    border-radius: 8px;
                    color: white;
                    cursor: pointer;
                    transition: all 0.3s ease;
                    margin-bottom: 15px;
                    background: linear-gradient(45deg, #22c55e, #16a34a);
                }
                #bot-toggle:hover {
                    transform: translateY(-2px);
                    box-shadow: 0 5px 15px rgba(0,0,0,0.2);
                }
                #bot-status {
                    background: rgba(255,255,255,0.05);
                    padding: 8px;
                    border-radius: 6px;
                    font-size: 12px;
                    margin-bottom: 15px;
                    text-align: center;
                    white-space: nowrap;
                    overflow: hidden;
                    text-overflow: ellipsis;
                }
                body.hide-game-ui .game-stats,
                body.hide-game-ui .statistics-panel,
                body.hide-game-ui .sidebar,
                body.hide-game-ui .side-panel,
                body.hide-game-ui .right-panel,
                body.hide-game-ui .notifications,
                body.hide-game-ui .toast-container,
                body.hide-game-ui .alert-container,
                body.hide-game-ui .chat-container,
                body.hide-game-ui .chat-panel,
                body.hide-game-ui .live-chat {
                    display: none !important;
                }
                body.bot-active .game-overlay {
                    opacity: 0.1 !important;
                    pointer-events: none !important;
                }
            `);
        }
        /** Attach event handlers for the UI. */
        static bindEvents() {
            this.toggleButton.addEventListener('click', () => {
                if (botState.active) {
                    this.stopBot();
                } else {
                    this.startBot();
                }
            });
            // Debounced window resize logging
            window.addEventListener('resize', Utils.debounce(() => {
                if (botState.active) Logger.info('Window resized. Bot continues to run.');
            }, 750));
        }
        /** Start the bot and update UI accordingly. */
        static startBot() {
            botState.active = true;
            this.toggleButton.textContent = 'Stop';
            this.toggleButton.style.background = 'linear-gradient(45deg, #ef4444, #b91c1c)';
            document.body.classList.add('bot-active');
            if (CONFIG.HIDE_UI_ELEMENTS) document.body.classList.add('hide-game-ui');
            this.updateStatus('Starting bot...');
            // Record start time for internal tracking even if not displayed
            botState.timerStartTime = Date.now();
            // No UI timer; only track elapsed time internally
            GameIntegration.findGameInstanceAndStart();
        }
        /** Stop the bot and reset UI. */
        static stopBot() {
            botState.active = false;
            // Clean up any timer interval if previously set (for backwards compatibility)
            if (botState.timerIntervalId) {
                clearInterval(botState.timerIntervalId);
                botState.timerIntervalId = null;
            }
            // Update paused elapsed time for internal tracking
            botState.timerPausedElapsed += Date.now() - botState.timerStartTime;
            botState.setState(BOT_STATES.IDLE);
            this.toggleButton.textContent = 'Start';
            this.toggleButton.style.background = 'linear-gradient(45deg, #22c55e, #16a34a)';
            document.body.classList.remove('bot-active');
            this.updateStatus('Bot stopped');
            GameIntegration._stopBot();
        }
        /** Update the status display and log the message. */
        static updateStatus(message) {
            Logger.info(message);
            if (this.statusDisplay) {
                this.statusDisplay.textContent = message;
            }
        }
    }

    // ============================================================================
    // INITIALIZATION
    // Wait for game elements to be present and then initialize the UI and detect platform.
    // ============================================================================
    function initializeBot() {
        Logger.info('Initializing CanvaJS Blackjack Bot v5.2.0');
        /**
         * Determine whether the user is on desktop or mobile based on URL path.
         */
        function detectPlatform() {
            const currentPath = window.location.pathname;
            if (currentPath.includes(PLATFORM_CONFIG.MOBILE.URL_SUFFIX)) {
                PLATFORM_CONFIG.CURRENT = PLATFORM_CONFIG.MOBILE;
                Logger.info('Platform detected: Mobile');
            } else {
                PLATFORM_CONFIG.CURRENT = PLATFORM_CONFIG.DESKTOP;
                Logger.info('Platform detected: Desktop');
            }
        }
        function mainInit() {
            detectPlatform();
            UI.init();
            Logger.info('CanvaJS Blackjack Bot initialized successfully');
            UI.updateStatus('Bot ready - Click Start to begin');
        }
        const observer = new MutationObserver((mutations, obs) => {
            if (document.getElementById('gameCanvas') && document.getElementById('app')?.__vue__) {
                Logger.info('Game canvas and Vue instance detected. Proceeding with initialization.');
                obs.disconnect();
                mainInit();
            }
        });
        Logger.info('Waiting for game canvas and Vue instance to be added to the DOM...');
        observer.observe(document.body, { childList: true, subtree: true });
    }
    initializeBot();

})();
