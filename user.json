// ==UserScript==
// @name         Crack 일본어 대사 한글 발음 호버
// @namespace    https://crack.wrtn.ai/
// @version      1.2.2
// @description  Crack의 일본어 대사에 마우스를 올리면 Gemini로 변환한 한글 발음을 표시합니다.
// @author       User
// @match        https://crack.wrtn.ai/*
// @run-at       document-idle
// @grant        GM_getValue
// @grant        GM_setValue
// @grant        GM_deleteValue
// @grant        GM_xmlhttpRequest
// @grant        GM_registerMenuCommand
// @connect      generativelanguage.googleapis.com
// @connect      www.gstatic.com
// @license      MIT
// ==/UserScript==

(function () {
    'use strict';

    const SCRIPT_ID = 'cjkh';
    const CONFIG_KEY = 'cjkh-config-v1';
    const CACHE_KEY = 'cjkh-cache-v1';
    const CACHE_FORMAT_VERSION = 3;
    const MAX_DIALOGUES_PER_ELEMENT = 200;
    const MAX_TEST_RESULTS = 30;
    const JAPANESE_RE = /[\u3040-\u30ff\u3400-\u9fff\uff66-\uff9d]/u;
    const API_ORIGIN = 'https://generativelanguage.googleapis.com';
    const TOOLTIP_FONT_STACKS = Object.freeze({
        pretendard: '"Pretendard Variable", Pretendard, system-ui, -apple-system, "Segoe UI", sans-serif',
        suit: '"SUIT Variable", SUIT, system-ui, -apple-system, "Segoe UI", sans-serif',
        noto: '"Noto Sans KR", system-ui, -apple-system, "Segoe UI", sans-serif',
        system: 'system-ui, -apple-system, "Segoe UI", sans-serif',
    });

    const LEGACY_DEFAULT_PROMPT = `당신은 일본어 발음의 한글 표기 전문가입니다.
입력된 일본어의 뜻을 번역하지 말고, 일본어로 실제 발음되는 소리만 자연스러운 한글로 적으세요.
문맥에 맞는 한자 읽기, 장음, 촉음, 탁음, 연탁, 감탄사, 인명과 고유명사의 읽기를 반영하세요.
원문의 의미를 요약하거나 재작성하지 마세요.
발음 문자열에 설명, 번호, 괄호, 따옴표, 원문, 번역문을 덧붙이지 마세요.
입력이 여러 단위로 주어지면 입력 순서와 index를 정확히 보존하세요.`;

    const DEFAULT_PROMPT = `당신은 일본어 발음을 읽기 쉬운 한글로 표기하는 전문가입니다.
입력된 일본어의 뜻을 번역하지 말고, 문맥에 맞는 실제 일본어 발음만 한글로 적으세요.
한자 읽기, 장음, 촉음, 탁음, 연탁, 감탄사, 인명과 고유명사의 읽기를 정확히 반영하세요.

[띄어쓰기와 문장부호]
1. 한글 발음을 공백 없이 이어 붙이지 마세요.
2. 일본어의 문절, 의미 단위와 실제로 숨을 쉬어 읽는 호흡 단위마다 일반 공백(U+0020)을 넣으세요.
3. 조사 하나하나를 억지로 모두 떼지는 말고, 한국어 사용자가 소리 내어 따라 읽기 쉬운 덩어리로 나누세요.
4. 원문의 마침표, 쉼표, 물음표, 느낌표와 말줄임표(……, …)의 순서와 느낌을 보존하세요.
5. 쉼표나 문장부호 뒤에는 읽기 편한 공백을 두되, 말줄임표 자체를 다른 기호로 바꾸지 마세요.

[출력 예시]
こんにちは。今日はいい天気ですね。
→ 콘니치와. 쿄오와 이이 텐키데스네.

おや。こちらのお連れ様が、ずっと隠し持っていらしたようですね。
→ 오야. 코치라노 오츠레사마가, 즛토 카쿠시못테 이라시타 요오데스네.

원문의 의미를 요약하거나 재작성하지 마세요.
설명, 번호, 괄호, 따옴표, 일본어 원문, 한국어 번역문을 덧붙이지 마세요.
입력이 여러 단위로 주어지면 입력 순서와 index를 정확히 보존하세요.`;

    const DEFAULT_RULES = [
        {
            id: 'jp-quotes',
            name: '일본어 인용부호 「」',
            enabled: true,
            selector: '[data-message-group-id]',
            pattern: '「([^「」]*[\\u3040-\\u30ff\\u3400-\\u9fff\\uff66-\\uff9d][^「」]*)」',
            flags: 'gu',
            group: 0,
            whole: false,
        },
        {
            id: 'double-quotes',
            name: '이중 인용부호 ""',
            enabled: false,
            selector: '[data-message-group-id]',
            pattern: '"([^"\\n]*[\\u3040-\\u30ff\\u3400-\\u9fff\\uff66-\\uff9d][^"\\n]*)"',
            flags: 'gu',
            group: 0,
            whole: false,
        },
        {
            id: 'single-quotes',
            name: "작은따옴표 ''",
            enabled: false,
            selector: '[data-message-group-id]',
            pattern: "'([^'\\n]*[\\u3040-\\u30ff\\u3400-\\u9fff\\uff66-\\uff9d][^'\\n]*)'",
            flags: 'gu',
            group: 0,
            whole: false,
        },
        {
            id: 'whole-message',
            name: '선택한 메시지 요소 전체',
            enabled: false,
            selector: '[data-message-group-id] .wrtn-markdown',
            pattern: '',
            flags: 'gu',
            group: 0,
            whole: true,
        },
    ];

    const DEFAULT_CONFIG = {
        uxVersion: 2,
        enabled: true,
        hoverMode: 'sentence',
        hoverDelay: 250,
        dialogueFormat: 'jp',
        customDialogueStart: '「',
        customDialogueEnd: '」',
        useAdvancedRules: false,
        customDelimiterPattern: '[。！？!?]+[」』”’）)\\]]*',
        customDelimiterFlags: 'gu',
        provider: 'google',
        apiKey: '',
        model: 'gemini-3.8-flash',
        firebaseConfig: {},
        firebaseConfigSource: '',
        firebaseSdkVersion: '12.12.0',
        firebaseLocation: 'global',
        prompt: DEFAULT_PROMPT,
        requestTimeout: 30000,
        retryCount: 1,
        rules: DEFAULT_RULES,
        tooltip: {
            fontFamily: 'pretendard',
            fontSize: 13,
            fontWeight: 500,
            textColor: '#27303f',
            backgroundColor: '#fffdf8',
            opacity: 98,
            maxWidth: 380,
            gap: 8,
            position: 'auto',
            showLoading: true,
        },
        cacheMax: 300,
        cacheMaxAgeDays: 30,
        menuPosition: { right: 18, bottom: 92 },
    };

    const clone = (value) => {
        if (typeof structuredClone === 'function') return structuredClone(value);
        return JSON.parse(JSON.stringify(value));
    };

    function mergeConfig(saved) {
        const base = clone(DEFAULT_CONFIG);
        if (!saved || typeof saved !== 'object') return base;
        const merged = { ...base, ...saved };
        merged.tooltip = { ...base.tooltip, ...(saved.tooltip || {}) };
        merged.menuPosition = { ...base.menuPosition, ...(saved.menuPosition || {}) };
        merged.firebaseConfig = saved.firebaseConfig && typeof saved.firebaseConfig === 'object'
            ? { ...saved.firebaseConfig }
            : {};
        merged.rules = Array.isArray(saved.rules) && saved.rules.length
            ? saved.rules.map((rule, index) => ({
                id: String(rule.id || `rule-${index}-${Date.now()}`),
                name: String(rule.name || `규칙 ${index + 1}`),
                enabled: rule.enabled !== false,
                selector: String(rule.selector || '[data-message-group-id]'),
                pattern: String(rule.pattern || ''),
                flags: String(rule.flags || 'gu'),
                group: Math.max(0, Number.parseInt(rule.group, 10) || 0),
                whole: !!rule.whole,
            }))
            : clone(DEFAULT_RULES);
        // 1.0 설정에서 사용하던 기본 따옴표 선택과 수정된 고급 규칙을 보존합니다.
        if (saved.dialogueFormat === undefined && Array.isArray(saved.rules)) {
            const jpOn = saved.rules.some((rule) => rule?.id === 'jp-quotes' && rule.enabled !== false);
            const doubleOn = saved.rules.some((rule) => rule?.id === 'double-quotes' && rule.enabled !== false);
            merged.dialogueFormat = jpOn && doubleOn ? 'both' : (doubleOn ? 'double' : 'jp');
            const compact = (rules) => JSON.stringify(rules.map((rule) => ({
                id: rule.id, name: rule.name, enabled: rule.enabled !== false, selector: rule.selector,
                pattern: rule.pattern, flags: rule.flags, group: Number(rule.group) || 0, whole: !!rule.whole,
            })));
            if (compact(saved.rules) !== compact(DEFAULT_RULES)) merged.useAdvancedRules = true;
        }
        // 1.2 가독성 마이그레이션: 사용자가 바꾼 값은 유지하고, 이전 기본값만 새 기본값으로 교체합니다.
        if ((Number(saved.uxVersion) || 0) < 2) {
            merged.uxVersion = 2;
            if (saved.hoverMode === undefined || saved.hoverMode === 'dialogue') merged.hoverMode = 'sentence';
            if (saved.hoverDelay === undefined || Number(saved.hoverDelay) === 350) merged.hoverDelay = 250;
            if (saved.prompt === undefined || saved.prompt === LEGACY_DEFAULT_PROMPT) merged.prompt = DEFAULT_PROMPT;
            const oldTooltip = {
                fontSize: 16, textColor: '#ffffff', backgroundColor: '#151823',
                opacity: 94, maxWidth: 520, gap: 10,
            };
            for (const [key, oldValue] of Object.entries(oldTooltip)) {
                if (saved.tooltip?.[key] === undefined || saved.tooltip[key] === oldValue) merged.tooltip[key] = base.tooltip[key];
            }
        }
        return merged;
    }

    function readStored(key, fallback) {
        try { return GM_getValue(key, fallback); }
        catch (_) { return fallback; }
    }

    function writeStored(key, value) {
        try { GM_setValue(key, value); }
        catch (_) { /* Tampermonkey 저장소가 막힌 경우 현재 세션에서만 동작 */ }
    }

    let config = normalizeConfig(readStored(CONFIG_KEY, null));
    writeStored(CONFIG_KEY, clone(config));
    let configRevision = 0;
    let messageCache = new WeakMap();
    let saveTimer = 0;

    function clamp(value, min, max, fallback) {
        const number = Number(value);
        return Number.isFinite(number) ? Math.min(max, Math.max(min, number)) : fallback;
    }

    function normalizeConfig(next) {
        const merged = mergeConfig(next);
        merged.hoverDelay = clamp(merged.hoverDelay, 0, 3000, DEFAULT_CONFIG.hoverDelay);
        merged.requestTimeout = clamp(merged.requestTimeout, 5000, 120000, DEFAULT_CONFIG.requestTimeout);
        merged.retryCount = Math.round(clamp(merged.retryCount, 0, 2, DEFAULT_CONFIG.retryCount));
        merged.cacheMax = Math.round(clamp(merged.cacheMax, 10, 2000, DEFAULT_CONFIG.cacheMax));
        merged.cacheMaxAgeDays = Math.round(clamp(merged.cacheMaxAgeDays, 1, 365, DEFAULT_CONFIG.cacheMaxAgeDays));
        merged.tooltip.fontSize = clamp(merged.tooltip.fontSize, 10, 36, DEFAULT_CONFIG.tooltip.fontSize);
        merged.tooltip.fontWeight = Math.round(clamp(merged.tooltip.fontWeight, 100, 900, DEFAULT_CONFIG.tooltip.fontWeight) / 100) * 100;
        merged.tooltip.opacity = clamp(merged.tooltip.opacity, 20, 100, DEFAULT_CONFIG.tooltip.opacity);
        merged.tooltip.maxWidth = clamp(merged.tooltip.maxWidth, 180, 1000, DEFAULT_CONFIG.tooltip.maxWidth);
        merged.tooltip.gap = clamp(merged.tooltip.gap, 0, 60, DEFAULT_CONFIG.tooltip.gap);
        if (!['dialogue', 'sentence'].includes(merged.hoverMode)) merged.hoverMode = 'sentence';
        if (!['jp', 'double', 'both', 'custom'].includes(merged.dialogueFormat)) merged.dialogueFormat = 'jp';
        merged.customDialogueStart = String(merged.customDialogueStart ?? '「');
        merged.customDialogueEnd = String(merged.customDialogueEnd ?? '」');
        merged.useAdvancedRules = !!merged.useAdvancedRules;
        if (!['google', 'firebase'].includes(merged.provider)) merged.provider = 'google';
        if (!['auto', 'top', 'bottom'].includes(merged.tooltip.position)) merged.tooltip.position = 'auto';
        if (!Object.hasOwn(TOOLTIP_FONT_STACKS, merged.tooltip.fontFamily)) merged.tooltip.fontFamily = DEFAULT_CONFIG.tooltip.fontFamily;
        return merged;
    }

    function persistConfig(next, immediate = false) {
        config = normalizeConfig(next);
        configRevision += 1;
        messageCache = new WeakMap();
        applyRuntimeStyles();
        syncMenuToggle();
        const save = () => writeStored(CONFIG_KEY, clone(config));
        clearTimeout(saveTimer);
        if (immediate) save();
        else saveTimer = window.setTimeout(save, 350);
    }

    function redact(message) {
        let text = String(message || '알 수 없는 오류');
        if (config.apiKey) text = text.split(config.apiKey).join('[API KEY]');
        if (config.firebaseConfig?.apiKey) text = text.split(config.firebaseConfig.apiKey).join('[FIREBASE API KEY]');
        return text.replace(/AIza[\w-]{20,}/g, '[API KEY]');
    }

    const FIREBASE_FIELDS = ['apiKey', 'authDomain', 'projectId', 'storageBucket', 'messagingSenderId', 'appId', 'measurementId', 'databaseURL'];
    const FIREBASE_REQUIRED = ['apiKey', 'projectId', 'appId'];
    const FIREBASE_APP_NAME = 'cjkh-firebase-ai';
    let firebaseModuleCache = null;

    // Firebase 콘솔에서 복사한 JSON 또는 `const firebaseConfig = {...}`를 eval 없이 읽습니다.
    function parseFirebaseConfigText(text) {
        const source = String(text || '');
        if (/private_key|-----BEGIN [A-Z ]*PRIVATE KEY-----|service_account/i.test(source)) {
            throw new Error('서비스 계정 키는 입력하지 마세요. Firebase 웹 앱 Config만 사용할 수 있습니다.');
        }
        const result = {};
        for (const key of FIREBASE_FIELDS) {
            const escaped = key.replace(/[.*+?^${}()|[\]\\]/g, '\\$&');
            const quoted = new RegExp(`["']?${escaped}["']?\\s*:\\s*(["'\\x60])([^"'\\x60]*?)\\1`);
            const numeric = new RegExp(`["']?${escaped}["']?\\s*:\\s*([0-9]+)`);
            const stringMatch = source.match(quoted);
            const numberMatch = source.match(numeric);
            if (stringMatch?.[2]) result[key] = stringMatch[2].trim();
            else if (numberMatch?.[1]) result[key] = numberMatch[1];
        }
        if (!Object.keys(result).length) {
            try {
                const parsed = JSON.parse(source);
                for (const key of FIREBASE_FIELDS) if (parsed?.[key] != null) result[key] = String(parsed[key]).trim();
            } catch (_) { /* 위의 안전한 필드 파서 결과를 사용 */ }
        }
        const missing = FIREBASE_REQUIRED.filter((key) => !result[key]);
        if (missing.length) throw new Error(`Firebase Config 필수 항목이 없습니다: ${missing.join(', ')}`);
        return result;
    }

    function firebaseConfigText(value) {
        const ordered = {};
        for (const key of FIREBASE_FIELDS) if (value?.[key]) ordered[key] = value[key];
        return Object.keys(ordered).length ? JSON.stringify(ordered, null, 2) : '';
    }

    function validateFirebaseSettings() {
        const sdkVersion = String(config.firebaseSdkVersion || '');
        const location = String(config.firebaseLocation || '');
        if (!/^\d+\.\d+\.\d+$/.test(sdkVersion)) throw new Error('Firebase SDK 버전 형식이 올바르지 않습니다. 예: 12.12.0');
        if (!/^[a-z0-9-]+$/.test(location)) throw new Error('Vertex AI 리전 형식이 올바르지 않습니다. 예: global, us-central1');
        const missing = FIREBASE_REQUIRED.filter((key) => !config.firebaseConfig?.[key]);
        if (missing.length) throw new Error(`Firebase Config 필수 항목이 없습니다: ${missing.join(', ')}`);
    }

    async function sha256(text) {
        try {
            const bytes = new TextEncoder().encode(text);
            const digest = await crypto.subtle.digest('SHA-256', bytes);
            return [...new Uint8Array(digest)].map((byte) => byte.toString(16).padStart(2, '0')).join('');
        } catch (_) {
            let hash = 2166136261;
            for (let i = 0; i < text.length; i += 1) {
                hash ^= text.charCodeAt(i);
                hash = Math.imul(hash, 16777619);
            }
            return `fnv-${(hash >>> 0).toString(16)}`;
        }
    }

    const CacheStore = {
        entries: new Map(),
        dirtyTimer: 0,

        load() {
            const raw = readStored(CACHE_KEY, []);
            if (!Array.isArray(raw)) return;
            const cutoff = Date.now() - config.cacheMaxAgeDays * 86400000;
            for (const item of raw) {
                if (!item || typeof item.key !== 'string' || !Array.isArray(item.results)) continue;
                if (item.version !== CACHE_FORMAT_VERSION) continue;
                if ((item.createdAt || 0) < cutoff) continue;
                this.entries.set(item.key, item);
            }
            this.prune();
        },

        get(key) {
            const item = this.entries.get(key);
            if (!item) return null;
            const cutoff = Date.now() - config.cacheMaxAgeDays * 86400000;
            if ((item.createdAt || 0) < cutoff) {
                this.entries.delete(key);
                this.scheduleSave();
                return null;
            }
            item.lastAccess = Date.now();
            this.scheduleSave();
            return item.results;
        },

        set(key, results) {
            const now = Date.now();
            this.entries.set(key, { version: CACHE_FORMAT_VERSION, key, results, createdAt: now, lastAccess: now });
            this.prune();
            this.scheduleSave();
        },

        prune() {
            const cutoff = Date.now() - config.cacheMaxAgeDays * 86400000;
            for (const [key, item] of this.entries) {
                if ((item.createdAt || 0) < cutoff) this.entries.delete(key);
            }
            if (this.entries.size <= config.cacheMax) return;
            const oldest = [...this.entries.values()].sort((a, b) => (a.lastAccess || 0) - (b.lastAccess || 0));
            while (this.entries.size > config.cacheMax && oldest.length) this.entries.delete(oldest.shift().key);
        },

        scheduleSave() {
            clearTimeout(this.dirtyTimer);
            this.dirtyTimer = window.setTimeout(() => {
                writeStored(CACHE_KEY, [...this.entries.values()]);
            }, 500);
        },

        clear() {
            this.entries.clear();
            clearTimeout(this.dirtyTimer);
            try { GM_deleteValue(CACHE_KEY); }
            catch (_) { writeStored(CACHE_KEY, []); }
        },
    };

    CacheStore.load();

    function buildTextMap(root) {
        const nodes = [];
        let text = '';
        const walker = document.createTreeWalker(root, NodeFilter.SHOW_TEXT, {
            acceptNode(node) {
                const parent = node.parentElement;
                if (!parent || !node.nodeValue) return NodeFilter.FILTER_REJECT;
                if (parent.closest(`#${SCRIPT_ID}-host, script, style, noscript, textarea, input, select, option`)) {
                    return NodeFilter.FILTER_REJECT;
                }
                return NodeFilter.FILTER_ACCEPT;
            },
        });
        let node;
        while ((node = walker.nextNode())) {
            const start = text.length;
            text += node.nodeValue;
            nodes.push({ node, start, end: text.length });
        }
        return { text, nodes };
    }

    function trimOffsets(text, start, end) {
        let left = start;
        let right = end;
        while (left < right && /\s/u.test(text[left])) left += 1;
        while (right > left && /\s/u.test(text[right - 1])) right -= 1;
        return [left, right];
    }

    function rangeFromOffsets(map, start, end) {
        if (!map.nodes.length || end <= start) return null;
        const first = map.nodes.find((item) => start >= item.start && start <= item.end)
            || map.nodes.find((item) => item.end > start);
        const last = [...map.nodes].reverse().find((item) => end >= item.start && end <= item.end)
            || [...map.nodes].reverse().find((item) => item.start < end);
        if (!first || !last) return null;
        const range = document.createRange();
        try {
            range.setStart(first.node, Math.max(0, Math.min(first.node.nodeValue.length, start - first.start)));
            range.setEnd(last.node, Math.max(0, Math.min(last.node.nodeValue.length, end - last.start)));
            return range;
        } catch (_) {
            return null;
        }
    }

    function normalizeRegexFlags(flags, { global = true, indices = false } = {}) {
        let next = String(flags || '').replace(/[^dgimsuvy]/g, '').replace(/y/g, '');
        if (global && !next.includes('g')) next += 'g';
        if (indices && !next.includes('d')) next += 'd';
        return [...new Set(next)].join('');
    }

    function validateRegex(pattern, flags, allowEmptyPattern = false) {
        if (!allowEmptyPattern && !pattern) return { ok: false, error: '정규식이 비어 있습니다.' };
        try {
            const regex = new RegExp(pattern, normalizeRegexFlags(flags));
            for (const sample of ['', '문장。다음! テスト']) {
                regex.lastIndex = 0;
                let match;
                let guard = 0;
                while ((match = regex.exec(sample)) && guard < 50) {
                    if (match[0].length === 0) return { ok: false, error: '빈 문자열과 일치하는 정규식은 사용할 수 없습니다.' };
                    guard += 1;
                }
            }
            return { ok: true };
        } catch (error) {
            return { ok: false, error: error.message };
        }
    }

    function compileRule(rule) {
        if (!rule.enabled) return { rule, ok: false, disabled: true };
        try { document.querySelector(rule.selector); }
        catch (error) { return { rule, ok: false, error: `CSS 선택자: ${error.message}` }; }
        if (rule.whole) return { rule, ok: true, whole: true };
        const validity = validateRegex(rule.pattern, rule.flags);
        if (!validity.ok) return { rule, ok: false, error: validity.error };
        try {
            return {
                rule,
                ok: true,
                regex: new RegExp(rule.pattern, normalizeRegexFlags(rule.flags, { global: true, indices: true })),
                hasIndices: true,
            };
        } catch (_) {
            return {
                rule,
                ok: true,
                regex: new RegExp(rule.pattern, normalizeRegexFlags(rule.flags, { global: true, indices: false }).replace(/d/g, '')),
                hasIndices: false,
            };
        }
    }

    function escapeRegexLiteral(value) {
        return String(value).replace(/[.*+?^${}()|[\]\\]/g, '\\$&');
    }

    function basicDetectionRules() {
        let pairs;
        if (config.dialogueFormat === 'double') pairs = [['"', '"', 'double']];
        else if (config.dialogueFormat === 'both') pairs = [['「', '」', 'jp'], ['"', '"', 'double']];
        else if (config.dialogueFormat === 'custom') pairs = [[config.customDialogueStart, config.customDialogueEnd, 'custom']];
        else pairs = [['「', '」', 'jp']];
        return pairs.filter(([start, end]) => start && end).map(([start, end, id]) => ({
            id: `basic-${id}`,
            name: '기본 대사 형식',
            enabled: true,
            selector: '[data-message-group-id]',
            pattern: `${escapeRegexLiteral(start)}([\\s\\S]*?)${escapeRegexLiteral(end)}`,
            flags: 'gu',
            group: 0,
            whole: false,
        }));
    }

    function compiledRules() {
        const rules = basicDetectionRules();
        if (config.useAdvancedRules) rules.push(...config.rules);
        return rules.map(compileRule).filter((item) => item.ok);
    }

    function extractDialogues(root, ruleItems = compiledRules()) {
        const map = buildTextMap(root);
        const found = [];
        const seen = new Set();
        for (const item of ruleItems) {
            const { rule } = item;
            if (!root.matches(rule.selector)) continue;
            if (item.whole) {
                if (!JAPANESE_RE.test(map.text)) continue;
                const [start, end] = trimOffsets(map.text, 0, map.text.length);
                if (end > start) found.push({ root, map, start, end, source: map.text.slice(start, end), ruleId: rule.id });
                continue;
            }
            item.regex.lastIndex = 0;
            let match;
            let guard = 0;
            while ((match = item.regex.exec(map.text)) && guard < MAX_DIALOGUES_PER_ELEMENT) {
                guard += 1;
                if (!match[0].length) break;
                const group = Math.min(rule.group, match.length - 1);
                const value = match[group];
                if (!value || !JAPANESE_RE.test(value)) continue;
                let start;
                let end;
                if (item.hasIndices && match.indices?.[group]) {
                    [start, end] = match.indices[group];
                } else {
                    const relative = match[0].indexOf(value);
                    start = match.index + Math.max(0, relative);
                    end = start + value.length;
                }
                [start, end] = trimOffsets(map.text, start, end);
                const key = `${start}:${end}`;
                if (end <= start || seen.has(key)) continue;
                seen.add(key);
                found.push({ root, map, start, end, source: map.text.slice(start, end), ruleId: rule.id });
            }
        }
        return found.sort((a, b) => a.start - b.start || a.end - b.end);
    }

    function splitByDelimiter(text, regex) {
        const parts = [];
        let start = 0;
        regex.lastIndex = 0;
        let match;
        let guard = 0;
        while ((match = regex.exec(text)) && guard < 500) {
            guard += 1;
            if (!match[0].length) throw new Error('분리 정규식이 빈 문자열과 일치합니다.');
            const end = match.index + match[0].length;
            if (end > start) parts.push({ start, end });
            start = end;
        }
        if (start < text.length) parts.push({ start, end: text.length });
        if (!parts.length && text.length) parts.push({ start: 0, end: text.length });
        return parts.map((part) => {
            const [startTrimmed, endTrimmed] = trimOffsets(text, part.start, part.end);
            return { start: startTrimmed, end: endTrimmed };
        }).filter((part) => part.end > part.start);
    }

    function segmentOffsets(text, mode = config.hoverMode) {
        if (mode === 'dialogue') return text ? [{ start: 0, end: text.length }] : [];
        if (mode === 'sentence') {
            return splitByDelimiter(text, /[。！？!?]+[」』”’）)\]]*/gu);
        }
        const validity = validateRegex(config.customDelimiterPattern, config.customDelimiterFlags);
        if (!validity.ok) throw new Error(validity.error);
        const regex = new RegExp(config.customDelimiterPattern, normalizeRegexFlags(config.customDelimiterFlags));
        return splitByDelimiter(text, regex);
    }

    function scanRoot(root) {
        const text = root.textContent || '';
        const signature = `${configRevision}:${text}`;
        const cached = messageCache.get(root);
        if (cached?.signature === signature) return cached.dialogues;
        const ruleItems = compiledRules();
        const dialogues = extractDialogues(root, ruleItems).map((dialogue) => {
            let offsets;
            try { offsets = segmentOffsets(dialogue.source); }
            catch (_) { offsets = [{ start: 0, end: dialogue.source.length }]; }
            const segments = offsets.map((part) => dialogue.source.slice(part.start, part.end));
            const units = offsets.map((part, index) => ({
                start: dialogue.start + part.start,
                end: dialogue.start + part.end,
                source: segments[index],
                segmentIndex: index,
                range: rangeFromOffsets(dialogue.map, dialogue.start + part.start, dialogue.start + part.end),
            })).filter((unit) => unit.range);
            return { ...dialogue, segments, units };
        });
        messageCache.set(root, { signature, dialogues });
        return dialogues;
    }

    function distanceToAncestor(element, ancestor) {
        let distance = 0;
        let current = element;
        while (current && current !== ancestor) {
            current = current.parentElement;
            distance += 1;
        }
        return current === ancestor ? distance : Number.POSITIVE_INFINITY;
    }

    function candidateRoots(target) {
        if (!(target instanceof Element)) return [];
        const roots = new Map();
        for (const item of compiledRules()) {
            try {
                const root = target.closest(item.rule.selector);
                if (root && !roots.has(root)) roots.set(root, distanceToAncestor(target, root));
            } catch (_) { /* invalid selectors are ignored */ }
        }
        return [...roots.entries()].sort((a, b) => a[1] - b[1]).map(([root]) => root);
    }

    function rectContains(rect, x, y) {
        const pad = 1.5;
        return x >= rect.left - pad && x <= rect.right + pad && y >= rect.top - pad && y <= rect.bottom + pad;
    }

    function hitTest(target, x, y) {
        const hits = [];
        for (const root of candidateRoots(target)) {
            for (const dialogue of scanRoot(root)) {
                for (const unit of dialogue.units) {
                    const rects = [...unit.range.getClientRects()];
                    if (rects.some((rect) => rectContains(rect, x, y))) {
                        hits.push({ root, dialogue, unit, rects, size: unit.end - unit.start });
                    }
                }
            }
        }
        return hits.sort((a, b) => a.size - b.size)[0] || null;
    }

    const pendingRequests = new Map();

    function gmRequest({ url, headers, data, timeout }) {
        return new Promise((resolve, reject) => {
            let settled = false;
            const finish = (fn, value) => {
                if (settled) return;
                settled = true;
                fn(value);
            };
            GM_xmlhttpRequest({
                method: 'POST',
                url,
                headers,
                data,
                timeout,
                anonymous: true,
                onload: (response) => finish(resolve, response),
                onerror: () => finish(reject, new Error('네트워크 요청에 실패했습니다.')),
                ontimeout: () => finish(reject, new Error(`요청 시간이 ${Math.round(timeout / 1000)}초를 초과했습니다.`)),
                onabort: () => finish(reject, new Error('요청이 취소되었습니다.')),
            });
        });
    }

    function parseApiError(response) {
        try {
            const json = JSON.parse(response.responseText || '{}');
            return json.error?.message || `HTTP ${response.status}`;
        } catch (_) {
            return `HTTP ${response.status}`;
        }
    }

    function extractCandidateText(payload) {
        const parts = payload?.candidates?.[0]?.content?.parts || [];
        const text = parts.map((part) => part.text || '').join('').trim();
        if (!text) {
            const reason = payload?.promptFeedback?.blockReason || payload?.candidates?.[0]?.finishReason;
            throw new Error(reason ? `응답이 생성되지 않았습니다: ${reason}` : 'Gemini가 빈 응답을 반환했습니다.');
        }
        return text.replace(/^```(?:json)?\s*/i, '').replace(/\s*```$/i, '').trim();
    }

    function buildApiPayload(segments, structured = true, spacingRepair = false) {
        const inputs = segments.map((text, index) => ({ index, text }));
        const spacingNotice = spacingRepair
            ? `이전 결과가 공백 없이 붙어 읽기 어려웠습니다. 이번에는 각 문절과 호흡 단위를 반드시 띄어 쓰세요.
예: 코치라노 오츠레사마가, 즛토 카쿠시못테 이라시타 요오데스네.\n`
            : '';
        const payload = {
            systemInstruction: { parts: [{ text: config.prompt || DEFAULT_PROMPT }] },
            contents: [{
                role: 'user',
                parts: [{
                    text: `${spacingNotice}아래 JSON의 각 text를 일본어 발음에 가까운 한글로만 변환하세요.
각 결과는 일본어 문절과 자연스러운 호흡 단위 사이에 반드시 공백을 넣어, 한국어 사용자가 바로 따라 읽을 수 있게 하세요.
문장부호와 말줄임표는 원문 순서대로 보존하세요. index와 순서를 바꾸지 마세요.\n${JSON.stringify(inputs)}`,
                }],
            }],
            generationConfig: {
                maxOutputTokens: Math.min(8192, Math.max(512, segments.join('').length * 4)),
                responseMimeType: 'application/json',
            },
        };
        if (/^gemini-3(?:[.-]|$)/.test(config.model)) {
            payload.generationConfig.thinkingConfig = { thinkingLevel: 'low' };
        }
        if (structured) {
            payload.generationConfig.responseSchema = {
                type: 'OBJECT',
                properties: {
                    results: {
                        type: 'ARRAY',
                        items: {
                            type: 'OBJECT',
                            properties: {
                                index: { type: 'INTEGER' },
                                pronunciation: {
                                    type: 'STRING',
                                    description: '의미·호흡 단위마다 일반 공백을 넣고 원문의 문장부호와 말줄임표를 보존한 읽기 쉬운 한글 발음',
                                },
                            },
                            required: ['index', 'pronunciation'],
                        },
                    },
                },
                required: ['results'],
            };
        }
        return payload;
    }

    function normalizeApiResults(parsed, segments) {
        const list = Array.isArray(parsed) ? parsed : parsed?.results;
        if (!Array.isArray(list)) throw new Error('응답 JSON에 results 배열이 없습니다.');
        const byIndex = new Map();
        for (const item of list) {
            const index = Number(item?.index);
            const pronunciation = formatPronunciation(item?.pronunciation || item?.result || '');
            if (Number.isInteger(index) && index >= 0 && pronunciation) byIndex.set(index, pronunciation);
        }
        const results = segments.map((_, index) => byIndex.get(index) || '');
        if (results.some((value) => !value)) throw new Error('일부 문장의 발음 결과가 누락되었습니다.');
        return results;
    }

    function formatPronunciation(value) {
        return String(value || '')
            .replace(/[\u00a0\t\r\n]+/g, ' ')
            .replace(/ {2,}/g, ' ')
            .replace(/([,，、;；:：])(?=\S)/g, '$1 ')
            .replace(/((?:…){1,}|\.{3,})(?=\S)/g, '$1 ')
            .replace(/([!?。！？]|(?<!\.)\.(?!\.))(?=[^\s」』”’）)\]])/g, '$1 ')
            .trim();
    }

    function needsSpacingRepair(source, pronunciation) {
        const readable = String(pronunciation || '').replace(/[.,!?。！？…，、;；:：]/g, '');
        const letters = (readable.match(/[가-힣A-Za-z0-9]/g) || []).length;
        const spaces = (readable.match(/ /g) || []).length;
        const sourceLength = (String(source || '').match(/[\u3040-\u30ff\u3400-\u9fff\uff66-\uff9d]/g) || []).length;
        return sourceLength >= 12 && letters >= 12 && (spaces === 0 || letters / (spaces + 1) > 15);
    }

    async function loadFirebaseModules() {
        if (firebaseModuleCache) return firebaseModuleCache;
        const base = `https://www.gstatic.com/firebasejs/${config.firebaseSdkVersion}/`;
        firebaseModuleCache = Promise.all([
            import(`${base}firebase-app.js`),
            import(`${base}firebase-ai.js`),
        ]).catch((error) => {
            firebaseModuleCache = null;
            throw new Error(`Firebase SDK(${config.firebaseSdkVersion})를 불러오지 못했습니다: ${error.message}`);
        });
        return firebaseModuleCache;
    }

    function firebaseError(error) {
        const message = String(error?.message || error || 'Firebase 요청 실패');
        const code = String(error?.code || '');
        const status = Number(error?.customErrorData?.status || 0);
        let hint = '';
        if (/app.?check/i.test(message + code)) hint = ' App Check가 강제 적용된 프로젝트는 유저스크립트에서 사용할 수 없습니다.';
        else if (/billing/i.test(message)) hint = ' Vertex AI 백엔드는 Blaze 결제 연결이 필요할 수 있습니다.';
        else if (/permission|forbidden|403/i.test(message) || status === 403) hint = ' Firebase AI Logic 및 Vertex AI 권한과 웹 API 키의 HTTP 리퍼러 제한을 확인하세요.';
        else if (/not found|404|model/i.test(message) || status === 404) hint = ' 모델 ID와 Vertex AI 리전을 확인하세요.';
        const result = new Error(`Firebase Vertex 오류: ${message}${hint}`);
        if (status) result.status = status;
        return result;
    }

    async function requestFirebaseOnce(segments, structured = true, spacingRepair = false) {
        validateFirebaseSettings();
        const [appMod, aiMod] = await loadFirebaseModules();
        try {
            let app = appMod.getApps().find((item) => item.name === FIREBASE_APP_NAME);
            const changed = app && Object.keys({ ...app.options, ...config.firebaseConfig })
                .some((key) => app.options[key] !== config.firebaseConfig[key]);
            if (changed) {
                await appMod.deleteApp(app);
                app = null;
            }
            if (!app) app = appMod.initializeApp({ ...config.firebaseConfig }, FIREBASE_APP_NAME);
            const VertexBackend = aiMod.AgentPlatformBackend || aiMod.VertexAIBackend;
            if (!VertexBackend) throw new Error(`Firebase SDK ${config.firebaseSdkVersion}에 Vertex AI 백엔드가 없습니다.`);
            const ai = aiMod.getAI(app, { backend: new VertexBackend(config.firebaseLocation) });
            const payload = buildApiPayload(segments, structured, spacingRepair);
            const generationConfig = { ...payload.generationConfig };
            if (generationConfig.thinkingConfig?.thinkingLevel && aiMod.ThinkingLevel) {
                const level = generationConfig.thinkingConfig.thinkingLevel.toUpperCase();
                generationConfig.thinkingConfig = { thinkingLevel: aiMod.ThinkingLevel[level] || generationConfig.thinkingConfig.thinkingLevel };
            }
            const model = aiMod.getGenerativeModel(ai, {
                model: config.model,
                systemInstruction: payload.systemInstruction,
                generationConfig,
            }, { timeout: config.requestTimeout });
            let timer;
            const timeout = new Promise((_, reject) => {
                timer = window.setTimeout(() => reject(new Error(`요청 시간이 ${Math.round(config.requestTimeout / 1000)}초를 초과했습니다.`)), config.requestTimeout + 1000);
            });
            let result;
            try { result = await Promise.race([model.generateContent({ contents: payload.contents }), timeout]); }
            finally { clearTimeout(timer); }
            const text = String(result.response.text() || '').replace(/^```(?:json)?\s*/i, '').replace(/\s*```$/i, '').trim();
            if (!text) throw new Error('Firebase가 빈 응답을 반환했습니다.');
            let parsed;
            try { parsed = JSON.parse(text); }
            catch (_) { throw new Error('Firebase가 올바른 JSON 결과를 반환하지 않았습니다.'); }
            return normalizeApiResults(parsed, segments);
        } catch (error) {
            throw firebaseError(error);
        }
    }

    async function requestGeminiOnce(segments, structured = true, spacingRepair = false) {
        const model = String(config.model || '').trim();
        if (!/^[A-Za-z0-9._-]+$/.test(model)) throw new Error('모델 ID 형식이 올바르지 않습니다.');
        if (config.provider === 'firebase') return requestFirebaseOnce(segments, structured, spacingRepair);
        const apiKey = String(config.apiKey || '').trim();
        if (!apiKey) throw new Error('설정에서 Gemini API 키를 입력해 주세요.');
        const url = `${API_ORIGIN}/v1beta/models/${encodeURIComponent(model)}:generateContent`;
        const response = await gmRequest({
            url,
            headers: {
                'Content-Type': 'application/json',
                'x-goog-api-key': apiKey,
            },
            data: JSON.stringify(buildApiPayload(segments, structured, spacingRepair)),
            timeout: config.requestTimeout,
        });
        if (response.status < 200 || response.status >= 300) {
            const error = new Error(parseApiError(response));
            error.status = response.status;
            throw error;
        }
        let payload;
        try { payload = JSON.parse(response.responseText || '{}'); }
        catch (_) { throw new Error('Gemini 응답을 JSON으로 읽지 못했습니다.'); }
        const text = extractCandidateText(payload);
        let parsed;
        try { parsed = JSON.parse(text); }
        catch (_) { throw new Error('Gemini가 올바른 JSON 결과를 반환하지 않았습니다.'); }
        return normalizeApiResults(parsed, segments);
    }

    async function requestGemini(segments) {
        let structured = true;
        let lastError;
        const attempts = Math.max(1, config.retryCount + 1);
        for (let attempt = 0; attempt < attempts; attempt += 1) {
            try {
                const results = await requestGeminiOnce(segments, structured);
                if (results.some((value, index) => needsSpacingRepair(segments[index], value))) {
                    try {
                        const repaired = await requestGeminiOnce(segments, structured, true);
                        if (!repaired.some((value, index) => needsSpacingRepair(segments[index], value))) return repaired;
                    } catch (_) { /* 첫 결과는 유효하므로 띄어쓰기 재요청 실패만으로 전체 요청을 버리지 않습니다. */ }
                }
                return results;
            } catch (error) {
                lastError = error;
                if (structured && error.status === 400 && /schema|response|generation/i.test(error.message || '')) {
                    structured = false;
                    continue;
                }
                const retryable = !error.status || error.status === 429 || error.status >= 500;
                if (!retryable || attempt >= attempts - 1) break;
                await new Promise((resolve) => window.setTimeout(resolve, 700 * (attempt + 1)));
            }
        }
        throw lastError || new Error('Gemini 요청에 실패했습니다.');
    }

    async function conversionKey(segments) {
        return sha256(JSON.stringify({
            version: CACHE_FORMAT_VERSION,
            provider: config.provider,
            backendProject: config.provider === 'firebase' ? config.firebaseConfig?.projectId || '' : '',
            backendLocation: config.provider === 'firebase' ? config.firebaseLocation : '',
            model: config.model,
            prompt: config.prompt,
            segments,
        }));
    }

    async function convertSegments(segments, { bypassCache = false } = {}) {
        const key = await conversionKey(segments);
        if (!bypassCache) {
            const cached = CacheStore.get(key);
            if (cached) return cached;
        }
        if (pendingRequests.has(key)) return pendingRequests.get(key);
        const promise = requestGemini(segments)
            .then((results) => {
                CacheStore.set(key, results);
                return results;
            })
            .finally(() => pendingRequests.delete(key));
        pendingRequests.set(key, promise);
        return promise;
    }

    const host = document.createElement('div');
    host.id = `${SCRIPT_ID}-host`;
    host.style.cssText = 'all:initial;position:fixed;inset:0;width:0;height:0;z-index:2147483646;';
    const shadow = host.attachShadow({ mode: 'open' });
    document.documentElement.appendChild(host);

    shadow.innerHTML = `
        <style id="runtime-style"></style>
        <style>
            :host{all:initial;color-scheme:light dark}
            *{box-sizing:border-box}
            .tab:focus-visible,button:focus-visible,input:focus-visible,select:focus-visible,textarea:focus-visible{outline:2px solid #818cf8;outline-offset:2px}
            #tooltip{position:fixed;display:none;z-index:3;padding:6px 9px;border:1px solid rgba(55,65,81,.16);border-radius:7px;box-shadow:0 3px 12px rgba(15,23,42,.14);font-family:system-ui,-apple-system,"Segoe UI",sans-serif;font-weight:500;line-height:1.5;letter-spacing:-.02em;white-space:pre-wrap;overflow-wrap:anywhere;word-break:keep-all;pointer-events:none}
            #tooltip.loading{color:#6b7280!important;box-shadow:0 2px 8px rgba(15,23,42,.10)}
            #tooltip.error{color:#991b1b!important;background:#fff7f7!important;border-color:#fecaca!important}
            #panel{position:fixed;inset:0;z-index:5;background:rgba(0,0,0,.48);display:grid;place-items:center;padding:18px;font-family:system-ui,-apple-system,"Noto Sans KR",sans-serif;color:#111827}
            #panel[hidden]{display:none}
            .box{width:min(760px,calc(100vw - 24px));height:min(720px,calc(100vh - 24px));background:#fff;border-radius:18px;box-shadow:0 24px 80px rgba(0,0,0,.35);display:flex;flex-direction:column;overflow:hidden}
            .head{display:flex;align-items:center;padding:18px 20px 12px;gap:12px}
            .title{font-size:18px;font-weight:900;flex:1}.status{font-size:11px;color:#059669;min-width:74px;text-align:right}
            .iconbtn{border:0;background:transparent;color:#6b7280;font-size:22px;cursor:pointer;padding:4px 7px}
            .tabs{display:flex;gap:4px;margin:0 16px 12px;padding:4px;background:#f3f4f6;border-radius:11px;overflow-x:auto}
            .tab{border:0;background:transparent;border-radius:8px;padding:9px 12px;color:#6b7280;font-size:12px;font-weight:700;white-space:nowrap;cursor:pointer;flex:1}
            .tab.on{background:#fff;color:#111827;box-shadow:0 1px 3px rgba(0,0,0,.12)}
            .body{flex:1;overflow:auto;padding:2px 20px 20px}.page{display:none}.page.on{display:block}
            .section{border:1px solid #e5e7eb;border-radius:12px;padding:14px;margin-bottom:12px}.section h3{font-size:13px;margin:0 0 12px}.row{display:flex;gap:10px;align-items:flex-end;margin-bottom:10px}.col{flex:1;min-width:0}
            label{display:block;font-size:11px;font-weight:800;color:#4b5563;margin:0 0 5px}.checkrow{display:flex;align-items:center;justify-content:space-between;gap:12px;margin-bottom:10px}.checkrow label{font-size:13px;color:#111827;margin:0}
            input[type=text],input[type=password],input[type=number],select,textarea{width:100%;border:1px solid #9ca3af;border-radius:8px;background:#fff;color:#111827;padding:9px 10px;font:12px/1.4 system-ui,sans-serif}textarea{resize:vertical;min-height:80px}.mono{font-family:ui-monospace,SFMono-Regular,Consolas,monospace!important}
            input[type=range]{width:100%}.small{font-size:10px;color:#6b7280;line-height:1.5;margin:6px 0}.errorline{font-size:10px;color:#dc2626;min-height:14px;margin-top:4px;white-space:pre-wrap}.result{background:#f9fafb;border:1px solid #e5e7eb;border-radius:8px;padding:10px;min-height:38px;max-height:170px;overflow:auto;white-space:pre-wrap;font-size:11px;line-height:1.55;color:#374151}
            .btn{border:1px solid #d1d5db;background:#f9fafb;color:#111827;border-radius:8px;padding:8px 11px;font-size:11px;font-weight:800;cursor:pointer}.btn.primary{background:#4f46e5;border-color:#4f46e5;color:#fff}.btn.danger{color:#dc2626;background:#fff7f7;border-color:#fca5a5}.btn:disabled{opacity:.5;cursor:default}
            .rule{border:1px solid #e5e7eb;border-radius:10px;padding:11px;margin-bottom:8px}.rulehead{display:flex;gap:8px;align-items:center;margin-bottom:9px}.rulehead input[type=text]{font-weight:800}.rulehead input[type=checkbox]{width:17px;height:17px}.rulegrid{display:grid;grid-template-columns:1fr 130px 72px;gap:8px}.rulegrid .wide{grid-column:1/-1}.ruleactions{display:flex;justify-content:flex-end;margin-top:7px}
            .foot{display:flex;justify-content:space-between;align-items:center;padding:12px 20px;border-top:1px solid #e5e7eb;background:#fafafa}.footnote{font-size:10px;color:#6b7280}.actions{display:flex;gap:8px}
            .toast{position:fixed;right:18px;bottom:18px;max-width:360px;padding:10px 13px;border-radius:9px;background:#111827;color:#fff;font:12px/1.45 system-ui,sans-serif;box-shadow:0 8px 28px rgba(0,0,0,.3);z-index:7;display:none;white-space:pre-wrap}
            #highlight-layer{position:fixed;inset:0;pointer-events:none;z-index:2}
            #hover-layer{position:fixed;inset:0;pointer-events:none;z-index:1}
            .hover-highlight{position:fixed;background:rgba(250,204,21,.13);border-bottom:1px solid rgba(180,83,9,.32);border-radius:2px}
            .detect-highlight{position:fixed;border:2px solid #ec4899;background:rgba(236,72,153,.20);border-radius:4px;box-shadow:0 0 0 1px rgba(255,255,255,.7) inset;animation:cjkh-pulse .75s ease-in-out alternate infinite}
            @keyframes cjkh-pulse{from{opacity:.55}to{opacity:1}}
            @media(max-width:640px){.row{display:block}.col{margin-bottom:9px}.rulegrid{grid-template-columns:1fr}.rulegrid .wide{grid-column:auto}.box{border-radius:12px}.tabs{justify-content:flex-start}.tab{flex:none}.body{padding-inline:12px}.head,.foot{padding-inline:12px}}
        </style>
        <div id="hover-layer"></div>
        <div id="highlight-layer"></div>
        <div id="tooltip" role="status" aria-live="polite"></div>
        <div id="panel" hidden>
            <div class="box" role="dialog" aria-modal="true" aria-label="일본어 발음 호버 설정">
                <div class="head"><div class="title">🗣️ 일본어 발음 호버 설정</div><div class="status" id="save-status"></div><button class="iconbtn" id="close" type="button" aria-label="닫기">×</button></div>
                <div class="tabs" id="tabs"></div>
                <div class="body" id="settings-body"></div>
                <div class="foot"><div class="footnote">변경 내용은 자동 저장됩니다.</div><div class="actions"><button class="btn" id="cancel" type="button">닫기</button><button class="btn primary" id="done" type="button">저장하고 닫기</button></div></div>
            </div>
        </div>
        <div class="toast" id="toast"></div>`;

    const tooltip = shadow.getElementById('tooltip');
    const panel = shadow.getElementById('panel');
    const toast = shadow.getElementById('toast');
    const runtimeStyle = shadow.getElementById('runtime-style');
    let settingsDraft = clone(config);
    let activeTab = 'general';
    let toastTimer = 0;
    let highlightTimer = 0;

    function rgba(hex, opacity) {
        const clean = String(hex || '#fffdf8').replace('#', '');
        const normalized = clean.length === 3 ? clean.split('').map((char) => char + char).join('') : clean.padEnd(6, '0').slice(0, 6);
        const value = Number.parseInt(normalized, 16);
        const red = (value >> 16) & 255;
        const green = (value >> 8) & 255;
        const blue = value & 255;
        return `rgba(${red},${green},${blue},${clamp(opacity, 0, 100, 98) / 100})`;
    }

    function applyRuntimeStyles() {
        const fontStack = TOOLTIP_FONT_STACKS[config.tooltip.fontFamily] || TOOLTIP_FONT_STACKS.system;
        runtimeStyle.textContent = `
            #tooltip{font-family:${fontStack};font-size:${config.tooltip.fontSize}px;font-weight:${config.tooltip.fontWeight};line-height:1.5;letter-spacing:-.02em;color:${config.tooltip.textColor};background:${rgba(config.tooltip.backgroundColor, config.tooltip.opacity)};max-width:min(${config.tooltip.maxWidth}px,calc(100vw - 20px));}
        `;
    }

    applyRuntimeStyles();

    function showToast(message, duration = 2600) {
        toast.textContent = redact(message);
        toast.style.display = 'block';
        clearTimeout(toastTimer);
        toastTimer = window.setTimeout(() => { toast.style.display = 'none'; }, duration);
    }

    function clearDetectionHighlights() {
        clearTimeout(highlightTimer);
        field('highlight-layer').textContent = '';
    }

    function highlightDialogues(dialogues, duration = 2400) {
        clearDetectionHighlights();
        const layer = field('highlight-layer');
        let rectCount = 0;
        for (const dialogue of dialogues) {
            const range = rangeFromOffsets(dialogue.map, dialogue.start, dialogue.end);
            if (!range) continue;
            for (const rect of range.getClientRects()) {
                if (rectCount >= 160) break;
                if (rect.width < 1 || rect.height < 1 || rect.bottom < 0 || rect.top > window.innerHeight || rect.right < 0 || rect.left > window.innerWidth) continue;
                const marker = document.createElement('div');
                marker.className = 'detect-highlight';
                marker.style.left = `${Math.max(0, rect.left - 2)}px`;
                marker.style.top = `${Math.max(0, rect.top - 2)}px`;
                marker.style.width = `${rect.width + 4}px`;
                marker.style.height = `${rect.height + 4}px`;
                layer.appendChild(marker);
                rectCount += 1;
            }
            if (rectCount >= 160) break;
        }
        highlightTimer = window.setTimeout(clearDetectionHighlights, duration);
        return rectCount;
    }

    function hideTooltip() {
        tooltip.style.display = 'none';
        tooltip.classList.remove('loading', 'error');
    }

    function clearHoverHighlight() {
        const layer = field('hover-layer');
        if (layer) layer.textContent = '';
    }

    function showHoverHighlight(range) {
        clearHoverHighlight();
        const layer = field('hover-layer');
        if (!layer || !range) return;
        for (const rect of range.getClientRects()) {
            if ((!rect.width && !rect.height) || rect.bottom < 0 || rect.top > window.innerHeight) continue;
            const marker = document.createElement('div');
            marker.className = 'hover-highlight';
            marker.style.left = `${Math.round(rect.left)}px`;
            marker.style.top = `${Math.round(rect.top)}px`;
            marker.style.width = `${Math.max(1, Math.round(rect.width))}px`;
            marker.style.height = `${Math.max(1, Math.round(rect.height))}px`;
            layer.appendChild(marker);
        }
    }

    function unionRect(rects) {
        if (!rects.length) return null;
        return rects.reduce((all, rect) => ({
            left: Math.min(all.left, rect.left),
            right: Math.max(all.right, rect.right),
            top: Math.min(all.top, rect.top),
            bottom: Math.max(all.bottom, rect.bottom),
        }), { left: rects[0].left, right: rects[0].right, top: rects[0].top, bottom: rects[0].bottom });
    }

    function positionTooltip(range) {
        const rects = [...range.getClientRects()].filter((rect) => rect.width || rect.height);
        const anchor = unionRect(rects);
        if (!anchor) return;
        const gap = config.tooltip.gap;
        const width = tooltip.offsetWidth;
        const height = tooltip.offsetHeight;
        let left = (anchor.left + anchor.right - width) / 2;
        left = Math.max(8, Math.min(window.innerWidth - width - 8, left));
        const topSpace = anchor.top;
        const bottomSpace = window.innerHeight - anchor.bottom;
        let placeTop = config.tooltip.position === 'top';
        if (config.tooltip.position === 'auto') placeTop = topSpace >= height + gap || topSpace > bottomSpace;
        let top = placeTop ? anchor.top - height - gap : anchor.bottom + gap;
        if (top < 8) top = Math.min(window.innerHeight - height - 8, anchor.bottom + gap);
        if (top + height > window.innerHeight - 8) top = Math.max(8, anchor.top - height - gap);
        tooltip.style.left = `${Math.round(left)}px`;
        tooltip.style.top = `${Math.round(top)}px`;
    }

    function showTooltip(text, range, state = '') {
        tooltip.textContent = text;
        tooltip.classList.toggle('loading', state === 'loading');
        tooltip.classList.toggle('error', state === 'error');
        tooltip.style.display = 'block';
        positionTooltip(range);
    }

    const rootIds = new WeakMap();
    let nextRootId = 1;
    const getRootId = (root) => {
        if (!rootIds.has(root)) rootIds.set(root, nextRootId++);
        return rootIds.get(root);
    };

    let hoverTimer = 0;
    let hoverToken = 0;
    let activeHit = null;
    let pointerFrame = 0;
    let latestPointer = null;

    function hitIdentity(hit) {
        if (!hit) return '';
        return `${getRootId(hit.root)}:${hit.unit.start}:${hit.unit.end}:${configRevision}`;
    }

    async function resolveHover(hit, token) {
        try {
            if (config.tooltip.showLoading) showTooltip('발음 변환 중…', hit.unit.range, 'loading');
            const results = await convertSegments(hit.dialogue.segments);
            if (token !== hoverToken || !activeHit || hitIdentity(activeHit) !== hitIdentity(hit)) return;
            const result = results[hit.unit.segmentIndex];
            showTooltip(result, hit.unit.range);
        } catch (error) {
            if (token !== hoverToken || !activeHit || hitIdentity(activeHit) !== hitIdentity(hit)) return;
            showTooltip(`오류: ${redact(error.message)}`, hit.unit.range, 'error');
        }
    }

    function setActiveHit(hit) {
        const before = hitIdentity(activeHit);
        const after = hitIdentity(hit);
        if (before === after) {
            if (hit && tooltip.style.display !== 'none') positionTooltip(hit.unit.range);
            return;
        }
        activeHit = hit;
        hoverToken += 1;
        clearTimeout(hoverTimer);
        hideTooltip();
        clearHoverHighlight();
        if (!hit) return;
        showHoverHighlight(hit.unit.range);
        const token = hoverToken;
        hoverTimer = window.setTimeout(() => resolveHover(hit, token), config.hoverDelay);
    }

    function processPointer() {
        pointerFrame = 0;
        if (!latestPointer || !config.enabled || !panel.hidden) {
            setActiveHit(null);
            return;
        }
        const hit = hitTest(latestPointer.target, latestPointer.clientX, latestPointer.clientY);
        setActiveHit(hit);
    }

    document.addEventListener('pointermove', (event) => {
        latestPointer = { target: event.target, clientX: event.clientX, clientY: event.clientY };
        if (!pointerFrame) pointerFrame = requestAnimationFrame(processPointer);
    }, { capture: true, passive: true });

    document.addEventListener('pointerleave', () => setActiveHit(null), { capture: true, passive: true });
    window.addEventListener('blur', () => setActiveHit(null), { passive: true });
    window.addEventListener('scroll', () => {
        if (activeHit) {
            showHoverHighlight(activeHit.unit.range);
            if (tooltip.style.display !== 'none') positionTooltip(activeHit.unit.range);
        }
    }, { capture: true, passive: true });
    window.addEventListener('resize', () => {
        applyRuntimeStyles();
        if (activeHit) {
            showHoverHighlight(activeHit.unit.range);
            if (tooltip.style.display !== 'none') positionTooltip(activeHit.unit.range);
        }
    }, { passive: true });

    const observer = new MutationObserver((records) => {
        const selectors = compiledRules().map((item) => item.rule.selector);
        for (const record of records) {
            const target = record.target instanceof Element ? record.target : record.target.parentElement;
            if (!target) continue;
            for (const selector of selectors) {
                try {
                    const root = target.closest(selector);
                    if (root) messageCache.delete(root);
                } catch (_) { /* ignored */ }
            }
        }
    });
    observer.observe(document.body, { childList: true, characterData: true, subtree: true });

    const TAB_DEFS = [
        ['general', '일반'],
        ['api', 'API'],
        ['detection', '대사 인식'],
        ['tooltip', '툴팁'],
        ['data', '캐시·데이터'],
    ];

    function field(id) { return shadow.getElementById(id); }

    function settingsTemplate() {
        return `
            <section class="page" data-page="general">
                <div class="section">
                    <h3>기본 동작</h3>
                    <div class="checkrow"><label for="cfg-enabled">기능 전체 사용</label><input id="cfg-enabled" type="checkbox"></div>
                    <div class="row"><div class="col"><label for="cfg-mode">발음 표시 범위</label><select id="cfg-mode"><option value="dialogue">대사 전체</option><option value="sentence">문장별</option></select></div><div class="col"><label for="cfg-delay">호버 대기 시간</label><select id="cfg-delay"><option value="0">즉시</option><option value="200">0.2초</option><option value="250">0.25초</option><option value="350">0.35초</option><option value="500">0.5초</option><option value="800">0.8초</option></select></div></div>
                    <div class="small">대사 전체는 어느 부분에 커서를 올려도 전체 발음을, 문장별은 커서가 놓인 문장만 보여 줍니다.</div>
                </div>
            </section>
            <section class="page" data-page="api">
                <div class="section">
                    <h3>Gemini 연결</h3>
                    <label for="cfg-provider">연결 방식</label><select id="cfg-provider"><option value="google">Google Gemini API 키 직접 연결</option><option value="firebase">Firebase AI Logic · Vertex AI 백엔드</option></select>
                    <div id="google-api-section" style="margin-top:10px">
                        <label for="cfg-api-key">Gemini API 키</label>
                        <div class="row"><div class="col"><input id="cfg-api-key" type="password" autocomplete="off" spellcheck="false"></div><button class="btn" id="toggle-key" type="button">표시</button></div>
                        <div class="small">키는 Tampermonkey 전용 저장소에 보관되며 공식 Gemini API에만 전송됩니다. 내보내기 파일에는 포함되지 않습니다.</div>
                    </div>
                    <div id="firebase-api-section" style="margin-top:10px">
                        <label for="cfg-firebase-config">Firebase 웹 앱 Config</label><textarea id="cfg-firebase-config" class="mono" style="min-height:150px" placeholder='{"apiKey":"AIza…","projectId":"…","appId":"1:…:web:…"}'></textarea>
                        <div class="errorline" id="firebase-config-error"></div>
                        <div class="row"><div class="col"><label for="cfg-firebase-sdk">Firebase JS SDK 버전</label><input id="cfg-firebase-sdk" class="mono" type="text" placeholder="12.12.0"></div><div class="col"><label for="cfg-firebase-location">Vertex AI 리전</label><input id="cfg-firebase-location" class="mono" type="text" placeholder="global"></div></div>
                        <div class="small">Firebase 콘솔의 웹 앱 Config만 입력하세요. 서비스 계정 JSON·비공개 키는 절대 입력하지 마세요. Vertex AI 백엔드는 프로젝트의 Firebase AI Logic 설정과 결제 연결이 필요할 수 있습니다.</div>
                    </div>
                    <div class="row"><div class="col"><label for="cfg-model-preset">모델 프리셋</label><select id="cfg-model-preset"><option value="gemini-3.8-flash">Gemini 3.8 Flash</option><option value="gemini-3.7-flash">Gemini 3.7 Flash</option><option value="gemini-3.6-flash">Gemini 3.6 Flash</option><option value="gemini-3.5-flash-lite">Gemini 3.5 Flash-Lite</option><option value="__custom__">직접 입력</option></select></div><div class="col" id="custom-model-box"><label for="cfg-model">모델 ID</label><input id="cfg-model" class="mono" type="text" spellcheck="false"></div></div>
                    <div class="row"><div class="col"><label for="cfg-timeout">타임아웃 (ms)</label><input id="cfg-timeout" type="number" min="5000" max="120000" step="1000"></div><div class="col"><label for="cfg-retry">재시도 횟수</label><input id="cfg-retry" type="number" min="0" max="2"></div></div>
                    <button class="btn primary" id="test-api" type="button">API 연결 테스트</button>
                    <div class="result" id="api-test-result">아직 테스트하지 않았습니다.</div>
                </div>
                <div class="section"><h3>변환 프롬프트</h3><textarea id="cfg-prompt" style="min-height:230px"></textarea><button class="btn" id="reset-prompt" type="button">기본 프롬프트 복원</button></div>
            </section>
            <section class="page" data-page="detection">
                <div class="section">
                    <h3>대사 형식</h3>
                    <label for="cfg-dialogue-format">사용하는 따옴표</label><select id="cfg-dialogue-format"><option value="jp">「일본어 대사」</option><option value="double">"일본어 대사"</option><option value="both">두 가지 모두</option><option value="custom">직접 지정</option></select>
                    <div class="row" id="custom-dialogue-symbols" style="margin-top:10px"><div class="col"><label for="cfg-dialogue-start">대사 시작 기호</label><input id="cfg-dialogue-start" type="text" maxlength="20"></div><div class="col"><label for="cfg-dialogue-end">대사 끝 기호</label><input id="cfg-dialogue-end" type="text" maxlength="20"></div></div>
                    <div class="errorline" id="dialogue-symbol-error"></div>
                    <label>미리보기</label><div class="result" id="format-preview"></div>
                </div>
                <div class="section"><h3>현재 채팅에서 확인</h3><button class="btn primary" id="test-detection" type="button">현재 채팅에서 대사 찾기</button><div class="result" id="detection-result">찾은 대사 수와 범위를 확인할 수 있습니다.</div></div>
                <details class="section" id="advanced-detection">
                    <summary style="cursor:pointer;font-size:13px;font-weight:800">고급 설정</summary>
                    <div style="margin-top:12px"><div class="checkrow"><label for="cfg-use-advanced">고급 감지 규칙 함께 사용</label><input id="cfg-use-advanced" type="checkbox"></div><div class="small">특수한 메시지 구조에서만 사용하세요.</div><div id="rules"></div><div class="row"><button class="btn" id="add-rule" type="button">+ 규칙 추가</button><button class="btn" id="restore-rules" type="button">고급 규칙 초기화</button></div></div>
                </details>
            </section>
            <section class="page" data-page="tooltip">
                <div class="section"><h3>모양과 위치</h3>
                    <div class="row"><div class="col"><label for="cfg-font-family">폰트</label><select id="cfg-font-family"><option value="pretendard">Pretendard</option><option value="suit">SUIT</option><option value="noto">Noto Sans KR</option><option value="system">시스템 기본</option></select></div><div class="col"><label for="cfg-font-size">글자 크기 (px)</label><input id="cfg-font-size" type="number" min="10" max="36"></div><div class="col"><label for="cfg-font-weight">글자 굵기</label><select id="cfg-font-weight"><option value="400">400 · 보통</option><option value="500">500 · 중간</option><option value="600">600 · 약간 굵게</option><option value="700">700 · 굵게</option></select></div></div>
                    <div class="row"><div class="col"><label for="cfg-max-width">최대 너비 (px)</label><input id="cfg-max-width" type="number" min="180" max="1000"></div><div class="col"><label for="cfg-gap">원문과 간격 (px)</label><input id="cfg-gap" type="number" min="0" max="60"></div></div>
                    <div class="row"><div class="col"><label for="cfg-text-color">글자 색</label><input id="cfg-text-color" type="color"></div><div class="col"><label for="cfg-bg-color">배경색</label><input id="cfg-bg-color" type="color"></div><div class="col"><label for="cfg-opacity">배경 투명도</label><input id="cfg-opacity" type="range" min="20" max="100"></div></div>
                    <div class="row"><div class="col"><label for="cfg-position">표시 위치</label><select id="cfg-position"><option value="auto">자동</option><option value="top">위</option><option value="bottom">아래</option></select></div><div class="col"><div class="checkrow"><label for="cfg-loading">로딩 표시</label><input id="cfg-loading" type="checkbox"></div></div></div>
                    <button class="btn" id="tooltip-preview" type="button">툴팁 미리보기</button>
                </div>
            </section>
            <section class="page" data-page="data">
                <div class="section"><h3>캐시</h3><div class="row"><div class="col"><label for="cfg-cache-max">최대 항목 수</label><input id="cfg-cache-max" type="number" min="10" max="2000"></div><div class="col"><label for="cfg-cache-days">보관 일수</label><input id="cfg-cache-days" type="number" min="1" max="365"></div></div><div class="result" id="cache-stats"></div><button class="btn danger" id="clear-cache" type="button">캐시 전체 삭제</button></div>
                <div class="section"><h3>설정 데이터</h3><div class="row"><button class="btn" id="export-settings" type="button">설정 내보내기</button><button class="btn" id="import-settings" type="button">설정 가져오기</button><input id="import-file" type="file" accept="application/json,.json" hidden></div><div class="small">내보내기에는 API 키와 캐시가 포함되지 않습니다.</div><button class="btn danger" id="reset-settings" type="button">모든 설정 기본값 복원</button></div>
            </section>`;
    }

    function renderTabs() {
        const tabs = field('tabs');
        tabs.innerHTML = '';
        for (const [id, label] of TAB_DEFS) {
            const button = document.createElement('button');
            button.type = 'button';
            button.className = `tab${id === activeTab ? ' on' : ''}`;
            button.dataset.tab = id;
            button.textContent = label;
            button.addEventListener('click', () => {
                activeTab = id;
                showSettingsPage();
            });
            tabs.appendChild(button);
        }
    }

    function showSettingsPage() {
        shadow.querySelectorAll('.tab').forEach((tab) => tab.classList.toggle('on', tab.dataset.tab === activeTab));
        shadow.querySelectorAll('.page').forEach((page) => page.classList.toggle('on', page.dataset.page === activeTab));
    }

    function setValue(id, value) {
        const element = field(id);
        if (element) element.value = value;
    }

    function setChecked(id, value) {
        const element = field(id);
        if (element) element.checked = !!value;
    }

    function ruleValidity(rule) {
        try { document.querySelector(rule.selector); }
        catch (error) { return `CSS 선택자 오류: ${error.message}`; }
        if (rule.whole) return '';
        return validateRegex(rule.pattern, rule.flags).error || '';
    }

    function renderRules() {
        const container = field('rules');
        container.innerHTML = '';
        settingsDraft.rules.forEach((rule, index) => {
            const card = document.createElement('div');
            card.className = 'rule';
            card.innerHTML = `
                <div class="rulehead"><input type="checkbox" data-prop="enabled"><input type="text" data-prop="name" aria-label="규칙 이름"></div>
                <div class="rulegrid"><div class="wide"><label>CSS 선택자</label><input class="mono" type="text" data-prop="selector"></div><div><label>정규식</label><input class="mono" type="text" data-prop="pattern"></div><div><label>플래그</label><input class="mono" type="text" data-prop="flags"></div><div><label>추출 그룹</label><input type="number" min="0" data-prop="group"></div></div>
                <div class="checkrow" style="margin-top:8px"><label>요소 전체를 대사로 사용</label><input type="checkbox" data-prop="whole"></div>
                <div class="errorline"></div><div class="ruleactions"><button class="btn danger" type="button" data-delete>삭제</button></div>`;
            const inputs = card.querySelectorAll('[data-prop]');
            inputs.forEach((input) => {
                const prop = input.dataset.prop;
                if (input.type === 'checkbox') input.checked = !!rule[prop];
                else input.value = rule[prop];
                input.addEventListener(input.type === 'checkbox' ? 'change' : 'input', () => {
                    rule[prop] = input.type === 'checkbox' ? input.checked : (prop === 'group' ? Math.max(0, Number.parseInt(input.value, 10) || 0) : input.value);
                    card.querySelector('[data-prop="pattern"]').disabled = !!rule.whole;
                    card.querySelector('[data-prop="flags"]').disabled = !!rule.whole;
                    card.querySelector('[data-prop="group"]').disabled = !!rule.whole;
                    card.querySelector('.errorline').textContent = ruleValidity(rule);
                    autoSaveDraft();
                });
            });
            card.querySelector('[data-prop="pattern"]').disabled = !!rule.whole;
            card.querySelector('[data-prop="flags"]').disabled = !!rule.whole;
            card.querySelector('[data-prop="group"]').disabled = !!rule.whole;
            card.querySelector('.errorline').textContent = ruleValidity(rule);
            card.querySelector('[data-delete]').addEventListener('click', () => {
                settingsDraft.rules.splice(index, 1);
                renderRules();
                autoSaveDraft();
            });
            container.appendChild(card);
        });
    }

    function dialogueFormatPreview(draft = settingsDraft) {
        const sample = 'こんにちは。今日はいい天気ですね。';
        if (draft.dialogueFormat === 'double') return `"${sample}"`;
        if (draft.dialogueFormat === 'both') return `「${sample}」\n"ありがとう。"`;
        if (draft.dialogueFormat === 'custom') return `${draft.customDialogueStart || ''}${sample}${draft.customDialogueEnd || ''}`;
        return `「${sample}」`;
    }

    function updateDetectionSimpleUi() {
        const custom = settingsDraft.dialogueFormat === 'custom';
        field('custom-dialogue-symbols').style.display = custom ? '' : 'none';
        const missing = custom && (!settingsDraft.customDialogueStart || !settingsDraft.customDialogueEnd);
        field('dialogue-symbol-error').textContent = missing ? '시작 기호와 끝 기호를 모두 입력해 주세요.' : '';
        field('format-preview').textContent = dialogueFormatPreview();
    }

    function fillSettings() {
        setChecked('cfg-enabled', settingsDraft.enabled);
        setValue('cfg-mode', settingsDraft.hoverMode);
        setValue('cfg-delay', settingsDraft.hoverDelay);
        setValue('cfg-dialogue-format', settingsDraft.dialogueFormat);
        setValue('cfg-dialogue-start', settingsDraft.customDialogueStart);
        setValue('cfg-dialogue-end', settingsDraft.customDialogueEnd);
        setChecked('cfg-use-advanced', settingsDraft.useAdvancedRules);
        setValue('cfg-provider', settingsDraft.provider);
        setValue('cfg-api-key', settingsDraft.apiKey);
        setValue('cfg-firebase-config', settingsDraft.firebaseConfigSource || firebaseConfigText(settingsDraft.firebaseConfig));
        setValue('cfg-firebase-sdk', settingsDraft.firebaseSdkVersion);
        setValue('cfg-firebase-location', settingsDraft.firebaseLocation);
        field('google-api-section').style.display = settingsDraft.provider === 'google' ? '' : 'none';
        field('firebase-api-section').style.display = settingsDraft.provider === 'firebase' ? '' : 'none';
        const presets = ['gemini-3.8-flash', 'gemini-3.7-flash', 'gemini-3.6-flash', 'gemini-3.5-flash-lite'];
        setValue('cfg-model-preset', presets.includes(settingsDraft.model) ? settingsDraft.model : '__custom__');
        setValue('cfg-model', settingsDraft.model);
        field('custom-model-box').style.display = field('cfg-model-preset').value === '__custom__' ? '' : 'none';
        setValue('cfg-timeout', settingsDraft.requestTimeout);
        setValue('cfg-retry', settingsDraft.retryCount);
        setValue('cfg-prompt', settingsDraft.prompt);
        setValue('cfg-font-family', settingsDraft.tooltip.fontFamily);
        setValue('cfg-font-size', settingsDraft.tooltip.fontSize);
        setValue('cfg-font-weight', settingsDraft.tooltip.fontWeight);
        setValue('cfg-max-width', settingsDraft.tooltip.maxWidth);
        setValue('cfg-gap', settingsDraft.tooltip.gap);
        setValue('cfg-text-color', settingsDraft.tooltip.textColor);
        setValue('cfg-bg-color', settingsDraft.tooltip.backgroundColor);
        setValue('cfg-opacity', settingsDraft.tooltip.opacity);
        setValue('cfg-position', settingsDraft.tooltip.position);
        setChecked('cfg-loading', settingsDraft.tooltip.showLoading);
        setValue('cfg-cache-max', settingsDraft.cacheMax);
        setValue('cfg-cache-days', settingsDraft.cacheMaxAgeDays);
        field('cache-stats').textContent = `저장된 발음: ${CacheStore.entries.size}개\n동시에 처리 중인 요청: ${pendingRequests.size}개`;
        renderRules();
        updateDetectionSimpleUi();
    }

    function bindSimpleField(id, setter, eventName = 'input') {
        field(id).addEventListener(eventName, (event) => {
            setter(event.target);
            autoSaveDraft();
        });
    }

    function autoSaveDraft() {
        const status = field('save-status');
        status.textContent = '저장 중…';
        persistConfig(settingsDraft);
        window.setTimeout(() => {
            if (!panel.hidden) status.textContent = '자동 저장됨';
        }, 430);
    }

    function bindSettingsEvents() {
        bindSimpleField('cfg-enabled', (el) => { settingsDraft.enabled = el.checked; }, 'change');
        bindSimpleField('cfg-mode', (el) => { settingsDraft.hoverMode = el.value; }, 'change');
        bindSimpleField('cfg-delay', (el) => { settingsDraft.hoverDelay = el.value; }, 'change');
        bindSimpleField('cfg-dialogue-format', (el) => {
            settingsDraft.dialogueFormat = el.value;
            updateDetectionSimpleUi();
        }, 'change');
        bindSimpleField('cfg-dialogue-start', (el) => {
            settingsDraft.customDialogueStart = el.value;
            updateDetectionSimpleUi();
        });
        bindSimpleField('cfg-dialogue-end', (el) => {
            settingsDraft.customDialogueEnd = el.value;
            updateDetectionSimpleUi();
        });
        bindSimpleField('cfg-use-advanced', (el) => { settingsDraft.useAdvancedRules = el.checked; }, 'change');
        bindSimpleField('cfg-provider', (el) => {
            settingsDraft.provider = el.value;
            field('google-api-section').style.display = el.value === 'google' ? '' : 'none';
            field('firebase-api-section').style.display = el.value === 'firebase' ? '' : 'none';
        }, 'change');
        bindSimpleField('cfg-api-key', (el) => { settingsDraft.apiKey = el.value.trim(); });
        bindSimpleField('cfg-firebase-config', (el) => {
            settingsDraft.firebaseConfigSource = el.value;
            try {
                settingsDraft.firebaseConfig = parseFirebaseConfigText(el.value);
                field('firebase-config-error').textContent = '';
            } catch (error) {
                field('firebase-config-error').textContent = error.message;
            }
        });
        bindSimpleField('cfg-firebase-sdk', (el) => {
            settingsDraft.firebaseSdkVersion = el.value.trim();
            firebaseModuleCache = null;
        });
        bindSimpleField('cfg-firebase-location', (el) => { settingsDraft.firebaseLocation = el.value.trim(); });
        bindSimpleField('cfg-timeout', (el) => { settingsDraft.requestTimeout = el.value; });
        bindSimpleField('cfg-retry', (el) => { settingsDraft.retryCount = el.value; });
        bindSimpleField('cfg-prompt', (el) => { settingsDraft.prompt = el.value; });
        bindSimpleField('cfg-font-family', (el) => { settingsDraft.tooltip.fontFamily = el.value; }, 'change');
        bindSimpleField('cfg-font-size', (el) => { settingsDraft.tooltip.fontSize = el.value; });
        bindSimpleField('cfg-font-weight', (el) => { settingsDraft.tooltip.fontWeight = el.value; }, 'change');
        bindSimpleField('cfg-max-width', (el) => { settingsDraft.tooltip.maxWidth = el.value; });
        bindSimpleField('cfg-gap', (el) => { settingsDraft.tooltip.gap = el.value; });
        bindSimpleField('cfg-text-color', (el) => { settingsDraft.tooltip.textColor = el.value; });
        bindSimpleField('cfg-bg-color', (el) => { settingsDraft.tooltip.backgroundColor = el.value; });
        bindSimpleField('cfg-opacity', (el) => { settingsDraft.tooltip.opacity = el.value; });
        bindSimpleField('cfg-position', (el) => { settingsDraft.tooltip.position = el.value; }, 'change');
        bindSimpleField('cfg-loading', (el) => { settingsDraft.tooltip.showLoading = el.checked; }, 'change');
        bindSimpleField('cfg-cache-max', (el) => { settingsDraft.cacheMax = el.value; CacheStore.prune(); CacheStore.scheduleSave(); });
        bindSimpleField('cfg-cache-days', (el) => { settingsDraft.cacheMaxAgeDays = el.value; CacheStore.prune(); CacheStore.scheduleSave(); });

        field('cfg-model-preset').addEventListener('change', (event) => {
            const custom = event.target.value === '__custom__';
            field('custom-model-box').style.display = custom ? '' : 'none';
            if (!custom) {
                settingsDraft.model = event.target.value;
                field('cfg-model').value = event.target.value;
                autoSaveDraft();
            }
        });
        bindSimpleField('cfg-model', (el) => { settingsDraft.model = el.value.trim(); });

        field('toggle-key').addEventListener('click', () => {
            const input = field('cfg-api-key');
            const showing = input.type === 'text';
            input.type = showing ? 'password' : 'text';
            field('toggle-key').textContent = showing ? '표시' : '숨김';
        });
        field('reset-prompt').addEventListener('click', () => {
            settingsDraft.prompt = DEFAULT_PROMPT;
            setValue('cfg-prompt', DEFAULT_PROMPT);
            autoSaveDraft();
        });
        field('test-api').addEventListener('click', async () => {
            const button = field('test-api');
            const result = field('api-test-result');
            button.disabled = true;
            result.textContent = '연결 테스트 중…';
            try {
                if (settingsDraft.provider === 'firebase') {
                    settingsDraft.firebaseConfig = parseFirebaseConfigText(field('cfg-firebase-config').value);
                }
                persistConfig(settingsDraft, true);
                const values = await convertSegments(['こんにちは。'], { bypassCache: true });
                result.textContent = `성공: こんにちは。 → ${values[0]}`;
            } catch (error) {
                result.textContent = `실패: ${redact(error.message)}`;
            } finally { button.disabled = false; }
        });
        field('add-rule').addEventListener('click', () => {
            settingsDraft.rules.push({
                id: `custom-${Date.now().toString(36)}`,
                name: '사용자 지정 규칙',
                enabled: true,
                selector: '[data-message-group-id]',
                pattern: '「([^「」]+)」',
                flags: 'gu',
                group: 0,
                whole: false,
            });
            renderRules();
            autoSaveDraft();
        });
        field('restore-rules').addEventListener('click', () => {
            settingsDraft.rules = clone(DEFAULT_RULES);
            renderRules();
            autoSaveDraft();
        });
        field('test-detection').addEventListener('click', () => {
            if (settingsDraft.dialogueFormat === 'custom' && (!settingsDraft.customDialogueStart || !settingsDraft.customDialogueEnd)) {
                field('detection-result').textContent = '시작 기호와 끝 기호를 모두 입력해 주세요.';
                return;
            }
            persistConfig(settingsDraft, true);
            const output = [];
            const detected = [];
            const rules = compiledRules();
            const roots = new Set();
            for (const item of rules) {
                try {
                    document.querySelectorAll(item.rule.selector).forEach((root) => roots.add(root));
                } catch (_) { /* ignored */ }
            }
            for (const root of roots) {
                for (const dialogue of extractDialogues(root, rules)) {
                    detected.push(dialogue);
                    if (output.length < MAX_TEST_RESULTS) output.push(dialogue.source);
                }
            }
            field('detection-result').textContent = detected.length
                ? `${detected.length}개를 찾았습니다.\n\n${output.slice(0, 12).map((text, index) => `${index + 1}. ${text}`).join('\n')}${detected.length > 12 ? '\n…' : ''}`
                : '감지된 일본어 대사가 없습니다. 선택자, 정규식, ON/OFF를 확인해 주세요.';
            if (detected.length) {
                const count = highlightDialogues(detected);
                setActiveHit(null);
                panel.hidden = true;
                showToast(`${detected.length}개 대사를 찾았습니다. 감지 범위를 분홍색으로 표시합니다.`, 2400);
                window.setTimeout(() => {
                    panel.hidden = false;
                    field('detection-result').textContent += count ? `\n\n화면에서 ${count}개 영역을 강조했습니다.` : '\n\n현재 화면 안에 보이는 대사가 없어 강조 표시는 생략했습니다.';
                }, 2400);
            }
        });
        field('tooltip-preview').addEventListener('click', () => {
            persistConfig(settingsDraft, true);
            const range = document.createRange();
            range.selectNodeContents(field('tooltip-preview'));
            showTooltip('타다시, 조오켄가 히토츠.', range);
            window.setTimeout(hideTooltip, 2600);
        });
        field('clear-cache').addEventListener('click', () => {
            if (!window.confirm('저장된 발음 캐시를 모두 삭제할까요?')) return;
            CacheStore.clear();
            field('cache-stats').textContent = '저장된 발음: 0개\n동시에 처리 중인 요청은 취소되지 않습니다.';
            showToast('캐시를 삭제했습니다.');
        });
        field('export-settings').addEventListener('click', () => {
            const exported = clone(settingsDraft);
            delete exported.apiKey;
            delete exported.firebaseConfig;
            delete exported.firebaseConfigSource;
            const data = JSON.stringify({ type: 'crack-japanese-pronunciation-settings', version: 1, exportedAt: new Date().toISOString(), config: exported }, null, 2);
            const url = URL.createObjectURL(new Blob([data], { type: 'application/json' }));
            const anchor = document.createElement('a');
            anchor.href = url;
            anchor.download = `crack-pronunciation-settings-${new Date().toISOString().slice(0, 10)}.json`;
            anchor.click();
            window.setTimeout(() => URL.revokeObjectURL(url), 1000);
        });
        field('import-settings').addEventListener('click', () => field('import-file').click());
        field('import-file').addEventListener('change', async (event) => {
            const file = event.target.files?.[0];
            if (!file) return;
            try {
                const parsed = JSON.parse(await file.text());
                const incoming = parsed.config || parsed;
                if (!incoming || typeof incoming !== 'object') throw new Error('설정 객체가 없습니다.');
                const currentKey = settingsDraft.apiKey;
                const currentFirebaseConfig = clone(settingsDraft.firebaseConfig);
                const currentFirebaseSource = settingsDraft.firebaseConfigSource;
                settingsDraft = mergeConfig({
                    ...incoming,
                    apiKey: currentKey,
                    firebaseConfig: currentFirebaseConfig,
                    firebaseConfigSource: currentFirebaseSource,
                });
                persistConfig(settingsDraft, true);
                renderSettings();
                showToast('설정을 가져왔습니다. API 키는 기존 값을 유지했습니다.');
            } catch (error) { showToast(`가져오기 실패: ${error.message}`, 4200); }
            event.target.value = '';
        });
        field('reset-settings').addEventListener('click', () => {
            if (!window.confirm('API 키를 포함한 모든 설정을 기본값으로 되돌릴까요? 캐시는 별도로 유지됩니다.')) return;
            settingsDraft = clone(DEFAULT_CONFIG);
            persistConfig(settingsDraft, true);
            renderSettings();
            showToast('설정을 기본값으로 복원했습니다.');
        });
    }

    function renderSettings() {
        field('settings-body').innerHTML = settingsTemplate();
        renderTabs();
        fillSettings();
        bindSettingsEvents();
        showSettingsPage();
    }

    function openSettings() {
        setActiveHit(null);
        settingsDraft = clone(config);
        panel.hidden = false;
        renderSettings();
        field('save-status').textContent = '';
    }

    function closeSettings() {
        persistConfig(settingsDraft, true);
        panel.hidden = true;
        hideTooltip();
        clearDetectionHighlights();
    }

    shadow.getElementById('close').addEventListener('click', closeSettings);
    shadow.getElementById('cancel').addEventListener('click', closeSettings);
    shadow.getElementById('done').addEventListener('click', closeSettings);
    panel.addEventListener('pointerdown', (event) => { if (event.target === panel) closeSettings(); });
    document.addEventListener('keydown', (event) => {
        if (event.key === 'Escape' && !panel.hidden) closeSettings();
    }, true);

    // Crack 햄버거 메뉴의 네이티브 항목을 복제해 설정 진입점을 만듭니다.
    const MENU_TARGET = '.py-4.overflow-y-auto.scrollbar > .px-2:first-of-type';
    const SETTINGS_ITEM_ID = 'crack-japanese-pronunciation-settings';
    const CHAT_PATH_RE = /^\/(?:stories\/[^/]+\/episodes|characters\/[^/]+\/chats|u\/[^/]+\/c)\/[^/?#]+/;
    const isChatPath = () => CHAT_PATH_RE.test(location.pathname);
    let settingsEl = null;
    let settingsSeen = false;
    let menuWatchUntil = 0;
    let drawerTries = 0;

    function syncMenuToggle(root = settingsEl) {
        const toggle = root?.querySelector('[data-cjkh-toggle]');
        if (!toggle) return;
        const checked = !!config.enabled;
        toggle.setAttribute('aria-checked', String(checked));
        toggle.setAttribute('aria-label', `일본어 발음 호버 ${checked ? '끄기' : '켜기'}`);
        toggle.title = `일본어 발음 호버 ${checked ? '끄기' : '켜기'}`;
        toggle.style.background = checked ? '#171717' : '#d1d5db';
        const thumb = toggle.querySelector('[data-cjkh-toggle-thumb]');
        if (thumb) thumb.style.transform = checked ? 'translateX(18px)' : 'translateX(2px)';
    }

    function toggleHoverEnabled(event) {
        event.preventDefault();
        event.stopPropagation();
        config.enabled = !config.enabled;
        persistConfig(config, true);
        if (!config.enabled) setActiveHit(null);
    }

    function createMenuToggle() {
        const toggle = document.createElement('span');
        toggle.dataset.cjkhToggle = '';
        toggle.setAttribute('role', 'switch');
        toggle.tabIndex = 0;
        toggle.style.cssText = 'position:relative;display:inline-block;flex:none;width:36px;height:20px;border-radius:999px;cursor:pointer;transition:background-color .16s ease;box-shadow:inset 0 0 0 1px rgba(0,0,0,.08);';
        const thumb = document.createElement('span');
        thumb.dataset.cjkhToggleThumb = '';
        thumb.setAttribute('aria-hidden', 'true');
        thumb.style.cssText = 'position:absolute;left:0;top:2px;width:16px;height:16px;border-radius:50%;background:#fff;box-shadow:0 1px 3px rgba(0,0,0,.28);transition:transform .16s ease;';
        toggle.appendChild(thumb);
        toggle.addEventListener('click', toggleHoverEnabled);
        toggle.addEventListener('keydown', (event) => {
            if (event.key === ' ' || event.key === 'Enter') toggleHoverEnabled(event);
        });
        return toggle;
    }

    function watchMenu(ms = 2200) {
        menuWatchUntil = Math.max(menuWatchUntil, Date.now() + ms);
        drawerTries = 0;
    }

    const menuNeedsWork = () => !(settingsEl && settingsEl.isConnected) && Date.now() < menuWatchUntil;

    function menuRows(group) {
        if (!(group instanceof Element)) return [];
        return [...group.children].filter((child) => {
            if (child.id === SETTINGS_ITEM_ID) return false;
            if (child.matches('button, a, [role="menuitem"]')) return (child.textContent || '').trim().length > 0;
            const controls = child.querySelectorAll('button, a, [role="menuitem"]');
            return controls.length === 1 && (controls[0].textContent || '').trim().length > 0;
        });
    }

    function findMenuSample(group) {
        const rows = menuRows(group);
        return rows.find((row) => /입력 번역기 설정/.test(row.textContent || '')) || rows[rows.length - 1] || null;
    }

    function findDrawerMenuGroup() {
        const vw = window.innerWidth;
        const vh = window.innerHeight;
        if (!(vw > 0 && vh > 0)) return null;
        for (const drawer of document.querySelectorAll('[role="dialog"], [data-state="open"], aside')) {
            if (drawer.closest(`#${SCRIPT_ID}-host, #ccs2-root, [id^="rpcm-"], [id^="wish-rp"]`) || drawer.closest('nav')) continue;
            const rect = drawer.getBoundingClientRect();
            if (rect.height < vh * 0.6 || rect.width < 160 || rect.width > Math.max(520, vw * 0.95)) continue;
            if (!(rect.right >= vw - 4 || rect.left <= 4)) continue;
            let scanned = 0;
            for (const group of drawer.querySelectorAll('div, ul')) {
                if (++scanned > 300) break;
                const rows = menuRows(group);
                if (rows.length >= 3 && rows.length >= group.children.length - 1) {
                    return { group, sample: findMenuSample(group) };
                }
            }
        }
        return null;
    }

    function createSettingsItem(sample) {
        let item;
        if (sample) {
            item = sample.cloneNode(true);
            for (const node of [item, ...item.querySelectorAll('*')]) {
                for (const attribute of [...node.attributes]) {
                    if (attribute.name === 'id' || attribute.name === 'href' || attribute.name === 'disabled'
                        || attribute.name.startsWith('on') || attribute.name.startsWith('data-')
                        || ['title', 'aria-label', 'aria-current', 'aria-describedby', 'aria-controls', 'aria-expanded'].includes(attribute.name)) {
                        node.removeAttribute(attribute.name);
                    }
                }
            }
            item.querySelectorAll('input, [role="switch"], [role="checkbox"]').forEach((node) => node.remove());
            const textNodes = [];
            const walker = document.createTreeWalker(item, NodeFilter.SHOW_TEXT);
            for (let node = walker.nextNode(); node; node = walker.nextNode()) {
                if (node.nodeValue.trim()) textNodes.push(node);
            }
            const label = textNodes.sort((a, b) => b.nodeValue.trim().length - a.nodeValue.trim().length)[0] || null;
            const oldIconText = textNodes.find((node) => node !== label && node.nodeValue.trim().length > 0 && node.nodeValue.trim().length <= 3) || null;
            for (const node of textNodes) node.nodeValue = node === label ? '일본어 발음 설정' : '';
            if (!label) item.textContent = '일본어 발음 설정';
            const icon = item.querySelector('svg, img');
            const blossom = document.createElement('span');
            blossom.textContent = '🌸';
            blossom.setAttribute('aria-hidden', 'true');
            blossom.style.cssText = 'font-size:16px;line-height:1;flex:none;';
            if (icon) icon.replaceWith(blossom);
            else if (oldIconText?.parentElement && oldIconText.parentElement !== item) oldIconText.parentElement.replaceWith(blossom);
            else if (label?.parentElement) label.parentElement.insertBefore(blossom, label);
            else item.prepend(blossom);
        } else {
            item = document.createElement('div');
            item.className = 'px-2.5 h-4 box-content py-[18px]';
            item.innerHTML = '<button type="button" class="w-full flex h-4 items-center justify-between typo-110-16-med space-x-2 ring-offset-4"><span class="flex space-x-2 items-center"><span style="font-size:16px;line-height:1">🌸</span><span class="whitespace-nowrap overflow-hidden text-ellipsis typo-text-sm_leading-none_medium">일본어 발음 설정</span></span></button>';
        }
        item.id = SETTINGS_ITEM_ID;
        item.style.cursor = 'pointer';
        item.setAttribute('aria-label', '🌸 일본어 발음 설정');
        const innerControl = item.matches('button, a, [role="menuitem"]') ? item : item.querySelector('button, a, [role="menuitem"]');
        if (innerControl) innerControl.setAttribute('aria-label', '🌸 일본어 발음 설정');
        const toggle = createMenuToggle();
        const toggleHost = innerControl || item;
        toggleHost.style.display = 'flex';
        toggleHost.style.alignItems = 'center';
        toggleHost.style.justifyContent = 'space-between';
        toggleHost.style.gap = '8px';
        toggleHost.appendChild(toggle);
        syncMenuToggle(item);
        item.addEventListener('click', (event) => {
            event.preventDefault();
            event.stopPropagation();
            openSettings();
        });
        return item;
    }

    function injectSettingsButton() {
        if (!menuNeedsWork()) return;
        const duplicate = document.getElementById(SETTINGS_ITEM_ID);
        if (duplicate && duplicate !== settingsEl) {
            settingsEl = duplicate;
            syncMenuToggle();
            settingsSeen = true;
            return;
        }
        let group = document.querySelector(MENU_TARGET);
        let sample = group ? findMenuSample(group) : null;
        if (!group && drawerTries++ < 6 && isChatPath() && document.visibilityState !== 'hidden') {
            const found = findDrawerMenuGroup();
            if (found) {
                group = found.group;
                sample = found.sample;
            }
        }
        if (!group) return;
        settingsEl = createSettingsItem(sample);
        group.appendChild(settingsEl);
        settingsSeen = true;
    }

    let menuInjectPending = false;
    let menuPath = location.pathname;
    let lastMenuInjectAt = 0;
    function scheduleMenuInject() {
        if (menuPath !== location.pathname) {
            menuPath = location.pathname;
            settingsEl = null;
            watchMenu(3000);
        }
        if (settingsSeen && settingsEl && !settingsEl.isConnected) {
            settingsSeen = false;
            watchMenu(2200);
        }
        if (menuInjectPending || !menuNeedsWork()) return;
        menuInjectPending = true;
        const run = () => requestAnimationFrame(() => {
            menuInjectPending = false;
            lastMenuInjectAt = Date.now();
            injectSettingsButton();
        });
        const wait = 100 - (Date.now() - lastMenuInjectAt);
        if (wait > 0) window.setTimeout(run, wait);
        else run();
    }

    document.addEventListener('click', () => {
        if (!(settingsEl && settingsEl.isConnected)) {
            watchMenu(2200);
            scheduleMenuInject();
        }
    }, true);
    document.addEventListener('visibilitychange', () => {
        if (document.visibilityState === 'visible') {
            watchMenu(1800);
            scheduleMenuInject();
        }
    });
    const menuObserver = new MutationObserver(scheduleMenuInject);
    menuObserver.observe(document.body, { childList: true, subtree: true });
    watchMenu(3000);
    injectSettingsButton();

    try {
        GM_registerMenuCommand('🌸 일본어 발음 설정', openSettings);
        GM_registerMenuCommand('일본어 발음 호버 켜기/끄기', () => {
            config.enabled = !config.enabled;
            persistConfig(config, true);
            setActiveHit(null);
            showToast(config.enabled ? '발음 호버를 켰습니다.' : '발음 호버를 껐습니다.');
        });
    } catch (_) { /* 메뉴 명령을 지원하지 않는 매니저에서도 본 기능은 동작 */ }
})();
